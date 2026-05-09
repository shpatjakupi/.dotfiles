---
name: kidsapp-team
description: >
  Full context for KidsApp — a Unity 6.3 educational app for children 1-4 år, shipped to
  Google Play (Android) and the App Store (iOS) via GameCI. Covers the project itself (repo
  structure, code conventions, mobile build constraints, build/deploy flow, mini-game
  checklist) AND the agent team that maintains it (manager, scouts, devs, code-reviewer,
  infra). Sound-only, no language/text, demo-quality first then design pass after
  ~10 mini-games. Use this skill for anything KidsApp-related. For deep deploy infrastructure
  prefer the kidsapp-infra skill.
---

# KidsApp

A Unity 6.3 mobile app containing many small mini-games for children aged 1-4, shipped to **Google Play (Android)** and the **App Store (iOS)**. Inspired by Bimi Boo's ecosystem of theme-apps, but consolidated into a single demo app. Sound-only, no language, no readable text — the app must be understandable globally without translation.

A team of specialized Claude Code agents researches ideas, builds mini-games, and reviews code. All agent actions require human approval via lab tickets at `lab.gomuos.com/kidsapp`.

## Project at a Glance

| | |
|--|--|
| Engine | Unity 6.3 LTS (`6000.3.0f1`) |
| Production targets | Android (Google Play) + iOS (App Store) |
| Dev preview target | WebGL at `https://kidsapp.gomuos.com` (test channel — every push lands here in ~13 min) |
| Repo | `https://github.com/shpatjakupi/kids-app` (private) |
| Local path | `C:\Users\shpat\Desktop\projects\kids-app` |
| VPS path | `/home/vegapunk/projects/kids-app` (CI checkout + WebGL preview host) |
| Distribution (prod) | Google Play Console + App Store Connect (TestFlight for iOS beta) |
| Build pipeline | GitHub Actions (GameCI): `webgl.yml` (preview), `android.yml` (.aab), `ios.yml` (.ipa) |
| Lab workspace | `kidsapp` (`workspaceId: 3`) — `https://lab.gomuos.com/kidsapp` |

## Vision and Constraints

Read `~/.claude/skills/kidsapp-team/GOALS.md` for the full vision and current priorities. Hard constraints (do not violate):

- **Målgruppe**: 1-4 år
- **Sprog**: ingen tale, intet tekst i UI — kun lyde og ikoner
- **Demo-quality first** — single mekanik per mini-game, placeholder-grafik OK; full design pass kommer efter ~10 mini-games
- **Engine version is locked** — do not bump Unity until we are explicitly ready

---

# Part 1 — The Project (kids-app repo)

## Repo Structure

```
kids-app/
├── Assets/
│   ├── Scenes/                    # .unity scene files (YAML)
│   ├── Scripts/                   # C# MonoBehaviours, ScriptableObjects, services
│   │   ├── Gameplay/              # mini-game logic, namespace KidsApp.Gameplay
│   │   ├── UI/                    # navigation/hub, namespace KidsApp.UI
│   │   ├── Audio/                 # sound playback, namespace KidsApp.Audio
│   │   └── Data/                  # ScriptableObject classes
│   ├── Prefabs/
│   ├── ScriptableObjects/         # asset instances (level data etc.)
│   ├── Audio/                     # royalty-free sound clips
│   └── UI/                        # UXML / USS for UI Toolkit
├── Packages/
│   ├── manifest.json
│   └── com.gametery.mcp-unity/    # local Unity MCP package (Editor-only, not bundled in build)
├── ProjectSettings/
│   ├── EditorBuildSettings.asset  # scene list (every shipping scene MUST be enabled here)
│   └── McpUnitySettings.json      # WebSocket port for MCP (Editor only)
├── docker/
│   ├── Dockerfile                 # nginx + WebGL build — for kidsapp.gomuos.com preview
│   └── nginx.conf                 # COOP/COEP for SharedArrayBuffer + br/gz/wasm MIME
├── .github/workflows/
│   ├── webgl.yml                  # GameCI WebGL build → GHCR → ArgoCD (dev preview)
│   ├── android.yml                # GameCI Android build → signed .aab → Play Console upload
│   └── ios.yml                    # GameCI iOS build → .ipa → App Store Connect / TestFlight
└── README.md
```

## Working at the file level (no Unity Editor needed for code)

All Unity assets are text — scenes, prefabs, ScriptableObjects, materials, meta files are YAML; scripts are C#. You can build entire mini-games by writing files. The GameCI runner does the actual headless build.

