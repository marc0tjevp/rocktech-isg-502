# Rocktech ISG-502

![PXL_20251004_203513867](https://github.com/user-attachments/assets/6ff13c6a-ba7f-4e6e-8f0f-ab2f130f6393)

This device can be found on eBay for ~25-40 EUR. It's described as being an "Industrial Smart Gateway".  
Inside is a Raspberry Pi 4B, making it a cheap and interesting dev box. The bundled daughterboard provides  
extra I/O including RS-232/RS-485 and an RX8010 RTC.

[Hackaday post](https://hackaday.io/project/195148-reverse-engineering-rocktech-isg-502) with hardware info.

---

## Device setup

**Device:** Rocktech ISG-502 (Raspberry Pi 4B inside)  
**OS:** Raspberry Pi OS **Bullseye** Lite (arm64, 2023-05-03 image)  
**Kernel:** `6.1.xx-v8+`

> ✅ This kernel is stable for both RX8010 RTC and CH432 UART drivers.  
> ❌ Newer kernels (6.6/6.12) break out-of-tree builds.

---

## Install flow

1. Flash image  
   - Download: `2023-05-03-raspios-bullseye-arm64-lite.img.xz`  
   - Flash with Raspberry Pi Imager or balenaEtcher.  
   - Boot and SSH in (`ssh pi@<ip>`).

2. Enable SPI + I²C  
   ```bash
   sudo raspi-config
   # → Interface Options → Enable I2C, Enable SPI
   sudo reboot
   ```

3. Update system  
   ```bash
   sudo apt-get update
   sudo apt-get upgrade -y
   ```

4. Build drivers  
   - [hwclock.md](./hwclock.md) → RX8010 RTC module  
   - [serial.md](./serial.md) → CH432 UART driver
   - [gpio.md](./gpio.md) → IO0-IO3 and Status LED
