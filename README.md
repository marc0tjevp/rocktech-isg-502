# Rocktech ISG-502 — RTC (RX8010) & CH432 UART Setup

**Device:** Rocktech ISG-502 (Raspberry Pi 4B inside)  
**OS:** Raspberry Pi OS **Bookworm** Lite, 64-bit  
**Kernel (example):** `6.12.47+rpt-rpi-v8`

This README captures exactly what worked: commands, configs, and Python helper scripts.

---

## 0) Prereqs (I²C, SPI, packages)

Enable I²C and SPI:

```bash
sudo raspi-config
# Interface Options → I2C → Enable
# Interface Options → SPI → Enable
sudo reboot
```

Install tools/libs:

```bash
sudo apt update
sudo apt install -y i2c-tools python3-spidev build-essential linux-headers-$(uname -r) git
```

Quick checks:

```bash
# I²C bus
i2cdetect -y 1     # expect to see 0x32 for the RX8010 if present

# SPI nodes
ls -l /dev/spidev0.0 /dev/spidev0.1   # spidev0.0 is the CH432 on this box
```

---

## 1) RX8010 RTC — build, load, bind

The kernel didn’t ship `rtc-rx8010.ko`, so build it:

```bash
mkdir ~/rx8010 && cd ~/rx8010
wget https://raw.githubusercontent.com/torvalds/linux/master/drivers/rtc/rtc-rx8010.c

cat > Makefile <<'EOF'
obj-m += rtc-rx8010.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
EOF

make
sudo mkdir -p /lib/modules/$(uname -r)/kernel/drivers/rtc
sudo cp rtc-rx8010.ko /lib/modules/$(uname -r)/kernel/drivers/rtc/
sudo depmod -a
sudo modprobe rtc-rx8010
```

Bind the device (address **0x32**):

```bash
echo rx8010 0x32 | sudo tee /sys/bus/i2c/devices/i2c-1/new_device
```

Verify:

```bash
ls -l /dev/rtc*
sudo hwclock -r
```

---

## 2) Prime the RTC with correct time (once)

Make sure system time is correct (NTP on), then write it to the RTC:

```bash
timedatectl status        # confirm NTP=active and time looks sane
sudo hwclock -w           # push system time → RTC
sudo hwclock -r           # read back to confirm
```

---

## 3) Re-bind RTC automatically on boot

Load the module at boot:

```bash
echo rtc-rx8010 | sudo tee /etc/modules-load.d/rtc-rx8010.conf
```

Create a tiny unit to (re)bind the device:

```bash
sudo tee /etc/systemd/system/rtc-rx8010.service >/dev/null <<'EOF'
[Unit]
Description=Bind RX8010 RTC on I2C-1 at 0x32
Requires=dev-i2c-1.device
After=dev-i2c-1.device systemd-modules-load.service

[Service]
Type=oneshot
ExecStart=/bin/sh -lc 'echo rx8010 0x32 > /sys/bus/i2c/devices/i2c-1/new_device'

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now rtc-rx8010.service
```

On boot, systemd can read time from the RTC when network isn’t up.

---

## 4) CH432 UART daughterboard over SPI

**What it is:** A WCH CH432-family SPI → (16550-style) UART chip on **/dev/spidev0.0**.  
**What works:** SPI **mode 3**, speed ~**500 kHz**.  
**Internal loopback works** (no wiring). Physical RS-232/RS-485 require wiring and (often) modem-control bits to enable the line driver.

### CH432 SPI encoding (important)

CH432 uses a compact address byte:

- **READ** register `reg` (0–15): send `[ (reg<<2) | 0b00, 0x00 ]` → data is 2nd byte  
- **WRITE** register `reg`: send `[ (reg<<2) | 0b10, value ]`

The UART registers are 16550-style:  
`RBR/THR=0x00, IER=0x01, IIR/FCR=0x02, LCR=0x03, MCR=0x04, LSR=0x05, MSR=0x06`.

> The board exposes two UART headers on the front:  
> **A2/B2 = RS-232**, **A1/B1 = RS-485** (plus GND).  
> We confirmed the UART cores work via **internal loopback**.

---

### `ch432_uart.py` (helper)

Save as `ch432_uart.py`:

```python
import time, spidev

class CH432UART:
    """
    Minimal CH432 (16550-like) UART over SPI.
    Confirmed working on Rocktech ISG-502 at spi0.0, mode=3, 500 kHz.
    """

    def __init__(self, bus=0, cs=0, mode=3, hz=500000):
        self.spi = spidev.SpiDev()
        self.spi.open(bus, cs)
        self.spi.mode = mode
        self.spi.max_speed_hz = hz
        # 16550 offsets
        self.THR = 0x00; self.RBR = 0x00
        self.IER = 0x01
        self.FCR = 0x02
        self.LCR = 0x03
        self.MCR = 0x04
        self.LSR = 0x05
        self.MSR = 0x06

    def close(self):
        try: self.spi.close()
        except: pass

    # Low-level SPI reg access (CH432 encoding)
    def _rw(self, reg, write, val=0):
        cmd = ((reg & 0x0F) << 2) | (0x02 if write else 0x00)
        if write:
            self.spi.xfer2([cmd, val & 0xFF])
            return None
        else:
            return self.spi.xfer2([cmd, 0x00])[1]

    def r(self, reg):  return self._rw(reg, False)
    def w(self, reg, v): self._rw(reg, True, v)

    # UART config
    def setup_8N1(self, baud=9600, clk_hz=22118400):
        """Set 8N1 and baud rate. CH432 has 22.1184 MHz ref on this board."""
        # LCR: DLAB=1 + 8N1
        self.w(self.LCR, 0x80 | 0x03)
        # divisor = clk/(16*baud)
        div = int(round(clk_hz/(16*baud)))
        dll = div & 0xFF
        dlm = (div >> 8) & 0xFF
        # When DLAB=1, THR->DLL, IER->DLM
        self.w(self.THR, dll)   # DLL
        self.w(self.IER, dlm)   # DLM
        # LCR: clear DLAB, keep 8N1
        self.w(self.LCR, 0x03)
        # FCR: enable + reset RX/TX FIFOs
        self.w(self.FCR, 0x07)

    def internal_loop(self, enable: bool):
        """Enable internal TX→RX loopback (MCR bit4)."""
        self.w(self.MCR, 0x10 if enable else 0x00)

    def set_mcr(self, value: int):
        """Directly set MCR (DTR=1, RTS=2, OUT1=4, OUT2=8, LOOP=16)."""
        self.w(self.MCR, value & 0x1F)

    def write_bytes(self, data: bytes, timeout_s=1.0):
        """Write with THR empty wait (LSR bit5)."""
        t_end = time.time() + timeout_s
        for b in data:
            while (self.r(self.LSR) & 0x20) == 0:
                if time.time() > t_end:
                    break
            self.w(self.THR, b)

    def read_available(self, max_ms=50):
        """Non-blocking read of whatever's queued (LSR bit0)."""
        out = bytearray()
        t0 = time.time()
        while (time.time() - t0) < (max_ms/1000.0):
            if (self.r(self.LSR) & 0x01) == 0:
                break
            out.append(self.r(self.RBR))
        return bytes(out)
```

---

### Internal loopback (no wiring)

Save as `internal_loop_test.py`:

```python
from ch432_uart import CH432UART
import time

def test_port(name, baud=9600):
    print(f"\n=== {name} (internal loop) ===")
    ser = CH432UART()  # spi0.0, mode 3, 500k
    try:
        ser.setup_8N1(baud=baud)
        ser.internal_loop(True)
        msg = f"LOOP_{name}\r\n".encode()
        print("TX:", msg)
        ser.write_bytes(msg)
        time.sleep(0.05)
        rx = ser.read_available()
        print("RX:", rx if rx else b"<nothing>")
    finally:
        ser.internal_loop(False)
        ser.close()

test_port("RS232_like")
test_port("RS485_like")
```

