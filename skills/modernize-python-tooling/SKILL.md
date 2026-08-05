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

Updated 2026-07-27 with a new Phase 0 (`src/` layout — confirmed in-scope decision
on #506 that this file was missing entirely) and two more confirmed lessons from
independently auditing `opaque-keys#461`, `ccx-keys#190`, `openedx-filters#381` — see
Phase 0 and the new Phase 2 lessons on `MANIFEST.in` and the
`upgrade-python-requirements.yml` reusable workflow.

Updated 2026-08-05 from a second review round on the #513 Enterprise track (reviewer
`salman2013`, the same person who wrote this skill's original 3-phase design).
**Two pieces of prior guidance are reversed, not just extended — read these even if
you've used this skill before:** the `pypa/gh-action-pypi-publish` pin now defaults
to the stable `@release/v1` tag, not a SHA (Phase 3); and Makefile targets must not
call `tox` at all, not even via `uv run tox` (Phase 2). Also added: a `codecov.yml`
threshold trap when `[tool.coverage.run]`'s `omit`/`branch=true` change shifts the
reported percentage, a `MANIFEST.in` `*.txt`-pattern false-positive, and a
`[tool.uv].default-groups = []` line reviewers keep mistaking for an empty
dependency group. See Phase 2/3 for detail on all of these.

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

### Phase 0 — Adopt `src/` layout (before Phase 1, library repos)

**Confirmed in-scope decision, not optional.** On 2026-07-15, `farhan` posted a
decision comment on #506 answering an explicit open question ("are we refactoring
all Python repos to use `src/` as the package root this cycle?"): **"Decision: `src/`
layout is in scope for this cycle. The precedent is already set in `xblock-core` and
`xblock-extras`. Folder structure will be reorganized and `pyproject.toml` updated
accordingly across repos."** The motivating reason (from `bradenmacdonald`, 2026-04-30
on the same issue): flat-layout editable installs of repos like `opaque-keys` aren't
resolvable by tools like mypy, while `src/`-layout repos work fine — see
[src-layout-vs-flat-layout](https://packaging.python.org/en/latest/discussions/src-layout-vs-flat-layout/).

**This file didn't mention `src/` layout at all until 2026-07-27** — it was missing
from three PRs done under this skill (`opaque-keys#461`, `ccx-keys#190`,
`openedx-filters#381`) before an independent audit caught it. Don't assume "the skill
covers everything the ticket asks for" — cross-check the parent issue's own comment
thread directly, especially for anything dated after this file's own "Updated" line.

**Verify the cited precedent before copying it** — the decision comment's own example
is unreliable. As of 2026-07-27: `openedx/XBlock` (xblock-core) does **not** have a
`src/` directory (flat layout: `xblock/`, `web_fragments/` at repo root) despite being
named as the precedent. `openedx/xblocks-extra` **does** have `src/`. Check whichever
repo you intend to copy from directly (`gh api repos/openedx/<repo>/contents`), don't
trust a decision comment's factual claim about another repo's current state.

**Also verify actual practice, not just the recorded decision** — as of 2026-07-27,
`openedx-events#590` (a sibling PR in this same effort, same author, still open) is
flat-layout, not `src/`. A dated decision existing on the tracking issue does not mean
it has been operationalized yet across the effort. If you're unsure whether `src/`
layout is still expected to be in scope by the time you're reading this, re-check
#506's comment thread for anything more recent than 2026-07-15 before assuming either
way — don't silently follow this file's guidance if the ticket has since moved on.

**How to do it, once confirmed in scope:**
1. Move the package directory (and any other importable top-level packages in the
   repo) into `src/<package_name>/` — e.g. `opaque_keys/` → `src/opaque_keys/`.
2. `pyproject.toml`: set `[tool.setuptools.packages.find] where = ["src"]`.
3. Update every other place that hardcodes the flat path: `MANIFEST.in`,
   `.coveragerc`/`[tool.coverage.run].source` (now `src/<package_name>`),
   `docs/conf.py` (`sys.path.insert` targets, if any), `tox.ini` if it references the
   package path directly, and any CI step that does e.g. `pylint <package_name>`
   instead of the tox/uv-run command that already resolves it correctly.
4. Verify locally: an editable install (`uv sync`) still makes the package importable
   from the repo root, `uv run mypy` (if configured) resolves cleanly, and the full
   test suite still passes — the whole point of this change is fixing mypy resolution
   under editable installs, so don't skip this specific check.
5. This is a substantial structural change on top of an otherwise mechanical
   migration — do it as its own commit (e.g. `refactor: move package to src/ layout`)
   separate from the Phase 1/2/3 commits, so it's easy for a reviewer to isolate.

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
   - `[tool.uv]`: once Phase 2's dependency groups exist, also set
     `default-groups = []` (with a comment explaining why) if every `uv sync`/
     `uv run` invocation in the repo already names an explicit `--group`. Without
     it, uv's implicit default group (named `dev`) gets synced *in addition* to
     whatever `--group` is passed, silently pulling the entire dev/test/quality/ci
     superset into every target — defeating the point of having separate groups at
     all. **A reviewer is likely to flag this line as "an empty dependency group" on
     sight** — it isn't one (it's a `[tool.uv]` setting, not a `[dependency-groups]`
     entry) — confirmed real on three separate #513 PRs (`edx-enterprise-data#693`,
     `enterprise-integrated-channels#180`, `enterprise-catalog#1136`), each
     independently flagged by the same reviewer. A comment directly above the line
     explaining the rationale (as above) heads this off; if asked anyway, that's the
     answer to give rather than removing or changing the setting.
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
   - **Do NOT also keep/re-add a `License ::` trove classifier alongside the
     SPDX `license` field — this is a hard, unconditional build failure on
     setuptools >=77, not a style choice.** Confirmed empirically (built a
     minimal package both ways): setuptools raises
     `InvalidConfigError: License classifiers have been superseded by license
     expressions (see https://peps.python.org/pep-0639/). Please remove: ...`
     the moment `pyproject.toml` has both `license = "<SPDX-expr>"` and a
     `"License :: OSI Approved :: ..."` entry in `classifiers`. A reviewer may
     ask you to "add the classifier back alongside the SPDX field for PyPI's
     classifier-based browsing" — that request cannot be satisfied as stated;
     the two are mutually exclusive under modern setuptools. Reply with this
     finding rather than complying, and don't assume "reviewer asked for it"
     means it's safe — verify by actually building the wheel.
   - `[tool.coverage.run]`: set `branch = true`, `source`, and `omit` patterns:
     ```toml
     omit = ["*/tests/*", "*/migrations/*", "*/__pycache__/*", "*/settings.py"]
     ```
   - `[tool.coverage.report]`: set `show_missing = true` and `exclude_lines`. Only add
     `fail_under` if the original `.coveragerc` already had one — **do not invent a
     threshold that didn't exist before**
   - `[tool.coverage.html]`: set `directory = "htmlcov"`
   - **If the repo has a `codecov.yml`, do not touch its coverage threshold as part of
     this migration — even implicitly.** Reviewer `salman2013` flagged this
     explicitly on two #513 PRs: "Please make sure we are not updating the code
     coverage threshold. It should match as before." The trap: the `omit`/
     `branch = true` change just above genuinely shifts the *reported* percentage
     (excluding test files' own trivially-self-covered statements, and/or counting
     partially-exercised branches) even though production-code coverage hasn't
     regressed at all — confirmed real on `edx-enterprise-data#693` (88.70% → 81.56%,
     a pure methodology change) and `taxonomy-connector#316` (98.91% → 98.70%). If
     this happens, don't silently add a `threshold:` buffer to `codecov.yml` to make
     the check pass again without explaining why — that reads exactly like "changed
     the threshold" from the reviewer's side even though the intent is compensating
     for a measurement-methodology shift, not loosening the real bar. Document the
     exact before/after percentages and the specific cause (which exclusion or
     `branch = true` produced the delta) directly in an inline comment in
     `codecov.yml` next to the change, so a reviewer can see it's compensating for a
     one-time reporting artifact and not a lowered bar.
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

   **Confirmed recurring bug — a shared group + `deps =` override is NOT a real
   version matrix.** A tox.ini pattern like this looks plausible and passes CI,
   but is broken:
   ```ini
   [testenv]
   dependency_groups =
       django42: test
       django52: test
   deps =
       django42: Django>=4.2,<4.3
       django52: Django>=5.2,<5.3
   ```
   Both factors sync the *same* `test` group — which `uv.lock` resolved against
   whatever Django version `test`/`[project].dependencies` actually pins (usually
   the newer one) — then `deps=` force-overrides just the `Django` package
   afterward via pip, on top of that already-locked environment. Every *other*
   transitive dependency stays resolved against the newer-Django-compatible
   graph, so the "django42" environment is a hybrid, not a real independent
   locked resolution — exactly the failure mode `[tool.uv].conflicts` exists to
   prevent. It reads as a working version matrix (CI is green, both envs run
   the test suite) but isn't testing what it claims to. **Confirmed present
   independently in `enterprise-subsidy#441`, `enterprise-access#1015`, and
   `openedx-ledger#242`** — three separate repos in the same effort, meaning
   this is a pattern an implementing agent will reach for by default unless
   explicitly warned off it. The fix: a real `django42` dependency-group
   (separate from `test`), `[tool.uv].conflicts` declaring the two mutually
   exclusive, `tox.ini` mapping each factor to its own group, and *no* `deps=`
   line at all. Verify the fix actually worked — don't just trust it — by
   checking `uv pip list | grep -i django` (or equivalent) inside each tox env
   to confirm the correct Django version is genuinely installed, and ideally
   confirm at least one transitive dependency actually diverges between the
   two locked resolutions (e.g. `django-filter` or `social-auth-app-django`
   at different versions) — if nothing diverges, the "fix" may not have taken.

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
6. Delete the `requirements/` directory. **Grep the whole repo for other references to
   deleted requirements files before considering this done** — `MANIFEST.in` in
   particular commonly has a stray `include requirements/base.in` /
   `include requirements/constraints.txt` line left over from packaging config that
   nothing else touches during this migration. Confirmed real:
   `ccx-keys#190` shipped with exactly this stale reference, caught only by an
   independent post-merge audit, not CI (it's not a hard build failure, just
   dead/wrong packaging metadata).

   **Don't over-apply this and strip `MANIFEST.in`'s per-package `*.txt` glob
   patterns too** (e.g. `recursive-include src/enterprise *.html ... *.txt`) — those
   are unrelated to the deleted `requirements/*.txt` files and package genuine
   runtime data files that ship inside the package itself (confirmed real:
   `src/enterprise/views_error_codes.txt` in `edx-enterprise#2654`). A reviewer may
   flag the `*.txt` pattern on sight assuming it's leftover pip-tools cruft — verify
   with `find src/<package> -name '*.txt'` whether any tracked file actually matches
   before touching it; if none currently do but the pattern predates this migration
   unchanged, leave it as a harmless no-op rather than pruning it as part of this PR.
