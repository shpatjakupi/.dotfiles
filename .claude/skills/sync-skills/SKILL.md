---
name: sync-skills
description: Sync skills from GitHub dotfiles repository to local Claude directory
context: fork
disable-model-invocation: true
---

# Sync Skills from GitHub Repository

Synchronizes skills from your dotfiles GitHub repository to your local Claude skills directory.

## Configuration

- **GitHub Repo**: `https://github.com/shpatjakupi/.dotfiles.git`
- **Local Clone Path**: `/c/Users/jakupi001/Projects/.dotfiles`
- **Skills Source**: `/c/Users/jakupi001/Projects/.dotfiles/.claude/skills/`
- **Skills Target**: `~/.claude/skills/`

## Execute
```bash
#!/bin/bash
set -e

# Configuration
GITHUB_REPO="https://github.com/shpatjakupi/.dotfiles.git"
LOCAL_REPO="/c/Users/jakupi001/Projects/.dotfiles"
SKILLS_SOURCE="$LOCAL_REPO/.claude/skills"
SKILLS_TARGET="$HOME/.claude/skills"

echo "🔄 Syncing skills from GitHub..."
echo ""

# Step 1: Ensure repository is up to date
if [ ! -d "$LOCAL_REPO" ]; then
    echo "📦 Cloning repository..."
    mkdir -p /c/Users/jakupi001/Projects
    exit 0
# Step 3: Scan and categorize skills
echo "📊 Scanning skills..."
echo ""
NEW_SKILLS=()
UPDATED_SKILLS=()
UNCHANGED_SKILLS=()

for skill_dir in "$SKILLS_SOURCE"/*/ ; do
    if [ -d "$skill_dir" ]; then
        skill_name=$(basename "$skill_dir")
        
        # Skip sync-skills itself to avoid recursion
        if [ "$skill_name" == "sync-skills" ]; then
            continue
        fi
        
        # Check if SKILL.md exists
        if [ ! -f "$skill_dir/SKILL.md" ]; then
            echo "⚠️  Skipping $skill_name (no SKILL.md found)"
            continue
        fi
        
        target_skill="$SKILLS_TARGET/$skill_name"
        
        if [ ! -d "$target_skill" ]; then
            NEW_SKILLS+=("$skill_name")
        elif ! diff -rq "$skill_dir" "$target_skill" > /dev/null 2>&1; then
            UPDATED_SKILLS+=("$skill_name")
        else
            UNCHANGED_SKILLS+=("$skill_name")
        fi
    fi
done

# Step 4: Report findings
echo "Results:"
echo "  ✨ ${#NEW_SKILLS[@]} new skills"
echo "  🔄 ${#UPDATED_SKILLS[@]} updated skills"
echo "  ✅ ${#UNCHANGED_SKILLS[@]} unchanged skills"


fi
    mkdir -p "$SKILLS_SOURCE"
    echo "⚠️  Directory created but is empty. Add skills to your repository first."

echo ""


if [ ${#NEW_SKILLS[@]} -gt 0 ]; then
    echo "📦 New skills found:"

    for skill in "${NEW_SKILLS[@]}"; do

        echo "  - $skill"
    done

    echo ""
fi

if [ ${#UPDATED_SKILLS[@]} -gt 0 ]; then
    echo "📝 Updated skills found:"
    for skill in "${UPDATED_SKILLS[@]}"; do

        echo "  - $skill"
    done
    echo ""
fi

# Step 5: Check if any sync needed
TOTAL_TO_SYNC=$((${#NEW_SKILLS[@]} + ${#UPDATED_SKILLS[@]}))
if [ $TOTAL_TO_SYNC -eq 0 ]; then
    echo "✅ All skills are already up to date!"
    echo ""
    echo "Available skills:"
    ls -1 "$SKILLS_TARGET" 2>/dev/null | grep -v "sync-skills" || echo "  (none yet)"
    exit 0
fi

# Step 6: Ask for confirmation
echo "Do you want to sync these skills? (y/n)"
read -r response

if [[ ! "$response" =~ ^[Yy]$ ]]; then
    echo "❌ Sync cancelled"
    exit 0
fi

# Step 7: Create target directory
mkdir -p "$SKILLS_TARGET"

# Step 8: Perform sync
echo ""
echo "🚀 Syncing skills..."

SYNCED_COUNT=0

for skill in "${NEW_SKILLS[@]}" "${UPDATED_SKILLS[@]}"; do
    echo "  📋 Copying $skill..."
    rm -rf "$SKILLS_TARGET/$skill"
    cp -r "$SKILLS_SOURCE/$skill" "$SKILLS_TARGET/"
    SYNCED_COUNT=$((SYNCED_COUNT + 1))
done

echo ""
echo "✅ Sync complete! Synced $SYNCED_COUNT skills"
echo ""
echo "Available skills:"
ls -1 "$SKILLS_TARGET" | grep -v "sync-skills"
echo ""
echo "💡 Run /skills to see all available skills in Claude Code"
```
