---
name: push-skills
description: Push new or updated skills from ~/.claude/skills/ to your dotfiles repository
context: fork
disable-model-invocation: true
---

# Push Skills to Repository

Detects new or updated skills in `~/.claude/skills/` and pushes them to your dotfiles repository on GitHub.

## Execute

```bash
#!/bin/bash
set -e

# Meta-skills to skip (not meant to be synced)
SKIP_SKILLS=("initial-setup" "sync-skills" "push-skills" "new-skill")

SKILLS_LOCAL="$HOME/.claude/skills"

echo "🚀 Push Skills to Repository"
echo ""

# Step 1: Ask for dotfiles path
echo "Where is your .dotfiles repository located? (full path)"
echo "  Example: /c/Users/jakupi001/Projects/.dotfiles"
echo ""
read -r DOTFILES_PATH

# Step 2: Validate path
if [ ! -d "$DOTFILES_PATH" ]; then
    echo ""
    echo "❌ Directory not found: $DOTFILES_PATH"
    exit 1
fi

if [ ! -d "$DOTFILES_PATH/.git" ]; then
    echo ""
    echo "❌ Not a git repository: $DOTFILES_PATH"
    exit 1
fi

SKILLS_REPO="$DOTFILES_PATH/.claude/skills"
mkdir -p "$SKILLS_REPO"

echo "✅ Repository: $DOTFILES_PATH"
echo ""

# Step 3: Check local skills exist
if [ ! -d "$SKILLS_LOCAL" ]; then
    echo "❌ No local skills directory found at $SKILLS_LOCAL"
    exit 1
fi

# Step 4: Scan and compare
echo "📊 Scanning skills..."
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

        # Skip meta-skills
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
echo "  ✨ ${#NEW_SKILLS[@]} new skill(s)"
echo "  🔄 ${#UPDATED_SKILLS[@]} updated skill(s)"
echo "  ✅ ${#UNCHANGED_SKILLS[@]} unchanged skill(s)"
echo ""

# Step 6: Show details
if [ ${#NEW_SKILLS[@]} -gt 0 ]; then
    echo "📦 New skills:"
    for skill in "${NEW_SKILLS[@]}"; do
        echo "  + $skill"
    done
    echo ""
fi

if [ ${#UPDATED_SKILLS[@]} -gt 0 ]; then
    echo "📝 Updated skills:"
    for skill in "${UPDATED_SKILLS[@]}"; do
        echo "  ~ $skill"
    done
    echo ""
fi

# Step 7: Check if anything to push
TOTAL=$((${#NEW_SKILLS[@]} + ${#UPDATED_SKILLS[@]}))
if [ $TOTAL -eq 0 ]; then
    echo "✅ All skills are up to date!"
    exit 0
fi

# Step 8: Ask for confirmation
echo "Do you want to push these skills to the repository? (y/n)"
read -r response

if [[ ! "$response" =~ ^[Yy]$ ]]; then
    echo ""
    echo "❌ Push cancelled."
    exit 0
fi

# Step 9: Copy skills to repo
echo ""
echo "📋 Copying skills to repository..."

for skill in "${NEW_SKILLS[@]}" "${UPDATED_SKILLS[@]}"; do
    echo "  📋 $skill"
    rm -rf "$SKILLS_REPO/$skill"
    cp -r "$SKILLS_LOCAL/$skill" "$SKILLS_REPO/"
done

# Step 10: Git operations
echo ""
echo "📝 Preparing git commit..."
cd "$DOTFILES_PATH"
git add .claude/skills/

# Build default commit message
if [ ${#NEW_SKILLS[@]} -gt 0 ] && [ ${#UPDATED_SKILLS[@]} -gt 0 ]; then
    DEFAULT_MSG="Add $(IFS=', '; echo "${NEW_SKILLS[*]}") and update $(IFS=', '; echo "${UPDATED_SKILLS[*]}") skills"
elif [ ${#NEW_SKILLS[@]} -gt 0 ]; then
    DEFAULT_MSG="Add $(IFS=', '; echo "${NEW_SKILLS[*]}") skill(s)"
else
    DEFAULT_MSG="Update $(IFS=', '; echo "${UPDATED_SKILLS[*]}") skill(s)"
fi

echo ""
echo "Default commit message: $DEFAULT_MSG"
echo ""
echo "Enter commit message (or press Enter for default):"
read -r COMMIT_MSG

if [ -z "$COMMIT_MSG" ]; then
    COMMIT_MSG="$DEFAULT_MSG"
fi

git commit -m "$COMMIT_MSG"

echo ""
echo "🚀 Pushing to origin/main..."
git push origin main

echo ""
echo "✅ Done! Pushed $TOTAL skill(s) to GitHub."
echo ""
echo "Skills pushed:"
for skill in "${NEW_SKILLS[@]}"; do
    echo "  ✨ $skill (new)"
done
for skill in "${UPDATED_SKILLS[@]}"; do
    echo "  🔄 $skill (updated)"
done
```
