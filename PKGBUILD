# Maintainer: 7Ji <pugokughin@gmail.com>

_desc="flippy's AArch64-focused fork aiming to increase usability"
_ver_major_minor=6.6
_flippy_repo="linux-${_ver_major_minor}.y"
_srcname="${_flippy_repo}"

pkgbase=linux-aarch64-flippy-git
pkgname=(
  "${pkgbase}"
  "${pkgbase}-headers"
  "${pkgbase}-dtb-allwinner"
  "${pkgbase}-dtb-amlogic"
  "${pkgbase}-dtb-rockchip"
)
pkgver=6.6.26
pkgrel=1
arch=('aarch64')
url="https://github.com/unifreq/${_flippy_repo}"
license=('GPL2')
makedepends=( # Since we don't build the doc, most of the makedeps for other linux packages are not needed here
  'kmod' 'bc' 'dtc' 'uboot-tools'
)
options=(!strip)
source=(
  "git+${url}.git"
  "git+https://github.com/unifreq/arm64-kernel-configs"
)
sha256sums=(
  'SKIP'
  'SKIP'
)

prepare() {
  cd arm64-kernel-configs
  local _config= _latest_config=
  for _config in config-"${_ver_major_minor}"-*-flippy-*; do
    if [[ -z "${_latest_config}" || $(vercmp "${_config}" "${_latest_config}") == 1 ]]; then
      _latest_config="${_config}"
    fi
  done
  if [[ -z "${_latest_config}" ]]; then
    echo 'ERROR: Cannot find latest Flippy config'
    return 1
  fi
  local _rev_config="$(git rev-list --count HEAD "${_latest_config}")"
  local _id_config="$(git rev-parse --short HEAD:"${_latest_config}")"
  echo "Using flippy's latest config ${_latest_config} (ver ${_rev_config}, id ${_id_config}})..."

  cd ../"${_srcname}"
  local _rev_kernel="$(git rev-list --count HEAD)"
  local _id_kernel="$(git rev-parse --short HEAD)"


  echo "Setting version..."
  scripts/setlocalversion --save-scmversion
  echo - > localversion.09-hyphen
  echo "r$(( "${_rev_kernel}" + "${_rev_config}" ))" > localversion.10-release-total
  echo - > localversion.19-hyphen
  echo "${_id_kernel}" > localversion.20-id-kernel
  echo - > localversion.29-hyphen
  echo "${_id_config}" > localversion.30-id-config
  echo "-${pkgrel}" > localversion.40-pkgrel
  echo "${pkgbase#linux}" > localversion.50-pkgname

  echo "Generating config..."
  cat ../arm64-kernel-configs/"${_latest_config}" > .config
  make olddefconfig
}

pkgver() {
  cd "${_srcname}"
  printf '%s.%s.%s.%s' \
    "$(make kernelversion)" \
    "$(<localversion.10-release-total)" \
    "$(<localversion.20-id-kernel)" \
    "$(<localversion.30-id-config)"
}

build() {
  cd "${_srcname}"

  # get kernel version, which will be used later for modules
  make prepare
  make -s kernelrelease > version

  # Host LDFLAGS or other LDFLAGS set by makepkg/user is not helpful for building kernel: it should links nothing outside of itself
  unset LDFLAGS
  # Only need normal Image, as most Amlogic devices does not need/support Image.gz
  # Image and modules are built in the same run to make sure they're compatible with each other
  # -@ enables symbols in dtbs, so overlay is possible
  make ${MAKEFLAGS} DTC_FLAGS="-@" Image modules dtbs
}

_dtb_common_pkg="${pkgbase}-dtb"

