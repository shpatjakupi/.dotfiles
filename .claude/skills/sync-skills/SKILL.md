---
name: sync-skills
description: Sync skills from GitHub dotfiles repository to local Claude directory. Works on any machine (Windows, Linux, VPS).
context: fork
disable-model-invocation: true
---

# Sync Skills from GitHub Repository

Synchronizes skills from your dotfiles GitHub repository to your local Claude skills directory. Works on any platform.

## Configuration

- **GitHub Repo**: `https://github.com/shpatjakupi/.dotfiles.git`
- **Skills Source**: `<dotfiles-repo>/.claude/skills/`
- **Skills Target**: `~/.claude/skills/`

## Execute
```bash
#!/bin/bash
set -e

# Configuration
GITHUB_REPO="https://github.com/shpatjakupi/.dotfiles.git"
SKILLS_TARGET="$HOME/.claude/skills"

# Auto-detect dotfiles location: check common paths
DOTFILES_PATH=""
CANDIDATES=(
    "$HOME/projects/.dotfiles"
    "$HOME/Desktop/projects/.dotfiles"
    "/c/Users/$USER/Desktop/projects/.dotfiles"
    "$HOME/.dotfiles"
)

for candidate in "${CANDIDATES[@]}"; do
    if [ -d "$candidate/.git" ]; then
        DOTFILES_PATH="$candidate"
        break
    fi
done

# If not found, clone it
if [ -z "$DOTFILES_PATH" ]; then
    DOTFILES_PATH="$HOME/projects/.dotfiles"
    echo "Dotfiles not found locally. Cloning..."
    mkdir -p "$HOME/projects"
    git clone "$GITHUB_REPO" "$DOTFILES_PATH"
fi

SKILLS_SOURCE="$DOTFILES_PATH/.claude/skills"

echo "Syncing skills from GitHub..."
echo ""
echo "Dotfiles: $DOTFILES_PATH"
echo "Target:   $SKILLS_TARGET"
echo ""

# Step 1: Pull latest changes
echo "Pulling latest changes..."
cd "$DOTFILES_PATH"
git pull --ff-only origin main 2>/dev/null || git pull --ff-only 2>/dev/null || echo "Warning: could not pull (may be on different branch)"
echo ""

# Step 2: Verify source exists
if [ ! -d "$SKILLS_SOURCE" ]; then
    echo "No skills directory found in dotfiles repo."
    exit 1
fi

# Step 3: Scan and categorize skills
echo "Scanning skills..."
echo ""
NEW_SKILLS=()
UPDATED_SKILLS=()
UNCHANGED_SKILLS=()

for skill_dir in "$SKILLS_SOURCE"/*/; do
    if [ -d "$skill_dir" ]; then
        skill_name=$(basename "$skill_dir")

        # Skip sync-skills itself to avoid recursion
        if [ "$skill_name" == "sync-skills" ]; then
            continue
        fi

        # Check if SKILL.md exists
        if [ ! -f "$skill_dir/SKILL.md" ]; then
            echo "Skipping $skill_name (no SKILL.md found)"
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
echo "  ${#NEW_SKILLS[@]} new skills"
echo "  ${#UPDATED_SKILLS[@]} updated skills"
echo "  ${#UNCHANGED_SKILLS[@]} unchanged skills"
echo ""

if [ ${#NEW_SKILLS[@]} -gt 0 ]; then
    echo "New skills:"
    for skill in "${NEW_SKILLS[@]}"; do
        echo "  + $skill"
    done
    echo ""
fi

if [ ${#UPDATED_SKILLS[@]} -gt 0 ]; then
    echo "Updated skills:"
    for skill in "${UPDATED_SKILLS[@]}"; do
        echo "  ~ $skill"
    done
    echo ""
fi

# Step 5: Check if any sync needed
TOTAL_TO_SYNC=$((${#NEW_SKILLS[@]} + ${#UPDATED_SKILLS[@]}))
if [ $TOTAL_TO_SYNC -eq 0 ]; then
    echo "All skills are already up to date!"
    echo ""
    echo "Available skills:"
    ls -1 "$SKILLS_TARGET" 2>/dev/null | grep -v "sync-skills" || echo "  (none yet)"
    exit 0
fi

# Step 6: Ask for confirmation
echo "Do you want to sync these skills? (y/n)"
read -r response

if [[ ! "$response" =~ ^[Yy]$ ]]; then
    echo "Sync cancelled"
    exit 0
fi

# Step 7: Create target directory
mkdir -p "$SKILLS_TARGET"

# Step 8: Perform sync
echo ""
echo "Syncing skills..."

SYNCED_COUNT=0

for skill in "${NEW_SKILLS[@]}" "${UPDATED_SKILLS[@]}"; do
    echo "  $skill"
    rm -rf "$SKILLS_TARGET/$skill"
    cp -r "$SKILLS_SOURCE/$skill" "$SKILLS_TARGET/"
    SYNCED_COUNT=$((SYNCED_COUNT + 1))
done

echo ""
echo "Sync complete! Synced $SYNCED_COUNT skills"
echo ""
echo "Available skills:"
ls -1 "$SKILLS_TARGET" | grep -v "sync-skills"
```
