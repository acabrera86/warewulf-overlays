---
date: 2025-03-05
category: Warewulf
draft: true
tags:
  - warewulf
  - kdump
author:
  name: Arian Cabrera
  role: Sr. Customer Support Engineer
contributors: Stephen Simpson, Howard Van Der Wal 
title: How to Enable Kdump in Warewulf Nodes
---

## Introduction

Kdump utilizes kexec to swiftly boot into a dump-capture kernel whenever a memory dump of the system kernel is required, such as during a system panic. This process preserves the system kernel's memory image across the reboot, making it accessible to the dump-capture kernel.

## Prerequisites

_This guide assumes that a Warewulf server is successfuly installed and nodes are able to boot from a Rocky Linux image._

In many cases, the kdump service is installed and activated by default on new Linux installations, but some installation options do not have to install or enable kdump by default. If you do not know whether kdump is installed on your system.

You can "_shell into_" an the image with `wwctl image shell`. For 4.5.x versions you can use `wwctl container shell`.

```shell
[root@Warewulf ~]# wwctl image shell rockylinux-9
Image build will be skipped if the shell ends with a non-zero exit code.
```

From here you can check if the rpm has been installed by running `rpm -q kexec-tools`

```shell
[warewulf:rockylinux-9] /# rpm -q kexec-tools
package kexec-tools is not installed
```

If you need to install kexec-tools please run `dnf install kexec-tools`. This command secures installation of the userspace tools for kexec.

```shell
[warewulf:rockylinux-9] /# dnf install kexec-tools

Last metadata expiration check: 0:01:12 ago on Wed Mar  5 18:07:55 2025.
Dependencies resolved.
=============================================================================================================================================================================================
 Package                                        Architecture                           Version                                                  Repository                              Size
=============================================================================================================================================================================================
Installing:
 kexec-tools                                    x86_64                                 2.0.27-16.el9_5.1                                        baseos                                 477 k

...

Complete!
```

## Installation Instructions

### Configure kdump

The Linux kernel accepts several boot-time arguments. For example, Warewulf currently specifies the following arguments by default:
`quiet crashkernel=no vga=791 net.naming-scheme=v238`.

You can specify a different set of kernel arguments for a node or profile using `--kernelargs`. This value is recorded on a node or profile as the Kernel.Args field.

**To set this at the profile level, run this command:**

```bash
wwctl profile set default --kernelargs 'quiet,crashkernel=512M,vga=791,net.naming-scheme=v238'
```

**Similarly, to set this at the node level, run this command:**

```bash
wwctl node set node01 --kernelargs 'quiet,crashkernel=512M,vga=791,net.naming-scheme=v238'
```

_To specify the memory reserved for `kdump` kernel, set the `crashkernel=` option to the required value. For example, to reserve 128 MB of memory, you can use `crashkernel=128M`._

---
**_NOTE_:** _For newest Warewulf versions, `kernel args` is set as a list to be able to combine them between profiles and nodes, therefore, these values may get appended. You may have to negate default values by adding "`~`". An example is: `~crashkernel=no`._

---

### Changing the `excludes` file inside the container to allow copying `/boot` directory to the nodes

Warewulf can exclude files from an image to prevent them from being delivered to the compute node. This is typically used to reduce the size of the image when some files are unnecessary. Patterns for excluded files are read from the file /etc/warewulf/excludes in the image itself. For example, the default Rocky Linux images exclude these paths:

```shell
/boot/
/usr/share/GeoIP
```

To remove `/boot/` from the `excludes` file shell into the image:

```shell
[root@Warewulf ~]# wwctl image shell rockylinux-9
Image build will be skipped if the shell ends with a non-zero exit code.
[warewulf:rockylinux-9] /# sed -i '/\/boot\//d' /etc/warewulf/excludes
[warewulf:rockylinux-9] /# cat /etc/warewulf/excludes
/usr/share/GeoIP
```

Since kdump needs the vmlinuz file to boot the dump kernel, you need to add a symlink to `/usr/lib/modules/<kernel_version>/<vmlinuz file>`, as this file is not present on `/boot`.

```shell
[warewulf:rockylinux-9] /# KERNEL_VERSION=$(ls /usr/lib/modules | head -n 1)
ln -s /usr/lib/modules/$KERNEL_VERSION/vmlinuz-$KERNEL_VERSION /boot/vmlinuz-$KERNEL_VERSION
[warewulf:rockylinux-9] /# ls /boot/ | grep vmlinuz
vmlinuz-5.14.0-503.19.1.el9_5.x86_64
```