7. **Check for a `.github/workflows/upgrade-python-requirements.yml` (or similarly
   named) caller of the org's shared reusable workflow
   (`openedx/.github/.github/workflows/upgrade-python-requirements.yml`) — if present,
   this migration silently breaks it and you cannot fix it from the individual repo.**
   That reusable workflow hardcodes `ADD_PATHS="requirements"` (plus
   `scripts/**/requirements*` if a `scripts/` dir exists) for its fork-friendly
   PR-creation step, with no `add_paths`-style input exposed in its `workflow_call`
   inputs — confirmed by reading the workflow source directly. Once `requirements/` is
   deleted, `make upgrade`'s real output (`uv.lock`/`pyproject.toml` changes) won't
   match any of the hardcoded `add-paths` globs, so the scheduled job will keep running
   but silently stop producing real dependency-upgrade PRs (empty diff, no output) —
   it won't fail, so nothing will alert anyone. Confirmed present and unaddressed in
   both `ccx-keys#190` and `openedx-filters#381`. Since you cannot patch a
   centrally-owned reusable workflow from a single repo's migration PR, **explicitly
   flag this as a known, out-of-scope gap in the PR description** (same treatment as
   the PyPI OIDC pre-merge-blocker) rather than silently leaving it — and consider
   raising the actual fix (parameterize `add_paths`, or auto-detect `uv.lock` vs.
   `requirements/`) as a separate issue/PR against `openedx/.github` once you've
   confirmed the same gap recurs across enough repos to justify a central fix.
