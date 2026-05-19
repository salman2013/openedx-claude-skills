# openedx-claude-skills

Claude Code skills for Open edX development.

## Skills

### `modernize-python-tooling`

Migrate an Open edX Python package from legacy tooling (pip-tools, setup.cfg/setup.py) to modern tooling (uv, pyproject.toml PEP 621/735, python-semantic-release).

**Usage:**
```
/modernize-python-tooling [repo-path]
```

The skill works through three phases in order, assessing the current state of the repo before making any changes. Items already implemented are skipped — only missing pieces are added.

---

#### Phase 1 — Consolidate metadata into `pyproject.toml`

Moves all package metadata out of `setup.cfg` / `setup.py` into a single `pyproject.toml`.

| What changes | Details |
|---|---|
| `pyproject.toml` | `[build-system]` uses `setuptools>=64` + `setuptools-scm`; `[project]` has static `dependencies` list; `dynamic = ["version"]` only |
| `[tool.setuptools_scm]` | `version_scheme = "only-version"`, `local_scheme = "no-local-version"` — prevents dirty suffixes being rejected by PyPI |
| `[tool.uv]` | `package = true` added |
| `license` | Converted to SPDX string format: `license = "AGPL-3.0"` + `license-files = ["LICENSE.txt"]` (PEP 639) |
| `setup.py` | Deleted |
| `setup.cfg` | Deleted |

---

#### Phase 2 — Switch dependency management to uv

Replaces `pip-compile` / `requirements/*.txt` with `uv` and a lockfile.

| What changes | Details |
|---|---|
| `pyproject.toml` — `[dependency-groups]` | Adds standard groups: `test-base`, `test` (current Django), legacy version group (e.g. `django42`), `quality`, `doc`, `ci` (tox + tox-uv only), `dev` |
| `pyproject.toml` — `[tool.uv].conflicts` | Allows two Django version groups to coexist in a single `uv.lock` |
| `pyproject.toml` — `[tool.edx_lint]` | `uv_constraints = []` added for repo-specific version overrides |
| `uv.lock` | Generated via `uv lock` |
| `requirements/` | Deleted |
| `tox.ini` | `requires = tox-uv>=1`; runner = `uv-venv-lock-runner`; uses `dependency_groups =` instead of `deps =` |
| `Makefile` | `upgrade` → `uv lock --upgrade`; `requirements` → `uv sync --group dev` |
| CI workflows | `astral-sh/setup-uv` step added; `pip install -r requirements/ci.txt` → `uv sync --group ci`; bare `tox` → `uv run tox` |

> If the repo already uses `[project.optional-dependencies]` instead of `[dependency-groups]`, `tox.ini` can use `extras =` instead of `dependency_groups =`. The Django version matrix requires `[dependency-groups]` with `[tool.uv].conflicts` to work correctly.

---

#### Phase 3 — Add semantic-release

Automates versioning, changelog, and PyPI publishing via GitHub Actions.

| What changes | Details |
|---|---|
| `pyproject.toml` — `[tool.semantic_release]` | `build_command` exports `SETUPTOOLS_SCM_PRETEND_VERSION=$NEW_VERSION` before building so the version is set correctly at build time |
| `.github/workflows/release.yml` | Runs CI via `workflow_call`, then PSR on `main`; uploads `dist/` artifact; `publish_to_pypi` job uses OIDC (`id-token: write`) |
| `.github/workflows/ci.yml` | `push: [main]` trigger removed — `release.yml` owns that path to avoid running tests twice on main |
| `.github/workflows/commitlint.yml` | Enforces conventional commit format on PRs |

> PyPI trusted publisher (OIDC) must be configured manually in PyPI project settings before the publish job will work. If the repo already uses a `PYPI_UPLOAD_TOKEN` secret and publishing is working, leave the token-based auth in place.

---

## Installation

Copy the skills into your Claude Code skills directory:

```bash
cp -r skills/* ~/.claude/skills/
```

Or clone and symlink:

```bash
git clone git@github.com:salman2013/openedx-claude-skills.git
ln -s $(pwd)/openedx-claude-skills/skills/modernize-python-tooling ~/.claude/skills/modernize-python-tooling
```
