# Serial (CH432) on Rocktech ISG-502

The Rocktech ISG-502 has a CH432 SPI → Dual UART chip on its daughterboard.  
With the out-of-tree `ch432` driver, we can expose two serial ports:

- **/dev/ttyWCH0** → RS-485 (via A0/B0, DE pin on GPIO5)  
- **/dev/ttyWCH1** → RS-232-like (via TXD0/RXD0)

---

## 1. Build and install the driver

```
sudo apt-get install -y git build-essential raspberrypi-kernel-headers

git clone https://github.com/WCHSoftGroup/ch432ser_linux.git
cd ch432ser_linux

cat > Makefile <<'EOF'
obj-m += ch432.o
KDIR := /lib/modules/$(shell uname -r)/build
all: ; $(MAKE) -C $(KDIR) M=$(PWD) modules
clean: ; $(MAKE) -C $(KDIR) M=$(PWD) clean
EOF

make
sudo mkdir -p /lib/modules/$(uname -r)/extra
sudo cp ch432.ko /lib/modules/$(uname -r)/extra/
sudo depmod -a
```

Enable autoload at boot:
```
echo ch432 | sudo tee /etc/modules-load.d/ch432.conf
```

---

## 2. Device Tree Overlay

Create `/boot/overlays/ch432.dtso`:

```dts
/dts-v1/;
/plugin/;
/ {
    compatible = "brcm,bcm2711";

    fragment@0 {
        target = <&spi0>;
        __overlay__ {
            status = "okay";
            #address-cells = <1>;
            #size-cells = <0>;

            ch432: ch432@0 {
                compatible = "wch,ch43x";
                reg = <0>;                 /* CE0 */
                spi-max-frequency = <1000000>;

                /* CH432 INT → GPIO7 */
                interrupt-parent = <&gpio>;
                interrupts = <7 2>;        /* falling edge */
            };
        };
    };

    fragment@1 {
        target = <&spidev0>;
        __overlay__ {
            status = "disabled";
        };
    };
};
```

Compile and enable:
```
sudo dtc -@ -I dts -O dtb -o /boot/overlays/ch432.dtbo /boot/overlays/ch432.dtso
echo "dtoverlay=ch432" | sudo tee -a /boot/config.txt
sudo reboot
```

---

## 3. Verify

```
dmesg -T | grep -i ch43
ls -l /dev/ttyWCH*
```

Expected:
```
/dev/ttyWCH0
/dev/ttyWCH1
```

---

## 4. RS-485 DE Control (GPIO5)

Install gpiod and set DE default LOW (receive mode):

```
sudo apt-get install -y gpiod
gpioset gpiochip0 5=0
```

Persist with a systemd oneshot:

```
# /etc/systemd/system/rs485-de.service
[Unit]
Description=RS-485 DE default LOW (GPIO5)
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/bin/gpioset gpiochip0 5=0

[Install]
WantedBy=multi-user.target
```

Enable it:
```
sudo systemctl enable --now rs485-de.service
```

---

## 5. Usage

- **RS-232**: use `/dev/ttyWCH1` directly with `stty` or `minicom`.  
- **RS-485**: toggle DE (GPIO5) manually before/after TX:
  ```
  stty -F /dev/ttyWCH0 115200 -echo
  gpioset gpiochip0 5=1; echo -ne 'HELLO485\r\n' > /dev/ttyWCH0; gpioset gpiochip0 5=0
  ```

Later this can be automated via `ioctl` or a wrapper script.

---
