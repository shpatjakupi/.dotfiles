# GomuOS Team Goals

These goals guide all agents — hunters prioritize audits here, devs prioritize implementation here.

---

## Priority 1: Checkout Flow (HIGHEST)

**Mål:** Det komplette checkout-flow skal være perfekt — design, UX og convenience.

Scope:
- Kunden lægger varer i kurven
- Kunden udfylder checkout-formularen (adresse, leveringstid, kontaktinfo)
- Kunden betaler via Bambora
- Kunden lander på bekræftelsessiden med en countdown-timer til ordren er færdig

Hvad "perfekt" betyder her:
- Designet skal være smukt, poleret og moderne — ikke bare funktionelt
- Flowet skal være friktionsfrit — færre klik, tydelige fejlbeskeder, ingen forvirring
- Loading states på alle async handlinger (tilføj til kurv, submit, betaling)
- Mobile-first — de fleste kunder bestiller på telefonen
- Countdown-timeren skal føles levende og tryg for kunden

Hunters der arbejder på dette mål:
- `gomuos-checkout-specialist` — backend/flow-problemer
- `gomuos-ui-reviewer` — design og UX

---

## Priority 2: Admin Dashboard (ORDER MANAGEMENT)

**Mål:** Admin-dashboardet skal fungere fejlfrit til ordremodtagelse og -håndtering.

Scope:
- Realtids-notifikationer når nye ordrer ankommer (WebSocket)
- Ordreoversigt er klar og handlingsbar
- Ordrestatus-opdateringer virker korrekt
- Ingen forsinkelser, crashes eller manglende opdateringer

Hunters der arbejder på dette mål:
- `gomuos-admin-specialist` — admin panel og konfiguration
- `gomuos-orders-specialist` — ordre-livscyklus og WebSocket

---

## Priority 3: Wolt Integration

**Mål:** Hele Wolt-leveringsflowet skal spille 100%.

Scope:
- Webhook-modtagelse og -behandling er robust
- Leveringsstatus opdateres korrekt i systemet
- Fejlhåndtering ved Wolt API-fejl
- Ingen tabte events

Hunters der arbejder på dette mål:
- `gomuos-wolt-specialist` — Wolt webhooks og delivery events

---

## Completed Goals

*(ingen endnu)*

---

## For Agents

Når du opretter tickets, angiv hvilket mål ticketen relaterer til i `description`-feltet.
Prioriter altid **Priority 1** højest — en ticket om checkout-design er vigtigere end en generel code quality-ticket.