When you add ANY new asset (`.cs`, `.unity`, `.prefab`, `.asset`, `.uxml`, image, audio):
1. Create the asset file
2. Create a matching `.meta` file with a unique 32-hex GUID
3. If it's a scene, add to `ProjectSettings/EditorBuildSettings.asset`
4. If it's a runtime asset referenced by code by path, ensure the path is correct

A scene file template lives in `Assets/Scenes/MainScene.unity`. The simplest meta file shape:
```yaml
fileFormatVersion: 2
guid: <32-hex-chars>
DefaultImporter:
  externalObjects: {}
  userData:
  assetBundleName:
  assetBundleVariant:
```
For `.cs` files use `MonoImporter:` instead of `DefaultImporter:` (see existing scripts for the exact shape).

## Code Conventions

- **Namespace**: `KidsApp.<Domain>` mirroring folder (`Scripts/Gameplay/Foo.cs` → `namespace KidsApp.Gameplay`)
- **Class name = file name**
- **Inspector fields**: `[SerializeField] private` + property — never raw `public` mutable fields
- **No `Find` / `GetComponent` in `Update`** — cache in `Awake` / `Start`
- **`[RequireComponent(typeof(...))]`** to declare component dependencies
- **ScriptableObjects** for any configuration data (level definitions, audio clip refs, character stats)
- **UI Toolkit** (UXML + USS) for UI — not legacy uGUI Canvas. Smaller WebGL bundle, modern direction.

## Build Constraints

Code must satisfy **all three** targets (WebGL preview + Android + iOS prod):

- **No threading / `Task.Run` / `ThreadPool`** — WebGL is single-threaded.
  `async/await` is fine.
- **No `System.IO.File`** for runtime persistence — WebGL has no real file
  system. Use `PlayerPrefs` for tiny state across all platforms.
- **No reflection that emits IL** (`Expression.Compile`, `DynamicMethod`) —
  WebGL strips IL emit, and iOS IL2CPP throws `PlatformNotSupportedException`
  on it. Use `[Preserve]` and avoid runtime code-gen.
- **`using UnityEditor;`** outside `#if UNITY_EDITOR ... #endif` blocks
  breaks every player build. Always guard editor-only code.
- **Touch input only** — design for one-finger taps. Mouse events still
  work in editor and WebGL desktop testing.
- **Build size matters everywhere** — WebGL target < 20 MB compressed, APK
  base < 150 MB (Play warning), IPA cellular < 200 MB. Audio: `Vorbis`
  compressed for mobile, `DecompressOnLoad` for WebGL.
- **Min SDK / iOS target**: Android `minSdkVersion 24` (Android 7.0+),
  iOS deployment target `13.0+`.
- **Background suspension** — iOS aggressively suspends; pause audio in
  `OnApplicationPause(true)` and resume on `false`.

## Build & Deploy Flow

There are two flows: a **fast dev preview** (WebGL) that fires on every push
and is the daily test loop, and **production release** to the mobile stores
which is invoked deliberately.

### Dev preview (every push to `main`)

```
edit C# / scene / prefab files
   ↓ git push origin main
GitHub Actions
   ├─ webgl.yml → GameCI WebGL build (~10 min)
   │              → docker build & push → ghcr.io/shpatjakupi/kids-app:latest
   ↓
ArgoCD auto-syncs apps/kidsapp/deployment.yaml (~3 min)
   ↓
kubectl rollout restart deployment/kidsapp -n gomuos   (forces :latest pull)
   ↓
https://kidsapp.gomuos.com  ← open on phone for touch test
```

Verify after pushing:
```bash
gh run list --workflow=webgl.yml --limit 1
ssh root@46.224.215.213 "kubectl rollout status deployment/kidsapp -n gomuos"
curl -sI https://kidsapp.gomuos.com | head -5
```

### Production release (manual, when ready)

```
gh workflow run android.yml   # → game-ci → signed .aab → fastlane supply → Play Internal Testing
gh workflow run ios.yml       # → game-ci → fastlane match+gym → pilot → TestFlight
   ↓
Smoke-test on Play Internal Testing + TestFlight
   ↓
Promote: Play Closed/Open → Production, App Store review submission
```

If any GameCI run fails: most failures are missing `.meta` files, scenes not in `EditorBuildSettings.asset`, or signing-credential drift (Android keystore secret, iOS provisioning profile expiry). See `kidsapp-infra` skill for full troubleshooting.

## Adding a New Mini-Game (typical flow)

