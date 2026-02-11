---
name: initial-setup
description: First-time setup - copies all skills from your dotfiles repo to ~/.claude/skills/
context: fork
disable-model-invocation: true
---

# Initial Setup - Claude Skills Installer

Sets up your Claude Code skills by copying them from your dotfiles repository to your local `~/.claude/skills/` directory so they are available globally.

## Execute

```bash
#!/bin/bash
set -e

echo "👋 Welcome to Claude Skills Initial Setup!"
echo ""
echo "This will copy all skills from your .dotfiles repository"
echo "to ~/.claude/skills/ so you can use them from any project."
echo ""

# Step 1: Ask for dotfiles path
echo "Where is your .dotfiles repository located? (full path)"
echo "  Example: /c/Users/jakupi001/Projects/.dotfiles"
echo ""
read -r DOTFILES_PATH

# Step 2: Validate the path exists
if [ ! -d "$DOTFILES_PATH" ]; then
    echo ""
    echo "❌ Directory not found: $DOTFILES_PATH"
    echo "   Please check the path and try again."
    exit 1
fi

echo "✅ Found: $DOTFILES_PATH"

# Step 3: Validate .claude/skills/ exists
SKILLS_SOURCE="$DOTFILES_PATH/.claude/skills"

if [ ! -d "$SKILLS_SOURCE" ]; then
    echo ""
    echo "❌ Skills directory not found: $SKILLS_SOURCE"
    echo "   Make sure your .dotfiles repo has a .claude/skills/ directory."
    exit 1
fi

echo "✅ Found skills directory: $SKILLS_SOURCE"
echo ""

# Step 4: List available skills
echo "📦 Available skills:"
SKILL_COUNT=0
for skill_dir in "$SKILLS_SOURCE"/*/; do
    if [ -d "$skill_dir" ] && [ -f "$skill_dir/SKILL.md" ]; then
        skill_name=$(basename "$skill_dir")
        if [ "$skill_name" != "initial-setup" ]; then
            echo "  - $skill_name"
            SKILL_COUNT=$((SKILL_COUNT + 1))
        fi
    fi
done

if [ $SKILL_COUNT -eq 0 ]; then
    echo "  (no skills found)"
    echo ""
    echo "⚠️  No skills to copy. Add skills to $SKILLS_SOURCE first."
    exit 0
fi

echo ""
echo "Found $SKILL_COUNT skill(s) to install."
echo ""

# Step 5: Ask for confirmation
echo "Do you want to copy all skills to ~/.claude/skills/? (y/n)"
read -r response

if [[ ! "$response" =~ ^[Yy]$ ]]; then
    echo ""
    echo "❌ Setup cancelled."
    exit 0
fi

# Step 6: Create target directory
SKILLS_TARGET="$HOME/.claude/skills"
mkdir -p "$SKILLS_TARGET"

# Step 7: Copy skills (skip initial-setup)
echo ""
echo "🚀 Installing skills..."
COPIED=()

for skill_dir in "$SKILLS_SOURCE"/*/; do
    if [ -d "$skill_dir" ] && [ -f "$skill_dir/SKILL.md" ]; then
        skill_name=$(basename "$skill_dir")

        # Skip initial-setup itself
        if [ "$skill_name" == "initial-setup" ]; then
            continue
        fi

        echo "  📋 Copying $skill_name..."
        rm -rf "$SKILLS_TARGET/$skill_name"
        cp -r "$skill_dir" "$SKILLS_TARGET/"
        COPIED+=("$skill_name")
    fi
done

# Step 8: Summary
echo ""
echo "✅ Setup complete! Installed ${#COPIED[@]} skill(s):"
for skill in "${COPIED[@]}"; do
    echo "  - $skill"
done

echo ""
echo "🎉 You can now use these skills from any project!"
echo "💡 Run /sync-skills anytime to pull updates from your dotfiles repo."
```