8. Update `tox.ini`: add `requires = tox-uv>=1`; change runner to
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
9. Update `Makefile`:
   - `upgrade`: `uv run --with edx-lint edx_lint write_uv_constraints pyproject.toml`
     then `uv lock --upgrade` — no pre-install of edx-lint needed
   - `compile-requirements`: remove or replace with `uv lock`
   - `requirements`: `uv sync --group dev` only — do **NOT** add
     `uv tool install tox --with tox-uv` (installs an unpinned global tox outside
     `uv.lock`)
   - **Don't call `tox` from Makefile targets at all — not even via `uv run tox`.**
     (Reversed 2026-08-05 — an earlier version of this skill said the opposite, "use
     `uv run tox` everywhere.") Confirmed as a standing org decision, reiterated by
     reviewer `salman2013` across the #513 Enterprise track (`edx-enterprise#2654`,
     `enterprise-integrated-channels#180`, `taxonomy-connector#316`): "We decided to
     not use tox commands in make file." The reasoning: `tox` builds its own
     virtualenv per environment (even with `tox-uv`), which is redundant with the
     project's own uv-managed venv for tasks that don't need multiple environments —
     `make quality`/`make docs`/`make pii_check` shouldn't spin up a second
     environment just to run `pylint`/`sphinx-build`/`code_annotations`, which are
     already installed in the venv `uv sync` set up. Instead, inline each tox
     environment's actual `commands =` directly via `uv run <tool>` (and
     `uv sync --group <name>` first, for any group not already covered by `dev`) —
     `enterprise-catalog`'s Makefile is a working example of this pattern (its `test`
     target is `uv run python3 -Wd -m pytest`, no tox involved at all). `tox.ini`
     itself is untouched by this — CI's own matrix testing still calls `tox` (or
     `uv run tox`) directly, this only affects the Makefile layer that a contributor
     runs locally. A target that inherently needs tox's actual parametrized matrix
     (e.g. a `test-all` that ran every Django version locally) has no direct
     replacement — either drop it to the single default-env command and rely on CI
     for the matrix, or leave it calling tox if the repo owner wants to keep true
     local matrix testing; check with them rather than guessing.
   - If you restructure any aggregate target (e.g. `test-quality: test-lint
     test-codestyle test-mypy test-format`), diff it against its pre-migration
     definition sub-target by sub-target — a rename/restructure can silently drop one
     with CI still green under the new name. Confirmed real: `forum` PR #283
     restructured `test-quality` down to `test-lint test-codestyle test-mypy`,
     silently dropping `test-format` (`black --check`) from the CI quality gate — not
     disclosed in the PR description, not caught by CI.
