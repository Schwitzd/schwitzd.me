+++
title = 'NVM on Enterprise'
date = 2024-06-11T19:44:00Z
draft = false
+++

[Node Version Manager for Windows](https://github.com/coreybutler/nvm-windows) is the de facto tool for managing multiple versions of [Node.js](https://nodejs.org/en), and is widely used by developers. In organisations where high security standards are in place, it can be a challenge to allow developers to use NVM.

## Getting started

The aim of this short guide is to enable your developers to use NVM for Windows without the need for administrator rights, applying the concept of least privilege.

In case you are using [AppLocker or WDAC](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/windows-defender-application-control/wdac-and-applocker-overview) you have to provide the developers a folder where they can download and excecute *.exe* binaries without restrictions.

## Silent installation

The installation process basically extracts the NVM files and creates two environment variables:

* **%NVM_HOME%**: NVM installation directory
* **%NVM_SYMLINK%**: Node.js symbolic link directory (hardcoded to: *c:\program files\nodejs*)

Unfortunately, during silent installation, you can only specify %NVM_HOME%, there is no switch to set %NVM_HOME%.

```cmd
# /DIR will be the %NVM_HOME%
nvm-setup.exe /SILENT /DIR=$installDir
```

This means that you should create a post-installation script that changes the environment variable %NVM_SYMLINK% to your defined path and edit the file *settings.txt* as follows:

```text
root: C:\tools\nvm 
path: C:\tools\nvm\nodejs
arch: 64 
proxy: none
```

## Non-admin usage

To switch between Node.js versions, NVM creates a new symbolic link to the folder defined at %NVM_SYMLINK%, but Windows requires administrator rights to create it and will prompt you with the UAC window. There is probably also a **Create Symbolic Links** privilege set that you can use to avoid giving admin access to your developers.

This setting is available on the **Local Security Policies** at *Computer Configuration → Windows Settings → Security Settings → Local Policies → User Rights Assignment → Create Symbolic Links*
