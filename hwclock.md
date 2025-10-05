# RX8010 RTC on Rocktech ISG-502

The daughterboard provides a **Seiko RX8010 RTC** on I²C bus 1 at address `0x32`.

---

## Build the driver

1. Install headers and tools:  
   ```bash
   sudo apt-get install -y build-essential raspberrypi-kernel-headers i2c-tools
   ```

2. Fetch RX8010 source (from Linux 6.1 tree):  
   ```bash
   mkdir ~/rx8010 && cd ~/rx8010
   curl -L 'https://chromium.googlesource.com/chromiumos/third_party/kernel/+/refs/tags/v6.1.92/drivers/rtc/rtc-rx8010.c?format=TEXT' \
     | base64 -d > rtc-rx8010.c
   ```

3. Makefile:  
   ```makefile
   obj-m += rtc-rx8010.o
   KDIR := /lib/modules/$(shell uname -r)/build
   all: ; $(MAKE) -C $(KDIR) M=$(PWD) modules
   clean: ; $(MAKE) -C $(KDIR) M=$(PWD) clean
   ```

4. Build + install:  
   ```bash
   make
   sudo mkdir -p /lib/modules/$(uname -r)/extra
   sudo cp rtc-rx8010.ko /lib/modules/$(uname -r)/extra/
   sudo depmod -a
   ```

5. Load dependencies and module:  
   ```bash
   echo regmap_i2c | sudo tee /etc/modules-load.d/regmap_i2c.conf
   echo rtc_rx8010 | sudo tee /etc/modules-load.d/rtc_rx8010.conf
   ```

---

## Bind with Device Tree (persistent)

Instead of manually instantiating with `echo new_device`, use a DT overlay so the kernel binds the chip at boot.

1. Create overlay source `/boot/overlays/rx8010.dtso`:  
   ```dts
   /dts-v1/;
   /plugin/;
   / {
     compatible = "brcm,bcm2711";
     fragment@0 {
       target = <&i2c1>;
       __overlay__ {
         #address-cells = <1>;
         #size-cells = <0>;
         status = "okay";
         rtc@32 {
           compatible = "epson,rx8010";
           reg = <0x32>;
         };
       };
     };
   };
   ```

2. Compile it:  
   ```bash
   sudo apt-get install -y device-tree-compiler
   sudo dtc -@ -I dts -O dtb -o /boot/overlays/rx8010.dtbo /boot/overlays/rx8010.dtso
   ```

3. Enable in `/boot/config.txt`:  
   ```bash
   echo "dtoverlay=rx8010" | sudo tee -a /boot/config.txt
   ```

4. Reboot:  
   ```bash
   sudo reboot
   ```

---

## Verify

After reboot, the kernel should automatically register the RTC:  
```bash
dmesg -T | grep -i rx8010
ls -l /dev/rtc*
sudo hwclock -r
```

Expected output:  
- `rtc-rx8010` registered as `rtc0`  
- `/dev/rtc0` present  
- `hwclock` reads the correct time

---

## First sync

On first boot the RX8010 usually reports **Frequency Stop** (no battery or cleared state).  
Sync system time to the chip:  

```bash
sudo timedatectl set-ntp true
sudo hwclock --systohc
sudo hwclock -r
```

Now the RTC retains correct time across reboots and power cycles.
