# standa/php-v8js

PIE-installable wrapper for [phpv8/v8js](https://github.com/phpv8/v8js) — V8 JavaScript Engine for PHP.

This repo packages the source from phpv8/v8js's `php8` branch (pinned at SHA `8a39efa3cf3b275e402ddf3c4f6b611a5f69a499`, 2026-04-19) as a Composer package that [PIE (PHP Installer for Extensions)](https://github.com/php/pie) can install directly. The C/C++ source itself is unmodified; this repo only adds packaging files (`composer.json`, `.gitattributes`, CI, README).

Upstream has no `composer.json` and is not on Packagist; this wrapper fills that gap.

## Supported PHP versions

The CI matrix builds and tests against **PHP 8.1, 8.2, 8.3, 8.4, and 8.5** on Linux (x86_64 and arm64, NTS and TS). 8.5 is the current stable as of June 2026; upstream phpv8/v8js's own CI doesn't cover 8.5 yet, so this package is the first place 8.5 gets continuous coverage.

PHP 8.0 is EOL (Nov 2023) and explicitly out of scope. macOS is currently unable to publish prebuilt binaries (V8 14.x incompatibility — see *Known limitations*).

## Install with PIE

### Fast path — prebuilt binary

The asset that gets downloaded depends on your platform tuple — PIE picks the right one automatically. **Do not pass `--with-v8js`** on this path: PIE refuses prebuilts when configure options are passed (it can't know whether the prebuilt was built with the flag you wanted) and falls back to a source build instead.

```bash
# Debian trixie / php:X.Y-cli/fpm/apache / Ubuntu 25.04+:
sudo apt-get install -y libnode-dev unzip
pie install standa/php-v8js

# Alpine / php:X.Y-cli-alpine / php:X.Y-fpm-alpine:
apk add --no-cache nodejs-dev unzip
pie install standa/php-v8js
```

The prebuilt was already linked against `/usr/lib/libnode.so.*` from the matching distro — no `--with-v8js` needed.

### What `pie install` does (no flags, the prebuilt path)

1. Resolves `standa/php-v8js` from Packagist
2. Detects your platform tuple — `php<X.Y>-<arch>-<os>-<libc>-<tsmode>` — e.g. `php8.4-arm64-linux-glibc-nts`
3. Matches and downloads the prebuilt `.so` archive from the release assets (asset name: `php_v8js-<ver>_php<X.Y>-<arch>-<os>-<libc>-<tsmode>.zip`)
4. Extracts and installs the `.so` into the active PHP's extension directory; enables it via an INI file

No `phpize`, no `./configure`, no `make`. ~10 seconds.

### What prebuilt binaries ship per release

Each tag produces these binaries via `.github/workflows/release.yml`:

| Platform | PHP versions | NTS | TS | libnode SOVERSION |
|---|---|---|---|---|
| `linux-glibc-x86_64` (Debian trixie / `php:X.Y-cli`) | 8.1, 8.2, 8.3, 8.4, 8.5 | ✅ | ✅ | `libnode.so.115` |
| `linux-glibc-arm64`  (Debian trixie / `php:X.Y-cli`) | 8.1, 8.2, 8.3, 8.4, 8.5 | ✅ | ✅ | `libnode.so.115` |
| `linux-musl-x86_64`  (Alpine / `php:X.Y-cli-alpine`) | 8.1, 8.2, 8.3, 8.4, 8.5 | ✅ | ✅ | `libnode.so.137` |
| `linux-musl-arm64`   (Alpine arm64) | — | ❌ | ❌ (GitHub Actions doesn't currently run JS actions inside arm64+musl containers — node20 runtime injection fails. Source build via `--with-v8js=/usr` is the fallback.) | — |
| `darwin-arm64` (macOS Apple Silicon) | — | ❌ | ❌ (until [v8js#546](https://github.com/phpv8/v8js/issues/546) lands) | — |
| Windows | — | ❌ | ❌ (out of scope — would need [php/php-windows-builder](https://github.com/php/php-windows-builder)) | — |

**30 Linux binaries per release**, each built inside the matching `php:X.Y-{cli,zts}{,-alpine}` container so the `.so` links against the libnode SOVERSION that actually ships with that base image. Users on hosts with a different SOVERSION (Ubuntu 22.04/24.04 = `.109`, custom Node build) won't be able to dynamically load these binaries — see the *source-build escape hatch* below.

### Source-build escape hatch (any libv8/libnode version)

If you're on a host without a matching libnode SOVERSION (Ubuntu 22.04/24.04 = `.109`, custom Node build) — or PIE installs a prebuilt that fails to load at startup with `cannot open shared object file` — pass `--with-v8js=PATH` to force a source build against your local V8/Node headers:

```bash
# Debian/Ubuntu (any version where libnode-dev is available):
sudo apt-get install -y libnode-dev pkg-config build-essential autoconf libtool
pie install standa/php-v8js --with-v8js=/usr

# Alpine (any version where nodejs-dev is available):
apk add --no-cache build-base autoconf libtool m4 pkgconfig nodejs-dev
pie install standa/php-v8js --with-v8js=/usr

# macOS (Homebrew — see warning below; build will fail until v8js supports V8 14.x):
brew install v8
pie install standa/php-v8js --with-v8js=$(brew --prefix v8)

# Custom V8 build (when you've built V8 from source):
pie install standa/php-v8js --with-v8js=/opt/v8
```

The `--with-v8js=PATH` flag disables the prebuilt-binary download (PIE skips prebuilts when configure options are passed), then PIE downloads the source archive and runs `phpize → ./configure → make` against your local V8/Node headers. Slower (~30–60s) but works on any libv8/libnode SOVERSION.

> **⚠️ macOS users:** Homebrew's `v8` formula is currently 14.x, which the
> upstream `php8` branch cannot build against yet (tracked at
> [phpv8/v8js#546](https://github.com/phpv8/v8js/issues/546)). Until that's
> resolved, the easiest macOS path is the Docker image at
> [`marekskopal/php-v8js`](https://hub.docker.com/r/marekskopal/php-v8js).
> If you want a native install, see *Known limitations → V8 14.x is not
> supported* below for how to use V8 12.x.

### Verify the install

```bash
php -m | grep v8js          # should print: v8js
php -r 'echo (new V8Js)->executeString("1+2"), PHP_EOL;'   # should print: 3
```

### If `pie` isn't installed yet

Follow [PIE's install guide](https://github.com/php/pie#installing-pie). On macOS: `brew install pie`. On Debian/Ubuntu the quickest path is:

```bash
curl -sSL https://github.com/php/pie/releases/latest/download/pie.phar -o /usr/local/bin/pie
chmod +x /usr/local/bin/pie
pie --version    # should print: 1.4.x or newer
```

PIE 1.4+ is required (1.3.x doesn't recognize the `php-64bit` platform package, among other resolver fixes).

## Configure options

| Flag | Required | What it does |
|---|---|---|
| `--with-v8js=PATH` | Strongly recommended | Path to your V8 install prefix. The upstream `config.m4` auto-searches `/usr/local` and `/usr`, but passing the path explicitly avoids surprises on macOS / non-standard layouts. |

## Prerequisites

V8 itself is **not** distributed here. You must have V8 headers + a linkable `libv8` (or, more commonly today, Node's bundled V8) installed before `pie install` runs. Recommended sources:

- **Debian / Ubuntu `libnode-dev`** — installs V8 headers at `/usr/include/node/v8.h` and the V8 ABI via `libnode.so`. Upstream's `config.m4` searches both `libv8.so` and `libnode.so`, so `--with-v8js=/usr` works. This is also what upstream v8js's own CI uses for its non-Alpine Linux job. Note: Debian's `libv8-dev` was removed years ago — `libnode-dev` is the supported path.
- **Homebrew `v8`** (currently V8 14.x — see "Known limitations" below)
- **Alpine** `apk add nodejs-dev` and link against Node's V8 (same idea as Debian's `libnode-dev`; this is what upstream v8js's own CI does on Alpine)
- **Build from source**: see [v8.dev/docs/build](https://v8.dev/docs/build), or use [marekskopal/php-v8js-docker](https://github.com/marekskopal/php-v8js-docker) as a reference build (`scripts/build-v8.sh` there is a working `depot_tools` build pipeline for V8 12.9.203 on Debian)

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
composer validate                                            # check composer.json
pie repository:add path .                                    # register this dir as a PIE source
pie build 'standa/php-v8js:*@dev' --with-v8js=/path/to/v8    # source build, no install
pie install 'standa/php-v8js:*@dev' --with-v8js=/path/to/v8  # source build + install
```

## License

MIT — same as upstream phpv8/v8js. See `LICENSE`.

## Credits

All actual extension code is from [phpv8/v8js](https://github.com/phpv8/v8js) and its contributors (see `CREDITS`). This repository only adds packaging metadata.
