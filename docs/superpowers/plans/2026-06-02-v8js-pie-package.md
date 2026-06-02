# `standa/php-v8js` PIE Package — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Scaffold a local repository at `/Users/standa/Documents/www/test/v8js/` that packages phpv8/v8js (php8 branch, pinned SHA) as a PIE-installable Composer package named `standa/php-v8js`, including a CI workflow that publishes prebuilt binaries via `php/pie-ext-binary-builder`.

**Architecture:** Snapshot-copy the upstream source at `phpv8/v8js@8a39efa3cf3b275e402ddf3c4f6b611a5f69a499` into the repo, add a PIE-compliant `composer.json` (type `php-ext`, single `--with-v8js` configure option, `download-url-method: ["pre-packaged-binary", "composer-default"]`, `os-families-exclude: ["windows"]`), wrap with project README, CI workflow, and `.gitattributes` that keep dev-only paths out of the `git archive` PIE downloads. Verify locally via `composer validate` and (best-effort) `pie build` against system libv8.

**Tech Stack:** PIE 1.3+ (we have 1.3.10 locally; docs from 1.5.x), Composer 2.x, GitHub Actions, `php/pie-ext-binary-builder@0.0.2`, `shivammathur/setup-php@v2`. Vendored source is C++/PHP-extension code from phpv8/v8js (no edits).

**Spec:** `docs/superpowers/specs/2026-06-02-v8js-pie-package-design.md`

**Pinned upstream SHA:** `8a39efa3cf3b275e402ddf3c4f6b611a5f69a499` (phpv8/v8js, `php8` branch, 2026-04-19).

---

## Task 1: Initialise the repository

**Files:**
- Create: `/Users/standa/Documents/www/test/v8js/.gitignore`
- Create: `/Users/standa/Documents/www/test/v8js/.gitattributes`

The working directory exists but is empty except for `docs/superpowers/`. We need a git repository and the two dotfiles that govern what's tracked and what ends up in PIE's downloaded archive.

- [ ] **Step 1: Init the git repo**

Run:
```bash
cd /Users/standa/Documents/www/test/v8js
git init -b main
```
Expected: `Initialized empty Git repository in /Users/standa/Documents/www/test/v8js/.git/`

- [ ] **Step 2: Write `.gitignore`**

The upstream phpv8/v8js `.gitignore` covers extension build artifacts. We add IDE/OS noise on top.

Write `/Users/standa/Documents/www/test/v8js/.gitignore`:
```gitignore
# PHP extension build artifacts (from phpize/configure/make)
*.lo
*.la
*.o
*.so
*.dep
.deps/
.libs/
modules/
autom4te.cache/
config.h
config.h.in
config.log
config.nice
config.status
configure
configure.ac
configure.in
build/
install-sh
ltmain.sh
missing
mkinstalldirs
acinclude.m4
aclocal.m4
include/
libtool
Makefile
Makefile.fragments
Makefile.global
Makefile.objects
run-tests.php
*.lock

# IDE / editor noise
.idea/
.vscode/
*.swp
*.bak

# OS noise
.DS_Store
Thumbs.db

# PHPUnit / Composer
/vendor/
.phpunit.result.cache
```

- [ ] **Step 3: Write `.gitattributes`**

This file is what governs PIE's `composer-default` download: anything `export-ignore`'d is stripped from the `git archive` ZIP that PIE downloads, so dev-only paths don't bloat user installs.

Write `/Users/standa/Documents/www/test/v8js/.gitattributes`:
```gitattributes
# Strip development-only paths from the PIE-downloaded archive.
/.github               export-ignore
/.gitattributes        export-ignore
/.gitignore            export-ignore
/.dockerignore         export-ignore
/docs                  export-ignore
/README.Linux.md       export-ignore
/README.MacOS.md       export-ignore
/README.Win32.md       export-ignore
/package.xml           export-ignore

# Force LF on shell + autoconf inputs
*.sh   text eol=lf
*.m4   text eol=lf
*.w32  text eol=lf
```

Note: we keep upstream's tests/ in the archive (PIE/PECL convention is to ship tests).

- [ ] **Step 4: Sanity check — no commit yet**

Run:
```bash
cd /Users/standa/Documents/www/test/v8js
git status --short
```
Expected: a list of untracked files including `.gitattributes`, `.gitignore`, `docs/`. No staged files. We commit after the rest of the scaffold is in place (Task 7).

---

