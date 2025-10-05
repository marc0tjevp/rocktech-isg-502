# GPIOs on Rocktech ISG-502

The daughterboard exposes 5 V-level digital IOs and a green LED:

| Label | Pi GPIO | Notes                    |
|------:|:-------:|--------------------------|
|  IO0  |  GPIO17 | 5 V logic, idles HIGH   |
|  IO1  |  GPIO18 | 5 V logic, idles HIGH   |
|  IO2  |  GPIO27 | 5 V logic, idles HIGH   |
|  IO3  |  GPIO22 | 5 V logic, idles HIGH   |
| LED G |  GPIO4  | On-board green LED      |

> ⚠️ **Caution**  
> These lines are level-shifted to **5 V** on the terminal side. Treat them as digital IO, not power. Do not feed 5 V directly into bare Pi GPIOs elsewhere. Keep load current small (a few mA typical) or drive a relay/SSR via a proper driver.

---

## Tools

```
sudo apt-get install -y gpiod
```

List pins:
```
gpioinfo
```

Read IOs (inputs):
```
gpioget gpiochip0 17 18 27 22
```

---

## Drive IOs (outputs)

Set HIGH / LOW (observe with multimeter between IOx and GND on the terminal block):

```
# IO0 → HIGH (≈5 V)
gpioset gpiochip0 17=1

# IO0 → LOW (≈0 V)
gpioset gpiochip0 17=0
```

Repeat with 18 (IO1), 27 (IO2), 22 (IO3) as needed.

---

## Green LED (GPIO4)

```
# On
gpioset gpiochip0 4=1
# Off
gpioset gpiochip0 4=0
```

---

## Optional: set safe defaults at boot

Create a oneshot that sets IOs LOW (and keeps LED off) on boot. Adjust to your needs.

```
# /etc/systemd/system/gpio-defaults.service
[Unit]
Description=Set Rocktech GPIO default states
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/bin/gpioset gpiochip0 17=0 18=0 27=0 22=0 4=0

[Install]
WantedBy=multi-user.target
```

Enable:
```
sudo systemctl enable --now gpio-defaults.service
```
