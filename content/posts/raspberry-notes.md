+++
title = 'Raspberry Pi Notes'
date = 2024-05-31T10:27:03Z
draft = true
+++

These evolving personal notes document my journey and discoveries as I explore the versatile Raspberry Pi. At present, all information pertains specifically to the **Raspberry Pi 5**.

## Power

* **Minimum required**: 5V / 3A (can't connect any bus-powered HDDs/SSDs)
* **Best performance**: 5v / 5A

Remainder: x Volt * y Amp = z Watt

[Pogo Pin](https://upload.wikimedia.org/wikipedia/commons/thumb/f/f3/Pogo_Pin_Connectors.jpg/1920px-Pogo_Pin_Connectors.jpg): A pogo pin is a spring-loaded connector used to create reliable electrical connections in electronics without the need for soldering, commonly for programming and connecting peripherals.

## Firmware

```bash
# Get current firmware
vcgencmd version

# Update Raspberry firmware
rpi-update

# Get bootloader details
sudo rpi-eeprom-update

# Update bootloader
sudo rpi-eeprom-update -a
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

## Hardware

### Geekworm X1011

Is a PCIe to NVMe Shield support 4x M.2, [offical wiki page](https://wiki.geekworm.com/X1011), it just works, nothing to do after following the mounting instructions.

```bash
lspci
0000:00:00.0 PCI bridge: Broadcom Inc. and subsidiaries Device 2712 (rev 21)
0000:01:00.0 PCI bridge: ASMedia Technology Inc. ASM1184e 4-Port PCIe x1 Gen2 Packet Switch
0000:02:01.0 PCI bridge: ASMedia Technology Inc. ASM1184e 4-Port PCIe x1 Gen2 Packet Switch
0000:02:03.0 PCI bridge: ASMedia Technology Inc. ASM1184e 4-Port PCIe x1 Gen2 Packet Switch
0000:02:05.0 PCI bridge: ASMedia Technology Inc. ASM1184e 4-Port PCIe x1 Gen2 Packet Switch
0000:02:07.0 PCI bridge: ASMedia Technology Inc. ASM1184e 4-Port PCIe x1 Gen2 Packet Switch
0000:03:00.0 Non-Volatile memory controller: Intel Corporation SSD Pro 7600p/760p/E 6100p Series (rev 03)
0001:00:00.0 PCI bridge: Broadcom Inc. and subsidiaries Device 2712 (rev 21)
0001:01:00.0 Ethernet controller: Device 1de4:0001

```

## Shops

* [Berrybase](https://www.berrybase.ch)
* [PI-SHOP](https://www.pi-shop.ch)
* [Pimoroni](https://shop.pimoroni.com)
* [Geekworm](https://geekworm.com) & [Geekworm on AliExpress](https://geekworm.aliexpress.com/store/1048722)
