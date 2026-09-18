# sioyek-dev — local Arch package

![build](https://github.com/hyperverse/sioyek-pkg/actions/workflows/build.yml/badge.svg)

Local [Arch Linux](https://archlinux.org) packaging for
[**sioyek**](https://github.com/ahrm/sioyek) — a PDF/EPUB viewer focused on
textbooks and research papers — built from the upstream **development** branch.

> This is a personal/local package, **not** published to the AUR. It builds
> sioyek against the **bundled mupdf** (statically linked) and installs a clean
> FHS layout under `/usr` and `/etc`.

---

## Contents

- [Installed files](#installed-files)
- [Dependencies](#dependencies)
- [Build](#build)
  - [How it builds](#how-it-builds)
  - [How sioyek finds files at runtime](#how-sioyek-finds-files-at-runtime)
- [Install](#install)
- [Update](#update)
- [Optional: `pacman -S sioyek-dev`](#optional-pacman--s-sioyek-dev)
- [Repo files](#repo-files)
- [Notes](#notes)
- [License](#license)
- [Attribution](#attribution)

---

## Installed files

```text
usr/bin/sioyek
usr/share/applications/sioyek.desktop
usr/share/pixmaps/sioyek-icon-linux.png
usr/share/sioyek/shaders/*            # 21 shader files
usr/share/sioyek/tutorial.pdf
usr/share/man/man1/sioyek.1.gz
etc/sioyek/keys.config                 # system-wide defaults
etc/sioyek/prefs.config
```

The package declares `provides=(sioyek)` and `conflicts=(sioyek)`, so it
replaces any AUR `sioyek` / `sioyek-git` install.

## Dependencies

**Runtime** (`depends`):

- `qt6-base`
- `qt6-declarative`
- `qt6-svg`
- `qt6-speech`
- `harfbuzz`
- `zlib`
- `libglvnd`

**Build** (`makedepends`):

- `git`

**Optional** (`optdepends`):

- `qt6-wayland` — native Wayland platform plugin (recommended on Wayland/niri).
- `speech-dispatcher` — text-to-speech backend.
- `espeak-ng` — a voice for `speech-dispatcher`.

> mupdf is built from the submodule and **statically linked**, so it is *not* a
> runtime dependency. The package ships only the binary plus data files.

## Build

```bash
cd /path/to/sioyek-pkg
makepkg -s          # -s installs any missing makedepends (needs sudo for deps)
```

This clones `https://github.com/ahrm/sioyek#branch=development`, initializes
submodules, builds mupdf, builds sioyek, and produces
`sioyek-dev-<ver>-x86_64.pkg.tar.zst`.

### How it builds

The important details, in case anything needs adjusting later:

- **mupdf** — `make -C mupdf USE_SYSTEM_HARFBUZZ=yes`
  - Uses system `harfbuzz`.
  - Bundles everything else (freetype, jpeg, openjpeg, jbig2dec, gumbo, mujs,
    lcms2, curl, brotli, …) and links statically into the sioyek binary.

- **sioyek** — `qmake6 "CONFIG+=linux_app_image" "DEFINES+=LINUX_STANDARD_PATHS"`
  then `make`.
  - `CONFIG+=linux_app_image` keeps the **local-mupdf** link line
    (`-Lmupdf/build/release -lmupdf -lmupdf-third -lmupdf-threads -lharfbuzz -lz`).
  - `DEFINES+=LINUX_STANDARD_PATHS` makes the binary read defaults from
    `/etc/sioyek` and read-only data from `/usr/share/sioyek` (FHS), instead of
    looking next to the executable.

- **Submodules** — `git submodule update --init --recursive` (**no `--remote`**).
  - Most nested submodules declare `branch = artifex` with `shallow = true`.
  - Using `--remote` makes git look for `refs/remotes/origin/artifex`, which the
    shallow clone never fetched, and aborts with:
    `fatal: Unable to find refs/remotes/origin/artifex`.
  - Without `--remote`, git checks out the exact pinned SHAs.

- **Config path quirk** — the `.pro` installs `keys.config`/`prefs.config` under
  `$$PREFIX/etc/sioyek` = `/usr/etc/sioyek`, but the binary (with
  `LINUX_STANDARD_PATHS`) reads `/etc/sioyek`. The `package()` function therefore
  drops `usr/etc` and installs the two configs to `/etc/sioyek` manually.

### How sioyek finds files at runtime

`parent_path = QCoreApplication::applicationDirPath()` (the real exe dir,
symlinks resolved). There are two modes:

| Mode | default config / keys | shaders / tutorial | user data | user overrides |
|------|-----------------------|--------------------|-----------|-----------------|
| **FHS** (`LINUX_STANDARD_PATHS`, this package) | `/etc/sioyek/` | `/usr/share/sioyek/` | `~/.local/share/sioyek/` (db, last doc) | `~/.config/sioyek/`, `~/.local/share/sioyek/` (`prefs_user.config`, `keys_user.config`) |
| **Portable** (no `LINUX_STANDARD_PATHS`) | next to binary (`parent_path`) | next to binary | `~/.local/share/sioyek/` | `~/.config/sioyek/`, `~/.local/share/sioyek/` |

Key facts:

- There is **no env var** to redirect the *default* config location — only user
  overrides are searched under `~/.config` and `~/.local/share`.
- This package uses **FHS mode**, so system defaults live in `/etc/sioyek` and
  your personal overrides/databases live under `~/.config/sioyek` and
  `~/.local/share/sioyek`.

## Install

Pick one:

**Install the built package directly** (no rebuild):

```bash
sudo pacman -U sioyek-dev-*.pkg.tar.zst
```

**Build + install with makepkg:**

```bash
makepkg -si
```

**Build + install with paru (from this directory):**

```bash
paru -Bi .
```

Run it:

```bash
sioyek                          # auto-selects platform plugin
QT_QPA_PLATFORM=wayland sioyek  # force Wayland (needs qt6-wayland)
```

## Update

Because the source is `git+https://github.com/ahrm/sioyek#branch=development`,
updating is just a rebuild — it re-clones the latest dev branch and re-pins
submodules:

```bash
makepkg -si        # re-clone + rebuild + reinstall
```

If a future upstream commit adds a dependency:

- `makepkg -s` installs missing `makedepends` automatically.
- pacman refuses the upgrade if a runtime `depend` is missing (and tells you
  what to install).

That dependency tracking is the main advantage over a bare `./build_linux.sh`
workflow.

## Optional: `pacman -S sioyek-dev`

To install/upgrade via plain `pacman -S` instead of `pacman -U`, set up a tiny
local repo:

```bash
sudo mkdir -p /var/cache/pacman-local
sudo cp sioyek-dev-*.pkg.tar.zst /var/cache/pacman-local/
repo-add /var/cache/pacman-local/sioyek-dev.db.tar.gz \
    /var/cache/pacman-local/sioyek-dev-*.pkg.tar.zst
```

Add to `/etc/pacman.conf` (anywhere):

```ini
[local]
SigLevel = Optional TrustAll
Server = file:///var/cache/pacman-local
```

Then:

```bash
sudo pacman -Sy
sudo pacman -S sioyek-dev
```

After each rebuild, copy the new `.pkg.tar.zst` into `/var/cache/pacman-local/`
and re-run `repo-add`. For a single local package, `pacman -U` is simpler.

## Repo files

- `PKGBUILD` — the recipe.
- `.SRCINFO` — generated metadata (`makepkg --printsrcinfo > .SRCINFO`).
- `README.md` — this file.
- `LICENSE` — MIT license for the packaging files.
- `.gitignore` — ignores `src/`, `pkg/`, the bare `sioyek/` clone, `*.pkg.tar.zst`, `*.log`.
- `.github/workflows/build.yml` — GitHub Actions workflow: builds the package and runs `namcap` on push to `master` and on manual trigger. Status badge above.

## Notes

- The build leaves `src/` at ~2 GB (full mupdf clone + build artifacts). Keep it
  for fast incremental rebuilds, or `rm -rf src/ pkg/` to reclaim space; `makepkg`
  re-clones cleanly next time.
- `options=('!strip' '!debug')` keeps the ~49 MB binary intact and avoids a huge
  debug package. Remove these if you want a stripped/smaller binary or debug
  symbols.

## License

The **packaging files** in this repository (PKGBUILD, README, .gitignore,
LICENSE) are MIT-licensed — see [LICENSE](LICENSE).

The **packaged software** (sioyek and its bundled mupdf) is GPL-3.0; the
PKGBUILD's `license=(GPL3)` field describes the packaged software, not these
packaging files.

## Attribution

Guided by a human, written by GLM-5.2.
