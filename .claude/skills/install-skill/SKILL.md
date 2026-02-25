---
name: install-skill
description: >
  Install a skill from GitHub into ~/.claude/skills/. Use when the user wants to
  add a skill from the internet, a GitHub URL, or a GitHub repo reference. Accepts
  formats like "owner/repo/path/to/skill", "owner/repo" (whole repo is the skill),
  or full GitHub URLs. After installing, always remind the user to run /push-skills
  to sync the new skill to their dotfiles at C:\Users\shpat\Desktop\projects\.dotfiles.
  Trigger phrases: "install skill from GitHub", "add skill from URL", "get skill from
  internet", "download skill", "install from repo".
---

# Install Skill from GitHub

Downloads a skill from a GitHub repository into `~/.claude/skills/` using the
bash script below. After successful install, always remind the user to run
`/push-skills` to sync it to dotfiles.

## Supported Input Formats

| Input | Meaning |
|---|---|
| `owner/repo/path/to/skill` | Skill lives in a subdirectory of a repo |
| `owner/repo` | The entire repo is the skill |
| `https://github.com/owner/repo/tree/main/skill-name` | Full GitHub URL |

## Execute

```bash
#!/bin/bash
set -e

SKILLS_DIR="$HOME/.claude/skills"

echo "Install Skill from GitHub"
echo ""
echo "Enter GitHub source (examples):"
echo "  anthropics/skills/skill-creator"
echo "  vercel-labs/skills/find-skills"
echo "  https://github.com/owner/repo/tree/main/skill-name"
echo "  owner/repo  (if the whole repo is one skill)"
echo ""
read -r INPUT

# Normalize: strip full GitHub URL prefix and /tree/<branch>/
INPUT="${INPUT#https://github.com/}"
INPUT="${INPUT%/}"
INPUT=$(echo "$INPUT" | sed 's|/tree/[^/]*/|/|')

# Split into parts
IFS='/' read -ra PARTS <<< "$INPUT"

if [ ${#PARTS[@]} -lt 2 ]; then
    echo "Invalid input. Minimum format: owner/repo"
    exit 1
fi

OWNER="${PARTS[0]}"
REPO="${PARTS[1]}"
REPO_URL="https://github.com/$OWNER/$REPO.git"

if [ ${#PARTS[@]} -ge 3 ]; then
    SUBPATH=$(IFS='/'; echo "${PARTS[*]:2}")
    SKILL_NAME="${PARTS[${#PARTS[@]}-1]}"
else
    SUBPATH=""
    SKILL_NAME="$REPO"
fi

echo ""
echo "Repository : $REPO_URL"
[ -n "$SUBPATH" ] && echo "Subpath    : $SUBPATH"
echo "Skill name : $SKILL_NAME"
echo ""

# Confirm if skill already exists
if [ -d "$SKILLS_DIR/$SKILL_NAME" ]; then
    echo "Skill '$SKILL_NAME' already exists. Overwrite? (y/n)"
    read -r response
    [[ ! "$response" =~ ^[Yy]$ ]] && echo "Cancelled." && exit 0
fi

# Clone to temp dir
TMP_DIR=$(mktemp -d)
echo "Cloning $REPO_URL ..."
git clone --depth=1 "$REPO_URL" "$TMP_DIR" 2>&1 | tail -1

# Copy skill files
if [ -n "$SUBPATH" ]; then
    if [ ! -d "$TMP_DIR/$SUBPATH" ]; then
        echo "Path '$SUBPATH' not found in repository."
        rm -rf "$TMP_DIR"
        exit 1
    fi
    rm -rf "$SKILLS_DIR/$SKILL_NAME"
    cp -r "$TMP_DIR/$SUBPATH" "$SKILLS_DIR/$SKILL_NAME"
else
    rm -rf "$SKILLS_DIR/$SKILL_NAME"
    cp -r "$TMP_DIR/." "$SKILLS_DIR/$SKILL_NAME"
fi

rm -rf "$TMP_DIR"

# Verify
if [ ! -f "$SKILLS_DIR/$SKILL_NAME/SKILL.md" ]; then
    echo "Warning: No SKILL.md found — double-check the path."
fi

echo ""
echo "Installed '$SKILL_NAME' to $SKILLS_DIR/$SKILL_NAME"
echo ""
echo "Next step: run /push-skills to sync to your dotfiles."
```
