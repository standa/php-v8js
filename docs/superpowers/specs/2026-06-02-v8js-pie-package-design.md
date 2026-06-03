# Design: `standa/php-v8js` — a PIE-installable wrapper for phpv8/v8js

**Date:** 2026-06-02
**Owner:** standa
**Status:** approved (pending spec review)

## Goal

Prepare a local repository at `/Users/standa/Documents/www/test/v8js/` that
packages the upstream [phpv8/v8js](https://github.com/phpv8/v8js) PHP
extension as a Composer/Packagist package installable by
[pie.phar](https://github.com/php/pie). The repo MUST be installable today
via `pie repository:add path .` + `pie install standa/php-v8js:*@dev`, and
SHOULD be ready to publish to Packagist and to ship prebuilt binaries via
[php/pie-ext-binary-builder](https://github.com/php/pie-ext-binary-builder)
once pushed to GitHub.

## Non-goals

- Building or distributing V8 itself. End users must have `libv8` available
  on their system (`apt install libv8-dev`, `brew install v8`, or a custom
  build). This package only wraps the v8js *extension*, not the engine.
- Windows support. Excluded via `os-families-exclude: ["windows"]` in
  `composer.json`. Adding it later means producing DLLs via
  [php/php-windows-builder](https://github.com/php/php-windows-builder) and
  removing the exclusion; out of scope here.
- Upstream maintenance of phpv8/v8js. This is a vendored snapshot at a
  pinned SHA; resyncing to a new upstream commit is a manual re-vendor
  operation (overwrite source files, bump version, tag).
- Publishing to Packagist or Docker Hub. The repo will be Packagist-ready
  but actual submission is a manual step the maintainer performs.

## Anchor facts (verified against upstream on 2026-06-02)

- **Upstream branch:** `phpv8/v8js@php8`, latest commit
  `8a39efa3cf3b275e402ddf3c4f6b611a5f69a499` (2026-04-19, "Merge pull
  request #545 from redbullmarky/php8 — Fixing memory leaks and
  deprecations").
- **Upstream has NO `composer.json`** and is **not on Packagist**. Latest
  formal tag is `2.1.2` but the `php8` branch is ahead of it and has the
  PHP 8.4 deprecation fixes.
- **Extension type:** standard PHP module (not a Zend extension). The
  registered module entry is in `v8js_main.cc`.
- **Extension name:** `v8js`.
- **Build system:** `phpize` + `./configure` + `make`. `config.m4` is at
  the repo root.
- **Configure options:** a single `PHP_ARG_WITH(v8js, ...)` macro,
  surfaced as `--with-v8js[=DIR]`. `config.m4` auto-searches `/usr/local`
  and `/usr` for `libv8.so` / `libnode.so` when no path is given.
- **License:** MIT.

## Approach (decided)

1. **Snapshot-copy** the upstream `php8` branch source files into the new
   repo. Not a submodule (`git archive` strips submodules and would break
   PIE's default `composer-default` download method), not a subtree
   (operationally heavier with no benefit at this scale).
2. **Add a PIE-compliant `composer.json`** declaring `type: php-ext`, an
   explicit `extension-name: v8js`, the single `--with-v8js` configure
   option, and `download-url-method: ["pre-packaged-binary",
   "composer-default"]` so prebuilt binaries are tried first with a
   source-build fallback.
3. **Add a fully active `release.yml`** that runs on every tag, drafts a
   GitHub release, and uses `php/pie-ext-binary-builder@0.0.2` to publish
   `.so` binaries for `{ubuntu-24.04, macos-14} × {php 8.1..8.4} × {nts, ts}`
   (with `macos-14 × ts` excluded since Homebrew PHP is NTS-only).
4. **Document the libv8 ABI risk** in the README so users with a different
   libv8 minor version than the CI builder don't get blindsided by the
   `Embedder-vs-V8 build configuration mismatch` runtime abort. The
   `composer-default` fallback handles them by rebuilding from source on
   their host.

## Repository layout

```
v8js/
├── .gitattributes              # export-ignore docs/, .github/, README.* dev-only
├── .gitignore                  # build artifacts (modules/, .libs/, *.lo, autom4te.cache/)
├── .github/workflows/release.yml
├── composer.json
├── LICENSE                     # MIT, copied verbatim from upstream
├── README.md                   # install instructions, libv8 prereqs, ABI note
├── docs/superpowers/specs/2026-06-02-v8js-pie-package-design.md   # this file
├── CREDITS                     # vendored
├── config.m4                   # vendored
├── config.w32                  # vendored
├── Makefile.frag               # vendored
├── package.xml                 # vendored (kept for PECL bridge / historical)
├── php_v8js.h                  # vendored
├── php_v8js_macros.h           # vendored
├── v8js_array_access.{cc,h}    # vendored
├── v8js_class.{cc,h}           # vendored
├── v8js_commonjs.{cc,h}        # vendored
├── v8js_convert.cc             # vendored
├── v8js_exceptions.{cc,h}      # vendored
├── v8js_generator_export.{cc,h}# vendored
├── v8js_main.cc                # vendored
├── v8js_methods.cc             # vendored
├── v8js_object_export.{cc,h}   # vendored
├── v8js_timer.{cc,h}           # vendored
├── v8js_v8.{cc,h}              # vendored
├── v8js_v8object_class.{cc,h}  # vendored
├── v8js_variables.cc           # vendored
├── README.Linux.md             # vendored (build prereqs for Linux)
├── README.MacOS.md             # vendored
├── README.Win32.md             # vendored (kept for reference even though Win is excluded)
└── tests/                      # vendored .phpt files
```

The "vendored" files are copied byte-for-byte from
`https://raw.githubusercontent.com/phpv8/v8js/8a39efa3cf3b275e402ddf3c4f6b611a5f69a499/<path>`.

## `composer.json`

```json
{
    "name": "standa/php-v8js",
    "description": "V8 JavaScript Engine for PHP (PIE-installable wrapper around phpv8/v8js, php8 branch)",
    "type": "php-ext",
    "license": "MIT",
    "keywords": ["v8", "javascript", "php-ext", "pie", "extension"],
    "require": {
        "php": "^8.1"
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

Rationale for each non-obvious field:

- **`extension-name: "v8js"`** — the package name contains a hyphen, so
  PIE's auto-derived extension name (`php-v8js`) would be invalid. Set it
  explicitly so `php -m` shows `v8js` and `extension=v8js` works.
- **`require.php: "^8.1"`** — tightened from `^8.0` to match the CI
  matrix (8.1–8.4). PHP 8.0 is EOL (Nov 2023) and not actively tested by
  upstream or by this repo's CI; better to refuse cleanly than to ship a
  half-tested claim.
- **`require.php-64bit` is deliberately NOT set.** Empirical finding from
  local verification on PIE 1.4.5 (2026-06): PIE's resolver does not
  synthesize the `php-64bit` virtual platform package into its target-PHP
  composer manifest, so any package declaring `"php-64bit": "*"` is
  unconditionally rejected ("missing from your platform") even on
  unambiguously 64-bit hosts. `--ignore-platform-req=php-64bit` is not
  honored. The original idea (reject 32-bit PHP up front) doesn't work as
  long as the constraint never resolves. Drop it; users on 32-bit PHP
  would surface a confusing link error, but 32-bit PHP is essentially
  extinct in 2026 and we don't pay any practical cost.
- **`download-url-method`** — try prebuilt binary first, fall back to
  source compile if no matching asset exists.
- **`os-families-exclude: ["windows"]`** — see Non-goals.

## CI: `.github/workflows/release.yml`

The workflow is the example from the pie-ext-binary-builder README, plus
two project-specific additions: a libv8 install step, and a matrix exclude
for the `macos × ts` cell (Homebrew PHP is NTS-only).

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
      fail-fast: false
      matrix:
        operating-system: [ubuntu-24.04, macos-14]
        php-versions: ['8.1', '8.2', '8.3', '8.4']
        zts-mode: [nts, ts]
        exclude:
          - operating-system: macos-14
            zts-mode: ts
    permissions:
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
        uses: php/pie-ext-binary-builder@0.0.2
        with:
          release-tag: ${{ github.ref_name }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          configure-flags: "--with-v8js=${{ env.V8_PREFIX }}"
```

**Matrix:** 2 OS × 4 PHP × 2 ZTS = 16 cells, minus 4 excluded macOS-TS
cells = **12 build cells per tag**.

**Initial PHP version range:** 8.1–8.4. PHP 8.5 is intentionally omitted
until verified locally — upstream v8js CI does not run against 8.5 yet
(per marekskopal/php-v8js-docker's notes). Add 8.5 to the matrix after a
local smoke test passes.

## Documentation: `README.md`

The README must cover:

1. **What this package is** — a PIE-installable wrapper around phpv8/v8js
   at the pinned upstream SHA. One-line, with a link to upstream.
2. **Install with PIE** — `pie install standa/php-v8js`. List the
   `--with-v8js=PATH` option.
3. **Prerequisites: libv8 on the host system.**
   - Debian/Ubuntu: `sudo apt install libv8-dev`
   - macOS (Homebrew): `brew install v8`, then
     `pie install standa/php-v8js --with-v8js=$(brew --prefix v8)`
   - Other: build V8 from source (link to marekskopal/php-v8js-docker for
     a reference build).
4. **The libv8 ABI caveat.** If a prebuilt binary is installed but the
   user's libv8 minor version doesn't match the CI builder's, `new V8Js()`
   may abort with "Embedder-vs-V8 build configuration mismatch". The
   `composer-default` fallback in `download-url-method` means PIE will
   automatically rebuild from source if no matching prebuilt asset exists
   for the user's platform — so most users hit a source build, not the
   abort. If a user does get a non-working prebuilt (their platform tuple
   matches a published asset but their libv8 differs), the documented
   workaround is to file an issue with their `php -i` and `apt-cache
   policy libv8-dev` (or `brew info v8`) output so a matching asset can
   be added.
5. **Upstream source pin** — the exact SHA the vendored code came from.
   A short note on how to resync.
6. **License** — MIT, same as upstream.

## Local verification flow

Once the scaffold is in place, the maintainer verifies it works *before
any push* by running:

```bash
cd /Users/standa/Documents/www/test/v8js
composer validate                                  # composer.json is valid
git init && git add . && git commit -m "Initial scaffold"
pie repository:add path .

# macOS (Homebrew):
pie build standa/php-v8js:*@dev --with-v8js=$(brew --prefix v8)

# Linux (apt-installed libv8-dev):
# pie build standa/php-v8js:*@dev --with-v8js=/usr
```

A successful local build produces `modules/v8js.so`. `pie install` (instead
of `pie build`) does the same plus installs the `.so` into the active PHP
installation's extension dir and adds `extension=v8js` to a PHP INI file.

## Decisions explicitly NOT made here (deferred to implementation or later)

- **Repo version numbering scheme.** Whether the first tag is `v0.1.0`,
  `v2.1.3` (continuing upstream's numbering), or `v3.0.0-php8`. Pick at
  first tag; PIE doesn't care as long as it's a valid SemVer string.
- **Whether to publish to Packagist now or hold.** Local install via
  `pie repository:add path .` works without Packagist.
- **PHP 8.5 support.** Defer until a local smoke test confirms the php8
  branch compiles cleanly on 8.5.
- **Bundling libv8 into the prebuilt binary.** Would eliminate the ABI
  mismatch problem but requires shipping a static libv8 or vendoring V8's
  multi-hour build into CI. Out of scope for v1; reconsider if users
  report mismatch issues frequently.

## Risks and how this design responds to them

| Risk | Response |
|---|---|
| libv8 ABI mismatch aborts `new V8Js()` for users on a different libv8 minor than CI's | `download-url-method` has `composer-default` fallback → PIE rebuilds from source automatically; README documents the manual `--force-source` escape hatch |
| Upstream phpv8/v8js advances; vendored snapshot drifts | Resync = overwrite source files at a new pinned SHA, bump version, tag. Re-vendor is a documented manual operation. |
| Windows users try to install and get an opaque error | `os-families-exclude: ["windows"]` makes PIE refuse cleanly with a clear message |
| `--with-v8js` not passed: config.m4 leaves PHP_V8JS=no and build skips | `needs-value: true` on the configure option strongly encourages passing a path; README's quick-start always shows it with a path |
| pie-ext-binary-builder action version 0.0.2 gets a breaking update | Pinned by tag; bump deliberately after testing |
