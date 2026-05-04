# Ticket Protocol

Full lifecycle and field conventions for lab tickets.

## Ticket Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | number | Auto-assigned |
| `title` | string | Short imperative description (max ~80 chars) |
| `description` | string | What was observed, why it matters, what to do |
| `severity` | `info \| warning \| error \| critical` | See severity guide below |
| `status` | `pending \| approved \| rejected \| done` | Lifecycle state |
| `jobName` | string | Which cron job or agent created the ticket |
| `assignedAgent` | string \| null | Which agent should handle this ticket |
| `delegatedBy` | string \| null | Agent that created this ticket for another agent |

## Ticket Lifecycle

```
[agent creates ticket]
        |
        v
   status: pending
        |
   human reviews in lab.gomuos.com
        |
   ┌────┴────┐
approved   rejected
   |
   v
agent picks up ticket, executes work
   |
   v
status: done  (agent marks via PATCH /api/tickets/{id}/done)
```

Human approval is **always required** before an agent acts on a ticket. Agents must not poll tickets faster than once per minute.

## Severity Guide

| Severity | When to use |
|----------|-------------|
| `info` | Routine observation, FYI, low-urgency improvement idea |
| `warning` | Something is suboptimal, degraded, or needs attention soon |
| `error` | Something is broken or failing, action required |
| `critical` | Production is down, data at risk, immediate action needed |

## Creating Tickets

**POST** `https://lab.gomuos.com/api/tickets`
Headers: `Authorization: Bearer $LAB_API_KEY`

```json
{
  "title": "Short imperative description",
  "description": "What was observed, why it matters, what the agent should do",
  "severity": "info | warning | error | critical",
  "jobName": "gomuos-manager"
}
```

After creation, assign the agent via PATCH:

**PATCH** `https://lab.gomuos.com/api/tickets/{id}`
```json
{ "assignedAgent": "gomuos-frontend-developer" }
```

## Fetching Open Tickets

**GET** `https://lab.gomuos.com/api/tickets?status=pending`

Returns all pending tickets. Filter by `assignedAgent: null` to find unrouted tickets.

## Polling for Approval

**GET** `https://lab.gomuos.com/api/tickets/{id}/poll`

Returns `{ status: "pending" | "approved" | "rejected" }`. Poll at most once per minute while waiting for human review.

## Marking Done

**PATCH** `https://lab.gomuos.com/api/tickets/{id}`
```json
{ "status": "done" }
```

## Conventions

- **One ticket per concern** — don't bundle unrelated work into one ticket
- **Title starts with a verb** — "Fix", "Add", "Review", "Investigate", "Refactor"
- **Description answers**: What did you observe? Why does it matter? What should the agent do?
- **Don't create duplicate tickets** — check `GET /api/tickets?status=pending` first
- **Severity reflects urgency**, not effort — a 5-minute fix can still be `critical`
