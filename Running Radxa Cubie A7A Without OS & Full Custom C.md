<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# Running Radxa Cubie A7A Without OS \& Full Custom Control

Complete guide to gaining root control, custom OS builds, bare-metal programming, and hardware-level access on your 12GB Radxa Cubie A7A for home AI and project server deployment.

This document covers everything from bare-metal programming to custom Linux builds, bootloader modification, root access, virtualization, and GPIO control—giving you full command over your SBC hardware and software stack.

## Understanding System Architecture

The Radxa Cubie A7A uses the **Allwinner A733 SoC** (octa-core ARM Cortex with 12GB LPDDR5), which boots through a multi-stage process. The chip contains a **BootROM** with built-in **FEL mode** (Flashing and Enabling Layer)—a low-level USB recovery mode that cannot be bricked, providing your ultimate safety net. This is critical: even if you completely destroy the bootloader or OS, FEL mode lets you recover via USB.[^1][^2][^3]

The boot chain follows: **BootROM → boot0 (FSBL) → U-Boot (FSBL) → Linux Kernel → Rootfs**. Every stage can be customized or replaced with your own compiled binaries, giving you complete control from silicon to userspace.[^1]

## Method 1: Building Custom OS From Source

### RadxaOS SDK Build (Recommended Starting Point)

Radxa provides their complete OS build system via Docker containers, making compilation straightforward.[^4]

**Prerequisites**: x86_64 PC with Ubuntu/Debian, VS Code, Docker installed.[^4]

**Clone the SDK**:

```bash
git clone --recurse-submodules https://github.com/RadxaOS-SDK/rsdk.git
cd rsdk
```

**Setup Development Container**: Open the directory in VS Code, install the Dev Containers extension, and click "Reopen in Container" when prompted. The first launch downloads all dependencies automatically.[^4]

**Build Complete OS Image**:

```bash
rsdk  # Launch TUI interface
# Select: "Build system image" → "radxa-cubie-a7a" → "Yes"
```

The compiled image appears in `out/radxa-cubie-a7a/output.img`. This gives you a baseline bootable image you fully control and can modify at the source level.[^4]

### Allwinner Tina SDK Build (Lower-Level Control)

For deeper customization, use Allwinner's complete SDK which includes boot0, U-Boot, kernel, and rootfs.[^1]

**Download SDK** (complete 1.4.6 repo):

- Mega: https://mega.nz/file/kFtD0BYY\#zm3FXLiLK9SfOFss3BGY1Kx714BFBqyyPeYeE5FvOw0
- BaiduPan: https://pan.baidu.com/s/1zcVq4l-rij7RPmJ92nccZg (password: 547b)[^1]

**Configuration**:

```bash
source ./build/envsetup.sh
./build.sh config
```

Select: `linux → debian → linux-5.15 → a733 → cubie_a7a → default → linaro-bullseye-xfce-arm64.tar.gz`[^1]

**Build Components Individually**:

- **Bootloader** (boot0 + U-Boot + sboot):

```bash
./build.sh bootloader
```

Outputs: `boot0_sdcard_sun60iw2p1.bin`, `u-boot-sun60iw2p1.bin`, `sboot_sdcard_sun60iw2p1.bin`[^1]

- **Kernel**:

```bash
./build.sh kernel
```

Uses defconfig from `device/config/chips/a733/configs/cubie_a7a/debian/linux-5.15/bsp_defconfig` and device tree from `device/config/chips/a733/configs/cubie_a7a/linux-5.15/board.dts`[^1]

- **Full Image**:

```bash
./build.sh
./build.sh pack
```

Find final image in `out/` directory.[^1]

## Method 2: Custom Kernel \& Device Tree Compilation

### Kernel Development Using Dev Containers

**Clone kernel source**:

```bash
git clone --recurse-submodules https://github.com/radxa-pkg/linux-a733.git
cd linux-a733
```

**Start Dev Container** (same VS Code + Docker workflow as RadxaOS SDK).[^5]

**Compile kernel**:

```bash
make deb
```

This generates `.deb` packages you can install on the running system:[^5]

```bash
sudo dpkg -i linux-headers-*.deb
sudo dpg -i linux-image-*.deb
sudo reboot
```


### Custom Device Tree Overlays for GPIO/Peripherals

Device tree overlays let you reconfigure pins, enable/disable peripherals, and customize hardware behavior without recompiling the entire kernel.[^6][^7]

**Create overlay** (example for custom I2C sensor on pins):

