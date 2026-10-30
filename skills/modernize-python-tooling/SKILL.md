---
name: modernize-python-tooling
description: Migrate an openedx Python package or platform repo from legacy tooling (pip-tools, setup.cfg/setup.py) to modern tooling (uv, pyproject.toml PEP 621/735, python-semantic-release, src layout).
---

Migrate an openedx Python package or platform repo from legacy tooling (pip-tools, setup.cfg/setup.py) to modern tooling (uv, pyproject.toml PEP 621/735, python-semantic-release, src layout).

## Usage
```
/modernize-python-tooling [repo-path]
```

## Context

Reference implementation: `openedx/sample-plugin` — `backend/pyproject.toml`, `backend/tox.ini`, `backend/Makefile`, `.github/workflows/backend-ci.yml`.

**Repo types:**
- **Library** (PyPI package): has `setup.py`/`setup.cfg` with full `install_requires`. All four phases apply.
- **Application** (not on PyPI): minimal `[project].dependencies`. Skip Phase 3. Phase 4 optional.

**Auto-detect**: no `setup.py`/`setup.cfg` and `[project].dependencies` ≤ 3 packages → application repo.

## Behavior

Create a feature branch first. Never commit to main/master.

```bash
git checkout master && git pull && git checkout -b modernize-python-tooling
```

Read all existing files before making changes. Assess each phase checklist item as already-done or missing — only implement what is missing.

After each phase, show the user a file-by-file summary and ask for commit approval before committing.

---

## Phase 1 — Metadata → `pyproject.toml`

Read `setup.cfg`/`setup.py`, then create/update `pyproject.toml`:

```toml
[build-system]
requires = ["setuptools", "setuptools-scm>8.1"]
build-backend = "setuptools.build_meta"

[project]
name = "package-name"
dynamic = ["version"]
description = "..."
readme = "README.rst"
license = "AGPL-3.0-only"      # SPDX string (PEP 639) — only "-or-later" if the
                               # original classifier said so; see Key rules below
license-files = ["LICENSE", "NOTICE"]  # include NOTICE too if the repo has one
requires-python = ">=3.11"
dependencies = ["django>=4.2", ...]   # static list, lower bounds required

[tool.setuptools_scm]
version_scheme = "only-version"
local_scheme = "no-local-version"
fallback_version = "0.0.0"   # what shallow CI checkouts and Docker mounts without .git build as
# If the repo carries org-wide named-release tags (release/verawood.1, ...), scope
# git describe to the repo's own tags, or setuptools_scm fails to parse them:
# git_describe_command = ["git", "describe", "--dirty", "--tags", "--long", "--match", "v[0-9]*"]

[tool.setuptools.packages.find]
include = ["<package>*"]
exclude = ["*tests*"]   # exclude-package-data (below) strips data files but NOT .py
                        # test subpackages. Not "*tests" alone: fnmatch only matches
                        # names *ending* in "tests", so "<package>.tests.fixtures"
                        # would still ship. Diff the wheel against the last release.
# where = ["src"] is added in Phase 4, once the package moves under src/

[tool.setuptools.package-data]
# Compare against old setup.py package_data to avoid missing files in the wheel.
# SYMLINK WARNING: check each dir with `git ls-tree HEAD -- <pkg>/<dir>`
#   mode 120000 = symlink — do NOT add it; setuptools fails with
#   "can't copy '<dir>': doesn't exist or not a regular file"
#   Include the real target path instead (e.g. conf/locale/** if translations/ -> conf/locale)
# NOTE: translations/config.yaml is an Atlas config — Atlas reads the source tree,
#   not the wheel, so it does NOT need to be packaged.
"<package>" = [
    "static/**",
    "templates/**",
    "public/**",
    "conf/locale/**",
]

[tool.setuptools.exclude-package-data]   # kebab-case — underscore causes schema error
"*" = ["tests*", "*.tests*"]

[tool.coverage.run]
branch = true
source = ["<package>"]    # flat layout; use source_pkgs = ["<package>"] instead
                          # once Phase 4 moves the package under src/ — coverage
                          # locates it via import machinery (__path__) rather
                          # than a filesystem path, which src/ layouts need
omit = ["*/tests/*", "*/migrations/*", "*/__pycache__/*", "*/settings.py"]

[tool.coverage.report]
show_missing = true
exclude_lines = ["pragma: no cover", "def __repr__", "raise AssertionError",
    "raise NotImplementedError", "if __name__ == .__main__.:", "if TYPE_CHECKING:"]
# Only add fail_under if the original .coveragerc had one — never invent a threshold
# Carry over [html]/[xml] output paths only if .coveragerc had them: the Makefile's
# coverage and diff_cover targets read htmlcov/ and coverage.xml at the root.
```