Expected output:

```
=== RS232_like (internal loop) ===
TX: b'LOOP_RS232_like\r\n'
RX: b'LOOP_RS232_like\r\n'

=== RS485_like (internal loop) ===
TX: b'LOOP_RS485_like\r\n'
RX: b'LOOP_RS485_like\r\n'
```

---

### Physical RS-232 echo (A2 ↔ B2)

Use a solid jumper under the Phoenix screws (DuPont tips don’t hold). Then:

Save as `rs232_loop_physical.py`:

```python
from ch432_uart import CH432UART
import time

ser = CH432UART()
try:
    ser.setup_8N1(9600)
    ser.internal_loop(False)

    # Many industrial hats require modem-control bits to enable the RS-232 transceiver.
    # Try the common enable combo first:
    ser.set_mcr(0x0B)  # DTR|RTS|OUT2

    msg = b'HELLO_PHYSICAL_RS232\r\n'
    print("TX:", msg)
    ser.write_bytes(msg)
    time.sleep(0.1)
    rx = ser.read_available()
    print("RX:", rx if rx else b"<nothing>")

finally:
    ser.set_mcr(0x00)
    ser.close()
```

If you still see `<nothing>`, try these alternates (edit the `set_mcr` line and retry):

- `0x03` (RTS|DTR)  
- `0x09` (DTR|OUT2)  
- `0x0F` (RTS|DTR|OUT1|OUT2)

---

### Physical RS-485 note (A1 / B1)

Half-duplex RS-485 typically uses **RTS** to drive **DE** (driver enable). A crude loopback is sometimes possible by bridging A1↔B1, but best is a second RS-485 device.

Sketch (send with driver on, then release):

```python
from ch432_uart import CH432UART
import time

ser = CH432UART()
try:
    ser.setup_8N1(9600)
    ser.internal_loop(False)

    # Assert RTS to enable TX driver, send burst, then release
    ser.set_mcr(0x02)   # RTS=1
    ser.write_bytes(b'PING485\r\n')
    ser.set_mcr(0x00)   # release driver

    time.sleep(0.1)
    print("RX485:", ser.read_available() or b"<nothing>")
finally:
    ser.set_mcr(0x00)
    ser.close()
```

---

## 5) Front panel pinout (quick)

- **RS-232:** A2 = TX, B2 = RX, plus GND  
- **RS-485:** A1/B1 = differential pair (+/–), plus GND  
- **Pi UART pass-through:** A0/B0 (separate from CH432)  
- Digital IO block (IO0–IO3, GND) and 9–24V power are on separate terminal blocks.

---

## 6) Troubleshooting

- **RTC doesn’t appear:** ensure `rtc-rx8010.ko` is present, module loaded, and device is bound at `0x32`. Re-run `echo rx8010 0x32 > …/new_device`.  
- **RTC time wrong after reboot:** make sure you ran `sudo hwclock -w` once **after** system time was correct.  
- **CH432 internal loop works but physical echo doesn’t:** use solid wire; toggle `MCR` enable combos (`0x0B`, `0x03`, `0x09`, `0x0F`).  
- **Weird serial framing:** try `setup_8N1(115200)`; if characters are off, tweak `baud`—base clock is 22.1184 MHz → divisors are exact for common rates.

---

## ✅ What’s verified working

- RX8010 RTC at **I²C 0x32**, kernel module built, bound, and persistent across reboots.  
- CH432 UART core over **SPI0.0, mode 3, 500 kHz** using the correct SPI register encoding.  
- **Internal loopback** echoes reliably from Python.  
- Physical RS-232/RS-485 wiring pending solid Phoenix jumpers / correct `MCR` enables.

---
