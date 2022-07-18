+++
title = "KVM RHCOS PXE Boot"
date = "2022-07-13T15:29:50+02:00"
author = "Kristjan Voje"
authorTwitter = "" #do not include @
cover = ""
tags = ["rhcos", "pxe", "boot", "coreos", "redhat"]
keywords = ["rhcos", "pxe", "boot", "coreos", "redhat"]
description = "Booting RHCOS on KVM using PXE boot"
showFullContent = false
+++

# KVM RHCOS PXE Boot
We will create a KVM network called `mylab` and launch a RHCOS instance using 
PXE boot. We will use KVM's integrated DHCP and TFTP server.   

Prerequisites:
* Linux system with kvm/libvirt installed (using Ubuntu 20.04 for this example).   
* 3GB RAM for the VM.

## Steps
1. Create a libvirt network with static DHCP and TFTP settings
2. Download initramfs, kernel and rootfs
3. Prepare a TFTP server for hosting initramfs (`initrd.img`) and the kernel (`vmlinuz`)
4. Configure PXE
5. Prepare a HTTP server for hosting the rootfs (`rhcos-live-rootfs.x86_64.img`)
6. Boot the VM

## Create a libvirt network
We define static DHCP entries in `virsh net` configuraion.   
Same goes for the TFTP server root dir and bootp file.   
```xml
<!-- mylab.xml -->
<network>
    <name>mylab</name>
    <forward mode='nat'>
        <nat>
        <port start='1024' end='65535'/>
        </nat>
    </forward>
    <ip address='192.168.132.1' netmask='255.255.255.0'>

        <tftp root='/var/www/tftpboot' />

        <dhcp>
        <range start='192.168.132.100' end='192.168.132.254'/>
        <host name='bootstrap' mac='52:54:00:90:6c:19' ip='192.168.132.19'/>

        <bootp file="pxelinux.0" />
        </dhcp>
    </ip>
</network>
```
```bash
virsh net-define mylab.xml
virsh net-start mylab
```

## Download initramfs, kernel and rootfs
We'll be using RedHat's official images.   
For this we need a RedHat account (it's free, so far).   
It's also possible to do this with Fedora CoreOS images, which is an upstream 
of RHCOS (or any other OS for that matter).   

Download initramfs, kernel and rootfs from:   
https://console.redhat.com/openshift/install/platform-agnostic

## Prepare a TFTP server
While we're using KVM to serve the contents of `/var/www/tftpboot`, we still 
need to populate that dir.   
```bash
# /var/www/tftpboot# tree
.
├── ldlinux.c32
├── libutil.c32
├── menu.c32
├── pxelinux.0
├── pxelinux.cfg
│   └── default         # <-- boot menu, config for pxelinux.0
└── rhcos
    ├── initrd.img      # <-- initramfs, served by TFTP
    └── vmlinuz         # <-- kernel, served by TFTP
```
(On Ubuntu), we can get the libraries by installing the following packages:   
```bash
apt-get install pxelinux syslinux
# /usr/lib/PXELINUX/pxelinux.0
# /usr/lib/syslinux/modules/bios/ldlinux.c32
# /usr/lib/syslinux/modules/bios/libutil.c32
# /usr/lib/syslinux/modules/bios/menu.c32
```
## Configure PXE
```bash
mkdir pxelinux.cfg
touch pxelinuc.cfg/default
```
`pxelinux.0` is a lightwaight OS that starts the bootstrapping process.   
`pxelinux.cfg/default` is configuration for that OS.   

```bash
# pxelinux.cfg/default
DEFAULT menu.c32
PROMPT 0

MENU TITLE ### MyLab boot menu ###
MENU AUTOBOOT Starting SuperOS in # seconds
TIMEOUT 5
TOTALTIMEOUT 10
ONTIMEOUT local

LABEL local
  MENU LABEL rhcos
  MENU DEFAULT
  KERNEL rhcos/vmlinuz
  APPEND initrd=rhcos/initrd.img ip=dhcp coreos.live.rootfs_url=http://192.168.132.1:8080/rhcos/rhcos-live-rootfs.x86_64.img

# In case we want to add an ignition file (RHCOS and Fedora CoreOS)
# coreos.inst.ignition_url=http://<HTTP_server>/bootstrap.ign  
```

## Prepare a HTTP server for serving rootfs
Last thing missing; we need to serve the rootfs defined in `pxelinux.cfg/default`:
```
coreos.live.rootfs_url=http://192.168.132.1:8080/rhcos/rhcos-live-rootfs.x86_64.img
```
Ideally, we would like a HTTPS server but this demo focuses on PXE.   
We'll be using NGINX to server our filesystem.   

```bash
# /etc/nginx/sites-enabled/pxe 
server {
	listen 8080 default_server;

	root /var/www/pxe;

	location / {
		autoindex on;
	}
}
```
We're serving the following filesystem image:   
```
/var/www/pxe/rhcos/rhcos-live-rootfs.x86_64.img
```

## Boot the VM
If all is well, we should be able to network boot a RHCOS VM.   
**Make sure to use at least 3GB of RAM**
```bash
virt-install \
    --pxe \
    --connect=qemu:///system \
    --name=bootstrap \
    --vcpus=2 \
    --memory=3072 \
    --os-variant=fedora31 \
    --disk=size=10 \
    --network network=mylab,mac=52:54:00:90:6c:19
```
The above command will get us to the login screen.   
We won't be able to login though - for that we need to configure COREOS with 
an `ignition` file (out of scope for this demo).   

## Bonus: Ignition
We can configure a COREOS machine at boot using Ignition.   
Generate an Ignition file, place it on your HTTP server (alongside rootfs), then 
add the following boot arguments to your PXE config:
```bash
# pxelinux.cfg/default 
DEFAULT menu.c32
PROMPT 0

MENU TITLE ### MyLab boot menu ###
MENU AUTOBOOT Starting SuperOS in # seconds
TIMEOUT 5
TOTALTIMEOUT 10
ONTIMEOUT local

LABEL local
  KERNEL rhcos/vmlinuz
  APPEND initrd=rhcos/initrd.img ip=dhcp coreos.live.rootfs_url=http://192.168.132.1:8080/rhcos/rhcos-live-rootfs.x86_64.img ignition.config.url=http://192.168.132.1:8080/ignition/bootstrap_test.ign ignition.firstboot ignition.platform.id=metal
```

## Sources
* https://docs.fedoraproject.org/en-US/fedora-coreos/live-reference/
* https://docs.openshift.com/container-platform/4.10/installing/installing_platform_agnostic/installing-platform-agnostic.html#installation-user-infra-machines-pxe_installing-platform-agnostic
* https://computingforgeeks.com/install-virtual-machines-on-kvm-using-pxe-and-kickstart/