**Key rules:**
- `license` SPDX string → remove the `License :: OSI Approved :: ...` classifier (newer setuptools rejects having both)
- Check the *original* classifier text before picking the SPDX string: `License :: OSI Approved :: GNU Affero General Public License v3` (no "or later") → `"AGPL-3.0-only"`; if it says "or later" → `"AGPL-3.0-or-later"`. Never use the bare deprecated `"AGPL-3.0"` form.
- After writing the `license` line, re-read `pyproject.toml` before committing — some repos have a formatter/linter hook that silently reverts it on save (e.g. back to the bare `"AGPL-3.0"` form or a previous value). Don't assume the edit stuck just because the tool reported success.
- `dependencies` must be a static list — never `dynamic = ["dependencies"]`
- Framework deps need a lower bound: `"django>=4.2"` (oldest tested version)
- No upper bounds in `dependencies` (no `Django<6.0`): they become `Requires-Dist` on PyPI and block every downstream consumer from upgrading. Caps go in `[tool.uv].constraint-dependencies` via `edx_lint write_uv_constraints`. Compare against the last release's `Requires-Dist` (`https://pypi.org/pypi/<dist>/json`) and match it. (feanil on openedx-filters#381, openedx-core#704, auth-backends#444)
- No direct-URL dependencies (`"pkg @ https://..."`) in `dependencies` or an extra: they land in `Requires-Dist` and PyPI rejects the upload, after PSR has already tagged and released. Use a plain `"pkg"` plus `[tool.uv.sources] pkg = { url = "..." }`. Grep the built METADATA for ` @ `.
- `include-package-data`: pyproject's default is `true`, and with setuptools-scm that ships every git-tracked file under the package: READMEs, COPYING, `conf/locale/config.yaml`, test data (`exclude-package-data` didn't catch nested test dirs on openedx-events#590). If the old `setup.py` never set `include_package_data=True`, set `[tool.setuptools] include-package-data = false` and list real data files in `package-data`. Either way, diff the built wheel's file list against the last PyPI wheel before committing.
- `authors = [{name = "Open edX Project", email = "oscm@openedx.org"}]` when the old author was edX/Open edX.
- Framework classifiers should match the tested versions (e.g. `Framework :: Django :: 4.2`, `Framework :: Django :: 5.2`)
- No `root` in `[tool.setuptools_scm]` (unlike sample-plugin which is a subdirectory)
- `[tool.uv]` must include `package = true`
- If the repo already uses ruff, carry over `[tool.ruff]` (`line-length = 120`, `target-version`), `[tool.ruff.lint]` (expanded `select`, `ignore = ["E501"]`), `[tool.ruff.lint.isort]` (`known-third-party`), and `[tool.ruff.format]` (`quote-style = "double"`, `indent-style = "space"`). Skip entirely if the repo doesn't use ruff — don't introduce it.

**Tags and changelog:**
- openedx uses `v{version}` tags, which is PSR's default. Still check the live tags: a `v` tag must exist for the latest PyPI version (`gh api repos/openedx/<repo>/tags`). If the repo only has bare tags (`1.2.3`), a maintainer has to cut a `v`-prefixed bridging release before merge, or PSR won't find the previous release. Setting it explicitly is fine; the merged repos do, with feanil's wording:
  ```toml
  [tool.semantic_release]
  # Explicitly set the tag format (same as default).
  tag_format = "v{version}"
  ```
- `CHANGELOG.rst` stops being updated: release notes go on the GitHub release (`changelog: "false"` in `release.yml`). Drop it from `readme` (README only) and add a note at the top, as openedx-ledger does: `.. DEPRECATED: ... see GitHub Releases`. Don't add a `.. changelog-insertion-marker` or `[tool.semantic_release.changelog]`, never delete old entries, and keep the file shipping in the sdist (feanil on DoneXBlock#388).

The `build_command`, `allow_zero_version`, and `major_on_zero` keys under `[tool.semantic_release]` are added in Phase 3.