```dts
/dts-v1/;
/plugin/;

/ {
    compatible = "allwinner,sun50i-a64";
    
    fragment@0 {
        target-path = "/";
        __overlay__ {
            my_device {
                compatible = "my-custom-device";
                pinctrl-names = "default";
                status = "okay";
            };
        };
    };
};
```

**Compile and activate**:

```bash
# Copy .dts file to board
armbian-add-overlay custom.dts  # If using Armbian
# Or manually: dtc -@ -I dts -O dtb -o custom.dtbo custom.dts
sudo cp custom.dtbo /boot/dtbs/allwinner/overlays/
```

**Enable via rsetup** (Radxa configuration tool): `rsetup → Overlays → Manage overlays → Select your overlay`[^7][^6]

## Method 3: U-Boot Customization \& Bootloader Control

### Building Custom U-Boot

**Using Dev Containers**:

```bash
git clone https://github.com/radxa/u-boot.git -b cubie-aiot-v1.4.6
cd u-boot
# Open in VS Code Dev Container
make deb
```

Install the resulting `.deb` on the board:[^8]

```bash
sudo dpkg -i u-boot-*.deb
cd /usr/lib/u-boot/radxa-cubie-a7a/
sudo ./setup.sh update_bootloader /dev/mmcblk1  # Adjust device as needed
sudo reboot
```


### U-Boot Environment Modification

Access U-Boot console via serial UART (pins 8/10/6 on 40-pin header at 115200 baud):[^9][^10]

```
# Interrupt boot by pressing any key during countdown
U-Boot> printenv  # Show all variables
U-Boot> setenv bootdelay 5  # Set boot delay to 5 seconds
U-Boot> saveenv
U-Boot> boot
```

**Custom boot commands** for testing kernels via USB/network without flashing:

```
U-Boot> usb start
U-Boot> fatload usb 0:1 ${kernel_addr_r} /zImage
U-Boot> fatload usb 0:1 ${fdt_addr} /sun60iw2p1-cubie-a7a.dtb
U-Boot> bootz ${kernel_addr_r} - ${fdt_addr}
```


## Method 4: FEL Mode Recovery \& USB Programming

**FEL mode** is your unbrickable recovery path built into the Allwinner A733 BootROM.[^3][^11]

### Entering FEL Mode

1. Press and hold the **UBOOT button** on the Cubie A7A (near USB-C port)[^11]
2. Connect USB Type-C cable from board to PC
3. Release UBOOT button
4. Verify: `lsusb` should show "Allwinner Technology"[^3]

### Using sunxi-fel Tools

**Install tools**:

```bash
git clone https://github.com/linux-sunxi/sunxi-tools.git
cd sunxi-tools
make
sudo make install
```

**Verify FEL mode**:

```bash
sunxi-fel ver
```

**Boot U-Boot from RAM** (testing without flashing):

```bash
sunxi-fel spl u-boot-sunxi-with-spl.bin
```

**Read/write memory** (dump firmware, patch bootloader in RAM):

```bash
sunxi-fel read 0x40000000 0x100000 firmware.dump
sunxi-fel write 0x40000000 custom_payload.bin
sunxi-fel exec 0x40000000
```

**Flash images using Phoenix tools** (for complete system images):

- Linux: `phoenixconsole` (CLI tool from Radxa)[^1]
- Windows: PhoenixCard v4.3.2[^1]

Boot into FEL mode and use Phoenix tools to flash complete `.img` files to eMMC/SD card.

## Method 5: Root Access \& System Modification

### Default Root Access

RadxaOS and Debian images typically ship with user `radxa` password `radxa`. Gain root:[^12]

```bash
sudo su -
```


### Modifying Root Filesystem

**Mount image on development PC**:

```bash
sudo losetup -fP radxa-cubie-a7a.img
sudo mount /dev/loop0p2 /mnt  # Root partition
sudo mount /dev/loop0p1 /mnt/boot  # Boot partition
```

**Customize**:

- Add packages: `sudo chroot /mnt apt install <packages>`
- Modify system files: Edit `/mnt/etc/*` as needed
- Add startup scripts: Place in `/mnt/etc/rc.local`
- Inject SSH keys: Copy to `/mnt/root/.ssh/authorized_keys`

**Unmount**:

```bash
sudo umount /mnt/boot /mnt
sudo losetup -d /dev/loop0
```


### Persistent Root via Kernel Command Line

