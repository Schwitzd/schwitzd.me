+++
title = 'Mikrotik - Tunnelbroker with Route64'
date = 2025-08-18T19:29:54Z
draft = false
+++

For learning purposes, I started looking into [IPv6](https://en.wikipedia.org/wiki/IPv6). First, I enabled a [Unique local address](https://en.wikipedia.org/wiki/Unique_local_address) subnet to leverage [K3s dual-stack](https://docs.k3s.io/networking/basic-network-options#dual-stack-ipv4--ipv6-networking) on my [Home Cluster](https://github.com/Schwitzd/IaC-HomeK3s). Then I thought it would be cool to be able to connect to my home from abroad, so I started investigating VPNs. It is at this point that I discovered that my [ISP](https://en.wikipedia.org/wiki/Internet_service_provider) currently is only offering IPv4 behind [CGNAT](https://en.wikipedia.org/wiki/Carrier-grade_NAT) for mobile devices (My Mikrotik connects to internet over [LTE](https://en.wikipedia.org/wiki/LTE_(telecommunication)).
Surfing the web I learned about [tunnel broker](https://en.wikipedia.org/wiki/Tunnel_broker) is a service that provides IPv6 connectivity over an existing IPv4-only internet connection by encapsulating IPv6 traffic inside IPv4 packets.

## Route64

[Route64](https://route64.org) is a free tunnel broker service that not only provides a public IPv6 address through a secure tunnel but also assigns you an entire IPv6 `/56` subnet. This subnet can for example be divided into multiple `/64` networks, which you can then use to segment your home network into different VLANs while keeping everything globally routable on the internet.

Here are all the steps you are required to do to optain the required information:

1. Register and login on the [Route64 Portal](https://manager.route64.org/)
1. Add a new **IPv6 Tunnelbroker**, by selecting the nearest [PoP](https://en.wikipedia.org/wiki/Point_of_presence) and **Wireguard** as tunnel type.
1. Enter your [public IPv4 address](https://ipv4.seeip.org) on **Remote endpoint** and create the service.
1. To get the IPv6 subnet go to *IPv6 Tunnelbroker → List IP subnets*

you will get a Wireguard configuration similar to the following:

```toml
[Interface]
PrivateKey = "your-private-key"
Address = "2a11:6c7:xxxx:xxxx::2/64"

[Peer]
PublicKey = "your-public-key"
AllowedIPs = "::/1, 8000::/1"
Endpoint = "route64-ipv4-endpoint:route64-port"
PersistentKeepAlive = 15
```

And also an IPv6 subnet like `2a11:6c7:xxxx:xxxx::/56`

## Mikrotik

In my case, all the Mikrotik configurations required are carried out in line with the [IaC principle](https://en.wikipedia.org/wiki/Infrastructure_as_code). On GitHub, you can find my IaC-HomeRouter[https://github.com/Schwitzd/IaC-HomeRouter] repository, where you can find my entire RouterOS configuration. As you continue reading this article, you will find the required manual steps.

### Configure WireGuard for Route64

To establish the tunnel with **Route64**, you first create a new WireGuard interface on your router. Think of the interface as a secure channel that will carry all IPv6 traffic through the tunnel. After that, you define a peer, which represents Route64's server. This is how your router knows who it's talking to, where to send the encrypted packets, and how to verify the other side.

1. Create a new WireGuard interface

```sh
/interface wireguard add  \
    mtu=1420 \
    name=wireguard-route64 \
    private-key="<route64-private-key>"
```

2. Add Route64 server as a WireGuard peer:

```sh
/interface wireguard peers add \
    allowed-address=::/1,8000::/1 \
    endpoint-address=<route64-ipv4-endpoint> \
    endpoint-port=<route64-port> \
    interface=wireguard-route64 \
    persistent-keepalive=15s \
    public-key="<route64-prublic-key>"
```

### IPv6 Setup

Once the WireGuard tunnel is up, you need to configure IPv6 on both the **WAN side** (the tunnel itself) and the **LAN side** (your internal network).

#### WAN Side

1. Assign the IPv6 address provided by Route64 to the WireGuard interface:

```sh
/ipv6 address add \
  address=2a11:6c7:xxxx:xxxx::2/64 \
  interface=wireguard-route64
```

2. Add a default IPv6 route that sends all global IPv6 traffic through the tunnel:

```sh
/ipv6 route add \
  dst-address=2000::/3 \
  gateway=Route64
```

> `2000::/3` is the range of all Global Unicast Addresses (GUA). Using this instead of `::/0` avoids affecting link-local and special-purpose addresses.

#### LAN Side

Route64 assigns you a full `/56` IPv6 subnet, which you can split into multiple `/64` networks. This is very handy if you already have different VLANs at home for example, one for regular devices and another for IoT.

Let's say Route64 delegated `2a11:6c7:abcd:1000::/56` to you:

* Use `2a11:6c7:abcd:1001::/64` for your **Home VLAN**
* Use `2a11:6c7:abcd:1002::/64` for your **IoT VLAN**

1. Add the subnets to your VLAN bridge interfaces and enable advertisement via [SLAAC](https://en.wikipedia.org/wiki/IPv6_address#Stateless_address_autoconfiguration_%28SLAAC%29):

```sh
/ipv6 address add \
  address=2a11:6c7:abcd:1001::/64 \
  advertise=yes \
  interface=vlan-home

/ipv6 address add \
  address=2a11:6c7:abcd:1002::/64 \
  advertise=yes \
  interface=vlan-iot
```

2. Adjust [Neighbor Discovery](https://en.wikipedia.org/wiki/Neighbor_Discovery_Protocol) (RA) so that each VLAN gets IPv6 advertisements, and set the [MTU](https://en.wikipedia.org/wiki/Maximum_transmission_unit) to match WireGuard (1420):

```sh
/ipv6 nd set [ find default=yes ] \
  interface=vlan-home \
  mtu=1420

/ipv6 nd add \
  interface=vlan-iot \
  mtu=1420
```

With this setup, your devices in both the Home and IoT VLANs will automatically receive globally routable IPv6 addresses from your prefix and use your router for connectivity.

#### Firewall

By default, the MikroTik IPv6 firewall blocks most inbound traffic, which also includes WireGuard handshakes. To make sure the tunnel with Route64 can be established, you must explicitly allow the WireGuard UDP port in the input chain before any general drop rules.

```sh
/ip firewall filter add \
  chain=input \
  action=accept \
  protocol=udp \
  dst-port=<route64-port> \
  comment="Allow WireGuard from Route64"
```

### Tests

1. Ping from any device your Route64 tunnel endpoint like `2a11:6c7:xxxx:xxxx::1`
1. Using a client device connected to your shiny new IPv6-enabled VLAN, browse the website [test-ipv6](test-ipv6.com)
1. On an mobile device also connected to your network install the app [Network Analyzer](https://techet.net/netanalyzer/)

## Credits

I would like to thanks for inspiring me to write this article: [Setup Route64 tunnel broker on MikroTik](https://www.animmouse.com/p/setup-route64-tunnel-broker-on-mikrotik/)
