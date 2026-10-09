# openedx-claude-skills

Claude Code skills for Open edX development.

## Skills

### `modernize-python-tooling`

Migrate an Open edX Python package or platform repo from legacy tooling (pip-tools, setup.cfg/setup.py) to modern tooling (uv, pyproject.toml PEP 621/735, python-semantic-release, src layout).

**Usage:**
```
/modernize-python-tooling [repo-path]
```

The skill works through four phases in order, assessing the current state of the repo before making any changes. Items already implemented are skipped — only missing pieces are added. Phase 3 (semantic release) only applies to library repos published to PyPI; Phase 4 (src layout) is optional for application repos.

---

#### Phase 1 — Consolidate metadata into `pyproject.toml`

Moves all package metadata out of `setup.cfg` / `setup.py` into a single `pyproject.toml`.

| What changes | Details |
|---|---|
| `pyproject.toml` | `[build-system]` uses `requires = ["setuptools", "setuptools-scm>8.1"]`; `[project]` has static `dependencies` list; `dynamic = ["version"]` only |
| `[tool.setuptools_scm]` | `version_scheme = "only-version"`, `local_scheme = "no-local-version"` — prevents dirty suffixes being rejected by PyPI. `fallback_version` only for repos with no git tags yet |
| `[tool.uv]` | `package = true` added |
| `license` | Converted to SPDX string format (PEP 639) — checked against the repo's *original* `License :: OSI Approved :: ...` classifier, not assumed: no "or later" in the classifier → `license = "AGPL-3.0-only"`; "or later" → `"AGPL-3.0-or-later"`. Never the deprecated bare `"AGPL-3.0"` form. `license-files = ["LICENSE", "NOTICE"]` (include `NOTICE` if the repo has one) |
| `[tool.setuptools.packages.find]` | `exclude = ["*.tests", "*.tests.*"]` so test subpackages don't ship in the wheel (a bare `"*tests"` only matches names *ending* in `tests` — nested fixture packages still leak through) |
| `[tool.coverage.run/report/html]` | Coverage config moves here from `.coveragerc`, which is deleted |
| `__version__` in `__init__.py` | Removed entirely if nothing imports it; if genuinely needed at runtime, read via `importlib.metadata.version(...)` with no `try/except` fallback (a silent fallback masks an uninstalled package) |
| `setup.py` / `setup.cfg` / `.coveragerc` | Deleted |

---

#### Phase 2 — Switch dependency management to uv

Replaces `pip-compile` / `requirements/*.txt` with `uv` and a lockfile.