Modify `/boot/extlinux/extlinux.conf` or U-Boot environment to add `init=/bin/bash` for emergency shell, or configure autologin.[^6]

## Method 6: Bare-Metal Programming (No OS)

### ARM Cortex Bare-Metal Workflow

For running custom code directly on the CPU without any OS layer, you write firmware that executes from reset.[^13][^14]

**Minimal bare-metal GPIO blink** (conceptual):

```c
#define GPIOB_BASE 0x40020400
#define RCC_AHB1ENR (*(volatile uint32_t*)0x40023830)

void main(void) {
    RCC_AHB1ENR |= (1 << 1);  // Enable GPIOB clock
    
    volatile uint32_t *GPIOB_MODER = (uint32_t*)(GPIOB_BASE + 0x00);
    *GPIOB_MODER |= (1 << 14);  // Set PB7 to output
    
    volatile uint32_t *GPIOB_ODR = (uint32_t*)(GPIOB_BASE + 0x14);
    
    while(1) {
        *GPIOB_ODR ^= (1 << 7);  // Toggle PB7
        for(volatile int i=0; i<100000; i++);  // Delay
    }
}
```

**Build** with ARM cross-compiler:

```bash
arm-none-eabi-gcc -mcpu=cortex-a53 -nostdlib -o firmware.elf main.c startup.s
arm-none-eabi-objcopy -O binary firmware.elf firmware.bin
```

**Load via FEL**:

```bash
sunxi-fel write 0x40000000 firmware.bin
sunxi-fel exec 0x40000000
```

For serious bare-metal work on A733, you'll need to study the Allwinner A733 User Manual for peripheral register addresses and write your own startup code (stack setup, interrupt vectors, etc.).[^13][^1]

## Method 7: Cross-Compilation Workflow

### Setting Up ARM64 Toolchain

**Install cross-compiler** (on Ubuntu x86_64 development PC):

```bash
sudo apt install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu
```

**Cross-compile kernel modules**:

```bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
cd /path/to/kernel/source
make -j$(nproc) Image modules
```

**Cross-compile userspace applications**:

```c
// hello.c
#include <stdio.h>
int main() { printf("Hello ARM64!\n"); return 0; }
```

```bash
aarch64-linux-gnu-gcc -o hello hello.c
file hello  # Verify: ELF 64-bit LSB executable, ARM aarch64
scp hello radxa@cubie-a7a:/home/radxa/
ssh radxa@cubie-a7a ./hello
```


### Building Kernel Modules for Cubie A7A

**Prepare kernel source** matching the running kernel:

```bash
uname -r  # Check running version on board
# Download matching kernel source on dev PC
git clone https://github.com/radxa/kernel.git -b allwinner-aiot-linux-5.15
cd kernel
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
make cubie_a7a_defconfig  # Or extract config from board: scp radxa@cubie:/proc/config.gz .
make modules_prepare
```

**Write module** (e.g., `hello_module.c`):

```c
#include <linux/module.h>
#include <linux/kernel.h>

static int __init hello_init(void) {
    printk(KERN_INFO "Hello from Cubie A7A module!\n");
    return 0;
}

static void __exit hello_exit(void) {
    printk(KERN_INFO "Goodbye from module!\n");
}

module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL");
```

**Makefile**:

