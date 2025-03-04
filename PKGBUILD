# Maintainer: Tobias Powalowski <tpowa@archlinux.org>
# Contributor: Sébastien "Seblu" Luttringer <seblu@seblu.net>

pkgbase=qemu-9p-patch-backport
pkgname=(qemu qemu-headless qemu-arch-extra qemu-headless-arch-extra
         qemu-block-{iscsi,rbd,gluster} qemu-guest-agent)
pkgdesc="A generic and open source machine emulator and virtualizer"
pkgver=5.0.0
pkgrel=1
arch=(x86_64)
license=(GPL2 LGPL2.1)
url="https://wiki.qemu.org/"
_headlessdeps=(seabios gnutls libpng libaio numactl libnfs
               lzo snappy curl vde2 libcap-ng spice libcacard usbredir libslirp
               libssh zstd liburing)
depends=(virglrenderer sdl2 vte3 libpulse brltty "${_headlessdeps[@]}")
makedepends=(spice-protocol python ceph libiscsi glusterfs python-sphinx xfsprogs)
source=(https://download.qemu.org/qemu-$pkgver.tar.xz{,.sig}
        iouring-1.patch::https://github.com/qemu/qemu/commit/de137e44f75d9868f5b548638081850f6ac771f2.patch
        iouring-2.patch::https://github.com/qemu/qemu/commit/ba607ca8bff4d2c2062902f8355657c865ac7c29.patch
        hostmem.patch::https://github.com/qemu/qemu/commit/70b6d525dfb51d5e523d568d1139fc051bc223c5.patch
        9p-iov.patch
        qemu-ga.service
        65-kvm.rules)
sha512sums=('21ef0cbe107c468a40f0fa2635db2a40048c8790b629dfffca5cd62bb1b502ea8eb133bfc40df5ecf1489e2bffe87f6829aee041cb8a380ff04a8afa23b39fcf'
            'SKIP'
            '533010ba4adb2678e232febaa0ae476556a2d319d431ab14c83985510e3a0f8159fca20a926df0f8b30e02c7859e1b33ffd8f7fcd6144dc87f09ea62a177b82b'
            'ffea3356fcc5c42a5e3d811f47ff1a0add6f3e3c96de7ee11a6a17c9667b4e5b2f1f0e9eabb59b448e421824d02a3038d1149d02398986e1ec7a752c7e71e9b1'
            'ddbd9e141ae918c52a97c1e28da372e939848223951f00dc84e1d0980ce87b90e4b9b2289c2100976c94042f04eaa234f201ab605e430e970da98e98879e4b2c'
            '9cbd1a9eff63443a0cbc2b38c2ac2b4e1b0a226ee1e8f6955a498c84e0dbc37f0717c4e982273d0f0dce87b520407895f845681dffd6b60b3c1ef1091acf544e'
            '269c0f0bacbd06a3d817fde02dce26c99d9f55c9e3b74bb710bd7e5cdde7a66b904d2eb794c8a605bf9305e4e3dee261a6e7d4ec9d9134144754914039f176e4'
            'bdf05f99407491e27a03aaf845b7cc8acfa2e0e59968236f10ffc905e5e3d5e8569df496fd71c887da2b5b8d1902494520c7da2d3a8258f7fd93a881dd610c99')
validpgpkeys=('CEACC9E15534EBABB82D3FA03353C9CEF108B584')

case $CARCH in
  i?86) _corearch=i386 ;;
  x86_64) _corearch=x86_64 ;;
esac

prepare() {
  mkdir build-{full,headless}
  mkdir -p extra-arch-{full,headless}/usr/{bin,share/qemu}

  cd ${pkgname}-${pkgver}

  # FS#66578 FS#66710
  patch -p1 < ../iouring-1.patch
  patch -p1 < ../iouring-2.patch

  # FS#66646
  patch -p1 < ../hostmem.patch

  patch -p1 < ../9p-iov.patch
}

build() {
  _build full \
    --audio-drv-list="pa alsa sdl"

  _build headless \
    --audio-drv-list= \
    --disable-sdl \
    --disable-gtk \
    --disable-vte \
    --disable-brlapi \
    --disable-opengl \
    --disable-virglrenderer
}

_build() (
  cd build-$1

  ../${pkgname}-${pkgver}/configure \
    --prefix=/usr \
    --sysconfdir=/etc \
    --localstatedir=/var \
    --libexecdir=/usr/lib/qemu \
    --extra-ldflags="$LDFLAGS" \
    --smbd=/usr/bin/smbd \
    --enable-modules \
    --enable-sdl \
    --enable-slirp=system \
    --enable-xfsctl \
    "${@:2}"

  make
)

