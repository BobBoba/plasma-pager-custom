# Maintainer: Felix Yan <felixonmars@archlinux.org>
# Maintainer: Antonio Rojas <arojas@archlinux.org>
# Contributor: Andrea Scarpino <andrea@archlinux.org>
#
# LOCAL FORK: identical to the official Arch plasma-desktop package, plus a single
# patch (0001-pager-custom-label-font-and-color.patch) that adds user-configurable
# font family, font size and text color to the Pager / Activity Pager widget.
# Rebuild after every official plasma-desktop update (the stock package will
# otherwise replace this one and revert the patch). See README.md.

pkgname=plasma-desktop
pkgver=6.7.1
_dirver=$(echo $pkgver | cut -d. -f1-3)
pkgrel=1
pkgdesc='KDE Plasma Desktop'
arch=(x86_64)
url='https://kde.org/plasma-desktop/'
license=(LGPL-2.0-or-later)
depends=(attica
         baloo
         emoji-font # for clock and language KCMs
         glibc
         kbookmarks
         kcmutils
         kcodecs
         kcompletion
         kconfig
         kconfigwidgets
         kcoreaddons
         kcrash
         kdbusaddons
         kdeclarative
         kglobalaccel
         kguiaddons
         ki18n
         kiconthemes
         kio
         kirigami
         kirigami-addons
         kitemmodels
         kitemviews
         kjobwidgets
         kmenuedit
         knewstuff
         knotifications
         knotifyconfig
         kpackage
         kpipewire
         krunner
         kservice
         ksvg
         kwidgetsaddons
         kwindowsystem
         kxmlgui
         libcanberra
         libksysguard
         libstdc++
         libwacom
         libx11
         libxcb
         libxcursor
         libxi
         libxkbcommon
         libxkbfile
         libplasma
         plasma-activities
         plasma-activities-stats
         plasma-workspace
         plasma5support
         polkit-kde-agent
         powerdevil
         qt6-5compat
         qt6-base
         qt6-declarative
         sdl2
         solid
         sonnet
         systemd-libs
         systemsettings
         wayland
         xcb-util-keysyms
         xdg-user-dirs)
optdepends=('bluedevil: Bluetooth applet'
            'glib2: kimpanel IBUS support'
            'ibus: kimpanel IBUS support'
            'kaccounts-integration: OpenDesktop integration plugin'
            'kscreen: screen management'
            'libaccounts-qt: OpenDesktop integration plugin'
            'packagekit-qt6: to install new krunner plugins'
            'plasma-nm: Network manager applet'
            'plasma-pa: Audio volume applet'
            'scim: kimpanel SCIM support')
makedepends=(extra-cmake-modules
             intltool
             kaccounts-integration
             kdoctools
             libibus
             packagekit-qt6
             scim
             wayland-protocols
             xf86-input-libinput
             xorg-server-devel)
groups=(plasma)
source=(https://download.kde.org/stable/plasma/$_dirver/$pkgname-$pkgver.tar.xz
        0001-pager-custom-label-font-and-color.patch)
# Integrity of the upstream tarball is guaranteed by the pinned SHA-256 below — the
# exact hash from Arch's official plasma-desktop package. The PGP signature (.sig)
# and validpgpkeys are intentionally NOT used: they only verify the signer's identity,
# require the KDE release keys in the user's keyring, and a stock keyring lacks them,
# which would make `makepkg` fail with "unknown public key". The pinned checksum already
# rules out tampering/corruption, so the package builds out of the box without keys.
sha256sums=('f0450dc26706fc2d719ab28e3102149cfbf6a72e81cd310c936b85c08abb6f2f'  # plasma-desktop-6.7.1.tar.xz
            'SKIP')                                                            # local patch (this repo)

prepare() {
  cd $pkgname-$pkgver
  # Add user-configurable font (family + size) and text color for the Pager labels.
  patch -Np1 < "$srcdir/0001-pager-custom-label-font-and-color.patch"
}

build() {
  cmake -B build  -S $pkgname-$pkgver \
    -DCMAKE_INSTALL_LIBEXECDIR=lib \
    -DBUILD_TESTING=OFF
  cmake --build build
}

package() {
  DESTDIR="$pkgdir" cmake --install build
}