**Handle `__version__` in `__init__.py`:**
Keep it (don't remove what existed — downstream code and docs may read it), reading the version from `importlib.metadata` without a `try/except` fallback — silent fallbacks mask uninstalled package errors:
```python
from importlib.metadata import version
__version__ = version("distribution-name")  # use the [project].name, NOT __package__
```
Using `version(__package__)` raises `PackageNotFoundError` — the distribution name often differs from the package directory name.

A common existing pattern to replace:
```python
# BEFORE (wrong — silent fallback hides uninstalled package)
from importlib.metadata import PackageNotFoundError, version
try:
    __version__ = version("pkg-name")
except PackageNotFoundError:  # pragma: no cover
    __version__ = "unknown"

# AFTER (correct — also drop PackageNotFoundError from the import)
from importlib.metadata import version
__version__ = version("pkg-name")
```

Update `docs/source/conf.py` if it reads `__version__` via regex — replace with `from importlib.metadata import version as get_version`.

Delete `setup.py`, `setup.cfg`, and `.coveragerc` (coverage config now lives in `pyproject.toml`).

If a reviewer asks for `MANIFEST.in` excludes (setuptools-scm puts every tracked file in the sdist), add them, then build the sdist and check nothing that shipped before stopped shipping.

Update `MANIFEST.in` once the package moves to `src/` layout in Phase 4 — paths must use `src/<package>/static` etc., and drop any `include requirements/...` lines referencing the now-deleted `requirements/` directory.

---

## Phase 2 — Switch to uv

### Dependency groups

**Library repos:**
```toml
[dependency-groups]
test-base = ["coverage", "ddt", ...]          # no version pins
test = [{include-group = "test-base"}, "Django>=5.2,<6.0"]   # current default
django42 = [{include-group = "test-base"}, "Django>=4.2,<4.3"]  # legacy matrix
quality = [{include-group = "test"}, "edx-lint", "pycodestyle", "pylint"]
doc = [{include-group = "test"}, "sphinx", "doc8", ...]
ci = ["tox", "tox-uv>=1"]                    # minimal — CI installs this first
dev = [{include-group = "quality"}, {include-group = "doc"}, {include-group = "ci"}]

[tool.edx_lint]
uv_constraints = []   # repo-specific version overrides go here — the ONLY
                      # place to add them; never edit [tool.uv].constraint-dependencies

[tool.uv]
package = true
default-groups = []   # otherwise `uv sync --group ci` implicitly pulls in
                      # the full `dev` superset too
conflicts = [[{group = "test"}, {group = "django42"}]]  # both pin Django
```

**Application repos:** add `base`, `bundled`, `assets`, `semgrep`, `coverage` groups as needed; strip `-r other.txt` lines (replaced by `include-group`) and `-c constraints.txt` lines (handled via constraint-dependencies).

**Carry over `dev.in` extras:** Check the old `dev.in` for packages not already covered by quality/ci groups (e.g. `diff-cover`, `edx-i18n-tools`) and add them to the `dev` group. Expose these tools through a tox env (`dependency_groups = dev` or a dedicated group) rather than a Makefile target that shells out to `uv run` — see the Makefile rules below.

**Rules:**
- `ci` group = just `tox` + `tox-uv`. Never add `uv tool install tox` to the Makefile.
- `quality` and `doc` must include `{include-group = "test"}`.
- Do NOT add ruff unless the repo already uses it.
- If `setup.cfg` had `[pycodestyle]`, migrate to `tox.ini` (pycodestyle reads tox.ini; pyproject.toml not supported).
- If bumping a framework's version range drops support for an older Python, bump `requires-python` accordingly and remove that Python from the tox envlist and CI matrix.
- edx-lint 6.2.0's PII checker crashes pylint on a `pylintrc` without `pii-terms`. If the lock resolves 6.2+, regenerate it: `uv run --with edx-lint edx_lint write pylintrc`.
- Declare every conflicting pair in `[tool.uv].conflicts` explicitly (each group that includes `test`, against `django42`), as openedx-ledger does. Otherwise uv derives them, the order varies by uv version, and `uv lock --check` passes or fails depending on who wrote the lock.

### Commands

```bash
# Populate [tool.uv].constraint-dependencies (machine-managed — never edit directly)
uv run --with edx-lint edx_lint write_uv_constraints pyproject.toml

uv lock   # generate uv.lock
```

**Seed `uv.lock` from the base branch's pins — never ship a fresh resolve.** A migration must not change what gets installed (feanil on enterprise-access#1015: a fresh lock pulled in 58 of 117 production upgrades the bot PRs hadn't merged yet). The first `make upgrade` after merge takes the upgrades.

1. Collect every `name==version` from the default branch's `requirements/*.txt` (`git show origin/master:requirements/<f>.txt`). Skip `pip.txt`, `pip-tools.txt` and `*constraints.txt`, and factor files that tox layers on top (`django.txt`, `celeryNN.txt`).
2. Add them temporarily to `[tool.uv].constraint-dependencies`, run `uv lock`, restore `pyproject.toml`, run `uv lock` again. uv keeps the locked versions as preferences, so the second lock keeps the base branch's versions.
3. Verify each group: `uv export --frozen --no-hashes --no-emit-project --no-default-groups --only-group <g>` against `requirements/<g>.txt`. Expected differences only: tooling the migration adds (tox-uv, setuptools-scm), packages that left the graph (pip-tools, wheel), platform-only markers (colorama), and packages the base branch itself pins at two versions in different files (e.g. docutils in doc.txt vs quality.txt).

