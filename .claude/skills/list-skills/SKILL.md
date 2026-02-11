---
name: list-skills
description: List all installed Claude Code skills with their descriptions
context: fork
disable-model-invocation: true
---

# List Skills

Lists all installed skills from `~/.claude/skills/` with their name, description, and configuration.

## Execute

```bash
#!/bin/bash
set -e

SKILLS_DIR="$HOME/.claude/skills"

echo "📋 Installed Claude Code Skills"
echo ""

if [ ! -d "$SKILLS_DIR" ]; then
    echo "❌ No skills directory found at $SKILLS_DIR"
    echo "   Run /initial-setup to get started."
    exit 0
fi

COUNT=0

for skill_dir in "$SKILLS_DIR"/*/; do
    if [ -d "$skill_dir" ] && [ -f "$skill_dir/SKILL.md" ]; then
        skill_name=$(basename "$skill_dir")
        COUNT=$((COUNT + 1))

        # Parse frontmatter
        desc=""
        context=""
        model_invocation=""

        in_frontmatter=false
        while IFS= read -r line; do
            if [ "$line" == "---" ]; then
                if $in_frontmatter; then
                    break
                else
                    in_frontmatter=true
                    continue
                fi
            fi
            if $in_frontmatter; then
                case "$line" in
                    description:*) desc="${line#description: }" ;;
                    context:*) context="${line#context: }" ;;
                    disable-model-invocation:*) model_invocation="${line#disable-model-invocation: }" ;;
                esac
            fi
        done < "$skill_dir/SKILL.md"

        # Build tags
        tags=""
        if [ "$context" == "fork" ]; then
            tags="fork"
        fi
        if [ "$model_invocation" == "false" ]; then
            [ -n "$tags" ] && tags="$tags, "
            tags="${tags}auto-invoke"
        fi

        echo "  /$skill_name"
        if [ -n "$desc" ]; then
            echo "    $desc"
        fi
        if [ -n "$tags" ]; then
            echo "    [$tags]"
        fi
        echo ""
    fi
done

if [ $COUNT -eq 0 ]; then
    echo "  (no skills installed)"
    echo ""
    echo "Run /initial-setup or /sync-skills to install skills."
else
    echo "─────────────────────────────────"
    echo "  $COUNT skill(s) installed"
    echo ""
    echo "  Manage skills:"
    echo "    /new-skill     Create a new skill"
    echo "    /sync-skills   Pull updates from repo"
    echo "    /push-skills   Push changes to repo"
fi
```
