# Agent Roster

Full list of available agents, their responsibilities, and routing rules.

## Agents

### gomuos-ui-reviewer
**Trigger**: Any commit touching frontend components, pages, CSS, or layout changes.
**Does**: Reviews screenshots or code diffs for UX issues, visual bugs, mobile responsiveness, and design consistency.
**Route when**: Recent git commits touch `next-app-template/` or the frontend pod has been redeployed.

### gomuos-code-reviewer
**Trigger**: Any new feature branch, PR, or significant refactor.
**Does**: Reviews code for correctness, security, performance, and adherence to project conventions (Spring Boot patterns, Next.js App Router conventions).
**Route when**: Ticket describes a code review request, or a feature ticket has been marked done and needs review before approval.

### gomuos-frontend-developer
**Trigger**: Feature requests, bug fixes, or UI changes on the Next.js frontend.
**Does**: Implements new pages, components, API routes, TypeScript interfaces, and service functions in `next-app-template`.
**Route when**: Ticket involves frontend code changes, new customer-facing features, or frontend bug fixes.

### gomuos-backend-developer
**Trigger**: Feature requests, bug fixes, or API changes on the Spring Boot backend.
**Does**: Implements entities, DTOs, DAOs, services, and controllers in `order-backend`. Handles DB schema changes, email templates, WebSocket events.
**Route when**: Ticket involves backend API, database, authentication, payment, Wolt integration, or server-side logic.

### gomuos-playwright-tester
**Trigger**: After a feature is implemented and deployed to staging (`testapp.gomuos.com`).
**Does**: Writes and runs Playwright end-to-end tests covering the happy path and edge cases for new or changed features.
**Route when**: A feature ticket has been implemented and needs E2E validation before production.

## Routing Decision Tree

```
Is it a visual/UX issue?
  → gomuos-ui-reviewer

Is it a code quality concern or needs review?
  → gomuos-code-reviewer

Is it a new feature or bug fix?
  → Frontend change?  → gomuos-frontend-developer
  → Backend change?   → gomuos-backend-developer
  → Both?             → Create two tickets, one per agent

Has a feature been deployed to staging and needs E2E testing?
  → gomuos-playwright-tester
```

## Unroutable Tickets

If a ticket doesn't clearly map to one agent:
- Split it into sub-tickets, one per agent
- Or escalate to the human with a `warning` severity ticket explaining the ambiguity

## Agent Communication Pattern

Agents do not call each other directly. They communicate via tickets:
1. Agent A creates a ticket for Agent B (sets `assignedAgent: "gomuos-agent-b"`)
2. Human approves the ticket
3. Agent B picks it up on next run

The manager's job is to ensure tickets are routed before agents run — not to coordinate them in real-time.
