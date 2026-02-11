---
name: new-skill
description: Scaffold a new Claude Code skill with proper frontmatter and template
context: fork
disable-model-invocation: true
---

# New Skill Generator

Creates a new skill scaffold with proper frontmatter and a bash template, ready for you to fill in.

## Execute

```bash
#!/bin/bash
set -e

echo "🛠️  New Skill Generator"
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

SKILLS_REPO="$DOTFILES_PATH/.claude/skills"
if [ ! -d "$SKILLS_REPO" ]; then
    echo "❌ Skills directory not found: $SKILLS_REPO"
    exit 1
fi

echo "✅ Repository: $DOTFILES_PATH"
echo ""

# Step 3: Ask for skill name
echo "What is the name of your new skill? (use kebab-case, e.g., 'code-review')"
read -r SKILL_NAME

# Step 4: Validate skill name
if [[ ! "$SKILL_NAME" =~ ^[a-z0-9]([a-z0-9-]*[a-z0-9])?$ ]]; then
    echo ""
    echo "❌ Invalid skill name: '$SKILL_NAME'"
    echo "   Must be kebab-case (lowercase letters, numbers, hyphens)."
    echo "   Cannot start or end with a hyphen."
    exit 1
fi

# Step 5: Check if skill already exists
if [ -d "$SKILLS_REPO/$SKILL_NAME" ]; then
    echo ""
    echo "❌ Skill '$SKILL_NAME' already exists at $SKILLS_REPO/$SKILL_NAME"
    exit 1
fi

echo "✅ Name: $SKILL_NAME"
echo ""

# Step 6: Ask for description
echo "Enter a short description of what this skill does:"
read -r SKILL_DESC

if [ -z "$SKILL_DESC" ]; then
    echo "❌ Description cannot be empty."
    exit 1
fi

echo ""

# Step 7: Ask about model invocation
echo "Should Claude invoke this skill automatically? (y/n)"
read -r AUTO_INVOKE

if [[ "$AUTO_INVOKE" =~ ^[Yy]$ ]]; then
    DISABLE_MODEL="false"
else
    DISABLE_MODEL="true"
fi

# Step 8: Ask about fork/context
echo "Should this skill run in a fork/subagent? (y/n)"
read -r USE_FORK

echo ""

# Step 9: Create skill directory
SKILL_DIR="$SKILLS_REPO/$SKILL_NAME"
mkdir -p "$SKILL_DIR"

# Step 10: Generate SKILL.md
SKILL_FILE="$SKILL_DIR/SKILL.md"

{
    echo "---"
    echo "name: $SKILL_NAME"
    echo "description: $SKILL_DESC"
    if [[ "$USE_FORK" =~ ^[Yy]$ ]]; then
        echo "context: fork"
    fi
    echo "disable-model-invocation: $DISABLE_MODEL"
    echo "---"
    echo ""
    echo "# ${SKILL_NAME}"
    echo ""
    echo "$SKILL_DESC"
    echo ""
    echo "## Instructions"
    echo ""
    echo "<!-- TODO: Write instructions for what this skill should do -->"
    echo "<!-- Claude will follow these instructions when the skill is invoked -->"
    echo ""
    echo "## Execute"
    echo ""
    echo '```bash'
    echo "#!/bin/bash"
    echo "set -e"
    echo ""
    echo "echo \"Running $SKILL_NAME...\""
    echo ""
    echo "# TODO: Add your skill logic here"
    echo ""
    echo "echo \"✅ Done!\""
    echo '```'
} > "$SKILL_FILE"

echo "✅ Created: $SKILL_FILE"

# Step 11: Copy to local skills
SKILLS_LOCAL="$HOME/.claude/skills"
mkdir -p "$SKILLS_LOCAL/$SKILL_NAME"
cp -r "$SKILL_DIR/"* "$SKILLS_LOCAL/$SKILL_NAME/"

echo "✅ Copied to: $SKILLS_LOCAL/$SKILL_NAME/"
echo ""

# Step 12: Next steps
echo "🎉 Skill '$SKILL_NAME' created successfully!"
echo ""
echo "Next steps:"
echo "  1. Edit:  nano $SKILL_FILE"
echo "  2. Test:  /$SKILL_NAME"
echo "  3. Push:  /push-skills"
```
