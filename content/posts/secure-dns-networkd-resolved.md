+++
title = 'Secure DNS Networkd Resolved'
date = 2025-08-23T11:42:42Z
draft = false
+++

This article will be very similar to [Integrate Dnscrypt-proxy with a Home DNS Server](https://www.schwitzd.me/posts/integrate-dnscrypt-proxy-with-a-home-dns-server/), with the main difference that instead of using [Dnscrypt-proxy](https://github.com/DNSCrypt/dnscrypt-proxy) to forward DNS queries to your preferred [DoH nameserver](https://github.com/curl/curl/wiki/DNS-over-HTTPS), I will use [systemd-resolved](https://www.freedesktop.org/software/systemd/man/latest/systemd-resolved.service.html).
At the moment, `systemd-resolved` only supports [DoT (DNS over TLS)](https://en.wikipedia.org/wiki/DNS_over_TLS). Support for DoH is still under development ([issue #8639](https://github.com/systemd/systemd/issues/8639)).

## Configure systemd-resolved

Make sure it's running and that `/etc/resolv.conf` points to it.

```sh
sudo systemctl enable --now systemd-resolved
```

Symlink `/etc/resolv.conf` to the stub resolver:

```sh
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

Now your system sends DNS queries to `127.0.0.53`, where `systemd-resolved` listens. Configure DoT in the file `/etc/systemd/resolved.conf.d/dns.conf`:

```toml
[Resolve]
DNS=DNS=9.9.9.9#dns.quad9.net 149.112.112.112#dns.quad9.net 2620:fe::fe#dns.quad9.net 2620:fe::9#dns.quad9.net
DNSOverTLS=yes
FallbackDNS=
```

Then restart the service:

```sh
sudo systemctl restart systemd-resolved
```

## Configure NetworkManager

In this setup I'm using the internal DHCP client provided by NetworkManager. By default, it accepts DNS servers from DHCP and forwards them to `systemd-resolved`. Since I want to enforce [Quad9](https://quad9.net) over DoT for all lookups, I need NetworkManager to only handle IP settings from DHCP and leave DNS entirely to systemd-resolved.

To configure NetworkManager to ignore DHCP-provisioned nameservers and to delegate DNS entirely to systemd-resolved. Create a file named `/etc/NetworkManager/conf.d/no-dns.conf` and add following content:

```toml
[main]
dns=none
systemd-resolved=false
```

Restart NetworkManager service to apply the settings:

```sh
sudo systemctl restart NetworkManager
```

### Dispatcher

Now, the juicy part... The trade-off of the above solution is that you only get the global DNS configuration, and none of the per-link DNS configurations are passed to systemd-resolved. In most scenarios, you won't even notice this, but if you connect to a network where you need to access local resources by their DNS names, you won't be able to because you're missing the local nameservers.

NetworkManager has a functionality called [Dispatcher](https://networkmanager.dev/docs/api/latest/NetworkManager-dispatcher.html), which runs user-defined scripts whenever a network interface changes state (such as up, down, or after a DHCP update) so you can automate custom actions on connectivity events.

I created a custom dispatcher script in `/etc/NetworkManager/dispatcher.d/50-local-dns-routing` that:

- On connect, applies IPv4/IPv6 DNS servers from DHCP server and disables DoT/default-route
- When at home, routes my personal domain through the router's DNS so local resources resolve correctly
- On disconnect, reverts any per-link DNS configuration

```sh
#!/usr/bin/env bash

LOG="/var/log/nm-dispatcher.localdns.log"
IFACE="<NETWORK-INTERFACE-NAME>"
HOME_SSID="<YOUR-HOME-SSID>"
MY_DOMAIN="<YOUR-DOMAIN>"

# Run only when the interface comes up
if [[ "$NM_DISPATCHER_ACTION" == "up" && "$DEVICE_IFACE" == "$IFACE" ]]; then
  # Grab DNSv4 from NetworkManager
  DNS4="$(nmcli -g IP4.DNS device show "$DEVICE_IFACE" 2>/dev/null \
    | tr '|' '\n' | awk 'NF' | xargs)"

  # DNSv6 can be slower to show up, so wait a little
  DNS6=""
  for i in {1..16}; do
    DNS6="$(nmcli -g IP6.DNS device show "$DEVICE_IFACE" 2>/dev/null \
      | tr '|' '\n' | sed 's/\\:/:/g' | awk 'NF' | xargs)"
    [[ -n "$DNS6" ]] && break
    sleep 0.5
  done

  echo "[$(date +"%F %T")] action=$NM_DISPATCHER_ACTION iface=$DEVICE_IFACE ssid=$CONNECTION_ID DNSv4='${DNS4:-none}' DNSv6='${DNS6:-none}'" >> "$LOG"

  # Apply DNS to this link only
  resolvectl dns "$DEVICE_IFACE" $DNS4 $DNS6
  resolvectl dnsovertls "$DEVICE_IFACE" no
  resolvectl default-route "$DEVICE_IFACE" no

  # When connected at home, forward queries for my domain to the router's DNS so home cluster resources resolve locally  
  if [[ "$CONNECTION_ID" == "$HOME_SSID" ]]; then
    resolvectl domain "$DEVICE_IFACE" "~$MY_DOMAIN"
    echo "[$(date +"%F %T")] at home: routing ~$HOME_DOMAIN via local DNS" >> "$LOG"
  fi
fi

# When the interface goes down, clean up its DNS settings
if [[ "$NM_DISPATCHER_ACTION" == "down" && "$DEVICE_IFACE" == "$IFACE" ]]; then
  resolvectl revert "$DEVICE_IFACE" >/dev/null 2>&1 || true
  echo "[$(date +"%F %T")] action=$NM_DISPATCHER_ACTION iface=$DEVICE_IFACE reverted" >> "$LOG"
fi
```

Ensure the script is executable:

```sh
sudo chmod +x /etc/NetworkManager/dispatcher.d/50-local-dns-routing
sudo systemctl restart NetworkManager
```

With this setup my laptop always uses [Quad9](https://quad9.net) as the default resolver. This has several advantages: DNS traffic is encrypted, improving privacy and protecting against eavesdroppers; it avoids the risk of [local resolver hijacking](https://www.first.org/global/sigs/dns/stakeholder-advice/detection/local-resolver-hijacking); and it keeps me independent from whatever DNS a network tries to enforce. At the same time, per-link DNS lets me reach resources provided by the current network, and when I'm at home I can resolve my internal cluster services by name without exposing them to the internet. But there are also some [limitations](#limitations).

## Verify the configuration

Use the command `resolvectl status` to verify that your configuration was applied successfully. In my case:

```
Global
         Protocols: +LLMNR +mDNS +DNSOverTLS DNSSEC=no/unsupported
  resolv.conf mode: stub
Current DNS Server: 149.112.112.112#dns.quad9.net
       DNS Servers: 9.9.9.9#dns.quad9.net 149.112.112.112#dns.quad9.net 2620:fe::fe#dns.quad9.net 2620:fe::9#dns.quad9.net

Link 2 (wlo1)
    Current Scopes: DNS LLMNR/IPv4 LLMNR/IPv6 mDNS/IPv4 mDNS/IPv6
         Protocols: -DefaultRoute +LLMNR +mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 192.168.12.1
       DNS Servers: 192.168.12.1 2a11:6c7:<redacted>::1 fd12:3456:789a:12::1
        DNS Domain: ~schwitzd.me
     Default Route: no
```

If you would like to perform an in-depth analysis, use a tool such as `dig`, `nslookup`, or `resolvectl query` to query a domain while monitoring traffic with `wireshark` or `tcpdump`.

```sh
sudo tcpdump -i any -vvv -s0 -n '(port 53 or port 853)'
```

## Limitations

Currently, the dispatcher script only works with the Wi-Fi network card because that's what I use most often. I plan to extend its functionality to include wired connections. Another limitation is handling VPNs. I haven't explored this area yet, but it's on my to-do list. Future versions of the script will include VPN support.