package_qemu() {
  optdepends=('qemu-arch-extra: extra architectures support')
  provides=(qemu-headless)
  conflicts=(qemu-headless)
  replaces=(qemu-kvm)

  _package full
}

package_qemu-headless() {
  pkgdesc="QEMU without GUI"
  depends=("${_headlessdeps[@]}")
  optdepends=('qemu-headless-arch-extra: extra architectures support')

  _package headless
}

_package() {
  optdepends+=('samba: SMB/CIFS server support'
               'qemu-block-iscsi: iSCSI block support'
               'qemu-block-rbd: RBD block support'
               'qemu-block-gluster: glusterfs block support')
  install=qemu.install
  options=(!strip !emptydirs)

  make -C build-$1 DESTDIR="$pkgdir" install "${@:2}"

  # systemd stuff
  install -Dm644 65-kvm.rules "$pkgdir/usr/lib/udev/rules.d/65-kvm.rules"

  # remove conflicting /var/run directory
  cd "$pkgdir"
  rm -r var

  cd usr/lib

  # bridge_helper needs suid
  # https://bugs.archlinux.org/task/32565
  chmod u+s qemu/qemu-bridge-helper

  # remove split block modules
  rm qemu/block-{iscsi,rbd,gluster}.so

  cd ../bin

  # remove extra arch
  for _bin in qemu-*; do
    [[ -f $_bin ]] || continue

    case ${_bin#qemu-} in
      # guest agent
      ga) rm "$_bin"; continue ;;

      # tools
      edid|img|io|keymap|nbd|pr-helper|storage-daemon) continue ;;

      # core emu
      system-${_corearch}) continue ;;
    esac

    mv "$_bin" "$srcdir/extra-arch-$1/usr/bin"
  done

  cd ../share/qemu
  for _blob in *; do
    [[ -f $_blob ]] || continue

    case $_blob in
      # provided by seabios package
      bios.bin|bios-256k.bin|vgabios-cirrus.bin|vgabios-qxl.bin|\
      vgabios-stdvga.bin|vgabios-vmware.bin|vgabios-virtio.bin|vgabios-bochs-display.bin|\
      vgabios-ramfb.bin) rm "$_blob"; continue ;;

      # provided by edk2-ovmf package
      edk2-*) rm "$_blob"; continue ;;

      # iPXE ROMs
      efi-*|pxe-*) continue ;;

      # core blobs
      bios-microvm.bin|kvmvapic.bin|linuxboot*|multiboot.bin|sgabios.bin|vgabios*) continue ;;

      # Trace events definitions
      trace-events*) continue ;;
    esac

    mv "$_blob" "$srcdir/extra-arch-$1/usr/share/qemu"
  done

  # provided by edk2-ovmf package
  rm -r firmware

  cd ..
  if [ "$1" = headless ]; then rm -r {applications,icons}; fi
}

package_qemu-arch-extra() {
  pkgdesc="QEMU for foreign architectures"
  depends=(qemu)
  provides=(qemu-headless-arch-extra)
  conflicts=(qemu-headless-arch-extra)
  options=(!strip)

  mv extra-arch-full/usr "$pkgdir"
}

package_qemu-headless-arch-extra() {
  pkgdesc="QEMU without GUI, for foreign architectures"
  depends=(qemu-headless)
  options=(!strip)

  mv extra-arch-headless/usr "$pkgdir"
}

package_qemu-block-iscsi() {
  pkgdesc="QEMU iSCSI block module"
  depends=(glib2 libiscsi)

  install -D build-full/block-iscsi.so "$pkgdir/usr/lib/qemu/block-iscsi.so"
}

package_qemu-block-rbd() {
  pkgdesc="QEMU RBD block module"
  depends=(glib2 ceph-libs)

  install -D build-full/block-rbd.so "$pkgdir/usr/lib/qemu/block-rbd.so"
}

package_qemu-block-gluster() {
  pkgdesc="QEMU GlusterFS block module"
  depends=(glib2 glusterfs)

  install -D build-full/block-gluster.so "$pkgdir/usr/lib/qemu/block-gluster.so"
}

package_qemu-guest-agent() {
  pkgdesc="QEMU Guest Agent"
  depends=(gcc-libs glib2 libudev.so)

  install -D build-full/qemu-ga "$pkgdir/usr/bin/qemu-ga"
  install -Dm644 qemu-ga.service "$pkgdir/usr/lib/systemd/system/qemu-ga.service"
  install -Dm755 "$srcdir/qemu-$pkgver/scripts/qemu-guest-agent/fsfreeze-hook" "$pkgdir/etc/qemu/fsfreeze-hook"
}

# vim:set ts=2 sw=2 et:
