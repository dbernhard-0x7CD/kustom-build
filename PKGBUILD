# Maintainer: David Bernhard <dbernhard@ethz.ch>
pkgbase=linux-kb
pkgver=6.2.13
pkgdesc="Custom kernel build (kustom build)"
kustom_build_id=501
pkgrel="$kustom_build_id"
_srcname="linux-$pkgver"

# Configuration file source commit in the archlinux 'linux' package
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
  "https://cdn.kernel.org/pub/linux/kernel/v6.x/$_srcname.tar.xz"
  "https://raw.githubusercontent.com/archlinux/svntogit-packages/$_linux_pkg_commit/repos/core-x86_64/config"
)
noextract=()
sha256sums=('c7dded14e368834b18bb2ad64af65560d8bcb9d2d6597e0f6ef151fded01e577'
            'd3cc4f935ee1794cc1f7bb17a9baaf039eec6791cad4bc6b24a4504953c4c8f8')
validpgpkeys=()

# Unset this to use gcc
llvm_path="/home/dbernhard/llvm-project/build_16_0_3/bin/"
# llvm_path="/mnt/Data/llvm-project/build_16_0_2/bin/"

# Values of 'full', 'thin' or 'none' allowed
llvm_lto="none"

gcc_lto=0

CFLAGS=""
CFLAGS="$CFLAGS -O3"
# CFLAGS="$CFLAGS -mllvm -polly"
# CFLAGS="$CFLAGS -march=native -mtune=native"

# End of configuration; Below only functional code

info="CFLAGS:$CFLAGS"
info+=",llvm_lto: $llvm_lto"
info+=",gcc_lto: $gcc_lto"

if [ -n "$kustom_build_id" ]; then
  info="$info,build id: $kustom_build_id"
fi

# Add additional sources based on configuration
if [[ 1 == $gcc_lto ]]; then
  _cachyos_repo="https://raw.githubusercontent.com/cachyos/kernel-patches/master/6.2/"
  source+=("${_cachyos_repo}/misc/gcc-lto/0001-gcc-LTO-support-for-the-kernel.patch"
           "${_cachyos_repo}/misc/gcc-lto/0002-gcc-lto-no-pie.patch")
fi

# Configure build environment
export KCFLAGS="$CFLAGS"
export KBUILD_BUILD_USER="$pkgbase"
export KBUILD_BUILD_HOST="$info"
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

prepare() {
  echo "Preparing in $(pwd)"

  cd $_srcname || echo "unable to find $_srcname" || exit

  echo "Setting version..."
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname

  echo "Setting config..."
  cp ../../config .config

  local src
  for src in "${source[@]}"; do
    src="${src%%::*}"
    src="${src##*/}"
    [[ $src = *.patch ]] || continue
    echo "Applying patch $src..."
    patch -Np1 < "../$src"
  done

  yes "" | make LLVM=$llvm_path KERNELRELEASE=$(<version) olddefconfig

  make -s kernelrelease > version

  case "$llvm_lto" in
    thin) scripts/config -e LTO -d LTO_NONE -e LTO_CLANG_THIN;;
    full) scripts/config -e LTO -d LTO_NONE -e LTO_CLANG_FULL;;
    none) ;;
    *) echo -e "Invalid value for \$llvm_lto: $llvm_lto\n"; exit 1;;
  esac

  # (from cachyos): Disable DEBUG, pahole is currently broken with GCC LTO
  scripts/config -d DEBUG_INFO \
                 -d DEBUG_INFO_BTF \
                 -d DEBUG_INFO_DWARF4 \
                 -d DEBUG_INFO_DWARF5 \
                 -d PAHOLE_HAS_SPLIT_BTF \
                 -d DEBUG_INFO_BTF_MODULES \
                 -d SLUB_DEBUG \
                 -d PM_DEBUG \
                 -d PM_ADVANCED_DEBUG \
                 -d PM_SLEEP_DEBUG \
                 -d ACPI_DEBUG \
                 -d SCHED_DEBUG \
                 -d LATENCYTOP \
                 -d DEBUG_PREEMPT

  # Disable BPF as this breaks the build for gcc with LTO
  scripts/config -d CONFIG_BPF_JIT

  if [ 1 == "$gcc_lto" ]; then
    echo "Enabling LTO_GCC"
    scripts/config -e LTO_GCC \
		   -d LTO_CP_CLONE
  fi

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

  echo "Prepared $pkgbase version $(<version)"
}

build() {
  cd $_srcname || exit

  echo "CFLAGS: $CFLAGS"
  
  time  (KBUILD_BUILD_TIMESTAMP='' V=2 make LLVM=$llvm_path all)
}

_package() {
  pkgdesc="The $pkgdesc kernel and modules"
  depends=(
    coreutils
    initramfs
    kmod
  )
  optdepends=(
    'wireless-regdb: to set the correct wireless channels of your country'
    'linux-firmware: firmware images needed for some devices'
  )
  provides=(
    KSMBD-MODULE
    VIRTUALBOX-GUEST-MODULES
    WIREGUARD-MODULE
  )
  replaces=(
    virtualbox-guest-modules-arch
    wireguard-arch
  )

  cd $_srcname
  local modulesdir="$pkgdir/usr/lib/modules/$(<version)"

  echo "Installing boot image..."
  # systemd expects to find the kernel here to allow hibernation
  # https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  install -Dm644 "$(make KERNELRELEASE=$(<version) -s image_name)" "$modulesdir/vmlinuz"

  # Used by mkinitcpio to name the kernel
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  echo "Installing modules..."
  make KERNELRELEASE=$(<version) INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 \
    DEPMOD=/doesnt/exist modules_install  # Suppress depmod

  # remove build and source links
  rm "$modulesdir"/{source,build}
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"
  depends=(pahole)

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
  cp -t "$builddir" -a scripts

  # required when STACK_VALIDATION is enabled
  install -Dt "$builddir/tools/objtool" tools/objtool/objtool

  # required when DEBUG_INFO_BTF_MODULES is enabled
  [[ -x tools/bpf/resolve_btfids/resolve_btfids ]] &&
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
    case "$(file -Sib "$file")" in
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

_package-docs() {
  pkgdesc="Documentation for the $pkgdesc kernel"

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing documentation..."
  local src dst
  while read -rd '' src; do
    dst="${src#Documentation/}"
    dst="$builddir/Documentation/${dst#output/}"
    install -Dm644 "$src" "$dst"
  done < <(find Documentation -name '.*' -prune -o ! -type d -print0)

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/share/doc"
  ln -sr "$builddir/Documentation" "$pkgdir/usr/share/doc/$pkgbase"
}

pkgname=(
  "$pkgbase"
  "$pkgbase-headers"
  "$pkgbase-docs"
)
for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done

