# Інструкція зі збірки

## 1. Підготовка інструментів
```bash
# Оновлення пакетів
sudo apt update

# Встановлення базових інструментів
sudo apt install git vim tree curl make libncurses-dev libssl-dev bc bison flex -y

# Встановлення gcc-9 (для уникнення помилки з gcc-10+)
sudo apt install gcc-9 g++-9 -y
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-9 9

# Встановлення ccache
sudo apt install ccache -y
ccache -M 5G

# Встановлення QEMU
sudo apt install qemu-system-arm cpio -y
```

## 2. Завантаження тулчейнів
```bash
cd ~/Downloads

# Baremetal toolchain
wget https://developer.arm.com/-/media/Files/downloads/gnu-a/8.3-2019.03/binrel/gcc-arm-8.3-2019.03-x86_64-arm-eabi.tar.xz
sudo tar xJf gcc-arm-8.3-2019.03-x86_64-arm-eabi.tar.xz -C /opt/

# Linux toolchain
wget https://developer.arm.com/-/media/Files/downloads/gnu-a/8.3-2019.03/binrel/gcc-arm-8.3-2019.03-x86_64-arm-linux-gnueabihf.tar.xz
sudo tar xJf gcc-arm-8.3-2019.03-x86_64-arm-linux-gnueabihf.tar.xz -C /opt/
```

## 3. Збірка U-Boot
```bash
cd ~/repos
git clone https://gitlab.denx.de/u-boot/u-boot.git
cd u-boot
git checkout v2019.07

export PATH=/opt/gcc-arm-8.3-2019.03-x86_64-arm-eabi/bin:$PATH
export CROSS_COMPILE='ccache arm-eabi-'
export ARCH=arm

make am335x_evm_defconfig
make -j4
```

**Результат:** MLO, u-boot.img

## 4. Збірка Linux Kernel
```bash
cd ~/repos
git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git
cd linux-stable
git checkout linux-4.19.y

export PATH=/opt/gcc-arm-8.3-2019.03-x86_64-arm-eabi/bin:$PATH
export CROSS_COMPILE='ccache arm-eabi-'
export ARCH=arm

# Створення конфігураційного фрагмента
mkdir fragments
cat > fragments/bbb.cfg << 'FRAGMENT'
CONFIG_USB_ANNOUNCE_NEW_DEVICES=y
CONFIG_USB_EHCI_ROOT_HUB_TT=y
CONFIG_AM335X_PHY_USB=y
CONFIG_USB_MUSB_TUSB6010=y
CONFIG_USB_MUSB_OMAP2PLUS=y
CONFIG_USB_MUSB_HDRC=y
CONFIG_USB_MUSB_DSPS=y
CONFIG_USB_MUSB_AM35X=y
CONFIG_USB_CONFIGFS=y
CONFIG_NOP_USB_XCEIV=y
CONFIG_USB_HID=y
CONFIG_USB_HIDDEV=y
CONFIG_USB_SERIAL=y
CONFIG_USB_SERIAL_PL2303=y
CONFIG_USB_SERIAL_GENERIC=y
CONFIG_USB_SERIAL_SIMPLE=y
CONFIG_USB_SERIAL_FTDI_SIO=y
CONFIG_USB_ULPI=y
CONFIG_USB_ULPI_BUS=y
CONFIG_BRIDGE=y
CONFIG_OF_OVERLAY=y
FRAGMENT

# Генерація конфігурації
./scripts/kconfig/merge_config.sh arch/arm/configs/multi_v7_defconfig fragments/bbb.cfg

# Збірка
make -j4 zImage modules am335x-boneblack.dtb
```

**Результат:** zImage (8.6 MB), am335x-boneblack.dtb (35 KB)

## 5. Збірка BusyBox
```bash
cd ~/repos
git clone git://git.busybox.net/busybox
cd busybox
git checkout 1_31_stable

export ARCH=arm
export PATH=/opt/gcc-arm-8.3-2019.03-x86_64-arm-linux-gnueabihf/bin:$PATH
export CROSS_COMPILE="ccache arm-linux-gnueabihf-"

make defconfig
make -j4
make install
```

## 6. Підготовка RootFS
```bash
cd ~/repos/busybox

# Створення директорій
mkdir -p _install/{boot,dev,etc/init.d,lib,proc,root,sys/kernel/debug,tmp}

# Створення init скрипта
cat > _install/etc/init.d/rcS << 'INITSCRIPT'
#!/bin/sh
mount -t sysfs none /sys
mount -t proc none /proc
mount -t debugfs none /sys/kernel/debug
echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s
INITSCRIPT

chmod +x _install/etc/init.d/rcS
ln -s bin/busybox _install/init

# Копіювання файлів ядра
cp ~/repos/linux-stable/arch/arm/boot/zImage _install/boot/
cp ~/repos/linux-stable/arch/arm/boot/dts/am335x-boneblack.dtb _install/boot/

# Встановлення модулів ядра
cd ~/repos/linux-stable
export INSTALL_MOD_PATH=~/repos/busybox/_install
export ARCH=arm
make modules_install

# Копіювання бібліотек
cd ~/repos/busybox/_install/lib
export CROSS_COMPILE="arm-linux-gnueabihf-"
libc_dir=$(${CROSS_COMPILE}gcc -print-sysroot)/lib
cp -a $libc_dir/*.so* .

# Створення конфігураційних файлів
cd ~/repos/busybox/_install
echo '$MODALIAS=.* root:root 660 @modprobe "$MODALIAS"' > etc/mdev.conf
echo 'root:x:0:' > etc/group
echo 'root:x:0:0:root:/root:/bin/sh' > etc/passwd
echo 'root::10933:0:99999:7:::' > etc/shadow
echo "nameserver 8.8.8.8" > etc/resolv.conf

# Створення архіву
cd ~/repos/busybox/_install
find . -print0 | cpio --null -o --format=newc | gzip -9 > ../rootfs.cpio.gz
```

**Результат:** rootfs.cpio.gz (24 MB)

## 7. Запуск в QEMU
```bash
cd ~/repos/busybox
qemu-system-arm -kernel _install/boot/zImage -initrd rootfs.cpio.gz \
  -machine virt -nographic -m 512 \
  -append "rdinit=/init console=ttyAMA0"
```

**Вихід:** Ctrl-A, потім X

## 8. Тестування

В запущеній системі виконати:
```bash
uname -a
ls -l
dmesg | grep init
busybox --help | head -15
poweroff
```