10. Update CI workflow(s):
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
     with `git_committer_name`, `git_committer_email`; upload
     `dist/` as artifact; expose `released` + `version` outputs;
     `permissions: contents: write` only (NOT `id-token`)
   - **Do NOT set `changelog: "false"`.** Earlier drafts of this skill set this because
     `openedx/sample-plugin`'s reference `release.yml` does — but there is no ticket
     (#506, or any child issue) that actually asks for changelog generation to be
     disabled; it was inherited from the reference template with no explicit
     justification. `CHANGELOG.rst` is genuinely useful historical documentation of
     what changed release-to-release — disabling changelog generation (or, worse,
     deleting the file — see below) throws that away for no technical reason. Enable
     it and configure it to update the *existing* RST file in place:
     ```toml
     [tool.semantic_release.changelog]
     mode = "update"
     insertion_flag = ".. changelog-insertion-marker"

     [tool.semantic_release.changelog.default_templates]
     changelog_file = "CHANGELOG.rst"
     output_format = "rst"
     ```
     Add the literal line `.. changelog-insertion-marker` at the very top of the
     existing `CHANGELOG.rst` (above all historical entries) — PSR's `update` mode
     looks for this exact string and inserts new version sections directly above it
     on each release, leaving everything below (the old, hand-written history)
     completely untouched. Verified empirically (`opaque-keys`, prototyped locally):
     produces a properly-formatted RST `Unreleased` section, grouped by
     conventional-commit type (Bug Fixes / Features / Chores / etc.), each entry
     linked to its commit SHA and PR number — and as a bonus, PSR also backfills a
     changelog section for the most recent *actual* tag if one is missing, computed
     from the commits between the two latest tags.
   - **If `CHANGELOG.rst` doesn't exist yet**, create one containing only the
     `.. changelog-insertion-marker` line — PSR will build everything above it from
     there. **If a previous pass of this migration already deleted `CHANGELOG.rst`**,
     restore it from git history (`git show <last-commit-before-deletion>:CHANGELOG.rst`)
     rather than starting fresh, so the historical record isn't lost. This is not a
     one-off risk — confirmed real on 4 separate #513 repos (`edx-enterprise#2654`,
     `edx-enterprise-data#693`, `enterprise-integrated-channels#180`,
     `taxonomy-connector#316`), each needing a dedicated follow-up commit to restore
     deleted entries after `salman2013` flagged it in review ("I don't think we
     should remove the old change release logs"). Treat "did this migration's diff
     remove any existing `CHANGELOG.rst` line" as something to check explicitly
     before opening the PR, not just something to react to once a reviewer notices.
   - **Verify `tag_format` matches this repo's actual tag convention before trusting
     PSR's computed next-version.** PSR's default is `"v{version}"`; several repos in
     this effort use bare version tags (`4.0.0`, not `v4.0.0`). If `tag_format` doesn't
     match, PSR silently fails to recognize any existing tag as a prior release and
     computes the next version as if there were none (confirmed: showed `0.1.0` as the
     "next version" for a repo actually on `4.0.0`, until `tag_format = "{version}"`
     was set to match the real tag style) — check `git tag --sort=-v:refname | head`
     against what's already configured (or absent) rather than assuming the default
     is correct.
   - **Whether to re-add `CHANGELOG.rst` to the `readme` field's PyPI-rendered content
     is a separate decision, not required for changelog generation to work.** The
     original reasoning for removing it from `readme = {file = [...]}` was to stop
     PyPI from showing an increasingly stale changelog once nobody was updating it
     manually — that reasoning no longer applies once semantic-release keeps it
     current automatically, but re-adding it is a judgment call for whoever owns the
     repo, not something to default to silently either way.
   - `publish_to_pypi` job (`needs: release`, `if: released == 'true'`): download
     artifact, publish via OIDC (`id-token: write`, no `user`/`password` inputs on the
     publish step — OIDC is used automatically once the permission is granted and no
     explicit credentials are passed)
   - **Do NOT SHA-pin `python-semantic-release/python-semantic-release`,
     `python-semantic-release/publish-action`, `actions/upload-artifact`, or
     `actions/download-artifact` in `release.yml` — use plain version tags
     (`@v10.6.1`, `@v7`, `@v8`) matching `openedx/XBlock`'s actual, currently-in-
     -production `release.yml` exactly.** This directly contradicts Phase 2's
     general "SHA-pin all actions" rule, but `release.yml` only runs on push to
     the default branch — never during PR CI — so a bad pin here is invisible
     until the very first real release attempt, and there is no automated check
     to catch it. Confirmed real and severe: independently auditing 5 sibling
     PRs (`opaque-keys#461`, `ccx-keys#190`, `openedx-filters#381`,
     `openedx-core#704`, `openedx-calc#217`) that each attempted to SHA-pin these
     4 actions anyway found **3 of 5 had at least one completely fabricated or
     mismatched SHA** — e.g. `opaque-keys`/`openedx-core` pinned
     `actions/upload-artifact` to a SHA that doesn't exist in that repo at all
     (verified via `gh api repos/actions/upload-artifact/commits/<sha>` → "No
     commit found"), and the *same* invalid SHA appeared, mislabeled, as
     `download-artifact`'s pin — i.e. the SHAs were swapped between two
     different actions' repos, not just wrong versions. Verifying every SHA
     against `gh api repos/<owner>/<repo>/commits/<sha>` for every action pin
     you add is possible but expensive and error-prone to do reliably by hand;
     since the actual working reference implementation doesn't pin these at
     all, don't introduce the risk. This now includes `pypa/gh-action-pypi-publish`
     too — see below, this skill previously recommended SHA-pinning it specifically,
     that guidance has been reversed.
   - **Use the stable `pypa/gh-action-pypi-publish@release/v1` tag — do NOT SHA-pin
     it.** (Reversed 2026-08-05 — this skill previously said the opposite. Read this
     whole note before picking either way, the history matters.) An earlier version
     of this skill recommended SHA-pinning this action, citing a real incident
     (`openedx/xblocks-core`'s 2026-07-14 release run failed with
     `docker: manifest unknown`, because the unpinned `@release/v1` ref had resolved
     to a commit whose Docker image wasn't published on ghcr.io yet). That incident
     was real, but pinning it introduced a *different*, confirmed, repeated
     production failure: across the #513 Enterprise track, reviewer `salman2013`
     independently required reverting a SHA-pinned `pypa/gh-action-pypi-publish` back
     to `@release/v1` on every single PR he reviewed (`edx-enterprise#2654`,
     `edx-enterprise-data#693`, `enterprise-integrated-channels#180`,
     `taxonomy-connector#316`), stating a hash-pinned version had broken PyPI
     publishing for the org before. Two confirmed incidents pointing opposite
     directions for the same action; the second is more recent, more repeated (4
     independent confirmations vs. 1), and comes directly from someone who has
     actually operated this exact publish step in production — so it's the current
     default. It's plausible the xblocks-core failure was actually caused by a bad
     *value* (a stale or mismatched SHA — this skill elsewhere documents 3 of 5
     audited PRs shipping a fabricated or swapped SHA for other actions in this same
     file) rather than SHA-pinning being unsafe in general, but that has not been
     verified against the actual xblocks-core failure logs. If you hit a PyPI-publish
     failure on either pin style, check the actual failure mode before assuming
     either side of this note is still correct — this has already flipped once.
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
4. **A version-matrix env shares a dependency-group with another factor instead of
   getting its own.** See the "Confirmed recurring bug" callout under Phase 2 —
   `dependency_groups = { django42: test, django52: test }` plus a `deps=` override
   looks like a real matrix and passes CI, but only one Django version is ever
   actually locked. Confirmed in 3 separate repos in the same effort
   (`enterprise-subsidy#441`, `enterprise-access#1015`, `openedx-ledger#242`).
5. **release.yml's action pins are unverifiable by CI, so verify them by hand.**
   `release.yml` only runs on push to the default branch, never PR CI — a bad SHA
   pin here is invisible until the first real release attempt. Confirmed severe:
   independently auditing 5 sibling PRs found 3 with a completely fabricated or
   swapped SHA for `python-semantic-release`/`upload-artifact`/`download-artifact`.
   Check every pin with `gh api repos/<owner>/<repo>/commits/<sha>` — see the
   detailed guidance under Phase 3, step 2.

None of these fail CI — that's exactly why they need a deliberate line-by-line check,
not a "tests pass" glance. As feanil put it reviewing a sibling PR in this same
effort (`mockprock` #66): "AI will not pick up when we change user behavior or
semantics without a lot of pre-definition by us... don't assume that the AI review is
sufficient."

---

## Handling reviewer feedback on an open PR

Confirmed lessons from a real second-round review (four Enterprise-group PRs,
reviewer `farhan`, 2026-07-2x):

1. **A reply saying "Fixed with commit X" doesn't mean it's on the PR.** A
   reviewer without push access may prototype a fix on their own personal fork
   and link that commit in a reply — that commit is not reachable from the
   actual PR branch and was never merged/cherry-picked. Don't take the reply's
   word for it: check out the actual PR head branch and read the real current
   file content. In the one case this happened, the substance turned out to be
   fixed anyway (via an independent commit on the real branch, same day) — but
   that was only established by direct inspection, not by trusting the comment
   thread.
2. **Verify a reviewer's suggested fix will actually do what they say before
   applying it verbatim — a plausible-sounding suggestion can be factually
   wrong, or right in principle but wrong as literally written.** Three
   confirmed real examples from one review round:
   - A comment claimed `--cov <pkg-name>` "may not resolve correctly" under a
     `src/` layout. Verified empirically (built an isolated src-layout test
     package) that pytest-cov/coverage.py resolve `--cov` by installed import
     name, not filesystem path — the claim was simply wrong. No change needed;
     replied with the evidence instead of complying.
   - A comment asked for `uv tool install tox --with tox-uv` to fix "tox not
     available on a fresh checkout." Adding it introduced a real regression:
     an unpinned, un-lockfiled tox living outside `uv.lock`, contradicting the
     repo's own established `uv run tox` pattern used everywhere else. It had
     already been added once (from round one) — the fix was to revert it.
   - A comment suggested adding `&& matrix.python-version == '3.12'` to a
     Codecov upload condition to prevent double-uploads if a second Python
     version were ever added. Applied literally, this would have permanently
     disabled Codecov — the matrix in that repo had no `python-version` key at
     all, only `toxenv`, so the condition could never be true. The real fix
     required adding the missing matrix dimension *first*, then the guard.
   In all three cases the mistake would have shipped clean-looking code that
   either did nothing or actively regressed something, and CI would not have
   caught it. Read the actual current file/config before writing the fix, and
   sanity-check that every field/key a suggested snippet references genuinely
   exists in this repo's config.
3. **A long-open migration PR can silently stop running CI entirely.** If
   `main` moves in the meantime (a real release, a routine dependency-bump PR),
   the migration branch can drift into `mergeable: CONFLICTING` — and GitHub
   does not run `pull_request`-triggered workflows at all once a PR has
   conflicts, so pushes silently produce zero check runs (not failures —
   nothing). Don't assume "no CI activity" means "nothing changed and it's
   still fine" on a PR that's been open more than a few days; check
   `gh pr view --json mergeable,mergeStateStatus` before trusting old green
   checks are still current. Resolve with a **merge**, not a rebase, if the PR
   already has review comments anchored to specific lines/commits — a rebase
   rewrites every commit SHA and risks detaching those anchors; a merge commit
   keeps them intact and only needs a normal (non-force) push. For deleted
   files the migration removed (`requirements/*.txt`, `CHANGELOG.rst`) that
   `main` happens to have also touched since, resolve modify/delete conflicts
   by keeping them deleted — that reflects the migration's actual intent.
4. **If you're mid-fix on a repo and a coordinating message reports another
   agent pushed new work to the same branch, refresh before you push.** Two
   agents (or a coordinator and an agent) can legitimately be working the same
   branch across a conflict-resolution handoff; blindly pushing from a stale
   local clone risks silently reverting the other's just-pushed commit. Always
   `git fetch` and diff against the actual current remote HEAD immediately
   before your final push, not just at the start of the session.

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

---

_This PR was created with Claude Code._
```

Put the "created with Claude Code" note at the very end of the description, not as a
top banner — don't imply human review happened unless it actually did.

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
- **The reference implementation's choices should be understood, not blindly
  copied.** 5 PRs reviewed independently of ours (`xblock-sdk` #519, `acid-block`
  #282, `xblock-lti-consumer` #685, `DoneXBlock` #388, `RecommenderXBlock` #136) all
  inherited `pypa/gh-action-pypi-publish@release/v1` unexamined from `sample-plugin`
  — which, per the reversed guidance above, has turned out fine on its own, but was
  copied without anyone confirming *why* `sample-plugin` made that choice, at a time
  when this skill was actively recommending the opposite. Two of the five also had
  separate, still-real bugs unrelated to the pin question: one had a leftover
  duplicate publish workflow (see Phase 3, "search broadly" note), one had the
  missing-tag gap (see Phase 1's `fallback_version`/tag pre-flight note). Check the
  parent issue's named reference implementation directly when in doubt, but verify
  it too — don't assume it's correct just because it's the source everyone copies
  from, and don't assume a given repo's copy is bug-free just because a PR merges.

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
Phase 0 - src/ layout (library repos, before Phase 1 — check #506 for current status first)
[ ] Confirmed src/ layout is still in scope per #506's comment thread (not just trusting this file)
[ ] Package(s) moved to src/<package_name>/, [tool.setuptools.packages.find] where = ["src"]
[ ] MANIFEST.in, coverage source path, docs/conf.py, tox.ini, CI steps updated for the new path
[ ] Editable install + mypy (if configured) + full test suite verified under the new layout
[ ] Done as its own commit, separate from Phase 1/2/3

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
[ ] codecov.yml coverage threshold left untouched unless the omit/branch=true change genuinely shifted the reported percentage — and if so, the shift is documented inline with before/after numbers, not silently absorbed
[ ] setup.py deleted
[ ] setup.cfg deleted
[ ] Lint tooling (pylint etc.) left untouched — ruff only added if explicitly requested

Phase 2 - Dependency management
[ ] [dependency-groups] covers expected groups for repo type
[ ] [tool.edx_lint].uv_constraints added (even if empty), edx_lint write_uv_constraints run
[ ] uv.lock generated and committed (only regenerated if deps actually changed)
[ ] requirements/ directory deleted
[ ] Grepped for stale references to deleted requirements files (MANIFEST.in especially)
[ ] MANIFEST.in's per-package *.txt (or other non-.py) glob patterns left alone unless verified genuinely unused — not stripped just because they look like requirements-file leftovers
[ ] [tool.uv].default-groups = [] set (with an explanatory comment) if every uv sync/uv run invocation in this repo already names an explicit --group
[ ] Checked for a caller of openedx/.github's upgrade-python-requirements.yml reusable workflow; if present, flagged in the PR description as a known gap (its hardcoded add-paths can't be fixed from this repo)
[ ] tox.ini uses tox-uv>=1 and uv-venv-lock-runner
[ ] pyproject.toml: framework deps have lower bound, framework classifiers added
[ ] Makefile: requirements uses uv sync --group dev only (no uv tool install tox)
[ ] Makefile: lint/test/docs targets do NOT call tox at all (not even via uv run tox) — inlined as direct uv run <tool> commands instead (reversed 2026-08-05 — see Phase 2 note)
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
[ ] release.yml: pypa/gh-action-pypi-publish uses the stable @release/v1 tag, NOT a commit SHA (reversed 2026-08-05 — see Phase 3 note)
[ ] commitlint.yml workflow added
[ ] Searched all workflow files (not just the expected filename) for any other pre-existing PyPI-publish-triggering workflow, and deleted it
[ ] PyPI trusted publisher (OIDC) flagged as a pre-merge blocker in the PR description, not silently assumed
[ ] changelog: "false" NOT set in release.yml's PSR step -- [tool.semantic_release.changelog] configured instead (mode="update", insertion_flag, RST output matching the existing CHANGELOG.rst)
[ ] CHANGELOG.rst exists with the insertion_flag marker at the top -- restored from git history if a prior pass deleted it, created fresh (marker only) if it never existed, otherwise just the marker added above existing history
[ ] tag_format verified against this repo's actual git tag convention (bare "X.Y.Z" vs "vX.Y.Z"), not left at PSR's untested default

PR
[ ] PR description follows the template (Summary / Removed / Not included / Versioning / Testing Notes / Code reviewer notes)
[ ] PR links back to its child tracking issue, and the child issue's checklist/comment is updated
```
