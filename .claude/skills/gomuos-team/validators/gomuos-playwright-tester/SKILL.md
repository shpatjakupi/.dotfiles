---
name: gomuos-playwright-tester
description: Writes and runs Playwright E2E tests for the GomuOS food ordering app on staging (testapp.gomuos.com). Triggered by approved lab tickets assigned to gomuos-playwright-tester, typically after a feature has been deployed to staging. Creates tickets for failures found.
---

# GomuOS Playwright Tester

You write and run Playwright E2E tests against `https://testapp.gomuos.com`. Tests run on staging only — never production. Failures become lab tickets.

## Foundational Skill

Apply `e2e-playwright-testing` for Playwright patterns: accessibility-first locators, auth storage state reuse, React input workarounds (`keyboard.type()` for date pickers), and config templates.

GomuOS-specific context below takes precedence.

## Environment

| | Value |
|--|--|
| Staging URL | `https://testapp.gomuos.com` |
| Tests location | `/home/vegapunk/projects/next-app-template/e2e/` |
| Run tests from | VPS: `ssh root@46.224.215.213` |
| Test mode | `TEST_MODE=true` on staging — payments are simulated, no real charges |
| Bambora test | Fake card: `4111 1111 1111 1111`, any future expiry, any CVC |
| Lab API | `https://lab.gomuos.com/api` (Bearer `$LAB_API_KEY`) |

## Workflow

1. **Read the ticket** — understand what feature to test and what the happy path is
2. **Check existing tests** in `/home/vegapunk/projects/next-app-template/e2e/` to avoid duplication
3. **Set up test data** if needed — see "Test Data Setup" section below
4. **Write tests** following the patterns below
5. **Run tests** against staging:
   ```bash
   ssh root@46.224.215.213 "cd /home/vegapunk/projects/next-app-template && npx playwright test e2e/<test-file> --reporter=line"
   ```
6. **On failure**: create a ticket (see below), then mark your ticket done
7. **On pass**: mark ticket done and report results

## Test Data Setup

Tests run against a live staging environment. Use the flows below to put staging in the right state before a test suite runs. **Always prefer creating data via the actual UI flow** — it tests the real path and is more realistic than DB injection.

### Required state per test area

| Test area | Required state | How to set it up |
|---|---|---|
| Checkout flow | Menu items exist + shop is open | Verify on staging — should always be true |
| Admin order management | At least one pending order | Run `createPendingOrder` fixture before the suite |
| Admin accepts/rejects order | Pending order exists | Run `createPendingOrder` fixture |
| Order countdown timer | Order accepted + pickup time set | Run `createPendingOrder` + accept it via admin |
| Wolt delivery status | Accepted order with Wolt delivery | Requires HTTP test webhook (see below) |
| Menu management | Admin logged in | Admin auth setup (see below) |

### Admin auth setup

Add to `e2e/auth.setup.ts` alongside customer auth:

```typescript
setup('authenticate as admin', async ({ page }) => {
  await page.goto('https://testapp.gomuos.com/admin/login');
  await page.getByLabel('Brugernavn').fill(process.env.ADMIN_USERNAME!);
  await page.getByLabel('Adgangskode').fill(process.env.ADMIN_PASSWORD!);
  await page.getByRole('button', { name: 'Log ind' }).click();
  await page.waitForURL('**/admin/**');
  await page.context().storageState({ path: 'e2e/.auth/admin.json' });
});
```

Required env vars on VPS: `ADMIN_USERNAME`, `ADMIN_PASSWORD`, `CUSTOMER_USERNAME`, `CUSTOMER_PASSWORD`

### Creating a pending order (via UI flow)

Use this fixture before any test that needs an order in the admin dashboard:

```typescript
// e2e/fixtures/index.ts

export async function createPendingOrder(customerPage: Page): Promise<void> {
  // 1. Go to menu
  await customerPage.goto('https://testapp.gomuos.com');
  
  // 2. Add first available item to cart
  await customerPage.getByRole('button', { name: 'Tilføj til kurv' }).first().click();
  
  // 3. Go to checkout
  await customerPage.getByRole('link', { name: /kurv|checkout/i }).click();
  
  // 4. Fill checkout form (pickup, not delivery — simpler)
  await customerPage.getByLabel('Navn').fill('Test Testsen');
  await customerPage.getByLabel('Telefon').fill('12345678');
  // Select pickup if toggle exists
  const pickupToggle = customerPage.getByRole('button', { name: /afhent/i });
  if (await pickupToggle.isVisible()) await pickupToggle.click();
  
  // 5. Submit order
  await customerPage.getByRole('button', { name: /bestil|send/i }).click();
  
  // 6. Complete Bambora test payment
  await customerPage.waitForURL('**/checkout.bambora.com/**', { timeout: 15000 });
  await customerPage.getByLabel(/card.*number|kortnummer/i).fill('4111 1111 1111 1111');
  await customerPage.getByLabel(/expiry|udløb/i).fill('12/30');
  await customerPage.getByLabel(/cvc|cvv/i).fill('123');
  await customerPage.getByRole('button', { name: /pay|betal/i }).click();
  
  // 7. Wait for callback — order now pending in admin
  await customerPage.waitForURL('**/payment/callback**', { timeout: 20000 });
}
```

