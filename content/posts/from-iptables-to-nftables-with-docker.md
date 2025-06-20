+++
title = 'From iptables to nftables with Docker'
date = 2025-06-20T06:53:41Z
draft = false
+++

This blog post was initially intended to explain how to migrate from `iptables` to `nftables` — there are plenty of similar posts all over the internet. However, I soon realised that I was also running Docker on my laptop, which still does not [natively support](https://github.com/docker/for-linux/issues/1472) `nftables` at the time of writing.
I therefore decided to write a dual-aim article: switching to `nftables` and allowing Docker containers to access the network.

## Disable `iptables`

To fully transition to `nftables` and prevent conflicts, it's important to stop and disable the legacy `iptables` services. You can do this with the following commands:

```sh
sudo systemctl stop iptables
sudo systemctl disable iptables

sudo systemctl stop ip6tables
sudo systemctl disable ip6tables
```

This ensures that `iptables` and `ip6tables` won't interfere with your nftables rules on boot.

(Optional) Remove legacy iptables package `iptables-nft`

## Docker with `nftables`

The Docker daemon can be configured using the `/etc/docker/daemon.json` file. If the file does not exist, create it and add the following content:

```json
{
  "iptables": false,
  "ip6tables": false
}
```

For more configuration options, refer to the [official documentation](https://docs.docker.com/engine/daemon/).

Restart the Docker deaemon for the changes to take effect:

```sh
sudo systemctl restart docker
```

Then, open the nftables configuration file (`/etc/nftables.conf`) with your preferred text editor.
This is my minimal firewall configuration, which I run on my everyday laptop and which blocks all incoming traffic while allowing Docker connectivity.
```nft
#!/usr/sbin/nft -f

flush ruleset

define docker_if = "docker0"
define wan_if = "eth0"  # Change this to your real external interface (e.g. wlan0)

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop

        # Allow loopback traffic
        iifname lo accept

        # Accept established and related connections
        ct state established,related accept

        # Allow ICMP (v4 and v6)
        ip protocol icmp accept
        ip6 nexthdr ipv6-icmp accept

        # Allow DHCPv6 client replies
        udp dport 546 udp sport 547 accept

        # Allow limited incoming multicast (IPv4 & IPv6)
        ip daddr 224.0.0.0/4 accept
        ip6 daddr ff00::/8 accept
    }

    chain forward {
        type filter hook forward priority 0; policy drop

        # Allow containers to forward to WAN
        iifname $docker_if oifname $wan_if accept
        # Allow return traffic
        ct state established,related accept
    }

    chain output {
        type filter hook output priority 0; policy accept
    }
}

table ip nat {
    chain postrouting {
        type nat hook postrouting priority 100; policy accept

        # Masquerade container traffic going to internet
        oifname $wan_if ip saddr 172.17.0.0/16 masquerade
    }
}
```

This configuration only supports containers connected to Docker’s default bridge network (`docker0`) using the default IP range (`172.17.0.0/16`). It does not apply to custom Docker networks or alternative runtimes.

## Closing words

I’ll update this post as soon as there are any relevant updates or official support from the Docker team.
