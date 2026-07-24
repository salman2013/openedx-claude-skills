---
name: modernize-python-tooling
description: Migrate an openedx Python package or platform repo from legacy tooling (pip-tools, setup.cfg/setup.py) to modern tooling (uv, pyproject.toml PEP 621/735, python-semantic-release). Use for any "Modernize Python repo" ticket under openedx/public-engineering.
---

Migrate an openedx Python package or platform repo from legacy tooling (pip-tools,
setup.cfg/setup.py) to modern tooling (uv, pyproject.toml PEP 621/735,
python-semantic-release).

## Origin

Consolidates three independently-converged implementations of the same org-wide
effort (openedx/public-engineering#506, tracked per repo-group in #511 XBlocks, #513
Enterprise, #520 Webhooks/Other). Base phases 1-3 come from
[salman2013/openedx-claude-skills](https://github.com/salman2013/openedx-claude-skills);
the PR template comes from #513/#520 PRs. This is a snapshot, not a live sync — if the
upstream skill has moved on, re-diff before trusting this file over it.

Updated 2026-07-24 with three lessons from reviewing #520 PRs (`forum` #283,
`credentials-themes` #1086, `xapi-db-load` #258) — see Phase 2 and the new
"Self-audit before requesting review" section.

## Usage
```
/modernize-python-tooling [repo-path]
```
If no path is provided, operate on the current working directory.

## Context

The openedx org is standardizing Python tooling. Reference implementation:
`openedx/sample-plugin` — specifically `backend/pyproject.toml`, `backend/tox.ini`,
`backend/Makefile`, `.github/workflows/backend-ci.yml`.

**Two repo types are supported:**

**Library repos** (PyPI packages):
- Published (or intended) to PyPI
- Use `pip-compile` / `requirements/*.txt`
- Use `setup.py` or `setup.cfg` (or `pyproject.toml` with `dynamic = ["dependencies"]`)
- All three phases apply.

**Application/platform repos** (not published to PyPI):
- Large Django applications/services (e.g. `edx-platform`, or a deployed service like
  `enterprise-catalog` that happens to live in a "library-shaped" repo)
- Use `pip-compile` / `requirements/*.txt`, possibly layered (base, testing, dev, etc.)
- May already have a `pyproject.toml` with minimal `[project].dependencies`
- **Phase 3 (semantic-release) does not apply — skip it entirely.**
- `[project].dependencies` stays minimal or absent; runtime deps go in a `base`
  dependency group instead.
- Map the existing layered requirements structure to dependency groups (Phase 2).

**Auto-detect repo type**: if there is no `setup.py`/`setup.cfg` and
`[project].dependencies` is minimal (≤ 3 packages) — or the repo is a deployed Django
*service* rather than an importable package regardless of what its requirements layout
looks like (confirmed independently on `enterprise-catalog`, PR #1136) — treat it as
an application repo. When in doubt, check whether the repo has ever been published to
PyPI and whether other repos install it as a dependency.

## Behavior

Work through the three phases in order. At the start, read existing `setup.cfg`,
`setup.py`, `pyproject.toml`, `tox.ini`, `Makefile`, `requirements/*.in`, and
`.github/workflows/*.yml` to understand current state before making changes.

**CRITICAL — assess before touching anything:**
Before making any change, compare the current state of each file against the phase
checklist. For every item, mark it as already done or missing. Only implement what is
genuinely missing. Never modify a file that already satisfies its phase requirement.
If the user asks why you are changing a working file, stop and explain — do not
proceed without their confirmation.

**Mechanical, all-or-nothing per repo:** this migration can't be split into partial
PRs — you can't half-delete `requirements/` or half-wire-up dependency-groups. Do the
whole repo as one PR with one commit per phase (see Committing below), even when the
diff is large (a single-repo PR here has run 5000+ lines and that's normal/expected).

---

### Phase 1 — Consolidate metadata into `pyproject.toml`

1. Read `setup.cfg` / `setup.py` to extract: `name`, `version`, `description`,
   `long_description`, `author`, `license`, `classifiers`, `python_requires`,
   `install_requires`, `extras_require`, `packages`, `entry_points`
2. Create/update `pyproject.toml`:
   - `[build-system]`: use `setuptools>=64` and `setuptools-scm>=8.0`
   - `[project]`: static `dependencies` list (not dynamic from requirements file);
     `dynamic = ["version"]`
     - Framework dependencies (Django, etc.) must include a lower bound matching the
       oldest tested version (e.g. `"django>=4.2"`) — never unconstrained. If a new
       framework version drops an older Python, bump `requires-python` and remove that
       Python from the tox envlist.
     - Include framework classifiers matching tested versions (e.g.
       `"Framework :: Django :: 4.2"`, `"Framework :: Django :: 5.2"`)
     - Turn any `extras_require` entries into real `[project.optional-dependencies]`
       extras (confirmed on edx-enterprise's `braze` extra, PR #2654)
   - `[tool.setuptools_scm]`: omit `root` setting (unlike sample-plugin which uses a
     subdirectory); set `version_scheme = "only-version"` and
     `local_scheme = "no-local-version"` — the local scheme prevents dirty version
     suffixes from being rejected by PyPI
   - `[tool.setuptools.packages.find]`: set `where` if needed
   - `[tool.uv]`: include `package = true` to explicitly mark the project as a
     package (not a bare workspace root)
   - `[tool.setuptools.exclude-package-data]`: exclude test files from the wheel —
     use **kebab-case** (not underscore, which causes a setuptools schema validation
     error: `configuration error: tool.setuptools must not contain
     {'exclude_package_data'} properties`):
     ```toml
     [tool.setuptools.exclude-package-data]
     "*" = ["tests*", "*.tests*", "spec*", "*.spec*"]
     ```
   - `license`: convert to SPDX string format (PEP 639):
     `license = "AGPL-3.0"` + `license-files = ["LICENSE.txt"]` — NOT the old
     `license = { text = "..." }` inline table
   - `[tool.coverage.run]`: set `branch = true`, `source`, and `omit` patterns:
     ```toml
     omit = ["*/tests/*", "*/migrations/*", "*/__pycache__/*", "*/settings.py"]
     ```
   - `[tool.coverage.report]`: set `show_missing = true` and `exclude_lines`. Only add
     `fail_under` if the original `.coveragerc` already had one — **do not invent a
     threshold that didn't exist before**
   - `[tool.coverage.html]`: set `directory = "htmlcov"`
   - **Ruff: out of scope by default.** Parent issue #506 only specifies
     pyproject.toml + uv + semantic-release — it does not mention linting tooling at
     all. The #520 PRs (openedx-webhooks, forum, etc.) replaced
     pylint/isort/pycodestyle/mypy with `ruff` on their own initiative; the #513
     Enterprise PRs correctly left pylint untouched, which matches the ticket's
     actual scope. **Default to leaving existing lint tooling exactly as it is.**
     Only add ruff if the repo owner or a separate ticket explicitly asks for it —
     don't bundle an unrequested tooling opinion into a mechanical migration PR. If a
     repo owner does ask for it, the config below is what the #520 PRs used:
     ```toml
     [tool.ruff]
     line-length = 120
     target-version = "py312"

     [tool.ruff.lint]
     select = ["E", "W", "F", "I", "B", "C4", "UP", "DJ"]
     ignore = ["E501"]  # line too long (handled by formatter)

     [tool.ruff.lint.isort]
     known-third-party = ["django", "xblock"]  # adjust per repo

     [tool.ruff.format]
     quote-style = "double"
     indent-style = "space"
     ```
3. Delete `setup.py` and `setup.cfg`
4. Remove hardcoded `__version__` from `__init__.py` — with `setuptools-scm` deriving
   the version from git tags, this will be permanently stale after any new tag. If the
   version is needed at runtime (confirmed pattern: `edx-enterprise`'s
   `cache_utils.versioned_cache_key` reads it), use `importlib.metadata` instead:
   ```python
   from importlib.metadata import version, PackageNotFoundError
   try:
       __version__ = version("your-package-name")
   except PackageNotFoundError:
       pass
   ```
   Also update `docs/conf.py` if it reads `__version__` via regex from `__init__.py`.
   Remove the now-unused `re` import and any `sys.path.append` that was only there to
   import the package for version reading. If this touches `README.rst`, do NOT
   introduce a new RST heading level that conflicts with existing ones — RST infers
   heading levels from the first occurrence of each underline character, so adding a
   new sub-section level (e.g. `^^^^`) under a section whose existing siblings use a
   different character (e.g. `----`) causes `doc8` to report `D000 Title level
   inconsistent`. Match whatever underline character is already used at that depth.
5. Set `fallback_version` in `[tool.setuptools_scm]` to a **plain, dot-separated
   integer string** like `"0.0.0"` — never `"0.0.0.dev0"` or any value with a
   non-numeric segment. This is not a style preference: `openedx-events` used
   `"0.0.0.dev0"` and it crashed 17 tests with
   `ValueError: invalid literal for int() with base 10: 'dev0'`, because
   `data.EventData.sourcelib` parses `__version__` via
   `tuple(map(int, __version__.split(".")))`. That specific repo's runtime code was
   the trigger, but the underlying risk (a fallback value that isn't safely
   int-parseable) applies to any repo that reads its own `__version__` at runtime
   for anything beyond display — grep for other consumers of `__version__` in the
   codebase (not just its assignment) before assuming `"0.0.0.dev0"` is harmless.

**Pre-flight check before enabling Phase 3 in any repo:** setuptools-scm derives the
build version from the latest git tag; semantic-release computes the *next* version
from conventional-commit history since that tag. If no tag exists at or above the
version currently live on PyPI, the first automated release could compute a version
*lower* than what's published, and `publish_to_pypi` fails (or targets the wrong
version line). Confirmed twice in this effort, not a one-off: `openedx-events` was
missing a tag for its actual latest release (11.2.0 published, only tagged to
11.1.1); `acid-block` has the same gap (0.4.1 published, only tagged to 0.4.0).
Before enabling Phase 3, always check `git tag --sort=-v:refname | head` against the
package's actual latest version on PyPI — **don't just check that some recent tag
exists**, confirm it matches the real latest published version. Use
`--sort=-v:refname`, not plain `git tag --list` piped through `tail` — lexicographic
sort silently misorders version numbers (e.g. `v9.20.0` sorts after `v10.5.0`
alphabetically, which produced a wrong "latest tag" reading here on the first pass).
If a tag is missing, this is a decision for a repo maintainer to make (create the
real tag, or accept the risk), not something to silently work around or push
yourself without explicit authorization — pushing a tag on someone else's behalf is
asserting release history on a repo you may not own.

**Key rule:** `dependencies` must be a static list in `[project]`, not
`dynamic = ["dependencies"]`.

---

### Phase 2 — Switch dependency management to uv

1. Read all `requirements/*.in` files to understand current deps
2. Add `[dependency-groups]` to `pyproject.toml`. Groups depend on repo type:

   **Library repos** — standard groups:
   - `test-base`: pytest, coverage, etc. (no version pins — let uv.lock handle it)
   - `test`: `{include-group = "test-base"}` + any test extras; pin the current
     default framework version directly (e.g. `Django>=5.2,<6.0`)
   - `quality`: pylint (or ruff, per the decision above), edx-lint, etc. — include
     `{include-group = "test"}` so linting runs against the same Django version
   - `doc`: Sphinx and doc-building deps — include `{include-group = "test"}`
   - `ci`: `{include-group = "test"}` + `{include-group = "quality"}` + tox, tox-uv
     — CI installs this via `uv sync --group ci`, then calls `uv run tox`; keep it
     minimal
   - `dev`: `quality` + `ci` + `doc` includes (not a chain through `ci`) + dev
     conveniences
   - Add version-matrix groups for older supported framework versions (e.g.
     `django42`) — two Django-version groups can coexist in one `uv.lock` via
     `[tool.uv].conflicts`; always use this instead of falling back to
     `uv-venv-runner`, and comment how to update the matrix

   **Application/platform repos** — extended groups:
   - `base`: all runtime deps from the production requirements file, including any
     GitHub-hosted deps. These stay out of `[project].dependencies`.
   - `bundled`: optional third-party add-ons (e.g. third-party XBlocks). Include
     `{include-group = "base"}`.
   - `test`: `{include-group = "base"}` + testing-specific deps
   - `coverage`: coverage measurement tools
   - `quality`: linting tools — include `{include-group = "test"}`
   - `doc`: Sphinx + doc deps — include `{include-group = "base"}`
   - `assets`: frontend build deps — standalone group
   - `semgrep`: semgrep and related tools — standalone group
   - `ci`: tox + tox-uv only
   - `dev`: `test` + `doc` + `assets` includes + dev-only tools
   - When reading `.in` files, strip `-r other.txt`/`-r other.in` references (replaced
     by `include-group`) and `-c constraints.txt` lines (handled via
     `[tool.uv].constraint-dependencies`)

3. Add `[tool.edx_lint]` section with `uv_constraints = []` for repo-specific version
   overrides (even if empty)
4. Run `edx_lint write_uv_constraints` to populate
   `[tool.uv].constraint-dependencies` — **this is machine-managed, never edit it
   directly**
5. Run `uv lock` to generate `uv.lock`. If the repo already has `uv.lock`, don't
   re-run unless `pyproject.toml` deps actually changed — re-running unnecessarily can
   downgrade pinned package versions.
6. Delete the `requirements/` directory
7. Update `tox.ini`: add `requires = tox-uv>=1`; change runner to
   `uv-venv-lock-runner`; replace `deps =` with `dependency_groups =`; remove
   `pip install` steps. (If the repo uses `[project.optional-dependencies]` instead of
   `[dependency-groups]`, `tox.ini` can use `extras =` instead — but a version matrix
   requires `[dependency-groups]` + `[tool.uv].conflicts`.)
   **`dependency_groups =` entries must name groups that exist in *this* repo's own
   `[dependency-groups]`** — don't copy a sibling repo's tox.ini verbatim. Confirmed
   real: `credentials-themes` PR #1086 copied `django52: test` from `forum`/
   `xapi-db-load` (where a `test` group exists) into a repo whose groups are actually
   named `django42`/`django52` with no `test` group at all. Breaks `tox -e django52`
   for anyone running it locally; invisible in CI whenever that repo drives tests via
   `uv sync --group` directly instead of through tox.
8. Update `Makefile`:
   - `upgrade`: `uv run --with edx-lint edx_lint write_uv_constraints pyproject.toml`
     then `uv lock --upgrade` — no pre-install of edx-lint needed
   - `compile-requirements`: remove or replace with `uv lock`
   - `requirements`: `uv sync --group dev` only — do **NOT** add
     `uv tool install tox --with tox-uv` (installs an unpinned global tox outside
     `uv.lock`; use `uv run tox` everywhere instead)
   - If you restructure any aggregate target (e.g. `test-quality: test-lint
     test-codestyle test-mypy test-format`), diff it against its pre-migration
     definition sub-target by sub-target — a rename/restructure can silently drop one
     with CI still green under the new name. Confirmed real: `forum` PR #283
     restructured `test-quality` down to `test-lint test-codestyle test-mypy`,
     silently dropping `test-format` (`black --check`) from the CI quality gate — not
     disclosed in the PR description, not caught by CI.
9. Update CI workflow(s):
   - Use `astral-sh/setup-uv` pinned to a full commit SHA (never a floating tag like
     `@v6`) — e.g. `@08807647e7069bb48b6ef5acd8ec9567f424441b # v8.1.0`
   - Pass `enable-cache: true` and `python-version` to `setup-uv` — this handles
     Python installation, so **remove** any separate `actions/setup-python` step
   - Replace `pip install -r requirements/ci.txt` with `uv sync --group ci`
   - Replace bare `tox` calls with `uv run tox` (or `uv run tox -e ${{ matrix.toxenv }}`
     for matrix jobs) — `uv sync` does not put tools on PATH, so bare `tox` won't find
     the synced environment
   - Set matrix job `name: ${{ matrix.toxenv }}` (not a static string) so failed jobs
     are identifiable in the GitHub UI
   - Add `permissions: contents: read` at the job level
   - Add a `workflow_call:` trigger so `release.yml` can reuse it (library repos)
   - If you add a new tox env to `envlist` that CI wasn't previously running (e.g.
     `docs`, `quality`), add it to the CI matrix in the same PR — an env that only
     exists in `tox.ini` provides no actual regression protection. Confirmed real:
     `xapi-db-load` PR #258 added `docs`/`quality` to `envlist` but left the CI matrix
     at `toxenv: [py]` only, so neither ever runs in CI.
   - **Add `fetch-depth: 0` to the CI workflow's `actions/checkout` step** (library
     repos using setuptools-scm). `actions/checkout` defaults to a shallow, tag-less
     clone, so setuptools-scm can't see any git tags during test runs and falls back
     to `fallback_version` instead of computing the real version. `release.yml`'s own
     checkout doesn't need this fix — `python-semantic-release`'s action auto-deepens
     a shallow clone before evaluating version history — but the CI/test job has no
     equivalent self-healing. Confirmed root cause of a real test crash in
     `openedx-events` (see Phase 1's `fallback_version` note above); confirmed present
     but not yet causing visible failures in 3 other repos in this effort
     (`xblock-sdk`, `xblock-lti-consumer`, `RecommenderXBlock`) — currently latent
     because their tests happen not to depend on the exact version string, not because
     the gap isn't there.

**edx-platform pin sync (rare — only build this if the repo already has it):**
confirmed unique to `edx-enterprise` among all repos checked so far; don't build this
for a repo that doesn't already have the old mechanism to replace. If a repo runs as
a plugin inside edx-platform and has a `requirements/edx-platform-constraints.txt` +
`check_pins.py` mechanism (refreshed via `curl` in `make upgrade` to pin its test
suite to whatever edx-platform actually installs it alongside), port it to a
`requirements/sync_platform_constraints.py` script: fetch edx-platform's
`requirements/edx/base.txt`, parse `==` pins by regex, merge into
`[tool.uv].constraint-dependencies` — run **after** `edx_lint write_uv_constraints`
(which owns/overwrites that list) and **before** `uv lock --upgrade`, so a
repo-specific override in `[tool.edx_lint].uv_constraints` still wins over a platform
pin for the same package. Fence the script's appended block with
`# --- begin/end openedx-platform sync ---` comment markers (built via a
`tomlkit.parse()` round-trip, not manual string edits) so platform-synced pins are
visually distinguishable from edx_lint-managed ones in the 300+ line array. Skip any
package with a hard pin from a narrow dependency group that conflicts with the
platform-derived constraint (e.g. a `js_test`-only `pyyaml` pin) — exclude it from the
merge and declare that group a conflicting resolution fork via `[tool.uv].conflicts`
instead of forcing one unresolvable graph.

---

### Phase 3 — Add semantic-release (library repos only — skip for application repos)

**If this is an application/platform repo, stop after Phase 2. Do not add
semantic-release.**

1. Add `[tool.semantic_release]` to `pyproject.toml`:
   - `build_command` must use `python -m build` (not `uv build`) —
     `python-semantic-release`'s action environment doesn't have uv available:
     ```
     build_command = "python -m pip install --upgrade build && SETUPTOOLS_SCM_PRETEND_VERSION=$NEW_VERSION python -m build"
     ```
   - Do NOT override `minor_tags` unless explicitly required
2. Create `.github/workflows/release.yml`:
   - `run_tests` job: `uses: ./.github/workflows/<ci-file>.yml` (reusable, no inline
     matrix) with `secrets: inherit` (so `CODECOV_TOKEN` etc. reach the called
     workflow) and `permissions: contents: read`
   - `release` job (`needs: run_tests`, `if: github.ref_name == 'main'`):
     `concurrency` with `cancel-in-progress: false`; checkout with
     `ref: ${{ github.ref_name }}` + `git reset --hard ${{ github.sha }}`; **no
     setup-uv step**; PSR action `python-semantic-release/python-semantic-release@v10.5.3`
     with `git_committer_name`, `git_committer_email`, `changelog: "false"`; upload
     `dist/` as artifact; expose `released` + `version` outputs;
     `permissions: contents: write` only (NOT `id-token`)
   - `publish_to_pypi` job (`needs: release`, `if: released == 'true'`): download
     artifact, publish via OIDC (`id-token: write`, no `user`/`password` inputs on the
     publish step — OIDC is used automatically once the permission is granted and no
     explicit credentials are passed)
   - **Pin `pypa/gh-action-pypi-publish` to a commit SHA, not `@release/v1`.**
     `@release/v1` is a floating branch, not a version tag — confirmed root cause of a
     real production failure (`openedx/xblocks-core`'s release run, 2026-07-14,
     `docker: manifest unknown`, because the ref resolved to a commit whose Docker
     image wasn't published on ghcr.io yet). This exact unpinned ref was present in
     `openedx/sample-plugin`'s own reference `release.yml` too (missed by that repo's
     own dedicated SHA-pinning PR #53 — every other action in the file uses a normal
     `@vX` tag that whatever tool drove that sweep could recognize; `@release/v1`
     looks like a branch name, not that pattern, so it slipped through) — check for
     this specifically, don't assume a repo copying the reference implementation got
     it right just because everything else is pinned.
3. **OIDC vs. token-based publish — target OIDC.** Parent issue #506 explicitly asks
   for OIDC ("publishes to PyPI via OIDC", "Confirm PyPI trusted publisher (OIDC) is
   configured for the repo"). Earlier drafts of this skill said to leave a working
   token-based publish in place — that was wrong; it papered over a real gap in the
   #513 batch (all 6 library repos shipped token-based auth against the ticket's
   explicit spec) rather than following it. Configure OIDC by default. This needs a
   PyPI-project-settings action (add a trusted publisher) that can't be done via API —
   flag it as a pre-merge blocker in the PR description (see template below), not a
   silent assumption, and don't merge until it's confirmed configured or the first
   release will simply fail.
4. **`major_on_zero`/`allow_zero_version`** — add to `[tool.semantic_release]` on
   every library repo, not just ones currently on a 0.x version:
   ```toml
   major_on_zero = false
   allow_zero_version = true
   ```
   `major_on_zero` only governs the 0.x → 1.0.0 transition specifically — once a repo
   is past 1.0.0 it's a no-op (a breaking-change commit still bumps 1.x → 2.0 normally
   regardless of this setting; there's no PSR config that suppresses major bumps past
   1.0, that's normal SemVer). Set it everywhere anyway for consistency and so a repo
   never auto-jumps to 1.0.0 as an accidental side effect if it's ever reset to 0.x —
   1.0.0 is supposed to signal a deliberate "first stable release" decision, not a
   side effect of a conventional-commit breaking-change marker.
5. Update the CI workflow: add `workflow_call:` trigger, remove `push: [main]`
   trigger — `release.yml` now owns that path (avoids running tests twice on main)
6. Create `.github/workflows/commitlint.yml`: enforce conventional commit format on
   PRs
7. **Search broadly for every pre-existing PyPI-publish-triggering workflow, not just
   the one you expect to find.** Different repos name theirs differently
   (`pypi-publish.yml`, `publish.yml`, `publish_pypi.yml`) and trigger on different
   events (`release: published`, `push: tags`) — grep every workflow file for
   `pypi-publish` and `pypa/gh-action` rather than deleting a single hardcoded
   filename. Missing one is a real, confirmed bug: `openedx/xblock-sdk` PR #519 added
   `release.yml` but left its old `pypi-publish.yml` (triggers on `push: tags: '*'`)
   untouched — once semantic-release pushes its first version tag, both workflows
   fire and race to publish the same version via two different auth mechanisms.

---

## Self-audit before requesting review

An AI review (yours or a teammate's) will not by default catch semantic/behavioral
drift — it wasn't told the rule existed. Before opening the PR, explicitly check for
these three failure modes, each confirmed in this effort's own PRs even with fully
green CI:

1. **A restructured aggregate target silently dropped a sub-check.** Diff any
   Makefile/tox target you touched (e.g. `test-quality`) against its pre-migration
   definition, sub-target by sub-target — not just "does it still run."
   (`forum` #283 dropped `test-format`/black from `test-quality` this way.)
2. **Config copied from a sibling repo references names that don't exist here.**
   Anything hand-copied between repos (tox.ini `dependency_groups =` lines, CI matrix
   snippets) must be checked against *this* repo's actual `[dependency-groups]`/env
   names, not assumed correct because it worked in the repo it came from.
   (`credentials-themes` #1086 copied a `test` group reference into a repo with no
   `test` group.)
3. **A newly-added tox env isn't wired into CI.** Adding `docs`/`quality` to
   `envlist` is a no-op for regression protection unless the CI matrix is updated in
   the same PR to actually run it. (`xapi-db-load` #258 left CI at `toxenv: [py]`
   after adding both.)

None of these fail CI — that's exactly why they need a deliberate line-by-line check,
not a "tests pass" glance. As feanil put it reviewing a sibling PR in this same
effort (`mockprock` #66): "AI will not pick up when we change user behavior or
semantics without a lot of pre-definition by us... don't assume that the AI review is
sufficient."

---

## Committing

After completing each phase, **pause and show the user a summary of all changes
made**, then ask for explicit approval before committing. Only commit once the user
confirms.

Suggested review prompt after each phase:
> "Here's what changed in Phase N — please review before I commit:"
> (list files created/modified/deleted with a one-line description of each change)
> "Shall I commit with message: `<commit message>`?"

Create a separate git commit per phase, all in one PR per repo:

- **Phase 1 commit:** `feat: consolidate package metadata into pyproject.toml`
- **Phase 2 commit:** `feat: switch dependency management from pip-compile to uv`
- **Phase 3 commit (library repos only):** `feat: add semantic-release and commitlint workflows`

Use conventional commit format (the whole point of semantic-release depends on it).
Do not squash phases into a single commit.

## PR description template

**Use selectively, not as a default reformat.** This template (from
openedx-webhooks#436, forum#280) earns its keep when a PR has a genuinely
non-obvious, repo-specific deviation worth calling out — e.g. an outlier release
workflow that needed reconciling, or why a test suite got excluded from CI. For a
repo where the change is exactly what the parent issue
already documents (delete setup.py/setup.cfg/requirements, add pyproject.toml +
uv.lock, done), the template mostly restates what the diff itself already shows
(GitHub's Files-changed view marks every deletion) — added structure without added
information. Reviewed and explicitly decided (2026-07-15) not to retrofit this onto
the #513 PRs: they're being reviewed as a batch of near-identical mechanical
migrations, where consistent brevity across all of them beats restructuring each one,
and reformatting an already-open PR's description with no code change reads as churn.
Reach for a short targeted callout on the one non-obvious point instead of a full
template swap, unless a specific PR actually has several deviations worth structuring.

Fill in only the sections that apply — app repos skip "Versioning"/semantic-release
mentions:

```markdown
> [!IMPORTANT]
> PR implemented with the assistance of Claude Code, human-reviewed and improved
> before pushing to code review.

## Summary

Modernize `<repo>` to uv + pyproject.toml (PEP 621/735)[ + python-semantic-release].

Part of <parent-issue-url> (tracked in <child-issue-url>).

- Replace `setup.py`/`setup.cfg` with `pyproject.toml` (PEP 621 static metadata)
- Switch from pip-compile to `uv` with PEP 735 dependency groups; commit `uv.lock`
- [Replace pylint/isort/... with ruff — only if actually done this repo]
- Update `tox.ini` to use `tox-uv` with `uv-venv-lock-runner`
- Update CI to use `astral-sh/setup-uv`; SHA-pin all actions
- [Add `python-semantic-release` + `release.yml` — library repos only]

## Removed

**Deleted files:** <list>

**Removed Makefile targets:**

| Target | Reason |
|---|---|
| `<target>` | <why it's gone> |

## Not included

<Anything the parent issue asks for that this repo doesn't need, and why —
e.g. "no release.yml, master had no PyPI publish workflow">

## Versioning

[Static|Dynamic] <how version is now determined, and why>

## Testing Notes

<What was actually verified: uv lock resolves, uv sync succeeds, which parts
could NOT be verified locally (e.g. native deps needing CI) and why>

## Code reviewer notes

<Anything a human reviewer should specifically double-check — e.g. new tox.ini
structure, CI matrix restructuring, ruff rule coverage vs. old pylint config>
```

## How much of this is actually proven (check before trusting blindly)

A skill existing, or being widely copied, isn't evidence it's correct — verify
outcomes, not that a PR merely opened or looks battle-tested.

- **Library-repo path (Phases 1-3): proven.** `openedx/XBlock` #914 merged with all
  CI green, and semantic-release then cut two real automated releases afterward
  (v6.2.0, v6.3.0) — the highest-risk part (OIDC publish, `build_command`,
  versioning) has actually run in production, not just been designed.
- **Application-repo path (Phases 1-2, no semantic-release): still unproven as of
  2026-07-24.** Re-checked: `enterprise-catalog` #1136, `xblock-sdk` #519,
  `acid-block` #282, `xblock-lti-consumer` #685, `DoneXBlock` #388 are all still open
  — no application/platform-repo PR in this effort has merged yet. Treat that branch
  as unverified until one merges with green CI.
- **The reference implementation itself isn't infallible — and the bug it seeded can
  outlive the PR review.** 5 PRs reviewed independently of ours (`xblock-sdk` #519,
  `acid-block` #282, `xblock-lti-consumer` #685, `DoneXBlock` #388, `RecommenderXBlock`
  #136) all inherited the same unpinned-`pypa/gh-action-pypi-publish@release/v1` bug
  from `sample-plugin`; one had a leftover duplicate publish workflow, one had the
  missing-tag gap. `RecommenderXBlock` #136 has since merged (2026-07-23) — its
  `release.yml` on `master` **still has the unpinned `@release/v1` ref today**,
  meaning this isn't just a review-time nitpick, it's a live risk sitting in
  production for that repo's next release. Check the parent issue's named reference
  implementation directly when in doubt, but verify it too — don't assume it's
  correct just because it's the source everyone copies from, and don't assume a
  known-bad pattern gets fixed just because a PR merges.

## Process notes — how this org-wide effort is tracked

- One parent issue (#506) defines the spec; work is split into per-repo-group child
  issues (e.g. #511 XBlocks, #513 Enterprise, #520 Webhooks/Other Services), each with
  a markdown checklist of repos.
- As PRs are opened, post them back as a comment on the child issue (either inline
  next to the checklist item, or as a consolidated "Created PRs" comment) — don't rely
  on GitHub's auto-linking alone, maintainers scan the issue for status.
  Check off the box once the PR is up (or once merged — conventions vary slightly by
  reporter, check how others in the same issue are marking theirs before choosing).
- A maintainer (e.g. FuaadZam) assigns a reviewer via an `@mention` comment on the
  child issue, separate from the actual PR review — check the child issue for a
  review assignment, not just the PR's reviewer field.
- Different teams working the same parent issue in parallel will independently
  reinvent very similar tooling. Before starting a new repo-group, check sibling child
  issues under the same parent for PRs already merged/opened — read one as a reference
  diff rather than starting from the parent issue text alone; it saves rediscovering
  the same edge cases (ruff-vs-pylint, app-vs-library detection, SHA-pinning) from
  scratch.

## Checklist (output at end)

```
Phase 1 - Metadata
[ ] pyproject.toml has [project] with static dependencies
[ ] setuptools-scm configured (no root setting unless subdirectory layout)  [library repos only]
[ ] license uses SPDX string format + license-files
[ ] [tool.uv] has package = true
[ ] [tool.setuptools.exclude-package-data] added (kebab-case) to exclude test files from wheel
[ ] Hardcoded __version__ removed from __init__.py; docs/conf.py updated to use importlib.metadata
[ ] fallback_version is a plain int-parseable string (e.g. "0.0.0"), never "0.0.0.dev0" or similar
[ ] Grepped codebase for other __version__ consumers beyond its own assignment, not just the __init__.py definition
[ ] fail_under only present in [tool.coverage.report] if original .coveragerc had one
[ ] setup.py deleted
[ ] setup.cfg deleted
[ ] Lint tooling (pylint etc.) left untouched — ruff only added if explicitly requested

Phase 2 - Dependency management
[ ] [dependency-groups] covers expected groups for repo type
[ ] [tool.edx_lint].uv_constraints added (even if empty), edx_lint write_uv_constraints run
[ ] uv.lock generated and committed (only regenerated if deps actually changed)
[ ] requirements/ directory deleted
[ ] tox.ini uses tox-uv>=1 and uv-venv-lock-runner
[ ] pyproject.toml: framework deps have lower bound, framework classifiers added
[ ] Makefile: requirements uses uv sync --group dev only (no uv tool install tox)
[ ] Makefile: lint/test/docs targets use uv run tox (not bare tox)
[ ] CI workflow: setup-uv pinned to SHA, no separate setup-python step
[ ] CI workflow: matrix job name is ${{ matrix.toxenv }}
[ ] CI workflow: jobs have permissions: contents: read
[ ] CI workflow: checkout step has fetch-depth: 0 (library repos, so setuptools-scm can see tags during test runs)
[ ] edx-platform constraint sync script added, if repo previously had one (Enterprise-style repos only)
[ ] release.yml (if Phase 3 applies): run_tests job has secrets: inherit and permissions: contents: read
[ ] Any restructured Makefile/tox aggregate target (e.g. test-quality) still runs the same set of sub-checks as before — diffed sub-target by sub-target, not just confirmed it still runs
[ ] tox.ini's dependency_groups mapping references group names that actually exist in this repo's own [dependency-groups] — not copied verbatim from a sibling repo
[ ] Any tox env newly added to envlist (docs, quality, etc.) is also added to the CI workflow's matrix in this same PR

Phase 3 - Semantic release (library repos only — skip for application repos)
[ ] Pre-flight: latest git tag (via --sort=-v:refname, not lexicographic) matches the actual latest version on PyPI
[ ] [tool.semantic_release] in pyproject.toml with build_command using python -m build
[ ] [tool.semantic_release]: major_on_zero = false and allow_zero_version = true set (all library repos, not just 0.x ones)
[ ] release.yml workflow added, targets OIDC (id-token: write, no user/password on publish step)
[ ] release.yml: pypa/gh-action-pypi-publish pinned to a commit SHA, not @release/v1
[ ] commitlint.yml workflow added
[ ] Searched all workflow files (not just the expected filename) for any other pre-existing PyPI-publish-triggering workflow, and deleted it
[ ] PyPI trusted publisher (OIDC) flagged as a pre-merge blocker in the PR description, not silently assumed

PR
[ ] PR description follows the template (Summary / Removed / Not included / Versioning / Testing Notes / Code reviewer notes)
[ ] PR links back to its child tracking issue, and the child issue's checklist/comment is updated
```
