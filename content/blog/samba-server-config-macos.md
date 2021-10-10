+++
title = "Samba Server Config for macOS"
date = "2021-09-26T05:06:24-04:00"
tags = ["sysadmin", "linux"]
audio = []
images = []
videos = []
draft = "true"
+++

This is a short guide on configuring a Samba server to provide files for macOS clients, figured out through trial, error, and careful poking at the documentation.

<!--more-->

This post was written in 2021 based on NixOS 21.05, Samba 4.14.4, Avahi 0.8, and macOS 11.6 (Big Sur).

# Samba

The first step is to load a Samba configuration that has all of the important parts.  Beyond the basic directives that are required for every Samba server, I found several additional directives to be important.

Globally:

* `fruit:advertise_fullsync = true` — Turns on advertisements for a feature that improves the performance of Time Machine backups.
* `fruit:aapl = yes` — Enables the `vfs_fruit` module's support for Apple clients.
* `fruit:model = Xserve` — Changes the icon displayed in Finder for this server, when connecting by IP/hostname.

Per Share:

* `fruit:time machine = yes` — A blanket flag that enables several Time Machine-related optimizations.
* `vfs objects = fruit streams_xattr` — Enables the `vfs_fruit` module for this share, which provides all of the Apple-related enhancements; enables the `streams_xattr` module for this share, which improves support for macOS extended attributes.

For more information, check [the documentation for `vfs_fruit`](https://www.samba.org/samba/docs/current/man-html/vfs_fruit.8.html).

# Avahi

If you also want your Samba share to have a nice icon in the Network tab (e.g. not a beige monitor displaying a Windows error), you have to install and enable Avahi.  macOS clients expect to see mDNS service advertisements for `_smb._tcp` and `_device-info._tcp`.  I provided these with a service definition in `/etc/avahi/services/smb.service` [(based on this post)](https://www.tumfatig.net/2017/let-mac-os-auto-discover-your-smb-shares/):

```xml
<?xml version="1.0" standalone='no'?><!--*-nxml-*-->
<!DOCTYPE service-group SYSTEM "avahi-service.dtd">
<service-group>
  <name>Your Display Name Here</name>
  <service>
	<type>_smb._tcp</type>
	<port>445</port>
  </service>
  <service>
	<type>_device-info._tcp</type>
	<port>0</port>
	<txt-record>model=Xserve</txt-record>
  </service>
</service-group>
```

This configuration makes the share appear to come from an Xserve.  If you want to pretend the share is coming from another Apple device, you can try other Apple model identifiers (`PowerBook`, `MacPro`, `Macmini`, and so forth).  A full list of these is in `/System/Library/CoreServices/CoreTypes.bundle/Contents/Info.plist`, look for the many instances of `com.apple.device-model-code` [(source)](https://askubuntu.com/questions/1109810/ubuntu-18-10-samba-4-8-4-smb-conf-what-are-the-valid-values-for-fruitmodel#1144356).

Originally, this had `<name replace-wildcards="yes">%h</name>`, but for some reason, that stopped working on my machine.  It would cause the Finder to say "connection failed" whenever you browsed to the host from the Network sidebar entry. Changing it to anything other than the machine's hostname made it work again.  I have no idea whether this is an actual problem, or just a weird caching issue.