## Task 2: Vendor upstream source at the pinned SHA

**Files:**
- Create (downloaded): every file from phpv8/v8js@8a39efa3 *except* `.github/` and `README.md`

We fetch the upstream tarball at the exact pinned SHA, extract into a scratch dir, copy the relevant files into the repo root, and delete what we'll replace ourselves (project README, upstream CI).

- [ ] **Step 1: Download the upstream tarball at the pinned SHA**

Run:
```bash
mkdir -p /tmp/v8js-scratch
curl -sSL --connect-timeout 30 --max-time 120 \
  -o /tmp/v8js-scratch/upstream.tar.gz \
  https://codeload.github.com/phpv8/v8js/tar.gz/8a39efa3cf3b275e402ddf3c4f6b611a5f69a499
ls -la /tmp/v8js-scratch/upstream.tar.gz
```
Expected: file present, ~200KB+.

- [ ] **Step 2: Extract the tarball and verify the pinned SHA's directory exists**

Run:
```bash
cd /tmp/v8js-scratch
tar xzf upstream.tar.gz
ls v8js-8a39efa3cf3b275e402ddf3c4f6b611a5f69a499/
```
Expected: a directory listing including `config.m4`, `v8js_main.cc`, `package.xml`, `tests/`, `.github/`, `LICENSE`, etc.

- [ ] **Step 3: Copy upstream files into the repo root, excluding `.github/` and `README.md`**

We use rsync with `--exclude` because we'll write our own `README.md` and our own CI workflow.

Run:
```bash
rsync -av \
  --exclude='.github' \
  --exclude='README.md' \
  /tmp/v8js-scratch/v8js-8a39efa3cf3b275e402ddf3c4f6b611a5f69a499/ \
  /Users/standa/Documents/www/test/v8js/
```
Expected: file copy log. No errors.

- [ ] **Step 4: Verify the vendor landed correctly**

Run:
```bash
cd /Users/standa/Documents/www/test/v8js
ls config.m4 v8js_main.cc v8js_class.cc php_v8js.h php_v8js_macros.h LICENSE
ls tests/ | head -10
# Must NOT exist:
test ! -e .github && echo ".github excluded OK"
test ! -e README.md && echo "README.md excluded OK"
```
Expected: the six file paths exist, the two negative checks both print "...OK".

- [ ] **Step 5: Sanity-check vendored content matches upstream**

Run:
```bash
cd /Users/standa/Documents/www/test/v8js
head -3 config.m4
grep -c "PHP_ARG_WITH(v8js" config.m4
grep -c "v8js" php_v8js.h
```
Expected:
```
PHP_ARG_WITH(v8js, for V8 Javascript Engine,
[  --with-v8js           Include V8 JavaScript Engine])

1
≥1
```

- [ ] **Step 6: Clean up scratch**

Run:
```bash
rm -rf /tmp/v8js-scratch
```

---

## Task 3: Write `composer.json`

**Files:**
- Create: `/Users/standa/Documents/www/test/v8js/composer.json`

The PIE manifest. Every field choice is justified in the spec (section "composer.json").

- [ ] **Step 1: Write `composer.json`**

Write `/Users/standa/Documents/www/test/v8js/composer.json`:
```json
{
    "name": "standa/php-v8js",
    "description": "V8 JavaScript Engine for PHP (PIE-installable wrapper around phpv8/v8js, php8 branch)",
    "type": "php-ext",
    "license": "MIT",
    "keywords": ["v8", "javascript", "php-ext", "pie", "extension"],
    "require": {
        "php": "^8.1",
        "php-64bit": "*"
    },
    "php-ext": {
        "extension-name": "v8js",
        "support-zts": true,
        "support-nts": true,
        "configure-options": [
            {
                "name": "with-v8js",
                "description": "Path to the V8 install prefix (e.g. /usr, /usr/local, /opt/homebrew). Auto-searches /usr/local and /usr if no value is given.",
                "needs-value": true
            }
        ],
        "download-url-method": ["pre-packaged-binary", "composer-default"],
        "os-families-exclude": ["windows"]
    }
}
```

- [ ] **Step 2: Validate the manifest**

Run:
```bash
cd /Users/standa/Documents/www/test/v8js
composer validate --strict
```
Expected: `./composer.json is valid`.

Composer may emit a warning about no `version` field — that's fine for a path-repo/tag-driven package and matches the recommendation in the PIE extension-maintainers doc. Ignore the warning.

