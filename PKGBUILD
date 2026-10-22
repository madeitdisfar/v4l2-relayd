# Maintainer: Minjun Kim <mjkim5216@gmail.com>

# v4l2-relayd, set up as a user service that relays the PipeWire camera into
# a v4l2loopback device, for apps that only read /dev/video (Discord) on the
# Samsung Galaxy Book4 Pro (940XGK). See README.md.

pkgname=v4l2-relayd
pkgver=0.2.0
pkgrel=5
pkgdesc="Relays a GStreamer source into a v4l2loopback device on demand (PipeWire camera user service)"
arch=(x86_64)
url="https://gitlab.com/vicamo/v4l2-relayd"
license=(GPL-2.0-only MIT)
# v4l2loopback ships with the CachyOS kernel; other kernels need v4l2loopback-dkms
depends=(glib2 gstreamer gst-plugins-base gst-plugins-base-libs gst-plugins-good gst-plugin-pipewire)
makedepends=(autoconf-archive)
source=(
  "$pkgname-$pkgver.tar.gz::https://gitlab.com/vicamo/v4l2-relayd/-/archive/upstream/$pkgver/v4l2-relayd-upstream-$pkgver.tar.gz"
  quick-restart-fixes.patch
  v4l2-relayd.service
  v4l2-relayd-modprobe.conf
  LICENSE
)
b2sums=('08a844ddae39b7ecdac9cd7094e0b4701808d5e60e1d0136de1be4da8c6b41df711c633fb5898d0c9f6e621ee979b02b3b3bc8232c092d9d5b07ef4ecc82f87a'
        '920f58f319023732d7a451bfc34dff8713bde5e9d619e6a46b6439967717ebc5af0d7e610129353f3586f55e763ccc6f11e0872d053b48e34690ec2ed5fde664'
        'c78f875f748aecd10864b5e456362c483af3e3fcb583b3082c9aa261b212bbcdbddfdaf593414c0e089ea375597c509a2952ae627115b0d0d133c431dd80fe31'
        '2e7de989ea169cc872c5f73359ec058ad8b80c12ca7c25b121be6b09a6b1d35c909698c2c0197815fc252c3f71a6e01aec94df1c22b8fd829456d50470b4d6dd'
        '9b7d79f59e653ba1537fccc2b1dad05e45c32511ad9937c2dba04ec392974ad46300aa0bc428b58e60ef312965d6788802089b28f21de6bef4300c80e560eee4')

prepare() {
  cd v4l2-relayd-upstream-$pkgver
  # Fresh input pipeline on every start, and a stall watchdog (-w)
  patch -Np1 -i ../quick-restart-fixes.patch
}

build() {
  cd v4l2-relayd-upstream-$pkgver
  ./autogen.sh
  ./configure --prefix=/usr --sysconfdir=/etc
  make
}

package() {
  cd v4l2-relayd-upstream-$pkgver
  # Only the daemon: upstream's system services and /etc config are replaced
  # by the user service below
  install -Dm755 src/v4l2-relayd -t "$pkgdir/usr/bin"
  install -Dm644 README.md -t "$pkgdir/usr/share/doc/$pkgname"

  cd "$srcdir"
  install -Dm644 v4l2-relayd.service -t "$pkgdir/usr/lib/systemd/user"
  install -Dm644 v4l2-relayd-modprobe.conf "$pkgdir/usr/lib/modprobe.d/v4l2-relayd.conf"
  echo v4l2loopback | install -Dm644 /dev/stdin "$pkgdir/usr/lib/modules-load.d/v4l2-relayd.conf"
  install -Dm644 LICENSE -t "$pkgdir/usr/share/licenses/$pkgname"
}
