---
name: push-skills
description: >
  Push new or updated skills from ~/.claude/skills/ to the dotfiles repository at
  C:\Users\shpat\Desktop\projects\.dotfiles. Use when the user says "push skill to dotfiles",
  "save skill to dotfiles", "sync skills", or anything about committing/pushing a skill
  to the dotfiles repo. Skills are stored as directories (not .skill packages) in
  .claude/skills/ inside the dotfiles repo. The skills-lock.json is only for GitHub-sourced
  skills — local/custom skills are tracked by directory presence in .claude/skills/.
---

# Push Skills to Dotfiles

## Dotfiles Structure

```
C:\Users\shpat\Desktop\projects\.dotfiles\
├── .claude\
│   └── skills\          <- Claude Code skills go HERE (as directories)
│       └── <skill-name>\
│           ├── SKILL.md
│           └── references\
├── .agents\
│   └── skills\          <- Agent skills (separate, do not mix)
├── skills-lock.json     <- Only for GitHub-sourced skills (has computedHash)
└── README.md
```

**Critical rules:**
- Skills are copied as **directories**, never as `.skill` packages
- Custom/local skills go in `.claude/skills/` — do NOT add them to `skills-lock.json`
- `skills-lock.json` only tracks skills installed from GitHub repos (with `computedHash`)
- Do NOT create any other top-level directories in the dotfiles root

## When Manually Pushing a Skill

1. Verify the skill exists in `~/.claude/skills/<skill-name>/`
2. Copy the directory to `.dotfiles/.claude/skills/`:
   ```bash
   cp -r ~/.claude/skills/<skill-name> /path/to/.dotfiles/.claude/skills/
   ```
3. Stage, commit, and push:
   ```bash
   cd /path/to/.dotfiles
   git add .claude/skills/<skill-name>
   git commit -m "Add <skill-name> skill"
   git push
   ```

## Automated Script (via /push-skills)

Run this script to detect and push all new/updated skills at once:

```bash
#!/bin/bash
set -e

# Meta-skills to skip (not meant to be synced)
SKIP_SKILLS=("initial-setup" "sync-skills" "push-skills" "new-skill")

SKILLS_LOCAL="$HOME/.claude/skills"

echo "Push Skills to Repository"
echo ""

# Step 1: Ask for dotfiles path
echo "Where is your .dotfiles repository located? (full path)"
echo "  Example: /c/Users/shpat/Desktop/projects/.dotfiles"
echo ""
read -r DOTFILES_PATH

# Step 2: Validate path
if [ ! -d "$DOTFILES_PATH" ]; then
    echo "Directory not found: $DOTFILES_PATH"
    exit 1
fi

if [ ! -d "$DOTFILES_PATH/.git" ]; then
    echo "Not a git repository: $DOTFILES_PATH"
    exit 1
fi

SKILLS_REPO="$DOTFILES_PATH/.claude/skills"
mkdir -p "$SKILLS_REPO"

echo "Repository: $DOTFILES_PATH"
echo ""

# Step 3: Check local skills exist
if [ ! -d "$SKILLS_LOCAL" ]; then
    echo "No local skills directory found at $SKILLS_LOCAL"
    exit 1
fi

# Step 4: Scan and compare
echo "Scanning skills..."
echo ""

NEW_SKILLS=()
UPDATED_SKILLS=()
UNCHANGED_SKILLS=()

is_skip_skill() {
    local name="$1"
    for skip in "${SKIP_SKILLS[@]}"; do
        if [ "$name" == "$skip" ]; then
            return 0
        fi
    done
    return 1
}

for skill_dir in "$SKILLS_LOCAL"/*/; do
    if [ -d "$skill_dir" ] && [ -f "$skill_dir/SKILL.md" ]; then
        skill_name=$(basename "$skill_dir")

        if is_skip_skill "$skill_name"; then
            continue
        fi

        repo_skill="$SKILLS_REPO/$skill_name"

        if [ ! -d "$repo_skill" ]; then
            NEW_SKILLS+=("$skill_name")
        elif ! diff -rq "$skill_dir" "$repo_skill" > /dev/null 2>&1; then
            UPDATED_SKILLS+=("$skill_name")
        else
            UNCHANGED_SKILLS+=("$skill_name")
        fi
    fi
done

# Step 5: Report
echo "Results:"
echo "  ${#NEW_SKILLS[@]} new skill(s)"
echo "  ${#UPDATED_SKILLS[@]} updated skill(s)"
echo "  ${#UNCHANGED_SKILLS[@]} unchanged skill(s)"
echo ""

if [ ${#NEW_SKILLS[@]} -gt 0 ]; then
    echo "New skills:"
    for skill in "${NEW_SKILLS[@]}"; do echo "  + $skill"; done
    echo ""
fi

if [ ${#UPDATED_SKILLS[@]} -gt 0 ]; then
    echo "Updated skills:"
    for skill in "${UPDATED_SKILLS[@]}"; do echo "  ~ $skill"; done
    echo ""
fi

TOTAL=$((${#NEW_SKILLS[@]} + ${#UPDATED_SKILLS[@]}))
if [ $TOTAL -eq 0 ]; then
    echo "All skills are up to date!"
    exit 0
fi

echo "Do you want to push these skills to the repository? (y/n)"
read -r response

if [[ ! "$response" =~ ^[Yy]$ ]]; then
    echo "Push cancelled."
    exit 0
fi

echo ""
echo "Copying skills to repository..."

for skill in "${NEW_SKILLS[@]}" "${UPDATED_SKILLS[@]}"; do
    echo "  $skill"
    rm -rf "$SKILLS_REPO/$skill"
    cp -r "$SKILLS_LOCAL/$skill" "$SKILLS_REPO/"
done

echo ""
echo "Preparing git commit..."
cd "$DOTFILES_PATH"
git add .claude/skills/

if [ ${#NEW_SKILLS[@]} -gt 0 ] && [ ${#UPDATED_SKILLS[@]} -gt 0 ]; then
    DEFAULT_MSG="Add $(IFS=', '; echo "${NEW_SKILLS[*]}") and update $(IFS=', '; echo "${UPDATED_SKILLS[*]}") skills"
elif [ ${#NEW_SKILLS[@]} -gt 0 ]; then
    DEFAULT_MSG="Add $(IFS=', '; echo "${NEW_SKILLS[*]}") skill(s)"
else
    DEFAULT_MSG="Update $(IFS=', '; echo "${UPDATED_SKILLS[*]}") skill(s)"
fi

echo "Default commit message: $DEFAULT_MSG"
echo "Enter commit message (or press Enter for default):"
read -r COMMIT_MSG

if [ -z "$COMMIT_MSG" ]; then
    COMMIT_MSG="$DEFAULT_MSG"
fi

git commit -m "$COMMIT_MSG"

echo ""
echo "Pushing to origin/main..."
git push origin main

echo ""
echo "Done! Pushed $TOTAL skill(s) to GitHub."
for skill in "${NEW_SKILLS[@]}"; do echo "  + $skill (new)"; done
for skill in "${UPDATED_SKILLS[@]}"; do echo "  ~ $skill (updated)"; done
```
