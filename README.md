# lumen-run

The "Run…" application launcher for **AspisOS**, a capability-based,
no-ambient-authority operating system built on the from-scratch
[Aegis](https://github.com/AspisOS/Aegis) kernel.

lumen-run is a small chromed dialog with a single text field: type an
application name and it launches the match. It speaks the
[lumen](https://github.com/AspisOS/lumen) external-window protocol, is a
standalone component of the Lumen desktop, and is distributed as a
[herald](https://github.com/AspisOS/AspisOS) package installed as an `/apps`
bundle.

## Role in the system

- An ordinary Lumen client: it connects to the compositor and draws a single
  460x120 dialog (its descriptor's display name is **Run**).
- The text field live-matches against the `/apps` bundle registry
  (`glyph_apps_scan`), case-insensitive prefix on either the bundle id or its
  display name; the first match is shown on a hint line.
- Enter asks Lumen to spawn the matched bundle via `lumen_invoke`
  (`LUMEN_OP_INVOKE`), then the dialog exits. Esc or the close button dismisses.
- It launches GUI app bundles only — that is what Lumen's invoke mechanism
  resolves. CLI commands need a terminal, so there is deliberately no
  arbitrary-command path.

## Capabilities

lumen-run's cap policy (`pkg/etc/aegis/caps.d/run`) is the baseline desktop-app
profile:

```
service
```

It carries no elevated capabilities of its own; launching the selected app is
delegated to the compositor via the invoke protocol, so lumen-run never spawns
processes directly.

Because its herald package id (`lumen-run`) differs from the bundle/exec name
(`run`) and it installs a binary plus a cap policy and an app descriptor across
`/apps` and `/etc`, it is a `class=system` package: first-party and
signature-trusted, installed verbatim by herald.

## Building

lumen-run fetches a pinned [glyph](https://github.com/AspisOS/glyph) toolkit
artifact (the GUI libraries it links) and builds against it, then packs a signed
herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

- `GLYPH_VERSION` pins the toolkit release fetched by `tools/fetch-glyph.sh`.
- `MUSL_CC` is the musl cross-compiler (the only toolchain assumption — point it
  at an Aegis-native `cc` to build on-device in the future).
- `HERALD_KEY` signs the `.hpkg`.

Output: `lumen-run.hpkg` (a `class=system` herald package) +
`lumen-run.hpkg.sig`.

## Package payload

```
/apps/run/run                        the app binary
/apps/run/app.ini                    the bundle descriptor
/etc/aegis/caps.d/run                its capability policy
```

## Repository layout

```
src/        run source
pkg/        install-tree skeleton shipped verbatim (app.ini + caps.d)
tools/      fetch-glyph.sh (toolkit fetch) + pack.sh (build the signed .hpkg)
Makefile    fetch toolkit -> build -> pack
VERSION         this component's version
GLYPH_VERSION   the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — lumen-run is a Lumen client and resolves the `/apps` registry
the compositor manages, so installing it pulls
[lumen](https://github.com/AspisOS/lumen) (which in turn provides the desktop
fonts).
