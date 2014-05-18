---
layout: post
title:  "External Storage and Shares"
date:	10/05/2014
---

In this post I explain how to configure automatic mount of external storage device (my Western Digital external hard drive). And how to Configure the Samba share for certain folders which it contains.

###  Install ntfs-3g

I am using ntfs on the external HD, so every non-computer-guy (A.K.A my parents) can
simply plug out the device and copy files to their computer.

_ntfs-3g_ is a package used to mount the NTFS filesystem, deprecating the
ntfs kernel module (_ntfs-3g_ is using FUSE).

```
pacman -S ntfs-3g
```

### Configure Automatic Mount of the HD

Create systemd's mount file for the HD.
Change the uuid to the uuid of the ntfs partition.

###### /etc/systemd/system/mnt-wd.mount
```
[Unit]
Description = Western Digital External Hard Drive
Requires = local-fs.target
After = local-fs.target
BindsTo = dev-disk-by\x2duuid-AAAAAAAAAAAAAAAA.device
After = dev-disk-by\x2duuid-AAAAAAAAAAAAAAAA.device

[Install]
WantedBy = dev-disk-by\x2duuid-AAAAAAAAAAAAAAAA.device

[Mount]
What = UUID=AAAAAAAAAAAAAAAA
Where = /mnt/wd
Options = noexec,nofail,umask=0
DirectoryMode = 0666
```

```bash
systemctl enable mnt-wd.mount
```

### Install and Configuring Samba

```bash
sudo pacman -S samba
sudo systemctl enable smbd.socket
sudo systemctl enable nmbd
sudo ufw allow from 192.168.1.0/24 to any app CIFS
	
```

configuration files:

###### /etc/samba/smb.conf
```
[global]
	server string = %h server (Samba, Arch)
	workgroup = WORKGROUP
	map to guest = Bad Password
	syslog = 0
	log file = /var/log/samba/log.%m
	max log size = 1000
	unix extensions = yes
	guest ok = yes
	guest only = yes
	read only = yes
	include = /etc/samba/smb-shares.conf
```

###### /etc/samba/smb-rw.conf
```
[global]
	server string = %h server (Samba, Arch)
	workgroup = WORKGROUP
	map to guest = Bad Password
	syslog = 0
	log file = /var/log/samba/rw-log.%m
	max log size = 1000
	unix extensions = yes
	guest ok = yes
	guest only = yes
	writeable = yes
	include = /etc/samba/smb-shares.conf
```

###### /etc/samba/smb-shares.conf
```
[share]
	path = /mnt/wd/share
	create mask = 0777
	directory mask = 0777

[local-share]
	path = /mnt/wd/local-share
	create mask = 0777
	directory mask = 0777
```

###### /etc/systemd/system/smbd-rw.socket
```
[Unit]
Description=Samba SMB/CIFS server socket

[Socket]
ListenStream=127.0.0.1:4455
Accept=yes

[Install]
WantedBy=sockets.target
```

###### /etc/systemd/system/smbd-rw@.service
```
[Unit]
Description=Samba SMB/CIFS server instance

[Service]
ExecStart=/usr/bin/smbd -F -s /etc/samba/smb-rw.conf
ExecReload=/bin/kill -HUP $MAINPID
StandardInput=socket
```

### windows port forwarding

```
c:\cygwin64\bin\ssh.exe -TN -o UserKnownHostsFile=/cygdrive/c/Users/%USERNAME%/.ssh/known_hosts -i /cygdrive/c/Users/%USERNAME%/.ssh/id_rsa-smb -L 4455:localhost:4455 192.168.1.50
```
