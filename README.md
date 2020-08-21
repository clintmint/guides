---
tags: guide
---

VirtualBox + Vagrant + WPDistillery using Windows 10 1903
===


[![hackmd-github-sync-badge](https://hackmd.io/cX5ZGiLzSBe0yZTd8zb8lA/badge)](https://hackmd.io/cX5ZGiLzSBe0yZTd8zb8lA)

## Introduction

This is a starter guide for quickly creating WordPress servers for penetration testing purposes. The guide assumes the reader has an update-to-date Windows 64-bit operating system.

[WPDistillery](https://github.com/flurinduerst/WPDistillery) will integrate with Vagrant in the downloading and configuration of WordPress by automatically installing a LAMP stack (via [ScotchBox](https://box.scotch.io/)) and other software dependencies, according to the WordPress version chosen. Vagrant will then automate the process of creating Virtual Machines using VirtualBox's hypervisor.

A *hypervisor* is required to be installed on your host machine in order to provide hardware virtualization services to Vagrant. In this guide we will use VirtualBox because it is free and cross-platform. Other hypervisors such as Hyper-V, VMWare and KVM may also work, however instructions for other platforms will *not* be covered in this guide.

## Table of Contents

[TOC]

## Install VirtualBox

1. [Download VirtualBox Platform Package for Windows hosts](https://www.virtualbox.org/wiki/Downloads)
2. Install VirtualBox-\*-Win.exe
3. [Download Oracle VM VirtualBox Extension Pack](https://www.virtualbox.org/wiki/Downloads)
4. Install Oracle_VM_VirtualBox_Extension_Pack-\*.vbox-extpack

## Install and configure Vagrant

Vagrant will be used for bootstrapping and configuring Virtual Machines on top of VirtualBox, with minimal user intervention.

1. [Download Vagrant](https://www.vagrantup.com/downloads.html)
2. Install vagrant_\*x86_64.msi
3. Bookmark this guide and restart your computer.
4. *Right* click Start Menu and select **Windows PowerShell (Admin)**.
5. Install **vagrant-hostsupdater** plugin so that Vagrant will automatically update the [hosts](https://en.wikipedia.org/wiki/Hosts_(file)) file when you bring up a new VM. This way you dont't have to remember IP addresses.
> NOTE: Some AV software (e.g. Webroot) may lock the `C:\Windows\System32\drivers\etc\hosts` file from being edited. If you run into errors during the `vagrant up` process you may need to remove the vagrant-hostsupdater plugin and use the IP address (192.168.33.10) instead of the domain name (wpdistillery.vm) when accessing WordPress. You may also get a Windows Firewall warning about ruby script, be sure to allow it.

```powershell
vagrant plugin install vagrant-hostsupdater
```

6. Leave the PowerShell window open.

## Install Git for Windows

**Git for Windows** will be used for downloading and configuring WPDistillery.

1. [Download Git for Windows](https://git-scm.com/download/win)
2. Run the Git for Windows installer.
3. Be sure to choose a default editor which suits you:

![](https://i.imgur.com/Smom6zk.png)


If you don't know how to use **Vim** then you should probably use something easier like **Nano** or **Notepad++**.

4. Going with the default selections for the rest of the installation unless you know what you're doing.

## Download and configure WPDistillery

**WPDistillery** allows you to download, install and easily switch to different versions of WordPress. The scripts are downloaded using git. For this part we must run our commands using Git Bash which we installed in the previous section. 

Git Bash emulates a Linux CLI environment to a degree and does automatic conversion between Windows and Linux [line endings](https://en.wikipedia.org/wiki/Newline#Issues_with_different_newline_formats) using dos2unix or unix2dos.

1. Open **Git Bash** from the Start Menu.
2. Clone WPDistillery Repository.

```bash
git clone https://github.com/flurinduerst/WPDistillery.git
```
3. Edit configuration.

```bash
cd WPDistillery
nano wpdistillery/config.yml
```

4. Set `wpversion:` to `4.2` or whatever you like.
5. Set timezone. Use `America/Chicago` for US Central. Look at the *TZ database name* column [here](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) for other timezones.
> NOTE: ***If*** you were unable to use the `vagrant-hostsupdater` plugin, you will need to modify the `url:` parameter under `wpsettings:`. Change`url: wpdistillery.vm` to `url: 192.168.33.10`. Otherwise you will run into issues with URL routing later on.
6. Goto `# WPDISTILLERY SETUP` at the bottom and make the following changes:
```bash
setup:
  wp: true
  settings: true
  theme: false
  plugins: false
  cleanup: false
  #...
  comment: false
  posts: false
  files: false
```
7. Press <kbd>CTRL</kbd> + <kbd>X</kbd> then <kbd>Y</kbd> and lastly <kbd>Enter</kbd>/<kbd>Return</kbd> to save & exit.

## Create WordPress Virtual Machine using Vagrant

WPDistillery was cloned inside your home directory, cd into the cloned repo and provison the machine. In the **Administrator: Windows PowerShell** window type the following commands.


```powershell
cd $HOME\WPDistillery

vagrant up
```

## Conclusion

If everything worked you should now have a live WordPress site at the following address: 

http://wpdistillery.vm

Admin login is here:

http://wpdistillery.vm/wp-admin

Set permalinks to simple otherwise you'll get URL routing issues:




To SSH into the VM:

```powershell
vagrant ssh
```

To stop the VM:

```powershell
vagrant halt
```

When you are done you can delete the VM and cleanup using PowerShell (as Administrator) in the WPDistillery in your home directory.

```powershell
cd $HOME\WPDistillery
vagrant destroy
Remove-Item -Recurse -Force .\public\
```




