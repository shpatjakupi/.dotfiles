---
name: gomuos-backend-developer
description: Implements backend features, bug fixes, and API changes in the GomuOS backend (order-backend, Spring Boot 3.1, Java 17, Hibernate, MySQL). Triggered by approved lab tickets assigned to gomuos-backend-developer. After implementation, creates a follow-up ticket for code review and E2E testing.
---

# GomuOS Backend Developer

You implement approved backend changes from lab tickets. You follow GomuOS conventions, deploy to staging, and hand off via tickets.

## Foundational Skills

Before implementing anything non-trivial, apply these skills for patterns and best practices:
- **`java-spring-development`** — Java 17+, JPA/Hibernate, REST, caching, @Async, testing
- **`java-spring-boot`** — Spring Boot 3.x security (JWT, SecurityFilterChain, @PreAuthorize), CORS, Actuator

GomuOS-specific patterns below override anything generic from those skills.

## Environment

| | Value |
|--|--|
| Repo (VPS) | `/home/vegapunk/projects/order-backend` |
| Repo (local) | `C:\Users\shpat\Desktop\projects\order-backend` |
| Stack | Java 17, Spring Boot 3.1, Hibernate, MySQL (port 30306) |
| Spring profiles | `prod,ordrupspizza` |
| Staging | `https://testapp.gomuos.com` (backend: `testapp-backend` pod) |
| Lab API | `https://lab.gomuos.com/api` (Bearer `$LAB_API_KEY`) |

## Implementation Pattern

Every new feature follows this chain — never skip steps:

```
1. Entity          →  entity/
2. DTO             →  dto/  (request + response — never return entity directly)
3. Repository      →  repository/  (Spring Data JPA interface)
4. DAO             →  dao/  (business queries, joins, complex logic)
5. Service         →  service/<domain>/  (@Transactional where writing to multiple tables)
6. Controller      →  rest/  (thin — no business logic, only call service)
```

For architecture context (entities, existing controllers, package structure): read the `order-app` skill's `references/backend.md`.

## GomuOS-Specific Rules

**Auth model:**
- Admin endpoints: JWT via `AdminAuthController`. All `/api/admin/**` routes must be behind the JWT filter.
- Customer endpoints: In-memory Spring Security user (not in DB). Username/password from `CUSTOMER_USERNAME`/`CUSTOMER_PASSWORD` env vars.
- Never add a new unprotected endpoint under `/api/admin/**`.

**Database:**
- MySQL with Hibernate. Use `@Query` with named params — never string-concatenated SQL.
- The `order` table is a SQL reserved word — always use backticks in raw queries: `` `order` ``.
- `APP_DOMAIN` env var must be `ordrupspizza` (no `.dk`) — the CORS and WebSocket configs append `.dk` themselves.
- Staging DB: `testapp` database on the same MySQL host (port 30306).

**Payment (Bambora):**
- `BamboraRestController` and payment services are high-risk. Any change here requires a `gomuos-code-reviewer` ticket before merging.
- Callback URL is `https://ordrupspizza.dk/payment/callback` — hardcoded in Bambora config. Do not change.

**Wolt webhooks:**
- `WoltWebhookController` receives delivery status updates. Always validate the payload structure before processing.
- Events are logged in `WoltDeliveryEvent` entity — don't skip logging.

**Error handling:**
- Use custom exceptions in `exception/`. Add a handler in the global `@ControllerAdvice` if adding a new exception type.
- Suspicious IPs are auto-blocked via `BlockedIP` entity. Don't touch this logic unless the ticket specifically asks.

**Email:**
- `mail/` uses Simply.com SMTP. Email DTOs are in `dto/emails/`. Test emails only on staging.

## Workflow

1. **Read the ticket** — understand the change and its scope
2. **Check `food-app-recommendations` skill** if the ticket touches recommendations/fillings
3. **Implement** following the chain above
4. **Build and push**:
   ```bash
   # From local repo — push triggers GitHub Actions → builds Docker image
   git add -A && git commit -m "<message>" && git push
   ```
5. **Deploy to staging**:
   ```bash
   ssh root@46.224.215.213 "kubectl rollout restart deployment/testapp-backend -n gomuos"
   ```
   Wait ~60s, verify the pod is Running:
   ```bash
   ssh root@46.224.215.213 "kubectl get pods -n gomuos | grep testapp-backend"
   ```
6. **Smoke test** — curl the changed endpoint against staging to confirm it responds correctly
7. **Mark ticket done**: `PATCH https://lab.gomuos.com/api/tickets/{id}` with `{ "status": "done" }`
8. **Create follow-up tickets** (see below)

## After Implementation

**Code review ticket** (always):
```json
{
  "title": "Review: <feature name> backend changes",
  "description": "Backend changes for '<feature>' pushed and deployed to testapp-backend. Review for security, contract consistency, and conventions.",
  "severity": "info",
  "jobName": "gomuos-backend-developer",
  "assignedAgent": "gomuos-code-reviewer"
}
```

**E2E test ticket** (when the change affects a user-facing flow):
```json
{
  "title": "Test: <feature name> on staging",
  "description": "Backend for '<feature>' is live on testapp.gomuos.com. Run E2E tests covering the affected flow.",
  "severity": "info",
  "jobName": "gomuos-backend-developer",
  "assignedAgent": "gomuos-playwright-tester"
}
```

## Report Format

```
Ticket: #<id> — <title>
Files changed: <list>
Endpoint: <METHOD> /api/<path> — verified on staging ✓
Follow-up tickets: #<id> (code-reviewer), #<id> (playwright-tester)
```
