---
name: kidsapp
description: >
  Full project context for KidsApp — a Unity 6.3 WebGL educational app for children 1-4 år, deployed
  to kidsapp.gomuos.com via GameCI → GHCR → ArgoCD → k3s. Multiple mini-games in one app, sound-only
  (no language/text), demo-quality first then design pass after ~10 mini-games. Use this skill when
  working on the kids-app repo, building or modifying mini-games, debugging the WebGL build, or
  navigating the project. For agent-team specifics (manager, scouts, devs, code-reviewer) prefer
  the kidsapp-team skill. For deep deploy infrastructure prefer the kidsapp-infra skill.
---

# KidsApp

A Unity 6.3 WebGL app containing many small mini-games for children aged 1-4. Inspired by Bimi Boo's ecosystem of theme-apps, but consolidated into a single demo app. Sound-only, no language, no readable text — the app must be understandable globally without translation.

## Project at a Glance

| | |
|--|--|
| Engine | Unity 6.3 LTS (`6000.3.0f1`) |
| Build target | WebGL (initially); Android + iOS later |
| Repo | `https://github.com/shpatjakupi/kids-app` (private) |
| Local path | `C:\Users\shpat\Desktop\projects\kids-app` |
| VPS path | `/home/vegapunk/projects/kids-app` |
| Live URL | `https://kidsapp.gomuos.com` |
| Build pipeline | GitHub Actions (GameCI) → `ghcr.io/shpatjakupi/kids-app:latest` |
| k3s manifests | `infra-gitops/apps/kidsapp/` → ArgoCD app `kidsapp` |
| Lab workspace | `kidsapp` (`workspaceId: 3`) — `https://lab.gomuos.com/kidsapp` |

## Vision and Constraints

Read `~/.claude/skills/kidsapp-team/GOALS.md` for the full vision and current priorities. Hard constraints (do not violate):

- **Målgruppe**: 1-4 år
- **Sprog**: ingen tale, intet tekst i UI — kun lyde og ikoner
- **Demo-quality first** — single mekanik per mini-game, placeholder-grafik OK; full design pass kommer efter ~10 mini-games
- **Engine version is locked** — do not bump Unity until we are explicitly ready

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
│   ├── Dockerfile                 # nginx + WebGL build artifact
│   └── nginx.conf                 # COOP/COEP for SharedArrayBuffer + br/gz/wasm MIME
├── .github/workflows/build.yml    # GameCI WebGL build + GHCR docker push
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

## WebGL Hard Constraints

These break the build or crash at runtime — never use them in runtime code:

- `System.Threading.Thread`, `Task.Run`, `ThreadPool` (single-threaded — `async/await` is fine)
- `System.IO.File` for file system access (use `PlayerPrefs` for tiny state)
- Reflection that emits IL (`Expression.Compile`, `DynamicMethod`)
- `using UnityEditor;` outside `#if UNITY_EDITOR ... #endif` blocks (does not exist in builds)

WebGL build size matters — every MB adds load time. Target < 20 MB compressed for v1. Audio: import as `DecompressOnLoad`.

## Build & Deploy Flow

```
edit C# / scene / prefab files
   ↓ git push origin main
GitHub Actions
   ├─ game-ci/unity-builder@v4 → WebGL output (~10 min, cached Library/)
   └─ docker build & push → ghcr.io/shpatjakupi/kids-app:latest
   ↓
ArgoCD auto-syncs apps/kidsapp/deployment.yaml (~3 min)
   ↓
kubectl rollout restart deployment/kidsapp -n gomuos  (forces :latest pull)
   ↓
https://kidsapp.gomuos.com
```

After pushing, you can verify:
```bash
gh run list --workflow=build.yml --limit 1
ssh root@46.224.215.213 "kubectl rollout status deployment/kidsapp -n gomuos"
curl -sI https://kidsapp.gomuos.com | head -5
```

If GameCI fails: read `gh run view <id> --log-failed`. Most failures are missing `.meta` files or scenes not in `EditorBuildSettings.asset`. See `kidsapp-infra` skill for full troubleshooting.

## Adding a New Mini-Game (typical flow)

1. **Plan**: read the approved idea-ticket on `lab.gomuos.com/kidsapp`. Mekanik, læringsmål, lyde, assets-estimat are already in the description.
2. **Scene**: create `Assets/Scenes/<MiniGame>.unity` (copy MainScene.unity as template, regenerate GUID). Add to `EditorBuildSettings.asset`.
3. **Scripts**: `Assets/Scripts/Gameplay/<MiniGame>/*.cs` for logic. Use ScriptableObjects for data, MonoBehaviours for runtime.
4. **Hub entry**: register the mini-game in the hub-screen (UI Toolkit) so the user can launch it.
5. **Audio**: place royalty-free clips in `Assets/Audio/<MiniGame>/`. Set import type to `DecompressOnLoad` in the `.meta`.
6. **Universal back button**: every mini-game scene must include the standard back button (returns to hub). Same prefab everywhere — same position.
7. **Stjerne-belønning**: on completion, trigger the shared reward animation/sound (lives in `Assets/Prefabs/Reward.prefab` or similar — check current state).
8. **Push**: GameCI builds, ArgoCD deploys, verify on `kidsapp.gomuos.com`.
9. **Code review**: `kidsapp-code-reviewer` reviews automatically when ticket is marked done.

## Lab Workspace `kidsapp`

All tickets for this project live in workspace `kidsapp`. Ideas come from three scout agents (run daily), implementations from `kidsapp-unity-developer`, validation from `kidsapp-code-reviewer`. The agent-side details are in the `kidsapp-team` skill — read that one when working on the agents themselves.

When creating tickets via API from this project, always include `"workspace": "kidsapp"` in the body. Without it the ticket lands in the default `gomuos` workspace and is invisible on `lab.gomuos.com/kidsapp`.

## Related Skills

| When you... | Use skill |
|---|---|
| Work on the agent team itself (manager, scouts, dev workflow, routing) | `kidsapp-team` |
| Touch build pipeline, GameCI, k3s manifests, Docker, ArgoCD | `kidsapp-infra` |
| Need general k3s / Traefik / cert-manager / ArgoCD knowledge | `vps-cluster` |
| Compare with the GomuOS food platform (different team, same VPS) | `gomuos-infra`, `gomuos-team`, `order-app` |

## Common Gotchas

- **Missing `.meta` file** → Unity ignores the asset, references break silently. Always create both.
- **Scene not in `EditorBuildSettings.asset`** → `SceneManager.LoadScene("Foo")` fails at runtime.
- **`replicas: 0` in k3s deployment** → has happened before via linter/auto-edit. Check `apps/kidsapp/deployment.yaml` if pod count is 0.
- **`:latest` image tag stale on pod** → `kubectl rollout restart deployment/kidsapp -n gomuos` to force pull.
- **mcp-unity package** in `Packages/com.gametery.mcp-unity/` is for the local Editor only (Editor-bound assemblies). Do not import its types from runtime code.
- **Bimi Boo "splash screen"** — the Unity Personal "Made with Unity" 4-second splash shows at app start. Cannot be removed without Pro license. Acceptable for now.
