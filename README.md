# Skill UV Enhancer

Convert an existing skill's Python scripts to use [uv](https://docs.astral.sh/uv/) for dependency management — no virtual environments, no bootstrap scripts, no global installs.

## What It Does

- Audits a skill's Python scripts for third-party imports
- Adds [PEP 723](https://peps.python.org/pep-0723/) inline metadata with dependency declarations
- Converts `python` / `python3` invocations to `uv run` in `SKILL.md`
- Adds `uv` to the skill's prerequisites
- Handles edge cases: stdlib-only scripts, existing metadata, shared requirements

## Prerequisites

- [uv](https://docs.astral.sh/uv/) — Python package runner
  ```bash
  curl -LsSf https://astral.sh/uv/install.sh | sh
  ```

## Installation

```bash
# Cursor
cp -r . ~/.cursor/skills/skill-uv-enhancer/

# Claude Code
cp -r . ~/.claude/skills/skill-uv-enhancer/

# Codex
cp -r . ~/.codex/skills/skill-uv-enhancer/
```

## Usage

Point the agent at a skill directory and say something like:

- "Enhance this skill with uv"
- "Convert this skill to use uv"
- "Add uv dependency management to this skill"
- "Fix skill dependencies"

The agent will audit the scripts, show you what it found, and wait for confirmation before making changes.

## License

MIT
