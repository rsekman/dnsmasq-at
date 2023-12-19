# This is an example PKGBUILD file. Use this as a start to creating your own,
# and remove these comments. For more information, see 'man PKGBUILD'.
# NOTE: Please fill out the license field for your package! If it is unknown,
# then please put 'unknown'.

# Maintainer: Robin Ekman <robin [dot] seth [dot] ekman [at] gmail [dot] com>
pkgname=dnsmasq-at
pkgver=0.1.0
pkgrel=1
pkgdesc="A dnsmasq@.service for easy per-interface configuration of dnsmasq"
arch=(any)
license=('GPL')
depends=('dnsmasq')
source=(
    'dnsmasq@.service'
    'dnsmasq@.conf'
)
sha256sums=('1835f8568dd1f4ce40ecb6d483ef7c3c7cb67e50c3a6c0b0942e24b0e8b51c90'
            '414aeaefceed6eb90b7ce2027c50ff72228575286353f2cca9795226b0e69779')

package() {
    mkdir -p "$pkgdir/usr/lib/systemd/system"
    install -Dm644 dnsmasq@.service \
        "$pkgdir/usr/lib/systemd/system"

    mkdir -p "$pkgdir/usr/share/dbus-1/system.d/"
    install -Dm644 dnsmasq@.conf \
        "$pkgdir/usr/share/dbus-1/system.d/"
}
