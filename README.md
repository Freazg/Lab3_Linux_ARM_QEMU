# Лабораторна робота 3: Розгортання Linux на ARM через QEMU

**Студент:** Сас Євгеній, група IO-31, номер 20  
**Викладач:** Каплунов Артем Володимирович  
**Дата:** 23 листопада 2025

## Мета роботи
Вивчити процес кросс-компіляції програмного забезпечення для ARM архітектури, зібрати повноцінну Linux систему (U-Boot, Kernel, BusyBox) та запустити її в емуляторі QEMU.

## Структура проекту
```
repos/
├── u-boot/          # Bootloader U-Boot v2019.07
├── linux-stable/    # Linux Kernel 4.19.y
└── busybox/         # BusyBox 1.31 + rootfs
    ├── _install/    # Підготовлений rootfs
    └── rootfs.cpio.gz  # Архів rootfs для QEMU
```

## Зібране ПЗ

### U-Boot (Bootloader)
- Версія: v2019.07
- Конфігурація: am335x_evm_defconfig
- Тулчейн: arm-eabi-gcc 8.3.0
- Результат: MLO, u-boot.img

### Linux Kernel
- Версія: 4.19.325
- Конфігурація: multi_v7_defconfig + BBB фрагмент
- Тулчейн: arm-eabi-gcc 8.3.0
- Результат: zImage (8.6 MB), am335x-boneblack.dtb (35 KB)

### BusyBox + RootFS
- Версія: 1.31.1
- Конфігурація: defconfig
- Тулчейн: arm-linux-gnueabihf-gcc 8.3.0
- Результат: rootfs.cpio.gz (24 MB)

## Запуск в QEMU
```bash
cd ~/repos/busybox
qemu-system-arm -kernel _install/boot/zImage -initrd rootfs.cpio.gz \
  -machine virt -nographic -m 512 \
  -append "rdinit=/init console=ttyAMA0"
```

## Тестування
Система успішно завантажилась та виконала всі тестові команди:
- `uname -a` → Linux 4.19.325 armv7l
- `ls -l` → Коректна структура директорій
- `dmesg | grep init` → Init process запущено
- `busybox --help` → BusyBox v1.31.1 працює

## Інструменти
- gcc-9 (host compiler)
- ARM toolchains (baremetal + Linux)
- ccache (кешування компіляції)
- QEMU (емуляція ARM)
