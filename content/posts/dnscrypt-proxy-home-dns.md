+++
title = 'Integrate Dnscrypt-proxy with a Home DNS Server'
date = 2024-07-31T04:47:14Z
draft = false
+++

I recently bought a MikroTik router and I'm spending a lot of time configuring it and trying to understand/learn as much as I can (some posts coming in the near future). With the new router I would like to go a step further and set up some static DNS entries within my home for my devices and for my lab.

In this post we will:

- Prevent dhcpd and NetworkManager to overwrite `/etc/resolv.conf`
- Configure [Dnscrypt-proxy](https://github.com/DNSCrypt/dnscrypt-proxy) to use [Quad9](https://quad9.net/) with [DoH](https://en.m.wikipedia.org/wiki/DNS_over_HTTPS)
- Use [Dnscrypt-proxy](https://github.com/DNSCrypt/dnscrypt-proxy) [forwarding](https://github.com/dnscrypt/dnscrypt-proxy/wiki/Forwarding) feature to resolve hostnames of home devices
- Manually configure `/etc/resolv.conf`

## Configure dhcpd

By default, dhcpd obtains nameservers from the DHCP server and configures them in `/etc/resolv.conf`. If you are a frequent traveller, this means that you will use whatever name servers are configured on the networks you connect to, with some potential security threads (MiTM, DNS Spoofing/Hijacking and privacy risks). To avoid this, we will configure dhcpd to ignore the DNS servers received from the dhcp server and always use Dnscrypt-proxy listening on `localhost:53`.

Edit the file `/etc/dhcpcd.conf` and append the following line:

```bash
# Prevent DNS servers overwrite
nohook resolv.conf
```

Restart the dhcp client service:

```bash
sudo systemctl restart dhcpcd.service
```

However, there is a downside to this change: if you are connected to a network and need to access local resources, you will not be able to resolve the hostname of those resources because you will be missing the local name servers. So be careful and think about what you are doing.

## Configure NetworkManager

To preventing also NetworkManager to overwrite the `/etc/resolv.conf` create a file named `/etc/NetworkManager/conf.d/dns.conf` and add following content:

```conf
[main]
dns=none
```

Restart the NetworkManager service:

```bash
sudo systemctl restart NetworkManager.service
```

## Configure Dnscrypt-proxy

I will not go into detail on how to install and configure Dnscrypt-proxy, the internet is full of tutorials on this. I will only mention that by default the name server with the lower latency is used, but I have chosen to use my preferred [resolver](https://download.dnscrypt.info/resolvers-list/v3/public-resolvers.md).

Edit the file `/etc/dnscrypt-proxy/dnscrypt-proxy.toml` and uncomment the following line if you would like to do the same.

```toml
server_names = ['quad9-doh-ip4-port443-filter-pri', 'cloudflare-security']
```

But back to my initial goal, I would need to configure the [Forwarding](https://github.com/dnscrypt/dnscrypt-proxy/wiki/Forwarding) feature in Dnscrypt-proxy to use my router's DNS server only to resolve the hostname of my home and lab devices. Uncomment the following line in the same file above:

```toml
## plugins can handle directly (forwarding, cloaking, ...)
## See the `example-forwarding-rules.txt` file for an example
forwarding_rules = '/etc/dnscrypt-proxy/forwarding-rules.txt'
```

Now it is time to edit the file `/etc/dnscrypt-proxy/forwarding-rules.txt`, as you can see at the beginning there is a good explanation of how to use this file, in my case I simply added the local TLD of my home with the relative nameserver:

```toml
home 192.168.12.1
lab  192.168.12.1
```

Restart the service to apply the desired configuration:

```bash
sudo systemctl restart dnscrypt-proxy.service`
```

## /etc/resolv.conf

Edit the file `/etc/resolv.conf` and add following line:

```bash
nameserver 127.0.0.1
```

## Bonus section

In case you find yourself in a situation where you are connecting to a local network and need to access local resources by their hostname, you can use this command to get the local DNS servers that are shipped with the DHCP offer package. (The `nmcli` package is required).

```bash
nmcli dev show | grep IP4.DNS
```

Manually add the local name server to `/etc/resolv.conf` and restart `NetworkManager`.
