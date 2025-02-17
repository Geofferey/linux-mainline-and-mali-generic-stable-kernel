# directories:
# - /compile/doc/stable - the files in this dir
# - /compile/source/linux-stable-cbq - the kernel sources checked out from gitrepo
# - /compile/result/stable - the resulting kernel, modules etc. tar.gz files
# - /compile/doc/kernel-config-options - https://github.com/hexdump0815/kernel-config-options
# name: stb-cbq

cd /compile/source/linux-stable-cbq

export ARCH=arm64
# defconfig + kernel-config-options based config will follow later
#scripts/kconfig/merge_config.sh -m arch/arm64/configs/defconfig /compile/doc/kernel-config-options/chromebooks-aarch64.cfg /compile/doc/kernel-config-options/mediatek.cfg /compile/doc/kernel-config-options/docker-options.cfg /compile/doc/kernel-config-options/options-to-remove-generic.cfg /compile/doc/stable/misc.cbq/options/options-to-remove-special.cfg /compile/doc/kernel-config-options/additional-options-generic.cfg /compile/doc/kernel-config-options/additional-options-aarch64.cfg /compile/doc/stable/misc.cbq/options/additional-options-special.cfg
#( cd /compile/doc/kernel-config-options ; git rev-parse --verify HEAD ) > /compile/doc/stable/misc.cbq/options/kernel-config-options.version
# lets use a cros config based with some required adjustments as a starting point
cp /compile/doc/stable/misc.cbq/misc/config-lazor-cros-olddefconfig-adjusted .config
make olddefconfig
make -j 8 Image dtbs modules
cd tools/perf
make
cd ../power/cpupower
make
cd ../../..
export kver=`make kernelrelease`
echo ${kver}
# remove debug info if there and not wanted
#find . -type f -name '*.ko' | sudo xargs -n 1 objcopy --strip-unneeded
make modules_install
mkdir -p /lib/modules/${kver}/tools
cp -v tools/perf/perf /lib/modules/${kver}/tools
cp -v tools/power/cpupower/cpupower /lib/modules/${kver}/tools
cp -v tools/power/cpupower/libcpupower.so.0.0.1 /lib/modules/${kver}/tools/libcpupower.so.0
# make headers_install INSTALL_HDR_PATH=/usr
cp -v .config /boot/config-${kver}
cp -v arch/arm64/boot/Image /boot/Image-${kver}
mkdir -p /boot/dtb-${kver}
cp -v arch/arm64/boot/dts/qcom/sc7180-trogdor*.dtb /boot/dtb-${kver}
cp -v System.map /boot/System.map-${kver}
# start chromebook special - required: apt-get install liblz4-tool vboot-kernel-utils
cp arch/arm64/boot/Image Image
lz4 -f Image Image.lz4
dd if=/dev/zero of=bootloader.bin bs=512 count=1
cp /compile/doc/stable/misc.cbq/misc/cmdline cmdline
mkimage -D "-I dts -O dtb -p 2048" -f auto -A arm64 -O linux -T kernel -C lz4 -a 0 -d Image.lz4 -b arch/arm64/boot/dts/qcom/sc7180-trogdor-coachz-r1-lte.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-coachz-r1.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-coachz-r3.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-lazor-r0.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-coachz-r3-lte.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-lazor-r1.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-lazor-r1-kb.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-lazor-r3.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-lazor-r1-lte.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-lazor-r3-kb.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-lazor-r3-lte.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-lazor-limozeen.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-lazor-limozeen-nots.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-lazor-limozeen-nots-r4.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-pompom-r1.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-pompom-r2-lte.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-pompom-r1-lte.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-pompom-r2.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-pompom-r3-lte.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-pompom-r3.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-r1.dtb -b arch/arm64/boot/dts/qcom/sc7180-trogdor-r1-lte.dtb kernel.itb kernel.itb
vbutil_kernel --pack vmlinux.kpart --keyblock /usr/share/vboot/devkeys/kernel.keyblock --signprivate /usr/share/vboot/devkeys/kernel_data_key.vbprivk --version 1 --config cmdline --bootloader bootloader.bin --vmlinuz kernel.itb --arch arm
cp -v vmlinux.kpart /boot/vmlinux.kpart-${kver}
rm -f Image Image.lz4 cmdline bootloader.bin kernel.itb vmlinux.kpart
# end chromebook special
cd /boot
update-initramfs -c -k ${kver}
#mkimage -A arm64 -O linux -T ramdisk -a 0x0 -e 0x0 -n initrd.img-${kver} -d initrd.img-${kver} uInitrd-${kver}
tar cvzf /compile/source/linux-stable-cbq/${kver}.tar.gz /boot/Image-${kver} /boot/System.map-${kver} /boot/config-${kver} /boot/dtb-${kver} /boot/initrd.img-${kver} /boot/vmlinux.kpart-${kver} /lib/modules/${kver}
cp -v /compile/doc/stable/config.cbq /compile/doc/stable/config.cbq.old
cp -v /compile/source/linux-stable-cbq/.config /compile/doc/stable/config.cbq
cp -v /compile/source/linux-stable-cbq/.config /compile/doc/stable/config.cbq-${kver}
cp -v /compile/source/linux-stable-cbq/*.tar.gz /compile/result/stable
