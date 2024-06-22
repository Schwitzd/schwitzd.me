+++
title = 'Install Arch with SSH'
date = 2024-06-22T07:33:40Z
draft = false
+++

My current Arch Linux installation has many years and I'd like to reinstall it using other technologies like LVM and BTRFS, but before reinstalling my laptop, I'm testing the installation process inside a VirtualBox VM. This morning I got bored of typing all the commands and in my head popped up [Powershell Direct with Hyper-V](https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/powershell-direct). Basically you can attach a Powershell session directly to the Hyper-V VM.

I found a similar approach by configuring port forwarding in VirtualBox and connecting via SSH.

## Getting started

1. If you have configured the VM with the NIC in NAT, you will need to create a port forwarding, go to *Setting → Network → Advanced → Port forwarding*. Add a new rule with the following settings:

    * **Name**: ssh
    * **Protocol**: TCP
    * **Host Port**: 2222
    * **Guest Port**: 22
    * Leave others empty

1. According to [Install Arch Linux via SSH](https://wiki.archlinux.org/title/Install_Arch_Linux_via_SSH) the default **root** password is empty, so we need to configure one:

    ```bash
    # passwd
    ```

1. From a terminal on the host system, run the following command to connect:

    ```bash
    ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -p 2222 root@localhost
    ```

    The `-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null` options will prevent verifying and writing the live environment's SSH host keys to `~/.ssh/known_hosts`
