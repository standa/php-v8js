# standa/php-v8js

PIE-installable wrapper for [phpv8/v8js](https://github.com/phpv8/v8js) — V8 JavaScript Engine for PHP.

This repo packages the source from phpv8/v8js's `php8` branch (pinned at SHA `8a39efa3cf3b275e402ddf3c4f6b611a5f69a499`, 2026-04-19) as a Composer package that [PIE (PHP Installer for Extensions)](https://github.com/php/pie) can install directly. The C/C++ source itself is unmodified; this repo only adds packaging files (`composer.json`, `.gitattributes`, CI, README).

Upstream has no `composer.json` and is not on Packagist; this wrapper fills that gap.

## Install with PIE

> **⚠️ macOS users:** Homebrew's `v8` formula is currently 14.x, which the
> upstream `php8` branch cannot build against yet (tracked at
> [phpv8/v8js#546](https://github.com/phpv8/v8js/issues/546)). Until that's
> resolved, the easiest macOS path is the Docker image at
> [`marekskopal/php-v8js`](https://hub.docker.com/r/marekskopal/php-v8js).
> If you want a native install, see *Known limitations → V8 14.x is not
> supported* below for how to use V8 12.x.

```bash
# macOS (Homebrew) — see warning above; build will fail until v8js supports V8 14.x:
brew install v8
pie install standa/php-v8js --with-v8js=$(brew --prefix v8)

# Debian / Ubuntu (recommended — verified end-to-end in a php:8.4-cli container):
sudo apt-get install libnode-dev pkg-config
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