1. **Plan**: read the approved idea-ticket on `lab.gomuos.com/kidsapp`. Mekanik, læringsmål, lyde, assets-estimat are already in the description.
2. **Scene**: create `Assets/Scenes/<MiniGame>.unity` (copy MainScene.unity as template, regenerate GUID). Add to `EditorBuildSettings.asset`.
3. **Scripts**: `Assets/Scripts/Gameplay/<MiniGame>/*.cs` for logic. Use ScriptableObjects for data, MonoBehaviours for runtime.
4. **Hub entry**: register the mini-game in the hub-screen (UI Toolkit) so the user can launch it.
5. **Audio**: place royalty-free clips in `Assets/Audio/<MiniGame>/`. Set import type to `DecompressOnLoad` in the `.meta`.
6. **Universal back button**: every mini-game scene must include the standard back button (returns to hub). Same prefab everywhere — same position.
7. **Stjerne-belønning**: on completion, trigger the shared reward animation/sound (lives in `Assets/Prefabs/Reward.prefab` or similar — check current state).
8. **Push**: `webgl.yml` runs automatically; ~13 min later the new mini-game is live at `kidsapp.gomuos.com`. Open it on your phone for touch test.
9. **Code review**: `kidsapp-code-reviewer` reviews automatically when ticket is marked done.
10. **Release** (when batch of mini-games is ready): manually trigger `android.yml` + `ios.yml` to ship to Play Internal Testing / TestFlight, then promote to production.

## Common Gotchas

- **Missing `.meta` file** → Unity ignores the asset, references break silently. Always create both.
- **Scene not in `EditorBuildSettings.asset`** → `SceneManager.LoadScene("Foo")` fails at runtime.
- **WebGL preview shows old build** → ArgoCD synced the manifest but the `:latest` tag didn't change. `kubectl rollout restart deployment/kidsapp -n gomuos` to force the pull.
- **`replicas: 0` in k3s deployment** → has happened before via linter/auto-edit on `apps/kidsapp/deployment.yaml`. ArgoCD will keep scaling the pod down until you fix it back to `1`.
- **Android keystore drift** → if the `.aab` upload to Play fails with "package already uses a different signing key", the keystore secret in GitHub Actions has rotated. Rebuild from the original keystore — never re-sign with a new one.
- **iOS provisioning profile expiry** → `fastlane match` regenerates; check `kidsapp-infra` for the renewal flow.
- **`bundleVersion` / `bundleVersionCode` not bumped** → both stores reject duplicate version numbers. CI auto-increments via the build number; manual pushes must bump it.
- **mcp-unity package** in `Packages/com.gametery.mcp-unity/` is for the local Editor only (Editor-bound assemblies). Do not import its types from runtime code.
- **"Made with Unity" splash** — Unity Personal's 4-second splash shows at app start. Cannot be removed without Pro license. Acceptable for now.

---

# Part 2 — The Agent Team

## Team Structure

```
kidsapp-team/
├── manager/
│   └── kidsapp-manager              ← orchestrator, routes tickets, reads GOALS.md
├── scouts/                           ← proactive idea-researchers (cron, daily)
│   ├── kidsapp-bimiboo-scout        ← Bimi Boo + lignende app-makere
│   ├── kidsapp-edu-research-scout   ← Montessori, Piaget, pædagogisk forskning
│   └── kidsapp-physical-toy-scout   ← fysisk legetøj → digital oversættelse
├── devs/
│   └── kidsapp-unity-developer      ← C# scripts, scenes, prefabs, ScriptableObjects
├── validators/
│   └── kidsapp-code-reviewer        ← Unity best practices, performance, security
└── infra/
    └── kidsapp-infra                ← deploy knowledge (build pipeline, k3s, ArgoCD)
```

Playwright-tester / WebGL-tester comes later when there is enough gameplay to audit.

## What Each Group Does

### Manager
Orchestrates the team: routes unassigned tickets to the right agent, reads `~/.claude/skills/kidsapp-team/GOALS.md` for current priorities, creates tickets for uncaptured work. Never writes code.

### Scouts (run on cron, daily)
Proactively research external sources for mini-game ideas and create idea-tickets in the kidsapp workspace. Each scout has a different angle so we don't get monoculture ideas.

| Scout | Angle |
|-------|-------|
| `kidsapp-bimiboo-scout` | What Bimi Boo and competitors already ship |
| `kidsapp-edu-research-scout` | What pedagogy (Montessori, Piaget, etc.) suggests |
| `kidsapp-physical-toy-scout` | Physical toys/games translated into digital mechanics |

Scouts create tickets with `assignedAgent: null` — the human approves the idea, then the manager routes approved ideas to `kidsapp-unity-developer`.

### Devs
Implement approved tickets. Do NOT write code until a ticket is approved by the human.

