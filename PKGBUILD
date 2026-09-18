# Maintainer: github.com/hyperverse
# Contributor: github.com/hyperverse
# Based on the AUR sioyek-git PKGBUILD by hrdl, Fabio 'Lolix' Loli, et al.
#
# Build notes:
#  * mupdf is built from the bundled submodule and statically linked into the
#    sioyek binary, so the package ships only the binary + data files (no mupdf
#    libraries). This guarantees the exact mupdf version sioyek's development
#    branch was tested against.
#  * We keep CONFIG+=linux_app_image (for the local-mupdf link line) but add
#    DEFINES+=LINUX_STANDARD_PATHS so the binary reads defaults from /etc/sioyek
#    and read-only data from /usr/share/sioyek at runtime (FHS layout), instead
#    of looking next to the executable.
#  * Submodules are fetched with `git submodule update --init --recursive`
#    (NO --remote), which checks out the exact pinned SHAs. Using --remote
#    breaks because most nested submodules declare `branch = artifex` with
#    `shallow = true`, so the artifex remote-tracking ref is never created
#    locally and git aborts with "Unable to find refs/remotes/origin/artifex".

pkgname=sioyek-dev
pkgver=2.0.0.r1158.gd0b2c191
pkgrel=1
pkgdesc="PDF viewer for research papers and technical books (development branch, bundled mupdf)"
arch=(x86_64)
license=(GPL-3.0-only)
url="https://github.com/ahrm/sioyek"
depends=(qt6-base qt6-declarative qt6-svg qt6-speech harfbuzz zlib libglvnd)
makedepends=(git unzip)
optdepends=(
  'qt6-wayland: native Wayland platform plugin'
  'speech-dispatcher: text-to-speech backend'
  'espeak-ng: voice for speech-dispatcher'
)
provides=(sioyek)
conflicts=(sioyek)
source=("git+https://github.com/ahrm/sioyek.git#branch=development")
sha256sums=('SKIP')
options=('!strip' '!debug')   # keep the 49 MB binary intact; avoid huge debug package

pkgver() {
  cd "sioyek"
  git describe --long --tags 2>/dev/null | sed 's/^v//;s/\([^-]*-g\)/r\1/;s/-/./g' \
    || printf "r%s.%s" "$(git rev-list --count HEAD)" "$(git rev-parse --short HEAD)"
}

prepare() {
  cd "sioyek"
  # Fetch nested submodules at their pinned SHAs. Do NOT use --remote.
  git submodule update --init --recursive
}

build() {
  cd "sioyek"

  # 1. Build only the mupdf libraries sioyek links against (libmupdf.a,
  #    libmupdf-third.a, libmupdf-threads.a). We deliberately skip the
  #    default `apps` target: it builds the mupdf-gl/mupdf-x11 viewers via the
  #    bundled freeglut, which needs GL/X11 dev headers (glu, libx11, ...) that
  #    are not needed by sioyek at all (sioyek does not link libmupdf-glut).
  make -C mupdf USE_SYSTEM_HARFBUZZ=yes libs libmupdf-threads -j"$(nproc)"

  # 2. Build sioyek. linux_app_image keeps the local-mupdf link line;
  #    LINUX_STANDARD_PATHS makes the binary use /usr/share/sioyek + /etc/sioyek.
  qmake6 \
    "CONFIG+=linux_app_image" \
    "DEFINES+=LINUX_STANDARD_PATHS" \
    pdf_viewer_build_config.pro
  make -j"$(nproc)"
}

package() {
  cd "sioyek"

  # `make install` uses the .pro's INSTALLS targets (PREFIX defaults to /usr):
  #   /usr/bin/sioyek
  #   /usr/share/applications/sioyek.desktop
  #   /usr/share/pixmaps/sioyek-icon-linux.png
  #   /usr/share/sioyek/shaders/*
  #   /usr/share/sioyek/tutorial.pdf
  # NOTE: the .pro puts keys/prefs under $$PREFIX/etc/sioyek = /usr/etc/sioyek,
  # but the binary (LINUX_STANDARD_PATHS) reads them from /etc/sioyek. So we drop
  # the misplaced /usr/etc and install the configs to /etc/sioyek ourselves.
  make INSTALL_ROOT="${pkgdir}/" install
  rm -rf "${pkgdir}/usr/etc"
  install -Dm644 -t "${pkgdir}/etc/sioyek" pdf_viewer/keys.config pdf_viewer/prefs.config

  # Man page (not covered by the .pro install targets).
  install -Dm644 resources/sioyek.1 -t "${pkgdir}/usr/share/man/man1"
}