| What changes | Details |
|---|---|
| `pyproject.toml` — `[dependency-groups]` | Adds standard groups: `test-base`, `test` (current Django), legacy version group (e.g. `django42`), `quality`, `doc`, `ci` (tox + tox-uv only), `dev` |
| `pyproject.toml` — `[tool.uv]` | `conflicts` allows two Django version groups to coexist in a single `uv.lock`; `default-groups = []` so `uv sync --group ci` doesn't implicitly pull in the full `dev` superset |
| `pyproject.toml` — `[tool.edx_lint]` | `uv_constraints = []` added — the *only* place for repo-specific constraint overrides; never edit `[tool.uv].constraint-dependencies` directly (machine-managed) |
| `uv.lock` | Generated via `uv lock` |
| `requirements/` | Deleted |
| `tox.ini` | `requires = tox-uv>=1`; runner = `uv-venv-lock-runner`; uses `dependency_groups =` instead of `deps =` |
| `Makefile` | `upgrade` → `uv run --with edx-lint edx_lint write_uv_constraints ...` then `uv lock --upgrade`; `requirements` → `uv sync --group dev`; `quality`/`test`/`docs`/`pii_check` targets **delegate to `tox -e <env>`** rather than inlining `uv run --group ...` commands; `clean` uses a bare `coverage erase`. The Makefile must never add a `uv run` prefix anywhere except `uv run --with <pkg>` in `upgrade` — `uv run` inside a target that tox itself invokes re-syncs default groups and corrupts the Django version matrix ([openedx/public-engineering#506](https://github.com/openedx/public-engineering/issues/506)) |
| CI workflows | `astral-sh/setup-uv` step added (SHA-pinned, no separate `setup-python`); `pip install -r requirements/ci.txt` → `uv sync --group ci --locked`; bare `tox` → `uv run --locked tox -e ${{ matrix.toxenv }}` (the `--locked` flags make a drifted `uv.lock` fail CI instead of silently re-resolving); job gets `permissions: contents: read`; `workflow_call:` trigger added so `release.yml` can reuse it |

> If the repo already uses `[project.optional-dependencies]` instead of `[dependency-groups]`, `tox.ini` can use `extras =` instead of `dependency_groups =`. The Django version matrix requires `[dependency-groups]` with `[tool.uv].conflicts` to work correctly.

---

#### Phase 3 — Add semantic-release

Automates versioning, changelog, and PyPI publishing via GitHub Actions.

| What changes | Details |
|---|---|
| `pyproject.toml` — `[tool.semantic_release]` | `build_command` uses `python -m build` (not `uv build` — PSR's action environment has no uv) and exports `SETUPTOOLS_SCM_PRETEND_VERSION=$NEW_VERSION` before building so the version is set correctly at build time. `allow_zero_version`/`major_on_zero` only added for repos still on 0.x |
| `.github/workflows/release.yml` | Rewritten to match [`openedx/sample-plugin`](https://github.com/openedx/sample-plugin/blob/main/.github/workflows/release.yml)'s reference exactly: a `run_ci` job calls `ci.yml` via `workflow_call` (no `secrets: inherit`, no `permissions` block on it); `release` job checks out, force-resets to the triggering sha, runs PSR with `vcs_release: "false"`, then `gh release create` attaches the built `dist/` as release assets (required for repos with immutable releases, since assets can't be attached after publish); `publish_to_pypi` job uses OIDC (`contents: read` + `id-token: write`, no token secret) |
| `.github/workflows/ci.yml` | `push: [main]` (or `[master]`) is **kept**, not removed, even after adding `workflow_call:` — it's what runs CI on direct pushes (dependabot merges, hotfixes) independently of the release flow, and doesn't conflict with `workflow_call` |
| `.github/workflows/commitlint.yml` | Enforces conventional commit format on PRs |
| `.readthedocs.yaml` | If the repo has Sphinx docs: installs via `method: uv` / `command: sync` / `groups: [doc]` instead of pip, using `uv.lock` |

> PyPI trusted publisher (OIDC) must be configured manually in PyPI project settings before the publish job will work. If the repo already uses a `PYPI_UPLOAD_TOKEN` secret and publishing is working, leave the token-based auth in place.

---

#### Phase 4 — Move to `src/` layout

Optional; skipped if `src/<package>/` already exists. Moves the package into `src/` to prevent accidentally importing an uninstalled copy of it.

| What changes | Details |
|---|---|
| Package directory | `git mv <package>/ src/<package>/` |
| `pyproject.toml` — `[tool.setuptools.packages.find]` | `where = ["src"]` added |
| `pyproject.toml` — `[tool.coverage.run]` | `source = [...]` changed to **`source_pkgs = [...]`**. This isn't cosmetic: Django's default (label-less) test discovery can't recurse past `src/` (`unittest`'s loader refuses to recurse into any directory without an `__init__.py`, and `src/` deliberately has none). `source_pkgs` locates the package via its import `__path__` instead of a filesystem path, independent of how the test runner was invoked. Getting this wrong — or invoking the test runner without an explicit package label — can silently drop real test coverage to near zero while CI stays green |
| `tox.ini` / `Makefile` | Filesystem paths updated to `src/<package>/...` |
| `MANIFEST.in` | Paths updated to `src/<package>/...` |
| `uv.lock` | Regenerated |

---

#### Handling reviewer feedback

Applies throughout all phases, not just at the end.

- No speculative "for consistency" changes during a batch fix — if a sibling file differs from the others, check its review threads first; the difference may be a decision a reviewer already made.
- Re-diff every file with a resolved review thread against what that thread concluded before requesting re-review — a later edit can silently undo an earlier fix.

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
