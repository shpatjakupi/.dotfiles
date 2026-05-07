# KidsApp Goals

## Vision

Én Unity 6.3 WebGL-app der indeholder mange små mini-spil til **børn 1-4 år**, hver med ét tydeligt læringsmål. Inspiration: Bimi Boo's økosystem af temaspil — men her samlet i én demo-app i stedet for spredt over 30+ apps.

## Konstanter (ændres ikke for v1)

- **Målgruppe**: 1-4 år
- **Sprog**: ingen tale, intet tekst i UI — kun lyde og ikoner. App'en skal kunne forstås uden at læse eller forstå et bestemt sprog
- **Engine**: Unity 6.3 LTS (6000.3.0f1), build target WebGL
- **Live URL**: kidsapp.gomuos.com
- **Funktionalitet før design**: hver mini-spil bygges som demo først (basic mekanik virker, placeholder-grafik OK), polering og unikt univers tilføjes når vi har ~10 mini-spil i kassen

## Læringsdomæner — hvert kan blive flere mini-spil

| Domæne | Eksempler på mekanikker |
|---|---|
| Bogstaver | Trace, matching, find-bogstavet, bogstav-til-lyd |
| Tal & tælling | Tæl-objekter, mængde-genkendelse, simple addition |
| Farver | Sortér efter farve, find-farven, farveblanding |
| Former | Match form til hul, sortér former, byg med former |
| Lyde | Dyrelyde, instrumenter, hverdagslyde |
| Finmotorik | Tegne med finger, drag-drop, tap-rytme |
| Hukommelse | Memory cards, vend-par, sekvenser |
| Logik | Sortering, mønstre, simpel årsag-effekt |

## Active priorities

- [ ] Mini-spil #1-3 i kassen — formentlig fra simpleste domæne (cause-effect, lyd-til-billede)
- [ ] Hub-skærm der viser alle mini-spil som ikoner
- [ ] Lyd-bibliotek (royalty-free) sat op i `Assets/Audio/`
- [ ] Stjerne-belønning ved gennemført spil (genbruges på tværs)
- [ ] Universel "tilbage"-knap (samme position i alle mini-spil)

## Pipeline

1. **Scouts** (kører ugentligt) opretter idé-tickets med spil-forslag
2. **Du** godkender de idéer du vil have bygget
3. **kidsapp-unity-developer** implementerer som demo (1-2 timer pr. mini-spil)
4. **kidsapp-code-reviewer** validerer at det bygger og ikke breaker noget
5. Når vi rammer **~10 mini-spil**: stop-pause for design, karakter-univers, lyd-design

## Completed

- [x] Unity 6.3 projekt + GameCI WebGL pipeline → kidsapp.gomuos.com live
- [x] kidsapp-team skills (manager, dev, code-reviewer, infra)
- [x] Lab workspace `kidsapp` + `/kidsapp` UI

## Out of scope (for v1)

- Original karakter-univers (kommer efter ~10 mini-spil)
- Tale eller tekst i UI
- Stort lyd-univers (vi bruger royalty-free for nu)
- Ægte progression-system med gem mellem sessioner
- App Store / Play Store udgivelse — vi tester WebGL i browser først