If validation FAILS with a JSON parse error, fix it inline and re-run.

---

## Task 4: Write the project `README.md`

**Files:**
- Create: `/Users/standa/Documents/www/test/v8js/README.md`

Replaces upstream's README. Must cover install via PIE, libv8 prereqs, and the ABI caveat.

- [ ] **Step 1: Write `README.md`**

Write `/Users/standa/Documents/www/test/v8js/README.md`:
````markdown
# standa/php-v8js

PIE-installable wrapper for [phpv8/v8js](https://github.com/phpv8/v8js) — V8 JavaScript Engine for PHP.

This repo packages the source from phpv8/v8js's `php8` branch (pinned at SHA `8a39efa3cf3b275e402ddf3c4f6b611a5f69a499`, 2026-04-19) as a Composer package that [PIE (PHP Installer for Extensions)](https://github.com/php/pie) can install directly. The C/C++ source itself is unmodified; this repo only adds packaging files (`composer.json`, `.gitattributes`, CI, README).

Upstream has no `composer.json` and is not on Packagist; this wrapper fills that gap.

## Install with PIE

```bash
# macOS (Homebrew):
brew install v8
pie install standa/php-v8js --with-v8js=$(brew --prefix v8)

# Debian / Ubuntu:
sudo apt-get install libv8-dev
pie install standa/php-v8js --with-v8js=/usr

# Other: build V8 yourself (see https://v8.dev/docs/build) and pass its install prefix:
pie install standa/php-v8js --with-v8js=/opt/v8
```

If `pie` is not yet on your system, follow [its install guide](https://github.com/php/pie#installing-pie). Verify after install:

```bash
php -m | grep v8js          # should print: v8js
php -r 'echo (new V8Js)->executeString("1+2"), PHP_EOL;'   # should print: 3
```

## Configure options

| Flag | Required | What it does |
|---|---|---|
| `--with-v8js=PATH` | Strongly recommended | Path to your V8 install prefix. The upstream `config.m4` auto-searches `/usr/local` and `/usr`, but passing the path explicitly avoids surprises on macOS / non-standard layouts. |

## Prerequisites

V8 itself is **not** distributed here. You must have `libv8` (or Node's bundled V8, on Alpine) installed before `pie install` runs. Recommended sources:

- Debian/Ubuntu `libv8-dev` (current versions ship V8 ~12.x)
- Homebrew `v8` (currently V8 14.x — see "Known limitations" below)
- Alpine: `apk add nodejs-dev` and link against Node's V8 (this is what upstream v8js's own CI does on Alpine)
- Build from source: see [v8.dev/docs/build](https://v8.dev/docs/build), or use [marekskopal/php-v8js-docker](https://github.com/marekskopal/php-v8js-docker) as a reference build (`scripts/build-v8.sh` there is a working `depot_tools` build pipeline for V8 12.9.203 on Debian)

## Known limitations

### V8 14.x is not supported by upstream v8js yet

The `php8` branch builds against V8 10.9, 11.x, 12.x, and 13.x. V8 14.6+ removed `Local::Holder()`, changed `SetAlignedPointerInInternalField`'s signature, and replaced `String::Write` with `WriteV2` — these require source patches in v8js that have not yet been merged. Track [phpv8/v8js#546](https://github.com/phpv8/v8js/issues/546).

This means **Homebrew users on a recent macOS may not get a working build** because Homebrew's `v8` formula is currently 14.8. Workarounds:

1. Use the Docker image from [marekskopal/php-v8js-docker](https://hub.docker.com/r/marekskopal/php-v8js) which pins V8 12.9.203.
2. Build an older V8 from source and pass `--with-v8js=/your/path`.

### libv8 ABI mismatch on prebuilt binaries

Releases of this package ship prebuilt `.so` binaries (built against the libv8 that ships with the CI runner: `ubuntu-24.04`'s `libv8-dev` and `macos-14`'s Homebrew `v8` at build time). If your system's libv8 minor version differs significantly, V8's `Embedder-vs-V8 build configuration mismatch` runtime check may abort `new V8Js()`.

PIE's `download-url-method: ["pre-packaged-binary", "composer-default"]` setting means PIE tries a prebuilt binary first, then falls back to a source build if no matching asset exists for your platform tuple. Most users will hit the source path and avoid the issue entirely.

If you get a prebuilt that doesn't work, please open an issue with your `php -i | head -30` and `apt-cache policy libv8-dev` (or `brew info v8`) output so a matching artifact can be added.

### Windows is not supported

`composer.json` has `os-families-exclude: ["windows"]`. Adding Windows requires DLLs built via [php/php-windows-builder](https://github.com/php/php-windows-builder); not done here. Windows users should use WSL2 + the Linux instructions above, or the Docker image.

## Updating the vendored source

To resync to a newer upstream commit:

```bash
NEW_SHA=<new-upstream-sha>
SCRATCH=$(mktemp -d)
curl -sSL "https://codeload.github.com/phpv8/v8js/tar.gz/${NEW_SHA}" \
  | tar xz -C "$SCRATCH"
rsync -av --exclude='.github' --exclude='README.md' \
  "$SCRATCH/v8js-${NEW_SHA}/" .
rm -rf "$SCRATCH"
# Update README.md's pinned-SHA reference, bump the version tag, commit.
```

## Local development

```bash
composer validate                              # check composer.json
git init && git add . && git commit -m "..."  # one-shot scaffold
pie repository:add path .                      # register this dir as a PIE source
pie build standa/php-v8js:*@dev --with-v8js=/path/to/v8   # source build, no install
pie install standa/php-v8js:*@dev --with-v8js=/path/to/v8 # source build + install
```

## License

MIT — same as upstream phpv8/v8js. See `LICENSE`.

## Credits

All actual extension code is from [phpv8/v8js](https://github.com/phpv8/v8js) and its contributors (see `CREDITS`). This repository only adds packaging metadata.
````

- [ ] **Step 2: Sanity check the README**

Run:
```bash
cd /Users/standa/Documents/www/test/v8js
wc -l README.md
grep -c "standa/php-v8js" README.md
grep -c "8a39efa3" README.md
```
Expected: line count > 50, both grep counts ≥ 1.

---

## Task 5: Write the CI workflow

**Files:**
- Create: `/Users/standa/Documents/www/test/v8js/.github/workflows/release.yml`

Tag-triggered, drafts a release, builds prebuilt `.so` per `{ubuntu-24.04, macos-14} × {php 8.1..8.4} × {nts, ts}` (macOS-ts excluded), uploads via `php/pie-ext-binary-builder@0.0.2`.

- [ ] **Step 1: Create the workflows directory**

Run:
```bash
mkdir -p /Users/standa/Documents/www/test/v8js/.github/workflows
```

- [ ] **Step 2: Write `release.yml`**

Write `/Users/standa/Documents/www/test/v8js/.github/workflows/release.yml`:
```yaml
name: Build and release PIE binaries

on:
  push:
    tags: ['*']

permissions:
  contents: read

jobs:
  create-draft-release:
    runs-on: ubuntu-latest
    permissions:
      # contents:write is required to create the draft release.
      contents: write
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-tags: 'true'
          ref: ${{ github.ref }}
      - name: Create draft release from tag
        env:
          GH_TOKEN: ${{ github.token }}
        run: gh release create "${{ github.ref_name }}" --title "${{ github.ref_name }}" --draft --notes-from-tag

  add-pie-binaries:
    needs: [create-draft-release]
    runs-on: ${{ matrix.operating-system }}
    strategy:
      # Don't cancel other cells if one fails; we want to see the full matrix.
      fail-fast: false
      matrix:
        operating-system: [ubuntu-24.04, macos-14]
        php-versions: ['8.1', '8.2', '8.3', '8.4']
        zts-mode: [nts, ts]
        exclude:
          # Homebrew PHP is NTS only; setup-php cannot install a ZTS PHP on macOS.
          - operating-system: macos-14
            zts-mode: ts
    permissions:
      # contents:write is required to upload to the draft release.
      contents: write
    steps:
      - uses: actions/checkout@v6

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php-versions }}
        env:
          phpts: ${{ matrix.zts-mode }}

      - name: Install libv8 (Linux)
        if: runner.os == 'Linux'
        run: |
          sudo apt-get update
          sudo apt-get install -y libv8-dev
          echo "V8_PREFIX=/usr" >> "$GITHUB_ENV"

      - name: Install libv8 (macOS)
        if: runner.os == 'macOS'
        run: |
          brew install v8
          echo "V8_PREFIX=$(brew --prefix v8)" >> "$GITHUB_ENV"

      - name: Build and release
        id: pie-binary-builder
        uses: php/pie-ext-binary-builder@0.0.2
        with:
          release-tag: ${{ github.ref_name }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          configure-flags: "--with-v8js=${{ env.V8_PREFIX }}"
```

- [ ] **Step 3: Lint with `actionlint` if available, else syntax-check via Python YAML**

`actionlint` may not be installed locally; fall back to YAML parse.

Run:
```bash
cd /Users/standa/Documents/www/test/v8js
if command -v actionlint >/dev/null 2>&1; then
  actionlint .github/workflows/release.yml
else
  python3 -c 'import yaml,sys; yaml.safe_load(open(".github/workflows/release.yml"))' && echo "YAML parses OK"
fi
```
Expected: `YAML parses OK` (or `actionlint` exits 0 with no findings).

---

## Task 6: Refresh the `.gitignore` to keep upstream's rules + our additions

**Files:**
- Modify: `/Users/standa/Documents/www/test/v8js/.gitignore`

The rsync in Task 2 brought in upstream's `.gitignore` and overwrote ours from Task 1. We need to merge them: keep upstream's rules and add our IDE/OS noise back.

- [ ] **Step 1: Verify which `.gitignore` is currently on disk**

Run:
```bash
cd /Users/standa/Documents/www/test/v8js
head -20 .gitignore
```
Expected: upstream's `.gitignore` (extension-specific rules like `.deps/`, `*.lo`, `*.la`, `modules/`).

- [ ] **Step 2: Append our additions if they're not already there**

The upstream `.gitignore` does NOT cover IDE/OS files or composer's `vendor/`. Append:

Run:
```bash
cd /Users/standa/Documents/www/test/v8js
cat >> .gitignore <<'EOF'

# --- standa/php-v8js packaging additions ---
.idea/
.vscode/
*.swp
*.bak
.DS_Store
Thumbs.db
/vendor/
.phpunit.result.cache
EOF
head -50 .gitignore
```
Expected: original upstream rules at top, our additions appended below the marker comment.

---

## Task 7: Initial commit

**Files:**
- All scaffolded files

- [ ] **Step 1: Review the staged set**

Run:
```bash
cd /Users/standa/Documents/www/test/v8js
git add .
git status --short
```
Expected: a list of new files: `.gitattributes`, `.gitignore`, `.github/workflows/release.yml`, `.dockerignore` (vendored), `CREDITS`, `LICENSE`, `Makefile.frag`, `README.md`, `README.Linux.md`, `README.MacOS.md`, `README.Win32.md`, `composer.json`, `config.m4`, `config.w32`, `docs/superpowers/{plans,specs}/*.md`, `package.xml`, `php_v8js.h`, `php_v8js_macros.h`, `tests/*`, `v8js_*.{cc,h}`. All marked `A`.

If anything unexpected shows up (e.g., a `.libs/` directory or a stray `modules/`), STOP and investigate before committing.

- [ ] **Step 2: Commit**

Run:
```bash
git commit -m "$(cat <<'EOF'
Initial scaffold: PIE-installable wrapper for phpv8/v8js

Packages phpv8/v8js@php8 (pinned at 8a39efa3) as standa/php-v8js, a
Composer package installable by pie.phar via type=php-ext.

- Vendors upstream source unmodified at the pinned SHA
- composer.json with extension-name=v8js, --with-v8js configure option,
  download-url-method=[pre-packaged-binary, composer-default],
  os-families-exclude=[windows]
- .github/workflows/release.yml uses php/pie-ext-binary-builder@0.0.2
  over {ubuntu-24.04, macos-14} x {php 8.1..8.4} x {nts, ts}
  (macos-14 + ts excluded; Homebrew PHP is NTS only)
- .gitattributes export-ignores dev-only paths from the archive PIE downloads
- Design spec at docs/superpowers/specs/2026-06-02-v8js-pie-package-design.md
- Implementation plan at docs/superpowers/plans/2026-06-02-v8js-pie-package.md
EOF
)"

git log --oneline
```
Expected: one commit hash followed by the first line of the message.

---

## Task 8: Local verification (best-effort)

**Files:**
- None modified; this is a verification task.

This step exercises the scaffold against the local PIE/PHP install. It is **best-effort**: locally we have PHP 8.5 (not in the CI matrix; composer.json's `^8.1` allows it) and Homebrew's V8 is 14.x (which the upstream `php8` branch may not build against — see issue #546). Even a *failed* `pie build` here is informative because it tells us **where** it fails (composer manifest → resolver → download → configure → make), and a failure at configure/make is a libv8/V8-version issue, not a packaging issue.

If you want a green local build, install an older V8 (12.9.203 known good) per the docker repo's `scripts/build-v8.sh` — out of scope for this plan.

- [ ] **Step 1: Validate `composer.json`**

Already done in Task 3, but re-run to be sure nothing drifted.

Run:
```bash
cd /Users/standa/Documents/www/test/v8js
composer validate --strict
```
Expected: `./composer.json is valid` (possibly with a no-version warning).

- [ ] **Step 2: Register the local repo as a PIE path repository**

Run:
```bash
pie repository:add path /Users/standa/Documents/www/test/v8js
```
Expected: a message confirming the path repository was added, plus PIE listing both the path repo and Packagist.

If PIE complains about the path repository already being added, skip ahead.

- [ ] **Step 3: Resolve the package (no build yet)**

We want to confirm PIE finds and parses our `composer.json` correctly before attempting an actual build.

Run:
```bash
pie show standa/php-v8js:*@dev 2>&1 | head -20
```
Expected: PIE prints package metadata including `extension-name: v8js`, the configure option, and a version like `dev-main`. If it errors with "package not found", recheck Task 7's commit was made (PIE may need a committed state to read the path repo).

- [ ] **Step 4: Attempt source build (best-effort, may fail on V8 14.x)**

```bash
# Try with Homebrew V8 path; --force may be needed because pie 1.3.10 occasionally
# refuses to build if it thinks something is "missing". If brew v8 is not installed,
# install it first: brew install v8.

V8_PREFIX="$(brew --prefix v8 2>/dev/null || echo /usr/local)"
pie build "standa/php-v8js:*@dev" --with-v8js="$V8_PREFIX" 2>&1 | tee /tmp/pie-build.log | tail -40
```

**Outcomes and what each means:**

- **`pie build` succeeds** → `.so` is in PIE's per-PHP build dir. Excellent. You can do `pie install` next.
- **Fails at `phpize`** → toolchain issue (autoconf, etc.). Unrelated to this scaffold.
- **Fails at `./configure` with "Please reinstall the v8 distribution"** → libv8 not at the path you passed. Install V8 or adjust `--with-v8js=`.
- **Fails at `make` with errors like `'Holder' is not a member of 'v8::Local<v8::Object>'`** → known V8 14.x incompatibility (#546). Out of scope for this scaffold; install V8 12.x instead.
- **Fails at the PIE composer-resolver stage** → packaging bug. STOP and inspect the resolver log; the scaffold needs a fix.

- [ ] **Step 5: Record the outcome**

Run:
```bash
echo "--- pie build outcome ---" >> /tmp/v8js-scaffold-notes.txt
tail -5 /tmp/pie-build.log >> /tmp/v8js-scaffold-notes.txt
cat /tmp/v8js-scaffold-notes.txt
```

If the build succeeded, mark the scaffold complete. If it failed at make/configure due to V8 version mismatch, also mark the scaffold complete (the failure is upstream, not us).

If it failed at the resolver, this plan has a packaging bug — investigate `composer.json` for typos / wrong field names.

---

## Self-review

Spec coverage check:

| Spec section | Implemented by |
|---|---|
| Repo layout | Tasks 1 (dotfiles), 2 (vendor), 3 (composer.json), 4 (README), 5 (CI) |
| composer.json field-by-field | Task 3 |
| CI workflow | Task 5 |
| README content (install, prereqs, ABI caveat, resync, Windows exclusion) | Task 4 |
| Local verification flow | Task 8 |
| Deferred decisions (version numbering, Packagist publish, PHP 8.5, libv8 bundling) | Documented in spec, not blocking |

Placeholder scan: no TBD/TODO/placeholders in any task. Every file has its full content inline.

Type consistency: `standa/php-v8js` (package name) and `v8js` (extension name) appear consistently across composer.json, README, and the commit message. The pinned SHA `8a39efa3cf3b275e402ddf3c4f6b611a5f69a499` appears identically in spec, plan, README, and commit message.

Risks the plan accepts:
- Local smoke test may fail on V8 14 (Homebrew default). This is a libv8 issue, not a packaging issue. The plan calls this out explicitly in Task 8 outcome notes.
- pie 1.3.10 is installed locally; the PIE docs are from 1.5.x. The core `php-ext` / `php-ext.*` schema is stable; if 1.3 rejects a field, drop it and re-validate. We'll know at Task 8 Step 3 (`pie show`).
