<!-- gen-readme:auto -->
# agents-skills-pure-skill-autofix-spec

A single-purpose [agentskills.io](https://agentskills.io) skill repo for Claude Code. It ships **`skill-autofix-spec`** — an agent-driven tool that refactors a target SKILL directory in place so it conforms to the AGENT SPEC standard (frontmatter, `<tier>-<slug>` directory naming, progressive-disclosure body extraction, the 500-line body limit, and relative-path verification). The skill lives under `.agents/skills/<name>/` and is mirrored into `.claude/skills/<name>` as a relative symlink so the harness auto-discovers it.

## Skills

| Skill | What it does |
|-------|--------------|
| [`skill-autofix-spec`](.agents/skills/skill-autofix-spec/SKILL.md) | Refactor a target SKILL directory to comply with the AGENT SPEC standard. |

## Install

### Per skill — `npx skills add`

Install any single skill into Claude Code:

```bash
npx skills add carlosmarte/agents-skills-pure-skill-autofix-spec \
  --skill skill-autofix-spec -a claude-code
```

### Usage

Once installed, point the skill at a skill directory:

```
use the skill-autofix-spec skill to refactor ./path/to/some-skill
```

It runs a dry-run-first audit (survey → propose → apply → verify) and asks for confirmation before any destructive change (directory rename, file move, or large content extraction).

## Layout

```
.agents/skills/<name>/SKILL.md                          # source of truth for each skill
.claude/skills/<name> -> ../../.agents/skills/<name>    # relative symlink (harness-discovered)
```

