+++
title = 'Raspberry Pi Notes'
date = 2024-05-31T10:27:03Z
draft = true
+++

These evolving personal notes document my journey and discoveries as I explore the versatile Raspberry Pi. At present, all information pertains specifically to the **Raspberry Pi 5**.

## Hardware

### Power

* **Minimum required**: 5V / 3A (can't connect any bus-powered HDDs/SSDs)
* **Best performance**: 5v / 5A

Remainder: x Volt * y Amp = z Watt

[Pogo Pin](https://upload.wikimedia.org/wikipedia/commons/thumb/f/f3/Pogo_Pin_Connectors.jpg/1920px-Pogo_Pin_Connectors.jpg): A pogo pin is a spring-loaded connector used to create reliable electrical connections in electronics without the need for soldering, commonly for programming and connecting peripherals.

## Firmware

```bash
# Update Raspberry firmware
rpi-update
```

## Networking

In Raspberry Pi OS 12 and later, dhcpcd is no longer used, everything goes through Network Manager, which is configured via nmcli or nmtui.

```bash
# View status
nmcli device status

# View NIC details
nmcli device show eth0

# Configure networking
sudo nmtui

# Apply changes
sudo systemctl restart NetworkManager
```

## Shops

* [Berrybase](https://www.berrybase.ch)
* [PI-SHOP](https://www.pi-shop.ch)
* [Pimoroni](https://shop.pimoroni.com)
* [Geekworm](https://geekworm.com) & [Geekworm on AliExpress](https://geekworm.aliexpress.com/store/1048722)