Edge cases:
- A package a conflict group requires directly (Django in a `django42` setup) can't be a global `==` pin: it would make the other group unresolvable. Put `Django==<base pin>` temporarily in each group on the other side of the conflict instead.
- A pin that needs a newer Python than `requires-python` allows (e.g. `sphinx==9.1.0` with `>=3.11`) needs a marker: `"sphinx==9.1.0; python_version >= '3.12'"`.
- If the base branch's CI never ran a group's environment (enterprise-access never ran Django 4.2), drop the group and its `conflicts` instead of locking an environment nobody tested.
- A group that hard-pins old versions of shared tools (edx-enterprise-data's `test-reporting`: `pytest==8.0.2`, `ddt==1.1.2`) leaks them into every group unless it's declared in `conflicts`, and must not be included by `dev`.
- A repo that syncs pins from openedx-platform (edx-enterprise) seeds that block from its own committed snapshot (`requirements/edx-platform-constraints.txt` on the base branch), not today's platform.
- Rerun `edx_lint write_uv_constraints` first. Stale constraints let `social-auth-app-django` 6 into enterprise-access, which broke login.

Then delete `requirements/`.

### `tox.ini`

```ini
[tox]
requires = tox-uv>=1
envlist = django{42,52}, quality, docs

[testenv]
runner = uv-venv-lock-runner
dependency_groups =
    django42: django42
    django52: test
commands = ...

[testenv:quality]
runner = uv-venv-lock-runner
dependency_groups = quality
commands =
    pycodestyle <package>/
    pylint <package>/

[testenv:docs]
runner = uv-venv-lock-runner
dependency_groups = doc
allowlist_externals =
    make
commands =
    doc8 --ignore-path docs/_build README.rst docs
    python -m build --wheel
    twine check dist/*
    make -C docs clean
    make -C docs html
```

Keep `allowlist_externals = make` in every env that calls `make`.

### `Makefile`

**Makefile targets must NOT use the `uv run` prefix** (openedx/public-engineering#506). The Makefile assumes the venv is already activated (e.g. via `uv sync` + direct `source .venv/bin/activate`, or because the caller already ran `uv run make ...`). Don't change which way the Makefile and tox call each other: if the repo's targets inline commands, call `tox`, or are called by tox envs, keep that. feanil prefers tox calling the Makefile in the long run but doesn't want migration PRs to flip either direction (2026-09-28). The example below is one repo's shape, not a target to rewrite towards.

Call Django management commands as `python manage.py ...`, not `./manage.py` (the file often isn't executable).

The **only** exception is `uv run --with <pkg>` in the `upgrade` target. Do not add `uv run` anywhere else in the Makefile — in particular, never put it inside a target that tox itself invokes (e.g. `clean`'s `coverage erase`): `uv run` re-syncs default dependency groups, which corrupts the Django version matrix when it runs inside an already-selected tox env.

No `uv sync` in any target except `requirements`: a target that syncs one group (e.g. `docs: uv sync --group doc`) shrinks the developer's venv and removes tools a later target needs (feanil on openedx-ledger#242). Put the missing group in `dev` instead.

```makefile
upgrade:
    uv run --with edx-lint edx_lint write_uv_constraints pyproject.toml
    uv lock --upgrade

requirements:
    uv sync --group dev

quality:
    tox -e quality

test:
    tox -e django52

test-all:
    tox -e django42,django52

docs:  ## generate Sphinx HTML documentation, including API docs
    tox -e docs
    $(BROWSER)docs/_build/html/index.html

pii_check:  ## check for PII annotations on all Django models
    tox -e pii_check

clean:
    coverage erase
```

`pii_check` and the `docs` target's `$(BROWSER)...` open-in-browser line are only applicable if the repo already has them (PII annotations on Django models / a `BROWSER` variable convention) — don't introduce either from scratch for a repo that never had them.

### CI workflow

```yaml
on:
  push:
    branches: [master]    # KEEP — CI still needs to run on direct pushes
  pull_request:
    branches: ['**']
  workflow_call:          # lets release.yml call this workflow and reuse its result

jobs:
  run_tests:
    name: ${{ matrix.toxenv }}
    runs-on: ${{ matrix.os }}
    permissions:
      contents: read
    steps:
    - uses: actions/checkout@<sha> # vX.Y.Z
    - name: Setup uv
      uses: astral-sh/setup-uv@1c37ad07a6a961277cf70c0d37d6f313000f5884 # v10.3.0
      with:
        enable-cache: true
        python-version: "${{ matrix.python-version }}"
    # setup-uv handles Python — do NOT add a separate actions/setup-python step

    - uses: actions/setup-node@<sha> # vX.Y.Z  (if repo has JS assets)
      with:
        node-version-file: '.nvmrc'
    - name: Install JS dependencies
      run: npm install              # required if repo has csslint/eslint tox envs

    - name: Install CI dependencies
      run: uv sync --group ci --locked

    - name: Run Tests
      run: uv run --locked tox -e ${{ matrix.toxenv }}
```

Other CI rules:
- Pin all actions to full commit SHAs: `@<sha> # vX.Y.Z`
- Job `name: ${{ matrix.toxenv }}` for identifiable matrix jobs, **unless** branch protection requires checks by their old names. Check first: `gh api repos/openedx/<repo>/branches/<default>/protection --jq .required_status_checks.contexts`. A renamed job never reports the required name, so the PR stays blocked with every check green (enterprise-access#1015 requires `tests (3.12, pinned)`). If anything is required, keep the old job name and matrix values.
- Add `workflow_call:` trigger so `release.yml` can call this workflow. **Keep** `push: [master]` (or `[main]`) too — it's what runs CI on direct pushes (dependabot merges, hotfixes) independently of the release flow; it does not conflict with `workflow_call`.
- Add `permissions: contents: read` to the job.
- Every `uv sync` and `uv run` in every workflow (CI and migration-check workflows alike) uses `--locked`, so a `uv.lock` that's drifted from `pyproject.toml` fails the build instead of being re-resolved on the runner (feanil, 2026-10-29). `uv run --with <tool>` one-offs are the only exception.
- If the workflow has a `concurrency` block, use sample-plugin's form once `workflow_call:` is added. Otherwise a push to the default branch and the release's `run_ci` share a group and `cancel-in-progress` can cancel the release:
  ```yaml
  concurrency:
    group: ${{ github.workflow }}-${{ github.event_name == 'pull_request' && github.ref || github.run_id }}
    cancel-in-progress: ${{ github.event_name == 'pull_request' }}
  ```
- Tests don't need tag history: don't add `fetch-depth: 0` to the test checkout. A shallow checkout builds the package as `fallback_version`, which is fine for tests.

**Do NOT touch `pypi-publish.yml`** in Phase 2 — it will be deleted in Phase 3.

---

## Phase 3 — Semantic release (library repos only)

### `pyproject.toml`

```toml
[tool.semantic_release]
# No version_toml — version is dynamic via setuptools-scm
build_command = "python -m pip install --upgrade build && SETUPTOOLS_SCM_PRETEND_VERSION=$NEW_VERSION python -m build"
major_on_zero = false
allow_zero_version = true
```

`build_command` must use `python -m build`, not `uv build` — PSR's action environment has no uv.

### `release.yml`

> **This block must match `openedx/sample-plugin`'s `.github/workflows/release.yml` exactly** (structure, key order, comments), minus its monorepo-only lines (`directory:`, `packages-dir:`, multiple dist paths, the npm/tutor-plugin jobs). Before writing it, fetch the live reference and diff against it:
> ```bash
> gh api repos/openedx/sample-plugin/contents/.github/workflows/release.yml --jq '.content' | base64 -d
> ```
> PSR's pinned SHA/version in particular changes over time — always take it from the live fetch, not from memory or this doc.

```yaml
name: Semantic Release

on:
  push:
    branches: [master]   # match this repo's default branch (sample-plugin uses main)

jobs:
  # The same aggregate a pull request has to pass, so the release runs exactly
  # the checks that were required to merge -- and picks up any package added to
  # ci.yml later without needing a change here.
  run_ci:
    uses: ./.github/workflows/ci.yml

  release:
    # The package this workflow publishes has to build before we tag
    # anything. The GitHub release and the PyPI upload cannot be taken back,
    # so a package that only fails to build in its publish job would leave the
    # release half-finished.
    needs: [run_ci]
    runs-on: ubuntu-latest
    if: github.ref_name == 'master'
    concurrency:
      group: ${{ github.workflow }}-release-${{ github.ref_name }}
      cancel-in-progress: false

    permissions:
      contents: write

    steps:
      # Note: We checkout the repository at the branch that triggered the workflow.
      # Python Semantic Release will automatically convert shallow clones to full clones
      # if needed to ensure proper history evaluation. However, we forcefully reset the
      # branch to the workflow sha because it is possible that the branch was updated
      # while the workflow was running, which prevents accidentally releasing un-evaluated
      # changes.
      - name: Setup | Checkout Repository on Release Branch
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          ref: ${{ github.ref_name }}

      - name: Setup | Force release branch to be at workflow sha
        run: |
          git reset --hard ${{ github.sha }}

      - name: Action | Semantic Version Release
        id: release
        uses: python-semantic-release/python-semantic-release@b700dbeb1f431a0fcc30a79f3f0869407fe07278 # v10.7.0
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          git_committer_name: "github-actions"
          git_committer_email: "actions@users.noreply.github.com"
          changelog: "false"
          # Commit, tag, push and build, but don't create the GitHub release.
          # We create it ourselves in the next step so that the distributions
          # are attached before the release is published. See that step for why.
          vcs_release: "false"

      # This repo has immutable releases enabled, which freezes a release's
      # assets the moment it is published, so assets cannot be attached
      # afterwards. Everything we want on the release has to be built by now
      # and passed to this one command, which creates the release as a draft,
      # uploads the assets, and only then publishes it:
      # https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/immutable-releases
      - name: Publish | Create GitHub Release with Assets
        if: steps.release.outputs.released == 'true'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          # Reuse the release notes python-semantic-release generated for us.
          RELEASE_NOTES: ${{ steps.release.outputs.release_notes }}
          TAG: ${{ steps.release.outputs.tag }}
        run: |
          # Output the release notes to a file
          printf '%s' "$RELEASE_NOTES" > "$RUNNER_TEMP/release_notes.md"
          # Creates the release as a draft, uploads the assets, and then
          # publishes it -- all within this one command.
          gh release create "$TAG" \
            --verify-tag \
            --title "$TAG" \
            --notes-file "$RUNNER_TEMP/release_notes.md" \
            dist/*

      - name: Upload | Distribution Artifacts
        uses: actions/upload-artifact@cf430e030ddbb5b0abf93d22962f4752f3646cd9 # v7.0.2
        if: steps.release.outputs.released == 'true'
        with:
          name: distribution-artifacts
          path: dist
          if-no-files-found: error

    outputs:
      released: ${{ steps.release.outputs.released || 'false' }}
      version: ${{ steps.release.outputs.version }}

  publish_to_pypi:
    # 1. Separate out the publish step from the github release step to run each step at
    #    the least amount of token privilege
    # 2. Also, publishing can fail, and its better to have a separate job if you need to retry
    #    and it won't require reversing the release.
    runs-on: ubuntu-latest
    needs: release
    if: github.ref_name == 'master' && needs.release.outputs.released == 'true'

    permissions:
      contents: read
      id-token: write

    steps:
      - name: Setup | Download Build Artifacts
        uses: actions/download-artifact@9000827ccba6bdab643e8b6fd33ac0654aef8333 # v8.0.2
        id: artifact-download
        with:
          name: distribution-artifacts
          path: dist

      - name: Publish to PyPi
        uses: pypa/gh-action-pypi-publish@dc37677b2e1c63e2034f94d8a5b11f265b73ba33 # v1.14.2
```

**Monorepo-only lines to drop for a single-package repo** (sample-plugin publishes a backend package + a tutor plugin + an npm frontend package from one workflow): the `directory:` input on the PSR step, the separate `Build | ...` step before the release, `packages-dir:` on the PyPI publish step, per-package `backend-plugin-sample/dist` style paths, and the extra `publish_tutor_plugin_to_pypi` / `publish_to_npm` jobs. None of those are divergences for a single-package repo — don't add them.

**Key rules:**
- `vcs_release: "false"` — PSR tags, pushes, builds but does NOT create the GitHub release
- `gh release create` replaces `publish-action`: draft → attach assets → publish (required by GitHub immutable releases)
- `printf '%s'` (not `echo`) — prevents backticks/`$(...)` in release notes being executed
- `outputs:` must be declared **after** `steps:` — otherwise downstream jobs see empty string
- `pypa/gh-action-pypi-publish` must be SHA-pinned (e.g. `@dc37677b2e1c63e2034f94d8a5b11f265b73ba33 # v1.14.2`) — pin to the latest SHA from the sample-plugin reference
- OIDC auth only — no `user: __token__` / `password:` fields
- Use `secrets.GITHUB_TOKEN` (not a PAT) — tag and release creation are covered by `contents: write`
- Delete `pypi-publish.yml` — release.yml replaces it
- Create `commitlint.yml` if it doesn't exist
- Remind user to configure OIDC trusted publisher on PyPI (workflow: `release.yml`)
- If the repo has Sphinx docs, update `.readthedocs.yaml` to install via `method: uv`, `command: sync`, `groups: [doc]` instead of pip — this uses `uv.lock` and avoids a heavy pip dependency resolution on every RTD build. No `[project.optional-dependencies].docs` needed with the uv method. Keep the `sphinx:` block and `fail_on_warning` if they were there, and pin `build.os` to a fixed version (`ubuntu-26.04`) rather than `ubuntu-lts-latest`, which RTD warns can break builds when it moves (event-bus-kafka#349).
- Remove the manual release steps that PSR now does: "Version bumped", "Changelog record added", "Create a tag", "Create a release on GitHub" and the PyPI check in `.github/PULL_REQUEST_TEMPLATE.md`; "bump `__version__`" / "tag and Travis releases" sections in `README.rst` (it's the PyPI description too). Say instead that merges to the default branch are released automatically from the conventional commit types, with notes on GitHub Releases.
- Replace "create/activate a virtualenv, then `make requirements`" in `README.rst` and `docs/getting_started.rst`/`docs/testing.rst` (the cookiecutter text) with the uv flow: `make requirements` creates `.venv`, then `source .venv/bin/activate`. `uv sync` ignores an activated virtualenv, so the old flow fails at `make test`.

---

## Phase 4 — src layout

Skip if `src/<package>/` already exists.

openedx-translations doesn't block the move: its extraction job is being changed to strip `src/` centrally (openedx/openedx-translations#82365), and `transifex.yml` stays as is. Do check other outside code that fetches in-repo paths: `gh search code "<repo>/master/<package>/" --owner openedx`. If that finds something, skip Phase 4 and say why in the PR.

1. `git mv <package>/ src/<package>/`
2. `pyproject.toml` — add `where = ["src"]` to `[tool.setuptools.packages.find]`
3. `pyproject.toml` — change `[tool.coverage.run]` from `source = ["<package>"]` to `source_pkgs = ["<package>"]`. This isn't cosmetic: `source` scans a filesystem path, but Django's default (label-less) test discovery can't recurse past `src/` (it has no `__init__.py`, so `unittest`'s loader stops there — `unittest/loader.py`'s `_find_test_path` returns `should_recurse=False` for any directory lacking `__init__.py`). `source_pkgs` instead locates the package via its import `__path__`, independent of how the test runner was invoked. Getting this wrong silently produces near-zero coverage (imports still execute, but no tests actually run) while CI stays green.
4. `tox.ini` — update filesystem paths: `pycodestyle src/<package>/`, `pylint src/<package>/`, `changedir = {toxinidir}/src/<package>`, csslint/eslint args
5. `Makefile` — update `module_root := src/<package>` (if used as a variable, this cascades automatically)
6. Settings files — update `LOCALE_PATHS` / `root()` calls that reference the package directory
7. `MANIFEST.in` — update paths to `src/<package>/static` etc.; drop any `include requirements/...` lines
8. If the test runner (e.g. a Django `test.py`) is invoked without an explicit app/package label anywhere (Makefile, tox, CI), add the label explicitly now — see point 3's note on why label-less discovery breaks under src/ layout
9. Non-Python references to the old path. Search every tracked file outside `src/` for `<package>/`, and fix what isn't an import path:
   - symlinks: `git ls-files -s | awk '$1==120000{print $4}'`, then check each target still exists (edx-enterprise's root `locale` -> `enterprise/conf/locale` broke `make extract_translations`)
   - docs: Sphinx directives with repo-relative paths (`:folder_path:`), `linkcode_resolve` in `docs/conf.py` (go up one more directory for `src/`), and `github.com/<org>/<repo>/blob/<branch>/<package>/` links (commit-pinned permalinks are fine)
   - `.tx/config`, and `.gitignore`/`.jshintignore` entries anchored at the root (a stale `<package>/static/bundles/*.js` stops ignoring built bundles)
10. Run `uv lock` (pyproject.toml changed)
11. Run `make quality` **and** `make test` (or the tox env that runs the test suite) and confirm the test count/coverage didn't silently drop before committing — a passing `make quality` alone won't catch a test-discovery regression

**Do NOT change:**
- `@patch('package.module')` in tests — Python import paths, not filesystem paths
- `--cov <package>` — installed package name, not path (this is unaffected by the `source` → `source_pkgs` change above, which is coverage-config-specific)

---

## Handling reviewer feedback

- **No speculative code during a batch fix.** Don't add anything "for consistency" or "just in case" as a side effect of a requested change, especially across sibling files. If one file differs from its siblings, check that file's review threads first — the difference may be a decision a reviewer already made. (openedx-platform#38915: a `GITHUB_PATH` line removed after review came back while fixing 12 other workflow files.)
- **Re-check resolved threads before asking for re-review.** After new commits, re-diff every file with a resolved thread against what that thread concluded. A later edit can silently undo an earlier fix, and later review rounds tend to check only new comments.

---

## Keeping a long-open PR mergeable

The requirements bot (`make upgrade` editing `requirements/*.txt`) and Dependabot (`setup-python` bumps in workflows you rewrote) cause conflicts, often across a whole batch at once. Merge rather than rebase once there are review threads:
1. `git merge origin/<default>`; `git rm` the files the migration deleted, keep the deleted `pypi-publish.yml` deleted.
2. Resolve conflicting workflow *hunks* to your side; don't take the whole file, which drops the default branch's unrelated bumps in it.
3. Carry any new upstream code into `src/` by hand.
4. Reseed or `uv lock --check` (see Phase 2), push with `--force-with-lease`, and confirm the PR's head SHA is what you pushed.

## Committing

One commit per phase, conventional format:
- Phase 1: `chore: migrate package metadata to pyproject.toml (PEP 621)`
- Phase 2: `chore: switch dependency management from pip-tools to uv`
- Phase 3: `chore: add python-semantic-release and commitlint workflows`
- Phase 4: `refactor: migrate to src layout`

Before pushing: run `make quality` — must pass with pylint 10.00/10.

Then compare the build with the last release:
```bash
uv lock --check
uv build
curl -s https://pypi.org/pypi/<dist>/json > meta.json   # save to a file; zsh's echo mangles the JSON
diff <(unzip -Z1 old.whl | grep -v dist-info/ | sort) <(unzip -Z1 dist/*.whl | grep -v dist-info/ | sort)
diff <(unzip -p old.whl '*.dist-info/METADATA' | grep ^Requires-Dist | sort) \
     <(unzip -p dist/*.whl '*.dist-info/METADATA' | grep ^Requires-Dist | sort)
tar tzf dist/*.tar.gz | grep -iE 'CHANGELOG|README|LICENSE'
uvx twine check dist/*
```
Then a PSR dry run, on a local branch named like the default branch, to confirm it finds the latest `v` tag and computes the next version: `uvx --from python-semantic-release semantic-release --noop version --print`.
The file lists should match except for differences you can explain in the PR (e.g. test files that shouldn't have shipped). Diff the lists, not just the counts.

Push to fork, open PR against the **upstream repo** (not the fork):
```bash
git push origin modernize-python-tooling
gh pr create --repo <upstream-owner>/<repo> --base master --head <fork-owner>:modernize-python-tooling
```

---

## Checklist

```
Phase 1
[ ] pyproject.toml: static dependencies, setuptools-scm, coverage config
[ ] pyproject.toml: [tool.setuptools.package-data] includes all dirs from old setup.py package_data (check translations/**)
[ ] License classifier removed (PEP 639: SPDX license = conflicts with License :: classifier)
[ ] license uses AGPL-3.0-only or AGPL-3.0-or-later (matching the original classifier) — never bare "AGPL-3.0"
[ ] license-files includes NOTICE if the repo has one
[ ] [tool.setuptools.packages.find] excludes test subpackages: exclude = ["*tests*"]
[ ] No upper bounds in [project].dependencies; Requires-Dist matches the last PyPI release
[ ] include-package-data set so the wheel's file list matches the last PyPI wheel
[ ] authors = Open edX Project <oscm@openedx.org> (if the old author was edX/Open edX)
[ ] __version__ kept, read via importlib.metadata without try/except
[ ] setup.py / setup.cfg / .coveragerc deleted
[ ] fail_under only if original .coveragerc had it
[ ] CHANGELOG.rst dropped from readme with a DEPRECATED note at the top; no insertion marker, no deleted entries, still in the sdist
[ ] A v-tag exists for the latest PyPI version (checked live); tag_format = "v{version}" optional

Phase 2
[ ] dependency-groups in pyproject.toml (test-base/test/django42/quality/doc/ci/dev)
[ ] [tool.edx_lint] section exists with uv_constraints (even if empty)
[ ] [tool.uv]: package=true, default-groups=[], conflicts declared for version-matrix groups
[ ] edx_lint write_uv_constraints run, uv.lock committed
[ ] uv.lock seeded from the base branch's requirements/*.txt; each group's uv export diffed against its .txt (only explainable differences)
[ ] requirements/ deleted
[ ] tox.ini: tox-uv>=1, uv-venv-lock-runner, dependency_groups
[ ] Makefile: NO uv run prefix anywhere except `uv run --with <pkg>` in upgrade; no uv sync except in requirements; Makefile↔tox calling direction unchanged; manage.py called as `python manage.py`; clean uses bare `coverage erase`
[ ] pylintrc regenerated if the lock resolves edx-lint 6.2+
[ ] CI: setup-uv SHA-pinned, no setup-python, npm install (if JS assets), --locked on every uv sync / uv run in every workflow, no fetch-depth: 0 on test checkouts
[ ] CI: an existing concurrency block uses sample-plugin's pull_request/run_id form once workflow_call: is added
[ ] CI: required status checks checked before renaming jobs; workflow_call: trigger added, push: [master] KEPT (not removed), permissions: contents: read on the job
[ ] pypi-publish.yml untouched (deleted in Phase 3)

Phase 3
[ ] [tool.semantic_release]: tag_format, build_command (python -m build), major_on_zero = false, allow_zero_version = true
[ ] release.yml fetched fresh from openedx/sample-plugin and structurally diffed against it (job/key order, comments, `run: |` vs `run:`) — monorepo-only lines (directory:, packages-dir:, multi-path dist, npm/tutor jobs) excepted
[ ] release.yml: job calling ci.yml is named run_ci, no secrets: inherit, no permissions block on it
[ ] release.yml: checkout step named "Setup | Checkout Repository on Release Branch", no fetch-depth: 0; reset step named "Setup | Force release branch to be at workflow sha" using `run: |`
[ ] release.yml: git_committer_name "github-actions" / git_committer_email "actions@users.noreply.github.com" (no [bot] suffix)
[ ] release.yml: PSR action pinned to the SHA/version currently on sample-plugin's release.yml (don't reuse an old pin from memory)
[ ] release.yml: vcs_release: "false", gh release create --verify-tag with printf release notes; upload-artifact@cf430e03 # v7.0.2 (uses: before if:, path with no trailing slash), download-artifact@9000827c # v8.0.2 (pinned SHAs, as on sample-plugin today); pypa@dc37677b # v1.14.2 step named "Publish to PyPi"; publish job if includes github.ref_name check matching the release job's branch; released output has || 'false' fallback
[ ] pypi-publish.yml deleted
[ ] commitlint.yml exists
[ ] .readthedocs.yaml uses method: uv / command: sync / groups: [doc], build.os pinned (if repo has Sphinx docs)
[ ] User reminded to configure OIDC trusted publisher on PyPI
[ ] PR template, README and docs no longer describe manual version bumps/tags/releases or a self-made virtualenv

Phase 4
[ ] No outside raw-file fetches of in-repo paths (`gh search code`); otherwise skip and say why (openedx-translations is not a blocker)
[ ] Package moved to src/<package>/
[ ] pyproject.toml: where = ["src"]
[ ] pyproject.toml: [tool.coverage.run] source_pkgs (not source) — required so default-label test discovery working (or not) doesn't silently zero out coverage
[ ] Test runner is invoked with an explicit app/package label everywhere (Makefile, tox, CI) — never label-less, since src/ blocks unittest's default discovery
[ ] tox.ini + Makefile filesystem paths updated
[ ] MANIFEST.in paths updated
[ ] Symlinks, docs paths/links/linkcode_resolve, .tx/config and root-anchored ignore files updated
[ ] uv.lock regenerated
[ ] uv run tox -e quality passes

Before pushing
[ ] uv lock --check clean; PSR --noop dry run finds the latest v tag; wheel file list and Requires-Dist diffed against the last PyPI release; sdist still has CHANGELOG/README/LICENSE; twine check passes

Review rounds
[ ] No unrequested "for consistency" or "just in case" changes in the latest commits
[ ] Every file with a resolved review thread re-diffed against what the thread concluded
```
