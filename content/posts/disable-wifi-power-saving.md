+++
title = 'Disable WiFi Power Saving in Linux'
date = 2024-07-15T06:01:20Z
draft = true
+++

We recently rearranged some space in our apartment and unfortunately my office desk ended up in the exact opposite corner of my router. My Arch Linux quickly started dropping the WiFi connection every 2-3 minutes while my Windows 11 laptop was working perfectly.

I started to investigate and the **Power Management** parameter in the `iwconfig` command caught my attention

```
wlp108s0  IEEE 802.11  ESSID:"WiFi"
          Mode:Managed  Frequency:5.5 GHz  Access Point: 21:AF:FE:23:E1:A4   
          Bit Rate=117 Mb/s   Tx-Power=20 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Power Management:off
          Link Quality=49/70  Signal level=-61 dBm  
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:0  Invalid misc:11   Missed beacon:0

```

Searching on the internet, I quickly realised that I was not alone: some people claim that this setting increases latency and others that WiFi drops out. So I decided to turn it off`

Create the file `/etc/NetworkManager/conf.d/wifi-powersave-off.conf` with the following content:

```
[connection]
# Values are 0 (use default), 1 (ignore/don't touch), 2 (disable) or 3 (enable).
wifi.powersave = 2
```

To apply the new setting restart `NetworkManager` deamon:

```bash
sudo systemctl restart NetworkManager
```

The downside is that your laptop will use more battery power as the WiFi card never goes into sleep mode.
