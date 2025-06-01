+++
title = 'Mikrotik Terraform - Part #1'
date = 2024-08-04T18:16:26Z
draft = false
+++

This is the first part of my MikroTik and Terraform series, where I will explain my old and current setup and answer a lot of questions about why.

## Old setup

Many, many years ago I decided to abandon the traditional two-pair wiring hDSL (most widespread, at least in Switzerland) because I was bored of paying the electrician at each house change due to the required changes on the building telephone panel.
To cut a long story short, I chose the LTE modem for its versatility, I have no need for high performance and therefore did not choose fibre optics.

I received a very poor Huawei LTE modem with almost no options and security features. I bought an Asus AX55 router, which allowed me to customise a lot of parameters and worked really well for many years.

For the past few years I have been unhappy with my setup because it requires two physical devices without all the features I was looking for.

## Why

### MikroTik

To be honest, my first choice of router was [Ubiquiti UDR](https://www.ui.com/cloud-gateways/wifi-integrated/dream-router), it was almost perfect, the only feature missing was the LTE modem, which was frustrating because I still had to keep the crappy Huawei modem. The other point was the price.. really too expensive and luckily it was not available in all stores in the end. I waited many months for the UDR to come back on the market without really looking for alternatives. That was until a few weeks ago when I first saw some MikroTik routers. The brand name was not new to me, but I never looked at their products. I don't know why, but as soon as I started reading the datasheets on the routers, I immediately felt stupid... wondering why I had never seen this manufacturer before.
I started digging through the [Routeros documentation](https://help.mikrotik.com/docs/display/ROS/RouterOS), YouTube videos and blog posts around the internet and the initial feeling was really mixed, from one side it looked really complicated and so a bit scary to have the possibility to customise every possible parameter, but from the other side it was exactly what I was looking for (LTE support, firewall and VLANs, static DNS, etc). Digging further I also discovered the possibility to run containers and this opened up some interesting scenarios with [CloudFlare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) to investigate in the future.
And finally, the icing on the cake... a Terraform provider currently in development.

### Terraform

Why Terraform? Simply because I love to automate everything I do.. as soon as I found this project [terraform-provider-routeros](https://github.com/terraform-routeros/terraform-provider-routeros) I understood that MikroTik would be my new router.

### Sharing

Why share [your entire router configuration](https://github.com/Schwitzd/IaC-HomeRouter) with the world, exposing your home network topology? That is a really good question. I believe that sharing is caring, and this project may help someone in the community who has similar needs to mine, or who is simply interested in increasing their home security.
