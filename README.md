# openedx-claude-skills

Claude Code skills for Open edX development.

## Skills

### `modernize-python-tooling`

Migrate an Open edX Python package from legacy tooling (pip-tools, setup.cfg/setup.py) to modern tooling (uv, pyproject.toml PEP 621/735, python-semantic-release).

**Usage:**
```
/modernize-python-tooling [repo-path]
```

## Installation

```bash
/plugin marketplace add https://github.com/salman2013/openedx-claude-skills
```

Or load locally:
```bash
claude --plugin-dir ./openedx-claude-skills
```
