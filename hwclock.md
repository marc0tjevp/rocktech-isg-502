# RX8010 RTC on Rocktech ISG-502

The daughterboard provides a **Seiko/EPSON RX8010 RTC** on I²C bus 1 at address `0x32`.

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

4. Build + insert:  
   ```bash
   make
   sudo modprobe regmap_i2c
   sudo insmod ./rtc-rx8010.ko
   ```

---

## Bind the device

1. Detect chip:  
   ```bash
   sudo i2cdetect -y 1
   # → should show 0x32
   ```

2. Bind:  
   ```bash
   echo rx8010 0x32 | sudo tee /sys/class/i2c-adapter/i2c-1/new_device
   ```

3. Verify:  
   ```bash
   ls -l /dev/rtc*
   sudo hwclock -r
   ```

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
