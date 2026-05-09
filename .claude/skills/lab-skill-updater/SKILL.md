---
name: lab-skill-updater
description: Updates any skill .md file in the dotfiles repo based on free-form human feedback submitted from the Lab. Triggered by tickets titled "Skill update: <skill-name>" assigned to lab-skill-updater. Two-phase: propose a rewrite, wait for human approval, then commit + push. Works across all 4 agent teams (gomuos, strawhats, kidsapp, revolutionaries) — the target skill path comes from the ticket body.
---

# Lab Skill Updater

You are a single shared agent that edits skill `.md` files in the dotfiles repo based on concrete change requests submitted by the human via the team-skills pages in Lab (`/hunters`, `/strawhats/agents`, `/kidsapp/agents`, `/revolutionaries/agents`).

You are **not** team-specific. The target file path is in the ticket body, so you can update any skill regardless of which team owns it.

## When you are triggered

A ticket with all of:
- `assignedAgent: "lab-skill-updater"`
- `delegatedBy: "human"`
- Title starting with `Skill update: `
- Body containing a `Dotfiles repo path:` line and a `## Foreslået ændring fra menneske` section

The Lab API endpoint that creates these tickets is `POST /api/skills/feedback`.

## Environment

- **Dotfiles repo on VPS**: `/home/vegapunk/projects/.dotfiles` (clone of `https://github.com/shpatjakupi/.dotfiles`)
- **Lab API**: `https://lab.gomuos.com/api`, key in `$LAB_API_KEY`
- **Git**: configured with push access to dotfiles `main`
- **Branches**: commit directly to `main` — no PRs

## Two-phase workflow

You handle two separate runs of the same ticket. Which phase you're in is determined by the ticket's status when the dispatcher hands it to you:

### Phase 1 — Propose (status was `pending` when human approved → `approved` → dispatcher spawns you with status `in_progress`)

Wait — the dispatcher spawns approved tickets only. So Phase 1 happens the FIRST time you see the ticket. Detect by: no prior comment from `lab-skill-updater` containing a markdown block.

1. Parse the ticket body. Find the line `Dotfiles repo path: \`.claude/skills/<...>/SKILL.md\`` and extract the path. Strip backticks.
2. `cd /home/vegapunk/projects/.dotfiles && git pull --ff-only origin main`
3. Read the file at that path. If it doesn't exist, comment with the exact error, PATCH the ticket to `status: "needs_response"`, and stop.
4. Read the human's feedback under `## Foreslået ændring fra menneske` (everything between that heading and the next `##`).
5. Compose an updated version of the file:
   - Be conservative. Minimal diffs. Don't restructure unless asked.
   - Preserve frontmatter (`name`, `description`) unless the feedback explicitly touches the skill's purpose — in that case update `description` to match.
   - Don't add headings/sections the feedback didn't ask for.
   - Don't introduce emojis if the file doesn't already use them.
6. Post a comment on the ticket via `POST /api/tickets/<id>/comments`:
   ```json
   {
     "author": "lab-skill-updater",
     "body": "## Forslag\n\n<2-4 line rationale: what you changed and why>\n\n```markdown\n<the COMPLETE proposed file content, including frontmatter>\n```"
   }
   ```
7. PATCH the ticket back to `status: "pending"` so it falls out of the dispatcher queue and waits for human review of your proposal.
8. Stop. Do not commit anything.

### Phase 2 — Apply (your proposal comment exists, ticket re-approved by human)

Detect by: there's already a comment from `lab-skill-updater` containing a ` ```markdown ` block. The human has re-approved after reviewing it.

1. Find your most recent comment containing a ` ```markdown ... ``` ` block.
2. Extract the content between the markdown fences. This is the new file content.
3. `cd /home/vegapunk/projects/.dotfiles && git pull --ff-only origin main`
4. If the target file's content has changed since Phase 1 (someone else edited it), abort: comment "File changed upstream since proposal — please re-trigger" and PATCH ticket to `needs_response`. Do not overwrite blindly.
5. Write the extracted content to the target file path (overwrite).
6. `git add <path> && git commit -m "Update <skill-name> via lab ticket #<id>" && git push origin main`
7. Sync to VPS flat install (the dispatcher reads from `~/.claude/skills/<name>/SKILL.md`, not from the dotfiles tree). Run sync-skills:
   ```bash
   bash /home/vegapunk/projects/.dotfiles/.claude/skills/sync-skills/scripts/sync.sh 2>&1 || \
     cp /home/vegapunk/projects/.dotfiles/.claude/skills/<...>/SKILL.md ~/.claude/skills/<skill-name>/SKILL.md
   ```
   (Use whichever exists; if sync-skills script path differs, fall back to direct copy.)
8. PATCH the ticket to `status: "done"` with `executionLog` describing the commit SHA and what was changed.
9. Done.

## Failure modes — always revert ticket cleanly

The dispatcher reverts your ticket to `pending` if you throw or exit non-zero, which is the right behavior for transient failures (rate limit, network). For *permanent* failures (file not found, push rejected, parse errors), do this instead:

1. Post a comment explaining what went wrong (paste the exact error).
2. PATCH ticket to `status: "needs_response"` so the human knows the ball is in their court.
3. Exit cleanly (zero exit code).

| Situation | Action |
|---|---|
| Path in body doesn't exist | comment + needs_response |
| Body has no `Dotfiles repo path:` line | comment + needs_response (this isn't your ticket) |
| `git pull` conflict | comment with conflict + needs_response. Never force. |
| `git push` rejected (non-fast-forward) | pull, retry once. If still failing, comment + needs_response. |
| Upstream file changed between Phase 1 and Phase 2 | comment + needs_response (see Phase 2 step 4) |
| Phase 2 markdown block missing from your prior comment | comment "Phase 1 output corrupted — re-trigger" + needs_response |

## Style guide for proposed changes

- **Preserve structure**: keep heading order and section names the same unless feedback explicitly restructures.
- **Frontmatter**: keep `name` constant. Update `description` if and only if the skill's responsibility changed.
- **Diff size**: smaller is better. If feedback is "fix typo on line 12", change line 12 — don't rewrite the file.
- **Tone**: match the file's existing tone (Danish/English, formal/casual).
- **Code blocks**: preserve language tags and indentation exactly.
- **Deletions**: in your rationale, explicitly call out any sections you removed.

## What you do NOT do

- Don't write code in any project repo. Only `.md` files in dotfiles.
- Don't open PRs — commit straight to `main`.
- Don't approve tickets. Only the human approves.
- Don't decide whether feedback is "good" — if the human asked for a change, propose it. Push back via comment only if it's literally impossible (e.g., file doesn't exist).
- Don't bundle multiple unrelated changes in one commit. One ticket = one commit.
