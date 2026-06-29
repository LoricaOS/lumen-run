# lumen-run

The "Run…" application launcher for **AspisOS**, a capability-based,
no-ambient-authority operating system built on the from-scratch
[Aegis](https://github.com/AspisOS/Aegis) kernel.

lumen-run is a small chromed dialog with a single text field: type an
application name and it launches the match. It is an external client of the
[lumen](https://github.com/AspisOS/lumen) compositor, distributed as a
[herald](https://github.com/AspisOS/AspisOS) package and installed as an `/apps`
bundle. Its descriptor's display name is **Run**.

## The AspisOS ecosystem

AspisOS is decomposed into independent repositories; lumen-run is one graphical
leaf of that tree:

| Repo | Role |
|------|------|
| [`AspisOS/Aegis`](https://github.com/AspisOS/Aegis) | The kernel. Provides the capability model and the `AF_UNIX` socket lumen-run uses to talk to the compositor. |
| [`AspisOS/lumen`](https://github.com/AspisOS/lumen) | The compositor / display server. lumen-run connects to its socket for a window and asks it to launch the chosen app via the invoke protocol. |
| [`AspisOS/glyph`](https://github.com/AspisOS/glyph) | The GUI toolkit. Supplies the renderer, the theme, the `/apps` bundle registry (`glyph_apps_scan`) that lumen-run matches against, and the client side of lumen's window protocol (`lumen_client.h`). |
| [`AspisOS/AspisOS`](https://github.com/AspisOS/AspisOS) | The OS: userland, rootfs, ISO/installer, and the herald package manager that installs this `.hpkg`. |

## What it does

Grounded in `src/main.c`:

- **One dialog.** Connects to lumen and draws a single 460×120 window with one
  text field and a hint line.
- **Live matching.** Each keystroke matches the typed text against the `/apps`
  bundle registry (`glyph_apps_scan`) — a case-insensitive prefix match on
  either the bundle id or its display name. The first match is shown on the
  hint line ("↵ <name>"); no match shows "No matching application".
- **Launch.** Enter asks the compositor to spawn the matched bundle via
  `lumen_invoke` (`LUMEN_OP_INVOKE`), then the dialog exits. Esc or the close
  button dismisses it.
- **GUI bundles only.** lumen-run launches `/apps` bundles, which is what
  lumen's invoke mechanism resolves. CLI commands need a terminal
  (`/apps/terminal`), so there is deliberately no arbitrary-command path here.

## Capabilities

AspisOS has no ambient authority: a process can do nothing except through
capabilities granted at exec time. lumen-run's policy
(`pkg/etc/aegis/caps.d/run`) is the baseline desktop-app profile:

```
service
```

It carries no elevated capabilities of its own. Notably, lumen-run never spawns
a process itself — launching the selected app is *delegated to the compositor*
through the invoke protocol, so the launcher needs nothing beyond a socket
connection to lumen.

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
- `HERALD_KEY` signs the `.hpkg` (ECDSA P-256).

Output: `lumen-run.hpkg` (a `class=system` herald package) +
`lumen-run.hpkg.sig`.

## Package payload

The `.hpkg` is a manifest-first, uncompressed POSIX `ustar` archive with a
detached signature. Its payload tree:

```
/apps/run/run                        the app binary (stripped)
/apps/run/app.ini                    the bundle descriptor (name=Run, exec=run)
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
[lumen](https://github.com/AspisOS/lumen) (which in turn ships the desktop fonts
every dependent inherits).

## Status

Early-stage and intentionally minimal: a single-field launcher with prefix
matching and no history, fuzzy ranking, or argument passing. It does one thing —
turn a typed name into a running app — and is expected to grow (recents,
multi-result selection) as AspisOS matures.
