# Skyeline Skills

Public install surface for Skyeline agent skills.

## Install

### Recommended for all supported agents

```bash
npx skills add kulkarniatharv/skyeline-skills --skill skyeline-sdk
```

### Codex only

```bash
$skill-installer install https://github.com/kulkarniatharv/skyeline-skills/tree/main/skills/skyeline-sdk
```

### Manual fallback

Codex:

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills/skyeline-sdk"
curl -fsSL https://skyeline.dev/agent-skills/skyeline-sdk/SKILL.md -o "${CODEX_HOME:-$HOME/.codex}/skills/skyeline-sdk/SKILL.md"
```

Claude Code:

```bash
mkdir -p "$HOME/.claude/skills/skyeline-sdk"
curl -fsSL https://skyeline.dev/agent-skills/skyeline-sdk/SKILL.md -o "$HOME/.claude/skills/skyeline-sdk/SKILL.md"
```

## Notes

- The manual fallback downloads the hosted skill from https://skyeline.dev/agent-skills/skyeline-sdk/SKILL.md.
- This repo is published from the private Skyeline monorepo.
- skills/skyeline-sdk/SKILL.md is synced from the private Skyeline monorepo and should not be edited here first.

## Source

- Repo: https://github.com/kulkarniatharv/skyeline-skills
- Skill path: https://github.com/kulkarniatharv/skyeline-skills/tree/main/skills/skyeline-sdk