package_linux-aarch64-flippy() {
  pkgdesc="The Linux Kernel and module - ${_desc}"
  depends=(
    "${_dtb_common_pkg}"
    'coreutils'
    'initramfs'
    'kmod'
  )
  optdepends=(
    'uboot-legacy-initrd-hooks: to generate uboot legacy initrd images'
    'linux-firmware: firmware images needed for some devices'
    'linux-firmware-amlogic-ophub: complete firmware set for devices with Amlogic SoCs'
    'wireless-regdb: to set the correct wireless channels of your country'
    "${pkgbase}-dtb-allwinner: dtbs for Allwinner SoCs"
    "${pkgbase}-dtb-amlogic: dtbs for Amlogic SoCs"
    "${pkgbase}-dtb-rockchip: dtbs for Rockchip SoCs"
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

  cd "${_srcname}"

  # Install modules
  echo "Installing modules..."
  make INSTALL_MOD_PATH="${pkgdir}/usr" INSTALL_MOD_STRIP=1 modules_install

  # Install DTBs, not to target pkg, but in srcdir, so the later package() routine could use them
  make INSTALL_DTBS_PATH="${srcdir}/dtbs" dtbs_install

  # Install pkgbase
  local _dir_module="${pkgdir}/usr/lib/modules/$(<version)"
  echo "${pkgbase}" | install -D -m 644 /dev/stdin "${_dir_module}/pkgbase"

  # Install kernel image (this is technically not vmlinuz, but I name it this way to utilize mkinitcpio's existing hooks)
  install -Dm644 arch/arm64/boot/Image "${_dir_module}/vmlinuz"

  # Remove hbuild and source links, which points to folders used when building (i.e. dead links)
  rm -f "${_dir_module}/"{build,source}
}

package_linux-aarch64-flippy-headers() {
  pkgdesc="Header files and scripts for building modules for linux kernel - ${_desc}"
  
  # Mostly copied from alarm's linux-aarch64 and modified
  cd "${_srcname}"
  local _builddir="${pkgdir}/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "${_builddir}" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "${_builddir}/kernel" -m644 kernel/Makefile
  install -Dt "${_builddir}/arch/arm64" -m644 arch/arm64/Makefile
  cp -t "${_builddir}" -a scripts

  echo "Installing headers..."
  cp -t "${_builddir}" -a include
  cp -t "${_builddir}/arch/arm64" -a arch/arm64/include
  install -Dt "${_builddir}/arch/arm64/kernel" -m644 arch/arm64/kernel/asm-offsets.s


  install -Dt "${_builddir}/drivers/md" -m644 drivers/md/*.h
  install -Dt "${_builddir}/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "${_builddir}/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "${_builddir}/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "${_builddir}/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "${_builddir}/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "${_builddir}/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "${_builddir}/{}" \;

  echo "Removing unneeded architectures..."
  local _arch
  for _arch in "${_builddir}"/arch/*/; do
    [[ ${_arch} = */arm64/ ]] && continue
    echo "Removing $(basename "${_arch}")"
    rm -r "${_arch}"
  done

  echo "Removing documentation..."
  rm -r "${_builddir}/Documentation"

  echo "Removing broken symlinks..."
  find -L "${_builddir}" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "${_builddir}" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -Sib "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v ${STRIP_SHARED} "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v ${STRIP_STATIC} "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v ${STRIP_BINARIES} "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v ${STRIP_SHARED} "$file" ;;
    esac
  done < <(find "${_builddir}" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Stripping vmlinux..."
  strip -v $STRIP_STATIC "${_builddir}/vmlinux"

  echo "Adding symlink..."
  mkdir -p "${pkgdir}/usr/src"
  ln -sr "${_builddir}" "$pkgdir/usr/src/$pkgbase"
}

_dtb_common_provides="${_dtb_common_pkg}=${pkgver}"

package_linux-aarch64-flippy-dtb-allwinner() {
  pkgdesc="DTB files for Allwinner SoCs for flippy's AArch64 kernel"
  provides=(
    "${_dtb_common_provides}"
  )
  echo 'Installing DTBs for Allwinner SoCs...'
  install -d -m 755 "${pkgdir}/boot/dtbs/${pkgbase}"
  cp -t "${pkgdir}/boot/dtbs/${pkgbase}" -a "${srcdir}/dtbs/allwinner"
}

package_linux-aarch64-flippy-dtb-amlogic() {
  pkgdesc="DTB files for Amlogic SoCs for flippy's AArch64 kernel"
  provides=(
    "${_dtb_common_provides}"
  )
  echo 'Installing DTBs for Amlogic SoCs...'
  install -d -m 755 "${pkgdir}/boot/dtbs/${pkgbase}"
  cp -t "${pkgdir}/boot/dtbs/${pkgbase}" -a "${srcdir}/dtbs/amlogic"
}

package_linux-aarch64-flippy-dtb-rockchip() {
  pkgdesc="DTB files for Rockchip SoCs for flippy's AArch64 kernel"
  provides=(
    "${_dtb_common_provides}"
  )
  echo 'Installing DTBs for Rockchip SoCs...'
  install -d -m 755 "${pkgdir}/boot/dtbs/${pkgbase}"
  cp -t "${pkgdir}/boot/dtbs/${pkgbase}" -a "${srcdir}/dtbs/rockchip"
}
