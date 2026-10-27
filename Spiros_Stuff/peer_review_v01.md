# Peer Review of `deid` (v0.4.13) - version 01

A contributor-oriented review of the `pydicom/deid` codebase, written from the
`spiroganas/deid` fork at commit `df5450f` ("Fix repeated field extraction and
parser cache invalidation (#307)").

The goal of this document is not just to list bugs. It is to answer the
question: *"What can I work on in this repository, in what order, so that each
pull request teaches me something new and moves me from first-time contributor
toward senior-engineer-level contributions?"*

Everything below was verified by actually installing the package, running the
test suite, measuring coverage, running linters and type checkers, and writing
small scripts to reproduce suspected bugs. The exact commands are in the
Appendix so you can reproduce every number.

---

## Table of contents

1. [Repository at a glance](#1-repository-at-a-glance)
2. [What I measured](#2-what-i-measured)
3. [First-time contributor changes](#3-first-time-contributor-changes)
4. [Beginner changes (small code fixes with a test)](#4-beginner-changes-small-code-fixes-with-a-test)
5. [Intermediate changes (behavioural bugs and refactors)](#5-intermediate-changes-behavioural-bugs-and-refactors)
6. [Advanced / senior-level changes](#6-advanced--senior-level-changes)
7. [Modern Python tooling assessment](#7-modern-python-tooling-assessment)
8. [Test coverage assessment](#8-test-coverage-assessment)
9. [CI/CD and release process](#9-cicd-and-release-process)
10. [Code quality and architecture observations](#10-code-quality-and-architecture-observations)
11. [Security and robustness notes](#11-security-and-robustness-notes)
12. [How to work with this project](#12-how-to-work-with-this-project)
13. [A learning roadmap: from first PR to senior-level contributor](#13-a-learning-roadmap-from-first-pr-to-senior-level-contributor)
14. [Appendix: commands and raw results](#14-appendix-commands-and-raw-results)

---

## 1. Repository at a glance

| Item | Observation |
|---|---|
| Purpose | "Best effort" de-identification of DICOM medical images: header scrubbing driven by a small recipe language, plus pixel-region blanking for burned-in text. Mirrors the CTP (RSNA Clinical Trial Processor) approach. |
| Upstream | `pydicom/deid` (part of the pydicom family). Your fork is `spiroganas/deid`. PRs ultimately go to upstream `master`. |
| Lead maintainer | Vanessa Sochat (`@vsoch`). `CONTRIBUTING.md` says the final merge is done or approved by her. |
| Size | ~5,200 lines of library code across 22 modules; ~7,600 lines of tests across 28 test files. |
| Language level | Python 3 only. `setup.py` claims `>=3.7`, but the code uses `functools.cache` (3.9+) and depends on `pydicom>=3.0` (which itself requires 3.10+). Effective minimum is **3.10**. |
| Runtime dependencies | `matplotlib`, `numpy>=1.20`, `pydicom>=3.0,<4`, `python-dateutil`. |
| Test data | Comes from a separate package `deid-data` (installed from PyPI in CI). |
| Activity | Actively maintained. 35 commits since January 2025; the most recent commit was three days before this review. Recent work is mostly correctness fixes to the header parser and performance work. |
| License | MIT |
| Docs | Jekyll site under `docs/` (published to GitHub Pages) plus Sphinx API docs under `docs/api_docs/`. |

**Overall impression.** This is a healthy, actively maintained scientific
Python project with a real user base, a thorough regression-test suite for its
core feature (header actions), and a maintainer who merges community PRs. Its
tooling, packaging and CI are circa 2019-2022 and have drifted: several config
files are stale or subtly broken. That drift is exactly where a newcomer can
contribute safely.

---

## 2. What I measured

All measurements were made on Python 3.13.17 in a fresh virtual environment
created with `uv`, with the package installed in editable mode.

### Test suite

| Metric | Value |
|---|---|
| Runner used by the project | `python -m unittest discover` (CI) |
| Also works with | `pytest` (no config needed) |
| Result | **274 passed, 1 skipped, 17 subtests passed, 17 warnings** in ~35 s |
| Skipped test | `deid/tests/test_dicom_pixels.py:287` - skipped with a long docstring describing a suspected bug (see §5.1) |
| Disabled test file | `deid/tests/Xtest_dicom_header.py` - renamed so the runner ignores it; imports a function that no longer exists |
| Test style | 100% `unittest.TestCase`; 0 files use pytest fixtures or parametrize; 238 `print()` calls in test bodies |

### Coverage (library code only, branch coverage on)

| Metric | Value |
|---|---|
| Overall | **72%** (2,175 statements, 529 missed; 970 branches, 148 partial) |
| Configured threshold (`setup.cfg`) | `fail_under = 50` |
| Is coverage run in CI? | **No.** The coverage config exists but no workflow invokes it. |

Per-module numbers are in §8.

### Static analysis

| Tool | Configuration used | Result |
|---|---|---|
| `flake8` | the repo's `setup.cfg` (ignores `E1 E2 E5 W5`, max line 100) | 0 findings (the ignore list is very permissive) |
| `black --check` (23.3.0, the pinned version) | the repo's `pyproject.toml` | **Formats nothing** - "No Python files are present to be formatted". The `exclude` setting is broken (see §3.6). |
| `isort --check-only` | the repo's `pyproject.toml` | clean |
| `ruff check --select E,F` | defaults (88-char lines) | 129 findings: 96 `E501` line-too-long, 33 `F401` unused-import (29 are intentional re-exports in `__init__.py` files; 3 are genuinely unused in `deid/main/__init__.py`) |
| `ruff check` (broad rule set) | - | 310 findings; the top categories are 134 `UP031` printf-style formatting, 36 `B018` useless expressions (stray string literals), 11 `PLW0604` `global` at module level, 10 `SIM103` needless bool, 6 `BLE001` blind `except Exception` |
| `ruff format --check` | defaults | 2 of 60 files would change |
| `mypy --ignore-missing-imports` | none in repo | 18 errors in 3 files (11 from monkey-patching `pydicom.Dataset` in `filter.py`, 6 in `detect.py`, 1 in `clean.py`) |
| `codespell` | the repo's ignore list | 10 hits (listed in §3.2) |

---

## 3. First-time contributor changes

These are safe, small, and self-contained. Each one is a complete PR on its
own. They teach you the mechanics: fork, branch, run the tests, run
`pre-commit`, write a clear PR description using the repo's template, respond
to review.

A good first-timer rule: **one topic per PR**. Do not bundle "fix five typos
and delete Travis config and add a test" into one PR; reviewers much prefer
three tiny PRs.

### 3.1 Documentation and grammar fixes

| # | File | Problem | Suggested change |
|---|---|---|---|
| 1 | `README.md` | Build badge points to `travis-ci.org`, which has been shut down. The project uses GitHub Actions. | Replace with the GitHub Actions badge for `.github/workflows/main.yaml`. |
| 2 | `README.md` | `pip install git+git://github.com/pydicom/deid` - GitHub disabled the unauthenticated `git://` protocol in 2022, so this command fails. | `pip install git+https://github.com/pydicom/deid`. |
| 3 | `README.md` | No "Development setup" or "Running the tests" section. A newcomer has to discover `.github/dev-requirements.txt` and `deid-data` by reading CI YAML. | Add a short section: clone, `pip install -e .`, `pip install deid-data pytest`, `pytest deid/tests`, `pre-commit install`. |
| 4 | `CONTRIBUTING.md` line 5-6 | "it should be first discussed with the project developers first" - duplicated "first". | Delete one. |
| 5 | `CONTRIBUTING.md` line 29 | "the final merge must be done by the Singularity project lead" - copy-pasted from the Singularity project. | "deid project lead". |
| 6 | `CONTRIBUTING.md` line 90 | Code of Conduct enforcement contact is `gmkurtzer@gmail.com`, which is the Singularity project's lead, not this project's. | Ask the maintainer what contact they want here; this is a one-line change but needs their input. |
| 7 | `docs/_docs/install/local.md` | Recommends `python setup.py install`, which is deprecated by setuptools and prints warnings. The version example `deid==0.1.19` is very old. | `pip install .` (or `pip install -e .` for development); update example version. |
| 8 | `docs/_docs/development/index.md` line 14 | Links to `/development/add-format/`, but the page file is `development/image-format.md` with no explicit permalink. Verify by building the Jekyll site; the link is probably broken. | Fix the link or add a `permalink:` to the page. |
| 9 | `docs/_docs/contributing/code.md` | Says "format your code with black" but does not link the linting page that explains `pre-commit`. | Add the cross-link. |
| 10 | `setup.py` classifiers | Lists only `Python :: 3.8`. The code cannot run on 3.8. | List 3.10 through 3.13 and fix `python_requires` (see §4.9). |

### 3.2 Spelling (found with `codespell`)

| File:line | Typo | Fix |
|---|---|---|
| `CHANGELOG.md:44` | "unecessary" | "unnecessary" |
| `deid/dicom/actions/uids.py:23` | "orignal" | "original" |
| `docs/_docs/user-docs/recipe-headers.md:255` | "supercede", "supercedes" | "supersede", "supersedes" |
| `docs/_docs/getting-started/dicom-pixels.md:107` | "implementeation" | "implementation" |
| `deid/config/utils.py:49` | `bot.exis(` | This one is a **bug**, not just a typo - see §4.1 |
| `deid/data/deid.dicom.xray.chest:22-24` | "EXTREMETY" | **Do not change blindly.** This is a filter value matched against real-world DICOM headers, and the misspelling may be deliberate to match vendor data. Ask the maintainer. |

Docstring and comment typos not caught by codespell (all trivial):

- `deid/dicom/filter.py:29` "filer_name" -> "filter_name"
- `deid/dicom/filter.py:225` and `:236` "the the"
- `deid/dicom/parser.py:152` "If if's a uid" -> "If it's a uid"
- `deid/dicom/utils.py:81` "pydicon" -> "pydicom"
- `deid/dicom/pixels/detect.py:24` "repor" -> "report"
- `deid/logger/message.py:60` "asci" -> "ANSI"
- `deid/dicom/tags.py:19` "take a string for a tag (e.g., )" - empty example
- `deid/dicom/actions/jitter.py:39` "Jitter bug!" - stray note in a docstring
- `deid/dicom/fields.py:292` "STOPPED HERE NEED TO ADDRESS EMPTY NAME" - leftover work-in-progress comment

### 3.3 Dead or stale configuration files

| File | Why it is stale | Action |
|---|---|---|
| `.travis.yml` | Tests Python 2.7 and 3.5; Travis is not used. | Delete. |
| `.pylintrc` (15 KB) | pylint is not in `dev-requirements.txt`, not in `pre-commit`, not in CI. | Delete, or wire it into pre-commit (ask maintainer which). |
| `.github/workflows/codespell.yaml` | Uses `crate-ci/typos` pinned to a 2021 commit ("version 1.0.4") and `actions/checkout@v3` while `main.yaml` uses `@v4`. | Bump both. |
| `.pre-commit-config.yaml` | `pre-commit-hooks` rev `v4.3.0` (2022). | Run `pre-commit autoupdate`, commit the result, confirm hooks still pass. |
| `.gitignore` | Missing `.coverage`, `htmlcov/`, `.pytest_cache/`, `.ruff_cache/`, `.mypy_cache/`, `.venv/`. | Add them. |
| `MANIFEST.in` | `prune *OLD` refers to directories that no longer exist. | Remove the line. |

### 3.4 Remove `print()` noise from the test suite

There are 238 `print()` calls inside test bodies (banners like
`######################START######################`, and "Case 1: ..." labels).
They predate pytest's `-v` and `subTest`. Converting the "Case N" prints into
`with self.subTest("Case 1: known tag"):` blocks, or just deleting the banners,
makes the test output readable. This is mechanical, touches only tests, and is
easy to review. Do it one file per PR, or a handful of files per PR.

### 3.5 Delete the disabled test file

`deid/tests/Xtest_dicom_header.py` has been renamed so the discovery pattern
`[t|T]est*.py` ignores it. It imports `get_fields` from `deid.dicom.header`,
which no longer exists, so it cannot be re-enabled as-is. Either delete it, or
(slightly harder) port its three test ideas onto the current API in
`test_dicom_fields.py`. Ask in the PR which the maintainer prefers.

### 3.6 Fix the broken `black` configuration in `pyproject.toml`

```toml
[tool.black]
profile = "black"      # <- not a black option (it is an isort option)
exclude = ["^env/"]    # <- black expects a regex *string*, not a list
```

With black 23.3.0 (the version pinned in `dev-requirements.txt`), the list is
stringified to `['^env/']`, which as a regex is a character class that matches
almost every path. Result: `black deid` reports "No Python files are present to
be formatted" and formats nothing. The pre-commit hook still works only because
pre-commit passes explicit file names, which bypass `exclude`.

Fix: delete `profile`, and make `exclude = '^env/'` (a string). Verify with
`black --check --verbose deid` before and after.

### 3.7 Add unit tests for untested helper functions

These functions have zero or near-zero coverage, are pure (no I/O, no DICOM
file needed), and have obvious expected behaviour. Each is a 10-30 line test.
This is the single best way to learn the codebase while producing something
the maintainer will happily merge.

| Target | Currently covered? | What to test |
|---|---|---|
| `deid/utils/fileio.py` `to_int`, `is_number` | no | ints, floats, numeric strings, booleans |
| `deid/utils/fileio.py` `write_json` / `read_json(ordered_dict=True)` | partly | round trip through a temp file; `OrderedDict` type when requested |
| `deid/utils/actions.py` `parse_keyvalue_pairs` | partly | `"a=1 b=true c=null"`, pairs without `=`, empty string, `None` |
| `deid/utils/actions.py` `convert_value` | partly | each VR family (`FL`, `US`, `OB`), and the "cannot cast returns None" path |
| `deid/logger/message.py` `convert2boolean`, `get_logging_level` | no | every string level name, numeric strings, invalid values |
| `deid/dicom/tags.py` `find_tag`, `_filter_tags`, `update_tag` | no (module is 32%) | search by regex, filter by VR/VM, update an existing vs missing field |
| `deid/dicom/filter.py` `equals`, `notEquals`, `empty`, `missing`, `startsWith`, `endsWith`, `apply_filter` | partly (47%) | each branch of `apply_filter`, including the "invalid filter name returns False" path |
| `deid/dicom/validate.py` `validate_dicoms` | partly | a non-DICOM file is skipped with a warning |
| `deid/dicom/actions/uids.py` `basic_uuid`, `dicom_uuid` (64-char limit) | partly | output length, prefix handling |
| `deid/logger/progress.py` | 20% | `ProgressBar` with `hide=True` (so nothing is written), `bar()` iterating a list |

See §8 for the full coverage table.

---

## 4. Beginner changes (small code fixes with a test)

Each of these is a confirmed defect that can be fixed in a few lines. The
skill being practised here is **"reproduce, fix, prove with a regression
test"**: write the failing test first, then make it pass. Mention the
reproduction in the PR description.

### 4.1 `bot.exis` typo raises `AttributeError` - `deid/config/utils.py:49`

```python
if found_format != next_deid["format"]:
    bot.exis("Mismatch in deid formats, %s and %s" % (...))
```

`DeidMessage` has `exit`, not `exis`. When two recipes with different formats
are combined the user gets an `AttributeError` instead of the intended error
message. Today only one format (`dicom`) exists, so this path is hard to reach,
but the test can monkeypatch a loaded recipe dict's `format` key.

### 4.2 `is_quiet()` logic is inverted - `deid/logger/message.py:272-276`

```python
def is_quiet(self):
    """is_quiet returns true if the level is under 1"""
    if self.level < 1:
        return False
    return True
```

The docstring says one thing and the code does the opposite. Verified: at
level 5 (DEBUG) it returns `True`. Nothing in the repo calls it, so fixing it
is safe. Add a test.

### 4.3 Stray debug `print` - `deid/utils/actions.py:53`

`print(extras)` runs every time a `deid_func:` value is evaluated, writing to
stdout from library code. Remove it (or demote to `bot.debug`). Test: capture
stdout while calling `parse_value` with a `deid_func:` and assert it is empty.

### 4.4 No-op string replace - `deid/dicom/actions/uids.py:53`

```python
prefix = field.lower().replace(" ", " ")   # replaces a space with a space
```

Verified output for field name "Patient Name": `patient name-<uuid>`. The
intent was almost certainly `replace(" ", "-")` or `replace(" ", "")`. Check
the documented example in `docs/_docs/examples/func-replace.md` to pick the
intended one, then add a test.

### 4.5 `bar()` ignores its `empty_char` / `filled_char` arguments - `deid/logger/progress.py:146-147`

The function accepts the arguments and then passes the module constants
`BAR_EMPTY_CHAR` / `BAR_FILLED_CHAR` to `ProgressBar` instead.

### 4.6 Duplicate branch in `get_logging_level` - `deid/logger/message.py:331-332`

`elif level == "LOG"` appears twice; the second is unreachable. Also `"CRITICAL"`
maps to `FLAG`, which looks accidental. Tidy and test the string-to-level
mapping.

### 4.7 Dead exception handler and unused imports - `deid/main/__init__.py`

```python
try:
    args = parser.parse_args()
except Exception:
    sys.exit(0)
```

`argparse` raises `SystemExit`, which is not a subclass of `Exception`, so this
handler never fires. If it did, it would exit with status 0 on a usage error,
which is wrong. Remove it. The same file imports `tempfile` and `glob` without
using them.

### 4.8 `--save` in `deid inspect` joins characters, not folders - `deid/main/inspect.py:56`

`base` is a list when the user passes folders but a plain string when the
default demo dataset is used. `"-".join(os.path.basename(folder) for folder in base)`
then iterates over the characters of the path. Also the loop variable `result`
on line 70 shadows the outer `result` dict. This module has **0% coverage**, so
a test that drives `main()` with a fake `args` namespace would be very welcome.

### 4.9 Wrong `python_requires` - `setup.py:74`

`python_requires=">=3.7"` but `deid/dicom/fields.py:9` uses
`functools.cache` (3.9+) and `pydicom>=3.0` needs 3.10+. Set `>=3.10` and fix
the classifiers. Add a CI matrix (see §9) so this cannot regress silently.

### 4.10 Missing config falls through to a raw traceback - `deid/dicom/parser.py:72-74`

```python
if not os.path.exists(config):
    bot.error("Cannot find config %s, exiting" % (config))
self.config = read_json(config, ordered_dict=True)   # FileNotFoundError
```

The message says "exiting" but execution continues. Use `bot.exit` (consistent
with the rest of the codebase) or raise a clear exception.

### 4.11 `get_identifiers` silently ignores two of its parameters - `deid/dicom/header.py:16-56`

`strip_sequences` and `remove_private` are accepted and documented but never
used. Either implement them (pass them through to `DicomParser.parse`, which
already supports both), or remove them and the docstring lines. This is a
"documentation lies" bug and a good conversation to have with the maintainer in
an issue before the PR.

### 4.12 Unbound name in an error path - `deid/dicom/fields.py:564-567`

```python
else:
    bot.warning("Unrecognized type %s in extract sequences, skipping." % type(item))
```

`item` is only defined inside the sequence loop above, so this branch would
raise `NameError`. In practice iterating a `Dataset` only yields
`DataElement`s so the branch is unreachable, but it should read
`type(contender)`.

### 4.13 Test leaves the process in a deleted working directory - `deid/tests/test_cli.py:38`

`os.chdir(self.tmpdir)` is never undone, and `tearDown` then `rmtree`s that
directory. Every test that runs afterwards executes with a cwd that no longer
exists. Save `os.getcwd()` in `setUp` and restore it in `tearDown` (or use
`self.addCleanup`).

---

## 5. Intermediate changes (behavioural bugs and refactors)

These require understanding how the pieces fit together, writing more than one
test, and in some cases deciding on semantics with the maintainer. These are
the PRs that demonstrate you can own a feature area.

### 5.1 `+` inside a filter value is treated as a regex quantifier (the skipped test)

Recipe filter values are passed to `re.search` in
`deid/dicom/filter.py:200` (`compareBase`). That is why `|` works as "or"
inside a value. But `+` is a regex quantifier, so the shipped default recipe
contains filters that can never match as intended:

- `deid/data/deid.dicom:778` - `contains ImageType DERIVED+SECONDARY+SCREEN SAVE`
- `deid/data/deid.dicom:783` - `contains ManufacturerModelName SOMATOM Definition AS+`

`DERIVED+SECONDARY` as a regex means "DERIVE, one or more D, SECONDARY" - it
matches `DERIVEDSECONDARY` but not a multi-valued ImageType of
`['DERIVED', 'SECONDARY']`. The skipped test at
`deid/tests/test_dicom_pixels.py:287` describes exactly this.

The complication: in the recipe grammar, `+` *between* filter statements means
AND (`+ contains Manufacturer GE`). `parse_member` in `deid/config/utils.py:360`
tries to tell the two uses apart (it only splits on `+` if the remainder
contains a filter keyword), so `DERIVED+SECONDARY` survives as a value. What
should it mean? Options:

1. Treat `+` inside a value as "all of these must be present" (what the skipped
   test expects, and arguably what the recipe author meant).
2. Escape values with `re.escape` unless a documented "regex mode" is used.
3. Document that values are regexes and fix the two shipped filters to use
   `(?=.*DERIVED)(?=.*SECONDARY)`.

This needs an issue first. Whichever is chosen, the PR should: un-skip the
test, add tests for `|`, `+`, and literal characters like `.`, and update
`docs/_docs/user-docs/recipe-filters.md`.

### 5.2 `_has_burned_pixels_single` mutates the loaded recipe

`deid/dicom/pixels/detect.py:197-199`:

```python
for coordset in item["coordinates"]:
    if "from:" in coordset[1]:
        coordset[1] = extract_coordinates(dicom, coordset[1])
```

`item` is the recipe's own dict. Verified: after one call the recipe's
coordinates changed from `[[0, 'from:SequenceOfUltrasoundRegions']]` to
`[[0, []]]`. Consequences when one `DeidRecipe` is reused across files (which
is what `DicomCleaner` and `has_burned_pixels(list_of_files)` do):

- the second file never sees `from:` and reuses the first file's coordinates;
- the `results` dict returned for file one is also aliased to the recipe and
  gets overwritten later.

Fix: build a new coordinates list per file instead of assigning into the
recipe (a `copy.deepcopy(item["coordinates"])` is the minimal change).
Regression test: run two different files through one recipe and assert the
recipe is unchanged and each result is independent.

### 5.3 A filter label with coordinates but no filter lines crashes

Verified reproduction: a recipe containing

```
%filter blacklist
LABEL only coords
  coordinates 0,0,10,10
```

raises `UnboundLocalError: cannot access local variable 'group'` at
`deid/dicom/pixels/detect.py:176`, because `group` is only assigned inside the
`for group in item["filters"]` loop, and the "no filters, assume True" branch
on line 134 skips that loop. The code *intends* to support this case. Fix the
scoping and add a test using a recipe like the one above.

### 5.4 Importing `deid.dicom` changes global process state

`deid/dicom/pixels/clean.py:23-27` runs at import time:

```python
matplotlib.use("pdf")
...
bot.level = 3
```

and `deid/dicom/__init__.py` imports `pixels`, so **`import deid.dicom`** (the
normal way to use the library) does both. Verified: with `MESSAGELEVEL=QUIET`
the logger level goes from 0 to 3 after the import, and the matplotlib backend
switches from `agg` to `pdf`. For a library this is a real problem: a user's
own logging configuration is silently overridden and their plotting backend
changes.

Fix: remove `bot.level = 3`; move the matplotlib import and backend selection
inside the functions that draw (`get_figure`, `save_png`, `save_animation`),
or use `matplotlib.use("pdf", force=False)` only when no backend is set. This
also makes `import deid.dicom` faster for the common header-only use case.
Test: import the module under a controlled `MESSAGELEVEL` and assert
`bot.level` is unchanged.

### 5.5 Zip handling extracts into the current directory

`deid/dicom/utils.py:51-61`: `get_files(..., tempdir=None)` is the default, and
for a `.zip` input it calls `compressedFile.extractall(tempdir)`. With
`tempdir=None` that extracts into the **current working directory**, then
picks "the first regular file in that directory", which may not even be the
extracted one. Fix: create a `tempfile.mkdtemp()` when `tempdir` is `None`,
and iterate `compressedFile.namelist()` rather than listing the directory.
Add a test with a zip containing one DICOM.

### 5.6 `get_tag` does a linear scan of the DICOM dictionary

`deid/dicom/tags.py:42-44` scans all ~5,000 entries of
`pydicom._dicom_dict.DicomDictionary` on every call. `add_field` calls it for
every ADD/REPLACE. Measured: 0.23 ms per call versus 0.1 µs for pydicom's
public `pydicom.datadict.tag_for_keyword`. Replace the scan with
`tag_for_keyword` + `dictionary_VR` / `dictionary_description` (public API,
also removes the dependency on a private module). Keep the existing
`test_dicom_tags.py` assertions passing.

### 5.7 Regex compiled per field instead of once per expression

`deid/dicom/fields.py:291-301`: `expression_re = None` is set and never
updated, so `field_matches_expander` recompiles the expression for every
contender (Python's `re` cache hides most of the cost, but the docstring
promises caching that does not happen). Compile once before the loop. Benchmark
with a large multi-frame dataset before and after and put the numbers in the
PR; the maintainers clearly value performance PRs (#289, #295, #304).

### 5.8 `deepcopy` of every matched field per action

`deid/dicom/parser.py:480-482` deep-copies the whole matched field dictionary
(including pydicom elements) to avoid "dictionary changed size during
iteration". A shallow `list(fields.items())` is enough for that purpose.
This needs care: confirm that `_run_action` never relies on the copies being
independent (it looks up `self.fields[uid]` for mutation, which suggests it
does not). Run the full suite and a benchmark.

### 5.9 Unify `get_identifiers` / `replace_identifiers` parameter handling

Related to §4.11. Once `strip_sequences` / `remove_private` are wired through
`get_identifiers`, document the behaviour in `docs/_docs/getting-started/dicom-get.md`.

### 5.10 Make `startsWith`, `endsWith`, `matches` reachable from recipes

`deid/dicom/filter.py` defines them, but `apply_filter` and
`deid/config/standards.py` `filters` do not list them, so a recipe cannot use
them. Either expose them (with docs and tests) or delete them. Ask first.

### 5.11 Convert the test suite to idiomatic pytest (incrementally)

`pytest` already runs the suite. A gradual conversion is a good intermediate
project: add a `conftest.py` with fixtures for the demo dataset and a temp
directory, replace `setUp/tearDown` banners, use `pytest.mark.parametrize`
where tests enumerate VRs. Keep CI running both runners until the conversion is
complete. Discuss with the maintainer before starting; some projects prefer to
stay on `unittest`.

---

## 6. Advanced / senior-level changes

These touch architecture, public API, packaging, or process. Each one should
start as an issue with a proposal, get agreement, and then be delivered as a
series of small PRs. The skill being demonstrated is **judgement**: scoping
work, managing backwards compatibility, writing a migration plan, and
communicating trade-offs.

### 6.1 Library code should raise, not `sys.exit`

`bot.exit(...)` (which calls `sys.exit`) is used throughout `deid/config/utils.py`,
`deid/dicom/parser.py`, and elsewhere. For a *library*, this means a malformed
recipe kills the host process (a web server, a notebook kernel, a batch job)
instead of raising something the caller can catch. The senior-level change is:

1. Define a small exception hierarchy (`DeidError`, `RecipeError`, ...).
2. Replace `bot.exit` in library modules with `raise`.
3. Catch at the CLI boundary (`deid/main/__init__.py`) and exit there.
4. Keep `bot.exit` working for backward compatibility for one release, with a
   deprecation warning.

This needs a CHANGELOG entry, docs, and probably a minor version bump.

### 6.2 Stop monkey-patching `pydicom.Dataset`

`deid/dicom/filter.py` attaches 11 methods to `pydicom.dataset.Dataset` at
import time (`Dataset.contains = contains`, ...). This is why mypy reports 11
errors there, why editors cannot find these methods, and why importing `deid`
changes the behaviour of every `Dataset` in the process. Refactor to plain
functions `contains(dataset, field, expression)` called from `apply_filter`.
Keep the patched attributes for one release behind a deprecation shim if
anyone might rely on them.

### 6.3 Lazy-load matplotlib and make it an optional extra

`matplotlib` is a hard dependency used only by `DicomCleaner.get_figure`,
`save_png`, and `save_animation`. Header-only users pay for a 40 MB install and
a slow import. Move the imports inside those methods and declare
`matplotlib` under `[project.optional-dependencies] viz = [...]`. This is a
packaging + API change that needs a deprecation note.

### 6.4 Migrate packaging to PEP 621 `pyproject.toml`

Replace `setup.py` + `setup.cfg` + `MANIFEST.in` + `deid/version.py`'s
`INSTALL_REQUIRES` with a single `[project]` table (see §7). Keep
`deid.version.__version__` importable (dynamic version from `pyproject` or keep
`version.py` as the single source and point `[tool.setuptools.dynamic]` at it).
Verify with `python -m build` and `twine check` that the sdist/wheel contain
`deid/data/*` and `deid/dicom/config.json`.

### 6.5 Introduce ruff, pytest, coverage, and a Python matrix in CI

See §7 and §9. This is several PRs: (a) add `ruff` config matching the
current `black`/`isort`/`flake8` settings so the diff is zero; (b) swap the
pre-commit hooks; (c) switch CI to `uv` + `pytest --cov` with a 3.10-3.13
matrix and upload coverage; (d) later, gradually enable more ruff rules
(`UP`, `B`, `SIM`) one family per PR with autofix.

### 6.6 Type annotations and `py.typed`

The public API (`get_identifiers`, `replace_identifiers`, `DeidRecipe`,
`DicomParser`, `DicomCleaner`, `has_burned_pixels`) has essentially no type
hints. Adding them module by module, fixing the 18 existing mypy errors, adding
`mypy` to CI, and shipping a `py.typed` marker is a multi-PR project that makes
the library much easier for downstream users and for you to reason about.

### 6.7 Decouple the recipe parser from the logger and the filesystem

`deid/config/utils.py` parses the recipe DSL but is interleaved with
`bot.exit`, `os.path` lookups and the `data_base` import. A `parse_recipe(text:
str) -> Recipe` pure function with a small dataclass model would make it
testable in isolation, enable a `deid validate-recipe` CLI command, and is a
natural precursor to §6.1. This is the kind of design PR where you write the
proposal first.

### 6.8 Release automation

There is no release workflow; `pypi.sh` is in `.gitignore`, so releases are a
manual script on the maintainer's machine. A `release.yaml` using PyPI trusted
publishing, triggered on a tag, with the CHANGELOG checked for the version, is
a contained but high-trust change. Propose it; the maintainer must set up the
PyPI side.

---

## 7. Modern Python tooling assessment

Short version: **functional but dated**. Nothing here is blocking contributors,
but almost every tool is one or two generations behind, and two config files
are broken in ways nobody noticed because CI does not exercise them.

| Area | Current state | Modern baseline | Gap |
|---|---|---|---|
| Build backend / metadata | `setup.py` + `setup.cfg` + `MANIFEST.in`; `pyproject.toml` holds only black/isort settings; version and dependencies live in `deid/version.py` and are `exec`'d by `setup.py` | PEP 621 `[project]` table in `pyproject.toml`; `setuptools>=61`, `hatchling`, or `flit` | Large. No `[build-system]` table at all, so pip falls back to legacy behaviour. |
| Environment / dependency manager | None. `pip install .`; dev deps in `.github/dev-requirements.txt`; `deid-data` only installed in CI YAML | `uv` (`uv sync`, `uv run pytest`), with `[project.optional-dependencies]` or `[dependency-groups]` for `dev`/`test` | Large. No lock file, no reproducible dev environment. |
| Formatter | `black==23.3.0` pinned (May 2023); `pyproject` config is **broken** (§3.6) | `ruff format` (or current black) | Medium. Works only via pre-commit. |
| Import sorting | `isort` with `profile = "black"` | `ruff check --select I` | Small; works. |
| Linter | `flake8` with `ignore = E1 E2 E5 W5` (disables most style checks); `.pylintrc` present but unused | `ruff` with `E,F,I,UP,B,SIM` as a start | Medium. Current lint catches almost nothing. |
| Type checker | None configured; 18 mypy errors | `mypy` or `pyright` in CI, `py.typed` | Large. |
| Test runner | `python -m unittest discover` in CI; pytest works but is unconfigured | `pytest` with `[tool.pytest.ini_options]`, `conftest.py` fixtures | Medium. |
| Coverage | `[coverage:*]` in `setup.cfg` with `fail_under = 50`; **never run in CI** | `pytest --cov` in CI; Codecov or `coverage report --fail-under` | Medium. Easy win. |
| Pre-commit | Present, hooks from 2022; runs black/isort/flake8 as "local" hooks | `pre-commit autoupdate`; `ruff-pre-commit` | Small. |
| Spell check | `typos` action pinned to a 2021 commit; `codespell` config in `setup.cfg` but not run | Current `typos` or `codespell` pre-commit hook | Small. |
| CI | GitHub Actions; creates conda environments to run pip; single unspecified Python version; mixed `checkout@v3`/`@v4` | `actions/setup-python` or `astral-sh/setup-uv`; matrix 3.10-3.13; cache | Medium. |
| Dependency updates | None (no Dependabot / Renovate) | Dependabot for actions + pip | Small to add. |
| Docs | Jekyll (Ruby 2.7.8 required - EOL) + Sphinx; docs job clones upstream from `/tmp` | Either is fine; pin Ruby via `.ruby-version`; or consider MkDocs | Medium, low priority. |
| Container | `continuumio/miniconda3` base, `python setup.py install`, no pin | `python:3.12-slim` + `pip install .` | Small. |
| Python version support | `python_requires>=3.7` (wrong), classifier `3.8` only, actual minimum 3.10 | Declare 3.10-3.13; test them | Small fix, needs matrix to enforce. |
| Repo hygiene | PR template yes; issue templates, `CODEOWNERS`, `SECURITY.md`, Dependabot no | All four | Small. |

**What a modernization series would look like** (each a separate PR, in order,
each discussed in an issue first):

1. Fix `pyproject.toml` black config; `pre-commit autoupdate`. (first-timer)
2. Add `pytest` + `coverage` to CI with a 3.10-3.13 matrix; upload the report.
   No behaviour change. (beginner/intermediate)
3. Add `ruff` configured to be a no-op against the current code
   (`line-length = 100`, `select = ["E", "F", "I"]`, same ignores as flake8),
   replace black/isort/flake8 in pre-commit. (intermediate)
4. Move metadata to PEP 621 `pyproject.toml`; delete `setup.py`/`setup.cfg`/
   `MANIFEST.in`; fix `python_requires`. (advanced)
5. Add `uv.lock` and document `uv sync` / `uv run pytest` in README. (intermediate)
6. Enable ruff rule families one at a time with `--fix`. (beginner, repeated)
7. Add Dependabot, `SECURITY.md`, issue templates. (first-timer)
8. Release workflow with trusted publishing. (advanced, maintainer-gated)

Step 3 is the one to be careful about: pick the ruff settings so the first PR
produces **zero** code changes. Reviewers accept tooling swaps far more readily
when the diff is only config.

---

## 8. Test coverage assessment

### 8.1 Is this a good area for a new contributor?

**Yes, strongly.** Three reasons:

1. The high-risk core (recipe parsing, header actions, nested sequences) is
   already well covered (81-99%), so you will not be expected to write the
   hardest tests first.
2. The low-coverage modules are small utilities with obvious contracts
   (`tags.py`, `filter.py`, `fileio.py`, `message.py`, `progress.py`,
   `inspect.py`). They are ideal for learning the codebase one function at a
   time.
3. Coverage is not enforced in CI, so any increase is pure upside and will not
   break anyone's workflow.

### 8.2 Per-module coverage (library code, branch coverage)

| Module | Stmts | Miss | Cover | Comment |
|---|---|---|---|---|
| `deid/main/inspect.py` | 41 | 41 | **0%** | The `deid inspect` CLI command. Untested entirely; contains the §4.8 bug. |
| `deid/logger/progress.py` | 67 | 50 | **20%** | Progress bar; contains §4.5. Easy to test with `hide=True`. |
| `deid/dicom/tags.py` | 80 | 49 | **32%** | `find_tag`, `update_tag`, `remove_sequences`, `get_private`, `has_private` untested. |
| `deid/dicom/filter.py` | 109 | 51 | **47%** | Filter primitives; only `contains` and `missing`/`present` are exercised. |
| `deid/logger/message.py` | 184 | 82 | **47%** | Level parsing, colour handling, `table`, `show_progress`. |
| `deid/main/identifiers.py` | 45 | 15 | 55% | CLI `identifiers` command; only the `all` path is exercised (via `test_cli.py`). |
| `deid/dicom/header.py` | 48 | 12 | 74% | `remove_private_identifiers` (lines 69-88) is untested. |
| `deid/main/__init__.py` | 52 | 10 | 74% | Argument parsing; `version`, `inspect` and help paths untested. |
| `deid/utils/actions.py` | 103 | 26 | 74% | `convert_value` failure branches, `func:` path. |
| `deid/dicom/pixels/clean.py` | 176 | 39 | 75% | `get_figure`, `save_png`, `save_animation`, compression fallbacks. |
| `deid/dicom/fields.py` | 251 | 48 | 76% | `extract_item`/`extract_sequence` (lines 166-220), `select_matches` GROUP branch. |
| `deid/utils/fileio.py` | 54 | 10 | 76% | `to_int`, `is_number`, `write_json` pretty=False. |
| `deid/config/utils.py` | 275 | 40 | 81% | Error/exit paths of the recipe parser. |
| `deid/data/__init__.py` | 11 | 2 | 82% | The "deid-data not installed" branch. |
| `deid/dicom/utils.py` | 54 | 7 | 84% | Zip-file branch of `get_files` (§5.5). |
| `deid/dicom/pixels/detect.py` | 114 | 15 | 84% | Multi-file path, `extract_coordinates` edge cases. |
| `deid/dicom/validate.py` | 19 | 2 | 87% | The "cannot read" warning path. |
| `deid/config/__init__.py` | 79 | 5 | 90% | - |
| `deid/dicom/groups.py` | 49 | 3 | 90% | - |
| `deid/dicom/actions/uids.py` | 33 | 1 | 90% | - |
| `deid/dicom/parser.py` | 265 | 20 | 91% | Core; well covered. |
| `deid/dicom/actions/jitter.py` | 43 | 1 | 95% | - |
| **Total** | **2,175** | **529** | **72%** | |

### 8.3 Suggested order of attack

1. `deid/utils/fileio.py` and `deid/utils/actions.py` - pure functions, no
   DICOM knowledge needed. One afternoon.
2. `deid/logger/message.py` and `progress.py` - still pure, teaches you how the
   project logs.
3. `deid/dicom/tags.py` and `deid/dicom/filter.py` - needs a `Dataset`; use
   `pydicom.Dataset()` built in the test, or the `dicom-cookies` demo data
   like the existing tests do. Teaches you pydicom basics.
4. `deid/main/inspect.py` and `identifiers.py` - construct an `argparse.Namespace`
   by hand and call `main(args, parser)`. Teaches you the CLI.
5. `deid/dicom/header.py` `remove_private_identifiers`, and the zip branch of
   `get_files`.
6. Then the behavioural bugs in §5, each of which comes with its own test.

### 8.4 Observations on the existing tests

- The core test files (`test_action_interaction.py`, 1,946 lines, and
  `test_replace_identifiers.py`, 1,351 lines) are long but very thorough. Read
  them to learn how recipes and actions are expected to behave; they are the
  real specification.
- `deid/tests/resources/*.dicom` are **recipe** files, not DICOM images,
  despite the extension. That naming surprises newcomers.
- Tests depend on the `deid-data` package (network install). It is not declared
  anywhere except the CI YAML. Adding `[project.optional-dependencies] test =
  ["deid-data", "pytest", "pytest-cov"]` would fix that.
- `self.subTest` is used in a few places (17 subtests); the "Case N:" `print`
  pattern elsewhere is the older equivalent and could be converted.
- Warnings: 17 pydicom `UserWarning`s about invalid VR values are emitted by
  tests that deliberately write odd values. Consider
  `pytest.warns`/`assertWarns` so they are asserted rather than leaked.

---

## 9. CI/CD and release process

`.github/workflows/main.yaml` has two jobs:

- **formatting**: creates a conda env with the latest `black`, then
  `pip install -r .github/dev-requirements.txt` (which pins black 23.3.0), then
  `pre-commit run --all-files`. Works, but the conda step is unnecessary.
- **testing**: creates an empty conda env, `pip install .`, `pip install deid-data`,
  `python -m unittest discover`. Runs on whatever Python the runner's miniconda
  ships. No matrix, no coverage, no caching.

`.github/workflows/docs.yaml` (on push to master) checks out `gh-pages`, clones
upstream `pydicom/deid` into `/tmp`, builds Sphinx API docs and force-moves
directories around. It works but is fragile and would not work from a fork.

`.github/workflows/codespell.yaml` runs `typos` pinned to a 2021 commit.

Release: manual. There is no tag-triggered workflow. The CHANGELOG is updated
by hand per PR, and the version in `deid/version.py` is bumped in the same PR
as the feature (see the CHANGELOG convention `(0.4.13)` suffixes). **Follow
this convention in your own PRs**: add a CHANGELOG line and, if the maintainer
asks, bump the patch version.

Concrete improvements, easiest first:

1. `actions/checkout@v4` everywhere; bump `typos`. (first-timer)
2. Replace conda steps with `actions/setup-python` (or `astral-sh/setup-uv`)
   and a `python-version: ["3.10", "3.11", "3.12", "3.13"]` matrix. (beginner)
3. Run `pytest --cov` and `coverage report --fail-under=70` (raise from 50 to
   the current 72% floor so it cannot regress). (beginner)
4. Dependabot for `github-actions` and `pip`. (first-timer)
5. Release workflow with trusted publishing. (advanced)

---

## 10. Code quality and architecture observations

### 10.1 Things done well (learn from these)

- **The recipe DSL** (`%header`, `%filter`, `%values`, `%fields` sections with
  `ADD/BLANK/JITTER/KEEP/REPLACE/REMOVE/LABEL` actions and `contains:`,
  `endswith:`, `select:vr:`, `except:` expanders) is a genuinely nice piece of
  design: small, readable by non-programmers, and testable from text files.
- **Action precedence** is documented inline where it matters
  (`deid/dicom/parser.py:648-651`: `KEEP > ADD > REPLACE > JITTER > REMOVE > BLANK`)
  and enforced by `excluded_from_deletion`. Read `test_action_interaction.py`
  to see how precedence is specified by tests.
- **`FieldsWithLookups`** (`deid/dicom/fields.py:344`) is a textbook example of
  adding O(1) lookup tables over an existing dict without changing callers.
  The PR that introduced it (#289) and the follow-up cache-invalidation fix
  (#307) are worth reading as examples of performance work done carefully.
- **`override_attr`** context manager (`fields.py:447`) - a small, correct,
  reusable utility with a clear docstring explaining *why* the hack exists.
- Pre-compiled module-level regexes (`fields.py:19`, `parser.py:30-31`) with a
  comment explaining the performance reason.
- A `CHANGELOG.md` that is actually maintained, with PR links and version
  numbers.
- Test resource recipes in `deid/tests/resources/` make filter tests
  data-driven.

### 10.2 Patterns to be aware of (and, eventually, to help change)

- **A global mutable logger singleton** (`bot`) whose level is read from an
  environment variable at import time and then mutated by other modules
  (§5.4). Senior-level fix: use the standard `logging` module with a
  `deid` logger; keep `bot` as a thin shim.
- **`sys.exit` inside library code** (§6.1).
- **Monkey-patching a third-party class** (§6.2).
- **`%`-formatting everywhere** (134 instances). Not wrong, but the project
  has started using f-strings in newer code; a `ruff --select UP031 --fix`
  PR is mechanical.
- **Inconsistent docstring styles**: numpy-ish `Parameters\n==========`,
  Sphinx `:param x:`, and free text all coexist. Pick one (the existing
  Sphinx API docs suggest Sphinx/numpydoc) and convert module by module.
- **camelCase methods** in `DeidMessage` (`useColor`, `addColor`, `emitError`,
  `isEnabledFor`) and in `filter.py` (`equalsBase`, `notEquals`) alongside
  snake_case elsewhere. Renaming public-ish methods is an API change; discuss.
- **Broad `except Exception`** in `deid/dicom/validate.py:32`,
  `deid/utils/actions.py:45`, `deid/dicom/parser.py:683`, `jitter.py:68,84`.
  Some are justified (reading arbitrary files); each should either narrow the
  exception or log it.
- **Module-level `global` state** flagged by ruff (`PLW0604`, 11 instances):
  the `here = os.path.dirname(...)` pattern is fine; check the others.
- `deid/dicom/__init__.py` **eagerly imports everything**, including
  matplotlib via `pixels`. Lazy imports (§6.3) would help.
- **Duplicated re-export lists**: `deid/utils/__init__.py`,
  `deid/dicom/__init__.py` etc. re-export without `__all__`, which is why ruff
  reports 29 `F401`s. Adding `__all__` lists is a first-timer PR that also
  documents the public API.
- `deid/data/__init__.py` raises `ValueError` when an optional package is
  missing; `ImportError` (or a custom `DeidError`) is more idiomatic.

### 10.3 Things I could not evaluate from this session

- The upstream issue tracker and open PRs on `pydicom/deid` (this session was
  scoped to your fork). Before starting any §5 or §6 item, search upstream
  issues; several may already be reported, and the maintainer may have
  opinions recorded there.
- Whether the Jekyll docs build cleanly (requires Ruby 2.7.8).
- Real-world performance on large multi-frame datasets (I only micro-benchmarked
  `get_tag`).

---

## 11. Security and robustness notes

`deid` handles protected health information, so "best effort" has to be taken
seriously. Nothing here is an exploitable vulnerability, but these are the
places a security-minded reviewer would look:

- **Recipe mutation (§5.2)** is the most important robustness issue found:
  reusing a recipe across files can apply the wrong coordinates to the second
  file, which in the worst case leaves burned-in text uncleaned. It deserves a
  regression test and a CHANGELOG line.
- **Regex quantifier in shipped filters (§5.1)** means two of the default
  filter rules in `deid/data/deid.dicom` cannot match the images they target.
- **Filter values are regexes**: a user-supplied recipe value like `(` raises
  `re.error` deep inside `compareBase`; a value like `.*` matches everything.
  If values are meant to be literal, escape them; if they are meant to be
  regexes, say so loudly in the docs.
- **Zip extraction into cwd (§5.5)**. Python's `zipfile` strips absolute paths
  and `..` components, so this is not a path-traversal hole, but writing user
  archives into the working directory is still surprising.
- `_get_clean_name` builds paths with `"%s/cleaned-%s.%s"` instead of
  `os.path.join` - portability, not security.
- Output directory permissions were recently tightened to `0o700` (#304); good
  precedent for the kind of change maintainers accept.
- There is no `SECURITY.md` describing how to report a vulnerability
  privately. Adding one is a first-timer PR (ask the maintainer for the
  contact).

---

## 12. How to work with this project

### 12.1 Local setup that actually works (verified)

```bash
git clone https://github.com/<you>/deid
cd deid
uv venv .venv && source .venv/bin/activate      # or python -m venv
uv pip install -e . deid-data pytest pytest-cov  # or plain pip
pip install -r .github/dev-requirements.txt      # black, isort, flake8, pre-commit
pre-commit install

pytest deid/tests -q                             # ~35 s
pytest deid/tests -q --cov=deid --cov-config=setup.cfg --cov-report=term-missing
pre-commit run --all-files
```

Note: `black deid` run directly will do nothing until §3.6 is fixed; use
`pre-commit run black --all-files` instead.

### 12.2 Conventions the maintainer expects (from `CONTRIBUTING.md`, the PR template, and recent merged PRs)

- PRs target `master` on upstream `pydicom/deid`.
- Fill in the PR template: Description, related issue, Checklist, Open
  questions. Recent merged PRs have short, specific titles
  ("Fix repeated field extraction and parser cache invalidation").
- Add a line to `CHANGELOG.md` under the `vxx (master)` heading, with the PR
  link, and the new version number in parentheses if you bump
  `deid/version.py`. Look at the last ten entries for the exact format.
- Every new source file carries the `__author__ / __copyright__ / __license__`
  header.
- No tabs; black formatting; keep the existing docstring style of the file you
  are editing.
- For anything bigger than a typo, open an issue first and describe what you
  intend to do. The maintainer is responsive and will tell you if it is
  wanted.
- User-visible behaviour changes need a docs update under `docs/_docs/`.

### 12.3 How to write a PR description that gets merged quickly

1. One sentence: what changed and why.
2. How you verified it (the exact command, and the test you added).
3. For bug fixes: a minimal reproduction, before and after.
4. For performance: numbers, with the dataset and command used.
5. Anything you were unsure about, under "Open questions".

### 12.4 Reading order to understand the code

1. `docs/_docs/user-docs/recipe-headers.md` and `recipe-filters.md` (the DSL).
2. `deid/config/utils.py` `load_deid` (how the DSL becomes a dict).
3. `deid/dicom/fields.py` (`DicomField`, `get_fields_with_lookup`,
   `expand_field_expression`).
4. `deid/dicom/parser.py` (`DicomParser.parse` -> `perform_action` ->
   `_run_action`).
5. `deid/dicom/header.py` (the public wrappers).
6. `deid/dicom/pixels/detect.py` then `clean.py`.
7. `deid/tests/test_action_interaction.py` as the executable spec.

---

## 13. A learning roadmap: from first PR to senior-level contributor

Each rung adds one new skill. Do them roughly in order; skip rungs you already
have. Expect the whole ladder to take months of part-time work, which is
normal.

| Rung | PR(s) | Skill you demonstrate |
|---|---|---|
| 1 | One typo/grammar PR from §3.1-3.2 (e.g. README install command and badge). | Fork, branch, commit, PR template, responding to review. |
| 2 | Delete `.travis.yml`; add missing `.gitignore` entries; `pre-commit autoupdate`. | Understanding project tooling; small config changes; explaining *why* in the PR. |
| 3 | Fix the `pyproject.toml` black config (§3.6) with before/after evidence in the PR. | Reproducing a tooling bug and proving the fix. |
| 4 | Add tests for `fileio.py` and `actions.py` helpers (§3.7). | Writing unit tests; reading coverage reports; using temp directories. |
| 5 | Fix `bot.exis`, `is_quiet`, `print(extras)`, `suffix_uuid`, each with a regression test (§4.1-4.4). One PR each. | Test-first bug fixing. |
| 6 | Fix `test_cli.py` cwd leak (§4.13); add tests for `inspect.py` and fix §4.8. | Test isolation; testing CLI code with `argparse.Namespace`. |
| 7 | Add pytest + coverage + Python matrix to CI (§9 items 2-3). | GitHub Actions; CI design; not breaking the maintainer's workflow. |
| 8 | `__all__` lists for all packages; `ruff --select UP031 --fix` conversion to f-strings. | Mechanical refactors at scale; keeping diffs reviewable. |
| 9 | Fix the recipe-mutation bug (§5.2) and the `UnboundLocalError` (§5.3) with multi-file tests. | Finding aliasing bugs; reasoning about shared state. |
| 10 | Remove import-time side effects (§5.4); lazy matplotlib import. | Library hygiene; backwards compatibility; import-time testing. |
| 11 | `get_tag` via `tag_for_keyword` (§5.6) with a benchmark in the PR; regex pre-compilation (§5.7). | Performance work with evidence. |
| 12 | Open an issue proposing semantics for `+` in filter values (§5.1); implement the agreed option; un-skip the test; update docs. | Driving a design decision to consensus; changing a DSL safely. |
| 13 | Introduce ruff as a zero-diff replacement for black/isort/flake8 (§7 step 3). | Tooling migration without disruption. |
| 14 | PEP 621 packaging migration (§6.4), verified with `python -m build`. | Packaging; release artifacts; what ends up in a wheel. |
| 15 | Exception hierarchy replacing `sys.exit` in library code (§6.1), delivered as an RFC issue + 3-4 PRs with deprecation shims. | API design, deprecation policy, multi-PR delivery, writing for reviewers. |
| 16 | De-monkey-patch `Dataset` (§6.2); add type hints + mypy to CI; `py.typed`. | Static typing at scale; incremental strictness. |
| 17 | Release automation proposal (§6.8). | Trust, process, and working with a maintainer on credentials you will never see. |

By rung 9 you are doing the work of a solid mid-level contributor. Rungs 12-17
are where senior-level judgement shows: you are choosing *what* to change,
writing the proposal, splitting the work, and shepherding it through review
rather than just executing a ticket.

Two habits that matter more than any single rung:

- **Open the issue before the big PR.** On a one-maintainer project, a 600-line
  surprise PR is a burden; a 10-line issue that says "I would like to do X,
  here is why, here is how I would split it" is a gift.
- **Review other people's PRs.** Reading the recent merged PRs (#289, #293,
  #295, #304, #307) and leaving a thoughtful comment on the next open one is
  how maintainers learn to trust your judgement.

---

## 14. Appendix: commands and raw results

Environment: Python 3.13.17, `uv 0.11.32`, Linux. Fresh venv.

```bash
uv venv venv && source venv/bin/activate
uv pip install -e . deid-data pytest coverage pytest-cov ruff black==23.3.0 isort flake8 mypy

# Tests + coverage (library only)
python -m pytest deid/tests -q \
  --cov=deid/config --cov=deid/data --cov=deid/dicom --cov=deid/logger \
  --cov=deid/main --cov=deid/utils --cov-branch --cov-report=term-missing
# -> 274 passed, 1 skipped, 17 subtests passed, 17 warnings; TOTAL 72%

# Lint / format / types
flake8 deid --count                       # 0
black --check --verbose deid              # everything excluded (broken config)
isort --check-only deid                   # clean
ruff check deid --isolated --select E,F --statistics   # 96 E501, 33 F401
ruff check deid --statistics              # 310 (broad default rule set in ruff 0.17)
ruff format --check deid                  # 2 files would be reformatted
mypy deid --ignore-missing-imports        # 18 errors in 3 files
uvx codespell deid docs/_docs docs/README.md examples README.md CONTRIBUTING.md CHANGELOG.md \
  --skip="*.dcm,*.dicom,lunr.min.js" -I .github/codespell_ignore_words.txt   # 10 hits
```

Bug reproductions (all run from the repo root with the venv active):

```python
# §4.1 / §4.2
from deid.logger import bot
hasattr(bot, "exis")          # False
bot.is_quiet()                # True at level 5 (DEBUG)  <- inverted

# §4.4
from deid.dicom.actions.uids import suffix_uuid
suffix_uuid(item={}, value=None, field="Patient Name")   # 'patient name-<uuid>'

# §5.4
import os; os.environ["MESSAGELEVEL"] = "QUIET"
from deid.logger import bot; print(bot.level)   # 0
import matplotlib; print(matplotlib.get_backend())   # agg
import deid.dicom
print(bot.level, matplotlib.get_backend())      # 3 pdf

# §5.3  (recipe with a LABEL that has coordinates but no filter lines)
# -> UnboundLocalError: cannot access local variable 'group'

# §5.2  (recipe with "coordinates from:SequenceOfUltrasoundRegions", run once)
# recipe.deid["filter"]["blacklist"][0]["coordinates"]
#   before: [[0, 'from:SequenceOfUltrasoundRegions']]   after: [[0, []]]

# §5.6
import timeit
from deid.dicom.tags import get_tag
from pydicom.datadict import tag_for_keyword
timeit.timeit(lambda: get_tag("PatientID"), number=100)          # ~0.023 s
timeit.timeit(lambda: tag_for_keyword("PatientID"), number=100)  # ~0.00001 s
```

Git history sampled (last ~60 commits): 12 distinct authors; the maintainer
and two or three regular contributors account for most commits, with several
one-commit contributors - evidence that first PRs from outsiders do get merged.