### Configuring the kdump target

When a kernel crash is captured, the core dump can be stored in various ways: as a file in a local file system, directly on a device, or sent over a network using NFS (Network File System) or SSH (Secure Shell) protocol. Only one of these options can be configured at a time. By default, the vmcore file is stored in the /var/crash directory of the local file system. To verify this, check the `/etc/kdump.conf` file. In this example, NFS is used since the Warewulf nodes will clear anything in memory after reboot, and anything added to /var/crash will get removed.

From `/etc/kdump.conf` file:

```shell
# nfs <nfs mount>
#           - Will mount nfs to <mnt>, and copy /proc/vmcore to
#             <mnt>/<path>/%HOST-%DATE/, supports DNS.
```

#### Configure `kdump` manually on the container

_Shell into_ the image and run the following commands:

```shell
[warewulf:rockylinux-9] /# grep -v ^# /etc/kdump.conf

auto_reset_crashkernel yes
path /var/crash
core_collector makedumpfile -l --message-level 7 -d 31

[warewulf:rockylinux-9] /# vi /etc/kdump.conf

# Edit the fields accordingly 
# If adding nfs please comment path

auto_reset_crashkernel yes
#path /var/crash
nfs nfs_server_ip:/kdump/path
core_collector makedumpfile -l --message-level 7 -d 31
```

#### Configuring kdump using overlays

To create the kdump overlay, please run:

```shell
$ wwctl overlay create kdump
```

Edit the kdump.conf.ww file (This command will create the overlay template)

```shell
$ wwctl overlay edit --parents kdump /etc/kdump.conf.ww
```

Add the following text after the `# This file is autogenerated by warewulf` line:

```bash
auto_reset_crashkernel yes
core_collector makedumpfile -l --message-level 7 -d 31

{{ if .Tags.kdump_nfs_location }}
nfs {{ .Tags.kdump_nfs_location }}
{{ else }}
path /var/crash
{{ end }}
```

Check that the new changes were applied successfully:

```shell
wwctl overlay show kdump /etc/kdump.conf.ww
```

Render the changes on a node to confirm it is working as expected.

```shell
$ wwctl overlay show -r node01 kdump /etc/kdump.conf

backupFile: true
writeFile: true
Filename: /etc/kdump.conf
# This is a Warewulf Template file.
#
# This file (suffix '.ww') will be automatically rewritten without the suffix
# when the overlay is rendered for the individual nodes. Here are some examples
# of macros and logic which can be used within this file:
#
# Node FQDN = node01
# Node Cluster = oso
# Network Config = <no value>, <no value>, etc.
#
# Go to the documentation pages for more information:
# https://warewulf.org/docs/main/contents/overlays.html
#
# Keep the following for better reference:
# ---
# This file is autogenerated by warewulf

auto_reset_crashkernel yes
core_collector makedumpfile -l --message-level 7 -d 31


path /var/crash
```

Change the tag to add the NFS path by using the `--tagadd` flag, in this example the path used was `172.16.131.5:/var/crash/`

```shell
$ wwctl profile set default --tagadd kdump_nfs_location=172.16.131.5:/var/crash/
Are you sure you want to modify 1 profile(s): y
```

Render the changes on a node to confirm the changes.

```shell
$ wwctl overlay show -r node01 kdump /etc/kdump.conf
# Removing extra lines to shorten the output
...

nfs 172.16.131.5:/var/crash/
```

Add the newly created overlay to the default profile, and the boot the node.

```shell
wwctl profile set default -O $(wwctl profile list default -a | grep SystemOverlay | awk '{print $3,kdump}')
```

### Testing the kdump configuration

---
**_⚠️ WARNING_** _The commands below cause the kernel to crash. Use caution when following these steps, and by no means use them on a production system._

---

To test the configuration, ssh into the node, and make sure that the service is running.

```shell
$ systemctl status kdump.service 

#If running please run the following

$ echo 1 > /proc/sys/kernel/sysrq
$echo c > /proc/sysrq-trigger
```

This forces the Linux kernel to crash, and the address-YYYY-MM-DD-HH:MM:SS/vmcore file is copied to the location you have selected in the configuration (that is, to `/var/crash/` by default)

## References & related articles

[Kdump docs](https://www.kernel.org/doc/Documentation/kdump/kdump.txt)
[kdump.conf man page](https://linux.die.net/man/5/kdump.conf)
[Warewulf documentation](https://warewulf.org/docs/)