| Agent | Domain |
|-------|--------|
| `kidsapp-unity-developer` | Unity 6.3, C# scripts, scenes, prefabs, UI Toolkit, ScriptableObjects |

### Validators
Review what the dev produced.

| Agent | Domain |
|-------|--------|
| `kidsapp-code-reviewer` | C# code quality, Unity API misuse, performance, security |

### Infra (knowledge skill)
Reference for build pipeline, GameCI, k3s deployment, ArgoCD wiring. Not a runnable agent — it is loaded by other agents who need deployment knowledge.

## Full Pipeline

```
[Scout opretter idé-ticket via cron] eller [Human writes ticket]
   ↓
   Auto-routed by kidsapp-manager → kidsapp-unity-developer (or human assigns directly)
   ↓
   Human approves
   ↓
   ticket-dispatcher (Vegapunk cron) spawns kidsapp-unity-developer
   ↓
   Implements in /home/vegapunk/projects/kids-app
   Pushes to main → GitHub Actions builds WebGL via GameCI (~6 min)
                  → Docker image to GHCR
                  → ArgoCD syncs (~3 min)
                  → Live at kidsapp.gomuos.com
   ↓
   Dev creates follow-up ticket → kidsapp-code-reviewer
   ↓
   Human approves → reviewer reads diff, creates issue tickets if needed
```

## Ticket Status Flow

```
pending ──→ approved ──→ in_progress ──→ done
   │                                      ↑
   │  human clicks "Spørg agent"
   ├──→ needs_response ──→ in_progress ──→ pending (agent answered)
   │
   └──→ rejected
```

## Lab API

- **URL**: `https://lab.gomuos.com/api`
- **Auth**: `Authorization: Bearer $LAB_API_KEY`
- **Workspace**: always `kidsapp` — every ticket created by this team MUST include `"workspace": "kidsapp"` in the body

| Endpoint | Use |
|----------|-----|
| `GET /api/tickets?workspace=kidsapp&status=<s>` | Fetch tickets |
| `POST /api/tickets` | Create ticket — body must include `"workspace": "kidsapp"` |
| `PATCH /api/tickets/{id}` | Update status, assignedAgent, executionLog |
| `POST /api/tickets/{id}/comments` | Post comment as agent |

**Create ticket:**
```json
{
  "title": "Verb-first short description",
  "description": "What was observed, why it matters, what to do",
  "severity": "info | warning | error | critical",
  "jobName": "kidsapp-<your-name>",
  "assignedAgent": "kidsapp-<target-agent>",
  "workspace": "kidsapp"
}
```

## Cron Integration (Vegapunk)

| Job | Interval |
|-----|----------|
| `kidsapp-manager` | Every 2 hours |
| `kidsapp-bimiboo-scout` | Daily 13:00 |
| `kidsapp-edu-research-scout` | Daily 14:00 |
| `kidsapp-physical-toy-scout` | Daily 15:00 |
| `ticket-dispatcher` | Every 3 min — picks up `approved` and `needs_response` for all workspaces |

The dispatcher chooses `cwd` based on agent name. For all `kidsapp-*` agents it uses `/home/vegapunk/projects/kids-app`.

## Project Goals

Active priorities live in `~/.claude/skills/kidsapp-team/GOALS.md`. The manager reads this to decide what to work on next. Format:

```markdown
# Current Goals
- [ ] <active goal>
- [x] <completed goal>
## Priority: <what matters most right now>
```

## Skills Location

```
dotfiles: .dotfiles/.claude/skills/kidsapp-team/
local:    ~/.claude/skills/  (installed flat by sync-skills)
```

## How to Add a New Agent

1. Create `dotfiles/.claude/skills/kidsapp-team/<group>/<agent-name>/SKILL.md`
2. Update this file's "Team Structure"
3. Update `kidsapp-manager/SKILL.md` routing rules
4. If reactive only — the dispatcher already maps `kidsapp-*` → `kids-app` cwd
5. If runs on schedule — add a cron job in `vegapunk/src/infra/cron.ts`
6. Commit + push, run sync-skills on VPS

## How to Modify an Existing Agent

1. Edit the skill file in `dotfiles/.claude/skills/kidsapp-team/<group>/<agent-name>/SKILL.md`
2. Commit + push → sync-skills on VPS

## Related Skills

| When you... | Use skill |
|---|---|
| Touch build pipeline, GameCI, k3s manifests, Docker, ArgoCD | `kidsapp-infra` |
| Need general k3s / Traefik / cert-manager / ArgoCD knowledge | `vps-cluster` |
| Compare with the GomuOS food platform (different team, same VPS) | `gomuos-team`, `order-app` |
