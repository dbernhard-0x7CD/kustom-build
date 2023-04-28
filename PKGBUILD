# Maintainer: David Bernhard <dbernhard@ethz.ch>
pkgbase=linux-kb
pkgver=6.2.12
pkgdesc="Custom kernel build (kustom build)"
kustom_build_id=101
pkgrel="$kustom_build_id"
linux_vername="linux-$pkgver"
module_name=$pkgver-$(echo $pkgbase | cut -d "-" -f 2-)
extraversion=$(echo $pkgbase | cut -d "-" -f 2-)

# update using: https://github.com/archlinux/svntogit-packages/tree/packages/linux/trunk
_linux_pkg_commit="1f2e6b4063bd9740021d9554055ccbcd41b72131"

arch=(x86_64)
url="https://n.ethz.ch/~dbernhard"
license=('GPL2')
groups=()
depends=()
makedepends=(
  bc libelf pahole cpio perl tar xz gettext
  xmlto python-sphinx graphviz imagemagick texlive-latexextra
  git
)
checkdepends=()
optdepends=()
provides=()
conflicts=()
replaces=()
backup=()
options=('!strip')
install=
changelog=
source=(
  "https://cdn.kernel.org/pub/linux/kernel/v6.x/$linux_vername.tar.xz"
  "https://raw.githubusercontent.com/archlinux/svntogit-packages/$_linux_pkg_commit/repos/core-x86_64/config"
)
noextract=()
sha256sums=('c7e146b52737adfa4c724bfa41bf4721c5ee3cf220c074fbc60eb3ea62b0ccc8'
            'c8b3fbb7664801bebc2d2d1fdf624524865a7817d0021c55c98523cb58dee201')
validpgpkeys=()

llvm_path="/mnt/Data/llvm/build/bin/"

CFLAGS=""
CFLAGS="$CFLAGS -O3"
CFLAGS="$CFLAGS -mllvm -polly"
CFLAGS="$CFLAGS -march=native"

info="CFLAGS:$CFLAGS"
if [ -n "$custom_build_id" ]; then
  info="$info\n,build id: $kustom_build_id"
fi

export KBUILD_BUILD_HOST="$info"
export KBUILD_BUILD_USER="$pkgbase"
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

prepare() {
    echo "Preparing in $(pwd)"

    cd $linux_vername || echo "unable to find $linux_vername" || exit

    # config comes from archlinux
    cp ../../config .config
    
    yes "" | LLVM=$llvm_path make olddefconfig
    # yes "" | LLVM=$llvm_path make localmodconfig
    
    diff -u ../../config .config || :

    # Copy actual config away
    archive_folder="../../builds/${kustom_build_id}/"
    if [ -d "$archive_folder" ]; then
      echo "$archive_folder already exists and means you've built this version before!"
      exit 1
    fi
    mkdir -p ${archive_folder} && \
      cp .config ${archive_folder}/config
      cp ../../PKGBUILD ${archive_folder}/PKGBUILD
}

build() {
    cd $linux_vername || exit

    # set correct version and store
    echo "extraversion: $extraversion"
    sed -i "s/EXTRAVERSION = *$/EXTRAVERSION = -$extraversion/g" Makefile 
    
    echo "CFLAGS: $CFLAGS"
    
    time (yes "" | KBUILD_BUILD_TIMESTAM='' V=2 make \
          LLVM=$llvm_path \
          KCFLAGS="$CFLAGS" \
          && V=2 make LLVM=$llvm_path bzImage)

    make -s image_name > bzimage_path
}

_package() {
    depends=(coreutils kmod initramfs)
    optdepends=('crda: to set the correct wireless channels of your country'
                'linux-firmware: firmware images needed for some devices')
    provides=(VIRTUALBOX-GUEST-MODULES WIREGUARD-MODULE)
    replaces=(virtualbox-guest-modules-arch wireguard-arch)

    bzimage_path=$(<$srcdir/$linux_vername/bzimage_path)

    cd $linux_vername || exit
    modulesdir="$pkgdir/usr/lib/modules/$module_name"
    
    echo "$pkgname" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"
    
    make INSTALL_MOD_PATH="$pkgdir/usr"  LLVM=$llvm_path INSTALL_MOD_STRIP=1 modules_install > /dev/null

    install -Dm644 "$srcdir/$linux_vername/$bzimage_path" "$modulesdir/vmlinuz"  

    rm -f "$modulesdir"/{source,build}
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"
  depends=(pahole)

  cd $linux_vername
  local builddir="$pkgdir/usr/lib/modules/$pkgbase/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
  cp -t "$builddir" -a scripts

  # required when STACK_VALIDATION is enabled
  install -Dt "$builddir/tools/objtool" tools/objtool/objtool

  # required when DEBUG_INFO_BTF_MODULES is enabled
  install -Dt "$builddir/tools/bpf/resolve_btfids" tools/bpf/resolve_btfids/resolve_btfids

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/x86" -a arch/x86/include
  install -Dt "$builddir/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "$builddir/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  echo "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */x86/ ]] && continue
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
  done

  echo "Removing documentation..."
  rm -r "$builddir/Documentation"

  echo "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -bi "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v $STRIP_SHARED "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v $STRIP_STATIC "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v $STRIP_BINARIES "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Stripping vmlinux..."
  strip -v $STRIP_STATIC "$builddir/vmlinux"

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

pkgname=("$pkgbase" "$pkgbase-headers")
for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done

