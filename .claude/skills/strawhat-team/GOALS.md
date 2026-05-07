# Straw Hats Team Goals

These goals guide the manager when deciding which projects to push forward and what quality bar to hold the team to.

---

## Priority 1: Velocity from wish to deployed

**Mål:** Et nyt ønske skal kunne gå fra menneskets idé til en kørende app på `<slug>.gomuos.com` så friktionsfrit som muligt.

Hvad det betyder for hvert led:
- **Intake** stiller maksimum 5 fokuserede spørgsmål før den skriver en spec. Spec skal være kort (én side) og handlings-orienteret.
- **Architect** vælger den simplest mulige stack der opfylder spec'en. Hvis Next.js full-stack er nok, ingen grund til at trække Spring Boot ind.
- **Devops** opretter altid struktur med både `backend/` og `frontend/` mapper, selv hvis kun én bruges — så er der plads til vækst.
- **Devs** committer hyppigt; små PR'er så human kan reviewe undervejs.
- **QA** tester success-kriterierne fra spec'en, ikke implementation-detaljer.

---

## Priority 2: Konsistens på tværs af apps

**Mål:** Alle Straw Hats apps skal føles som søsterskibe — samme designsprog, samme login-flow, samme ops-mønstre.

- Frontend: Mantine UI 7 default tema, sammenlignelig med ordrupspizza
- Backend: Spring Boot 3.1, Hibernate, samme MySQL host
- Deploy: Domain `<slug>.gomuos.com`, ArgoCD app, cert-manager TLS
- README.md skal altid have: hvad app'en gør, lokal dev-kommandoer, deploy-status badge

---

## Priority 3: Skib hellere end at perfektionere

Et v1 deployed slår en perfekt v3 i design. Hvis architect kan se at en feature kan vente til v2, så skal den vente. Ingen analyse-paralyse.

---

## For Agents

Når du opretter tickets, angiv hvilken priority de relaterer til i `description`-feltet hvis det er relevant. Manager prioriterer **Priority 1** højest når der skal vælges mellem to projekter samtidigt.