```makefile
obj-m += hello_module.o

KDIR := /path/to/linux-a733
PWD := $(shell pwd)

default:
	$(MAKE) ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

**Build and deploy**:

```bash
make
scp hello_module.ko radxa@cubie-a7a:/home/radxa/
ssh radxa@cubie-a7a
sudo insmod hello_module.ko
dmesg | tail  # See "Hello from Cubie A7A module!"
sudo rmmod hello_module
```


## Method 8: GPIO \& Hardware Interface Control

### Direct GPIO Access via sysfs/libgpiod

The 40-pin header exposes UART, SPI, I2C, PWM, and GPIO pins.[^15]

**Using libgpiod** (modern, recommended):

```bash
sudo apt install gpiod
gpiodetect  # List GPIO chips
gpiofind PJ23  # Find chip and line number for pin
gpioget gpiochip0 23  # Read state
gpioset gpiochip0 23=1  # Set high
gpioget $(gpiofind PJ23)  # Combined command
```

**In C** using libgpiod:

```c
#include <gpiod.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    struct gpiod_chip *chip = gpiod_chip_open_by_name("gpiochip0");
    struct gpiod_line *line = gpiod_chip_get_line(chip, 23);  // PJ23
    
    gpiod_line_request_output(line, "blink", 0);
    
    for(int i=0; i<10; i++) {
        gpiod_line_set_value(line, 1);
        sleep(1);
        gpiod_line_set_value(line, 0);
        sleep(1);
    }
    
    gpiod_line_release(line);
    gpiod_chip_close(chip);
    return 0;
}
```

Compile: `gcc -o blink blink.c -lgpiod`[^16]

### UART Serial Console Access

The **UART0** interface (pins 6/8/10 on 40-pin header) provides low-level system access for debugging and bootloader interaction.[^10][^9]

**Hardware connection**:

- Pin 6 (GND) → USB-TTL GND (black)
- Pin 8 (UART0_TX) → USB-TTL RX (white)
- Pin 10 (UART0_RX) → USB-TTL TX (green)
- Do NOT connect VCC[^9]

**Connect via minicom** (Linux):

```bash
sudo apt install minicom
sudo minicom -s  # Configure: /dev/ttyUSB0, 115200 8N1, no flow control
```

**Or via Tabby/screen**:

```bash
sudo screen /dev/ttyUSB0 115200
```

This gives you console access from bootloader through kernel boot to login prompt—critical for debugging failed boots.[^17][^18]

## Method 9: Virtualization with KVM/QEMU

### Running VMs on ARM64 SBC

The A733's ARM Cortex cores support **hardware virtualization** (if enabled in kernel config).[^19][^20][^21]

**Install KVM/QEMU**:

```bash
sudo apt install qemu-system-aarch64 qemu-kvm libvirt-daemon-system virtinst
sudo systemctl enable --now libvirtd
sudo usermod -aG kvm,libvirt $USER
```

**Verify KVM support**:

```bash
lsmod | grep kvm
# Should show kvm and kvm_arm (or similar)
dmesg | grep -i kvm
```

**Create VM**:

```bash
qemu-system-aarch64 \
  -enable-kvm \
  -M virt \
  -cpu host \
  -smp 4 \
  -m 4096 \
  -nographic \
  -kernel /path/to/Image \
  -append "console=ttyAMA0 root=/dev/vda" \
  -drive file=debian-arm64.qcow2,if=none,id=hd0 \
  -device virtio-blk-device,drive=hd0
