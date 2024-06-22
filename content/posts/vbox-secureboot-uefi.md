+++
title = 'VirtualBox Secure Boot & UEFI'
date = 2024-05-20T16:44:23Z
draft = true
+++

This short article explains how to enable [Secure Boot](https://wiki.debian.org/SecureBoot) and [UEFI](https://wiki.debian.org/UEFI) for a Debian VM in VirtualBox.

## Getting Started

In the VirtualBox VM settings go to **System** and enable Extended Features: **Enable EFI (special OSes only) and **Enable Secure Boot**.

## Inside the VM

{{< gist Schwitzd c819f172ea2d407a47711172ddec0de1 >}}