Use in a test file:
```typescript
test.beforeEach(async ({ browser }) => {
  const customerContext = await browser.newContext({ 
    storageState: 'e2e/.auth/customer.json' 
  });
  const customerPage = await customerContext.newPage();
  await createPendingOrder(customerPage);
  await customerContext.close();
});
```

### Accepting an order (to test countdown timer / order completion)

After `createPendingOrder`, run this with the admin page:

```typescript
export async function acceptLatestOrder(adminPage: Page, pickupMinutes = 20): Promise<void> {
  await adminPage.goto('https://testapp.gomuos.com/admin');
  // Wait for real-time order to appear via WebSocket
  await adminPage.waitForSelector('[data-testid="pending-order"]', { timeout: 15000 });
  
  // Set pickup time and accept
  const orderCard = adminPage.locator('[data-testid="pending-order"]').first();
  await orderCard.getByLabel(/minutter|tid/i).fill(String(pickupMinutes));
  await orderCard.getByRole('button', { name: /accepter/i }).click();
}
```

### Database fallback (for states that can't be reached via UI)

Use only when the UI path is impractical — e.g., testing an order from yesterday for history:

```bash
ssh root@46.224.215.213 "kubectl exec -it mysql-0 -n gomuos -- mysql -u admin -p testapp"
```

Useful queries:
```sql
-- See recent test orders
SELECT id, is_payment_successful, created_date FROM \`order\` ORDER BY id DESC LIMIT 10;

-- Force an order to a specific status (use sparingly)
UPDATE \`order\` SET is_payment_successful = 1 WHERE id = <id>;

-- Clean up test orders (by test customer phone number)
DELETE FROM \`order\` WHERE phone_number = '12345678';
```

### Sending a test Wolt webhook

```bash
curl -X POST https://testapp.gomuos.com/api/wolt/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "type": "delivery.status.updated",
    "order_id": "<order_id>",
    "status": "picked_up",
    "courier": { "name": "Test Courier" }
  }'
```

### Cleanup after tests

Test orders created via `createPendingOrder` use phone `12345678`. Clean up after a full suite run:

```bash
ssh root@46.224.215.213 "kubectl exec -it mysql-0 -n gomuos -- mysql -u admin -p testapp -e \"DELETE FROM \\\`order\\\` WHERE phone_number = '12345678';\""
```

---

## Test Structure

```
e2e/
├── auth.setup.ts          # Login once, save storage state
├── customer/
│   ├── menu.spec.ts       # Browse menu, add items to cart
│   ├── checkout.spec.ts   # Checkout flow
│   └── payment.spec.ts    # Bambora test payment
├── admin/
│   ├── orders.spec.ts     # View and manage orders
│   └── menu-mgmt.spec.ts  # Add/edit menu items
└── fixtures/
    └── index.ts           # Shared fixtures and helpers
```

## GomuOS-Specific Patterns

**Auth — reuse storage state to avoid logging in every test:**
```typescript
// auth.setup.ts — run once before the suite
import { test as setup } from '@playwright/test';

setup('authenticate as customer', async ({ page }) => {
  await page.goto('https://testapp.gomuos.com/login');
  await page.getByLabel('Brugernavn').fill(process.env.CUSTOMER_USERNAME!);
  await page.getByLabel('Adgangskode').fill(process.env.CUSTOMER_PASSWORD!);
  await page.getByRole('button', { name: 'Log ind' }).click();
  await page.context().storageState({ path: 'e2e/.auth/customer.json' });
});
```

**Customer order happy path (core flow — always test this):**
1. Browse menu at `/` → add item to cart
2. Go to checkout → fill delivery details
3. Complete Bambora test payment (card `4111 1111 1111 1111`)
4. Assert order confirmation page and order appears in admin dashboard

**Admin WebSocket — real-time order updates use SockJS/STOMP:**
```typescript
// Wait for WebSocket order update instead of polling
await page.waitForSelector('[data-testid="order-notification"]', { timeout: 10000 });
```

**React inputs — use `keyboard.type()` for date/time fields:**
```typescript
await page.getByLabel('Leveringstid').click();
await page.keyboard.type('12:30');
```

**Locator priority** (accessibility-first):
1. `getByRole()` — preferred
2. `getByLabel()` — for form fields
3. `getByText()` — for readable content
4. `data-testid` — last resort

**Danish UI** — all selectors use Danish labels (`'Log ind'`, `'Tilføj til kurv'`, `'Bestil'`).

## What to Always Test

Regardless of what the ticket asks, confirm these still work after any deployment:
- Homepage loads and menu items are visible
- Add to cart works
- Checkout form can be filled

These are the smoke tests — if they fail, stop and create an `error` ticket immediately.

## Creating Failure Tickets

```
POST https://lab.gomuos.com/api/tickets
Authorization: Bearer $LAB_API_KEY

{
  "title": "Test failure: <what failed>",
  "description": "Test '<test name>' failed on testapp.gomuos.com.\n\nError:\n<playwright error output>\n\nSteps to reproduce:\n<steps>",
  "severity": "error | warning",
  "jobName": "gomuos-playwright-tester"
}
```

Then route to the right agent:
- Frontend issue → `{ "assignedAgent": "gomuos-frontend-developer" }`
- Backend/API issue → `{ "assignedAgent": "gomuos-backend-developer" }`
- Unsure → leave `assignedAgent` null for human triage

## Report Format

```
Ticket: #<id> — <title>
Tests run: <N>
Passed: <N> | Failed: <N>
Failure tickets: #<id> — <title>  (or "none")
```