```

**Nested containerization** with Docker/Podman works natively on ARM64:

```bash
sudo apt install docker.io
sudo systemctl enable --now docker
sudo docker run --rm -it arm64v8/ubuntu bash
```


## Method 10: ARM Trusted Firmware (TF-A/ATF)

For true low-level control, you can rebuild **BL31** (the EL3 secure firmware).[^22][^23][^24]

**Clone TF-A**:

```bash
git clone https://git.trustedfirmware.org/TF-A/trusted-firmware-a.git
cd trusted-firmware-a
```

**Build for A733** (sun50i platform):

```bash
make CROSS_COMPILE=aarch64-linux-gnu- PLAT=sun50i_a64 DEBUG=1
# Output: build/sun50i_a64/debug/bl31.bin
```

**Inject into U-Boot build**:

```bash
export BL31=/path/to/trusted-firmware-a/build/sun50i_a64/debug/bl31.bin
cd /path/to/u-boot
make CROSS_COMPILE=aarch64-linux-gnu- <cubie_defconfig>
make
```

This gives you control over PSCI (Power State Coordination Interface), secure boot, and low-level SoC initialization.[^24]

## Advanced Topics

### Boot Priority \& Multi-Boot Setup

The Cubie A7A boot priority: **MicroSD Card > NVMe SSD > UFS Module > eMMC Module**. Use this to test experimental OS builds on SD card while keeping stable OS on eMMC.[^25]

### SPI Flash Programming

The board has 128Mbit SPI flash (Winbond W25Q128). You can store U-Boot here for instant boot without SD/eMMC:[^1]

```bash
# From U-Boot console
sf probe
sf erase 0 +<size>
sf write ${load_addr} 0 <size>
```


### Android System Modification

The A733 supports Android 13. Build custom Android using Allwinner SDK (select `android` instead of `linux` during config).[^26][^27][^1]

## Safety \& Recovery

**Never lose access**:

1. **FEL mode** is always available via hardware button—you cannot brick the board[^11][^3]
2. Keep a **working SD card image** as backup before experimenting
3. **Serial console** access (UART0) lets you debug even if networking/display fails[^9]
4. **Multi-boot**: Test on SD card before flashing eMMC[^25]

## Summary Workflow for Full Control

1. **Build custom OS** using RadxaOS SDK or Allwinner Tina SDK[^4][^1]
2. **Modify bootloader** (U-Boot) for custom boot logic, environment, or splash screens[^28]
3. **Compile custom kernel** with your drivers, security modules, and optimizations[^5]
4. **Create device tree overlays** to reconfigure GPIO, enable custom hardware[^7][^6]
5. **Cross-compile applications \& modules** on x86_64 host for ARM64 target[^29][^30]
6. **Flash via FEL mode** for unbrickable updates[^11]
7. **Access via serial console** for low-level debugging[^9]
8. **Control GPIO** directly via libgpiod for sensors, actuators[^16]
9. **Run VMs/containers** for isolated AI workloads[^20]
10. **Build ARM TF-A** for ultimate secure boot control[^24]

You now have complete sovereignty over your Cubie A7A from silicon to userspace—ideal for home AI server, library management, and custom embedded Linux projects requiring full hardware and software control.
<span style="display:none">[^31][^32][^33][^34][^35][^36][^37][^38][^39][^40][^41][^42][^43][^44][^45][^46][^47][^48][^49][^50][^51][^52][^53][^54][^55][^56][^57][^58][^59][^60][^61][^62][^63][^64][^65][^66][^67][^68][^69][^70][^71][^72][^73][^74][^75][^76]</span>

<div align="center">⁂</div>

[^1]: https://linux-sunxi.org/Radxa_Cubie_A7A

[^2]: https://github.com/Siragon/exdroid-driver-fel

[^3]: https://linux-sunxi.org/FEL

[^4]: https://docs.radxa.com/en/cubie/a7a/low-level-dev/system-build/radxa_os

[^5]: https://docs.radxa.com/en/cubie/a7a/low-level-dev/system-build/kernel

[^6]: https://docs.radxa.com/en/zero/zero3/os-config/rsetup

[^7]: https://docs.armbian.com/User-Guide_Armbian_overlays/

[^8]: https://docs.radxa.com/en/compute-module/nx5/radxa-os/low-level-dev/u-boot

[^9]: https://docs.radxa.com/en/cubie/a7a/other-system/android/uart_debug

[^10]: https://docs.radxa.com/en/cubie/a7z/system-config/uart_debug

[^11]: https://docs.radxa.com/en/cubie/a7a/low-level-dev/install-system/fel-install-system/fel_mode

[^12]: https://forum.armbian.com/topic/56130-radxa-cubie-a7aa7z-allwinner-a733/

[^13]: https://github.com/cpq/bare-metal-programming-guide

[^14]: https://www.reddit.com/r/learnprogramming/comments/16qbnhe/is_it_possible_to_run_programs_without_an_os/

[^15]: https://docs.radxa.com/en/cubie/a7a/hardware-use/pin-gpio

[^16]: https://docs.radxa.com/en/rock5/rock5b/app-development/gpiod

[^17]: https://docs.radxa.com/en/rock5/rock5b/radxa-os/serial

[^18]: https://docs.radxa.com/en/zero/zero/low-level-dev/serial

[^19]: https://stackoverflow.com/questions/66570811/virtualizing-multiple-sbcs-on-an-arm-based-instance

[^20]: https://community.arm.com/oss-platforms/w/docs/510/spawn-a-linux-virtual-machine-on-arm-using-qemu-kvm

[^21]: https://aircconline.com/ijcnc/V15N2/15223cnc08.pdf

[^22]: https://git.congatec.com/arm-nxp/imx8-family/atf-imx8-family/-/blob/cgtimx8__imx_4.14.98_2.2.0/docs/plat/allwinner.rst

[^23]: https://docs.u-boot.org/en/stable/board/allwinner/sunxi.html

[^24]: https://trustedfirmware-a.readthedocs.io/en/latest/plat/allwinner.html

[^25]: https://docs.radxa.com/en/cubie/a7a/getting-started/quickly_start

[^26]: https://docs.radxa.com/en/cubie/a7a/other-system/android/install_system

[^27]: https://docs.radxa.com/en/cubie/a7a

[^28]: https://docs.radxa.com/en/cubie/a7a/low-level-dev/system-build/uboot

[^29]: https://doc.dpdk.org/guides-19.11/linux_gsg/cross_build_dpdk_for_arm64.html

[^30]: https://dev.to/samar_lass_27db28cecec23c/cross-compiling-linux-kernel-module-a-hands-on-guide-b7d

[^31]: https://docs.radxa.com/en/rock4/rock4c+/getting-started/install-os

[^32]: https://docs.radxa.com/en/cubie/a7a/getting-started

[^33]: https://xdaforums.com/t/tutorial-how-to-unlock-bootloader-and-install-custom-recovery-and-root.2895763/

[^34]: https://wiki.radxa.com/Zero/dev/u-boot

[^35]: https://docs.radxa.com/en/rock3/rock3a/low-level-dev/kernel

[^36]: https://docs.radxa.com/en/cubie/a7a/getting-started/install-system

[^37]: https://docs.radxa.com/en/rock5/rock5c/other-os/buildroot

[^38]: https://docs.radxa.com/en/rock4/rock4d/other-os/buildroot

[^39]: https://docs.radxa.com/en/cubie/a7a/download

[^40]: https://docs.radxa.com/en/cubie/a7a/low-level-dev/system-build

[^41]: https://forum.radxa.com/t/custom-kernel-build-using-radxa-repo-bsp/13945

[^42]: https://www.radxa.com/products/cubie/a7a/

[^43]: https://xdaforums.com/t/a-pain-to-root-cheap-allwinner-a33-tablet-please-help-me.3564543/

[^44]: https://tinyhack.com/2024/01/18/using-u-boot-to-extract-boot-image-from-pritom-p7/

[^45]: https://www.youtube.com/watch?v=ksbRMo00uTA

[^46]: https://www.youtube.com/watch?v=y40MHUkyJ4s

[^47]: https://www.youtube.com/watch?v=v7zUCqCqr38

[^48]: https://community.st.com/t5/stm32-mcus-products/gpio-not-working-by-baremetal-programming/td-p/686486

[^49]: https://wiki.debian.org/InstallingDebianOn/Allwinner

[^50]: https://docs.radxa.com/en/cubie/a7z/hardware-use/pin-gpio

[^51]: https://www.youtube.com/watch?v=UQWaiZde_Tg

[^52]: https://www.linkedin.com/posts/kusuma-kiran-kumar_embeddedsystems-baremetalprogramming-arm-activity-7386996559584006144-1BjG

[^53]: https://www.qemu.org/2025/08/26/qemu-10-1-0/

[^54]: https://tinyhack.com/2024/01/

[^55]: https://www.skysilk.com/blog/kvm-vs-qemu/

[^56]: https://whycan.com/t_12059.html

[^57]: https://docs.radxa.com/en/cubie/a5e/hardware-use/pin-gpio

[^58]: https://docs.radxa.com/en/rock4/rock4ab-se/low-level-dev/rootfs-armhf

[^59]: https://www.reddit.com/r/Arcade1Up/comments/g5380p/linux_hacking_arcade1up_fel_mode_usbboot/

[^60]: https://xor.co.za/post/2018-12-01-fel-bootprocess/

[^61]: https://git.ti.com/cgit/atf/arm-trusted-firmware/tree/docs/plat/allwinner.rst?h=ti-atf\&id=65f78887a637b15ec6dc99cbd8871413fdaf64b3

[^62]: https://docs.radxa.com/en/rock5/rock5c/radxa-os/headless

[^63]: https://docs.radxa.com/en/cubie/a5e/other-system/tina-os/build-system

[^64]: https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842305/Build+ARM+Trusted+Firmware+ATF?view=blog

[^65]: https://docs.radxa.com/en/cubie/a7a/system-config

[^66]: https://docs.radxa.com/en/rock4/rock4d/system-config/uart_debug

[^67]: https://forum.beagleboard.org/t/how-to-set-gpio-in-a-custom-device-tree-overlay/33843

[^68]: https://docs.radxa.com/en/cubie/a5e/system-config/uart_login

[^69]: https://community.toradex.com/t/changing-gpio-behaviour-using-device-tree-overlay/26426

[^70]: https://www.youtube.com/watch?v=3gnPfqhToMY

[^71]: https://gist.github.com/artizirk/96f3db0e389ffa6a3c2cc00ae8e4c9e8

[^72]: https://stackoverflow.com/questions/25572395/cross-compiling-a-kernel-module-arm

[^73]: https://evelta.com/cubie-a7a-edge-ai-innovation-sbc-radxa/

[^74]: https://www.mistrasolutions.com/page/device-tree-overlay/

[^75]: https://embear.ch/posts/compiling-a-kernel-module/

[^76]: https://docs.radxa.com/en/cubie/a7a/other-system/android/quickly_start

