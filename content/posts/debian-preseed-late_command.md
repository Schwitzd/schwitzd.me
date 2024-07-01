+++
title = 'Debian Preseed late_command'
date = 2024-07-01T16:05:02Z
draft = false
+++

In Debian installation process, the `preseed.cfg` file allows for automated installations by pre-configuring various installation parameters. In this article I will focus on the `d-i preseed/late_command string` parameter, which is used to run custom commands at the end of the installation process.

Here is an example that I recently used in my [packer-vbox-debian-latest](https://github.com/Schwitzd/packer-vbox-debian-latest) project:

```
d-i preseed/late_command string \
mkdir --mode=700 /target/home/testuser/.ssh; \
wget -q http://10.0.2.2:8081/key.pub -O /target/home/testuser/.ssh/authorized_keys; \
in-target chown testuser:testuser /home/testuser/.ssh; \
in-target chown testuser:testuser /home/testuser/.ssh/authorized_keys; \
in-target chmod 0600 /home/testuser/.ssh/authorized_keys
```

As you can see, sometimes `/target` is used, other times `in-target`, I must admit I struggled a bit to understand the difference and when to use one instead of the other.

## The `/target` directory

When you are installing Debian, the installer needs a place to set up the new system, and this place is called `/target`. During the installation process, `/target` acts as a temporary workspace where the new system's root filesystem is prepared. Essentially, the installer mounts the new system's root partition to `/target`, allowing it to configure and manipulate the filesystem and its contents as if it were the actual running system.

## The `in-target` command

The `in-target` command is used within the preseed file to execute commands in the context of the newly installed system, rather than within the installer’s environment. When you use `in-target`, it effectively chroots into `/target` and runs the specified command as if you were executing it on the new system.

## Conclusion

- Use `/target` when you need to create or manipulate files and directories directly on the new system’s filesystem during the installation process.
- Use `in-target` when you need to execute commands within the context of the new system, as it effectively chroots into `/target` and runs commands as if you were operating within the new system itself.
