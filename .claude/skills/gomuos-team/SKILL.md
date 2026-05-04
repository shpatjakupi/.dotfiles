---
name: gomuos-team
description: Team of specialized GomuOS agents — sub-skills for backend, frontend, infra, code review, UI review, playwright testing, project management, and domain specialists for checkout, wolt, menu, orders, and admin.
---

# GomuOS Team

Umbrella skill containing specialized sub-skills for the GomuOS food ordering platform.

## Implementors

These agents write code and deploy to staging:

- **gomuos-backend-developer** — Spring Boot backend implementation
- **gomuos-frontend-developer** — Next.js frontend implementation

## Domain Specialists (Tech Leads)

These agents do NOT write code. They analyze tickets and create precise sub-tickets for implementors:

- **gomuos-checkout-specialist** — Checkout flow, Bambora payment, promo codes
- **gomuos-wolt-specialist** — Wolt delivery webhooks and API client
- **gomuos-menu-specialist** — Food, beverages, fillings, sauces, combos, recommendations
- **gomuos-orders-specialist** — Order lifecycle, real-time WebSocket updates, order history
- **gomuos-admin-specialist** — Shop config, work hours, employees, account management

## Reviewers & QA

- **gomuos-code-reviewer** — Security and convention review
- **gomuos-ui-reviewer** — UI/UX review
- **gomuos-playwright-tester** — E2E testing on staging

## Orchestration

- **gomuos-manager** — Routes tickets to domain specialists or implementors
- **gomuos-team-setup** — Team orchestration setup

## Ticket Routing Logic

```
New ticket arrives
    ↓
gomuos-manager reads ticket
    ↓
Domain-specific? → route to domain specialist
    ├── checkout/payment/promo → gomuos-checkout-specialist
    ├── wolt/delivery webhook → gomuos-wolt-specialist
    ├── menu/food/beverage/filling/combo → gomuos-menu-specialist
    ├── order lifecycle/admin orders → gomuos-orders-specialist
    └── admin panel/shop config/employees → gomuos-admin-specialist
    ↓
Specialist creates [BE] and [FE] sub-tickets
    ↓
gomuos-backend-developer ← [BE] tickets
gomuos-frontend-developer ← [FE] tickets
    ↓
gomuos-code-reviewer ← after implementation
gomuos-playwright-tester ← after staging deploy
```
