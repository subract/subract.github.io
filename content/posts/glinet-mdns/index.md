---
title: "Use mDNS with GL.iNet guest networks"
date: 2025-05-09T10:50:00-06:00
---

Recently, I've been using [a GL.iNet Beryl travel router](https://www.gl-inet.com/products/gl-mt1300/) for my home network. I purchased it for a road trip, then liked it well enough to continue using it as my primary home router.

I use its guest network feature for my IoT devices, to keep them segregated from my main LAN (I also use it as my only 2.4 GHz network, since I don't really trust band steering). This works well enough, but comes at a cost: mDNS broadcasts from devices on the guest network don't reach devices on the LAN. In particular, I have a Brother laser that I'd like to be able to use with AirPrint from the LAN.

I didn't find much on the Internet on how to set this up, so I decided to write it up.

## Step 1: Install `avahi-dbus-daemon`

Open the router admin interface. Navigate to Applications > Plug-ins. Search for and install `avahi-dbus-daemon` (note there are many results for `avahi` in the default repos, but you only need `avahi-dbus-daemon`):

![Dashy demo screenshot](install-avahi.png#center)

## Step 2: Configure Avahi

SSH into the router (using username `root` and the same password you use to sign into the web interface). Edit `/etc/avahi/avahi-daemon.conf`. It should be pre-populated with some default settings. Find and change `enable-reflector=no` to `yes`. Finally, restart the daemon with `/etc/init.d/avahi-daemon restart`

## Step 3: Test!

This was all I needed to get things working. Printing from my iPhone to an AirPrint printer on the guest network began working immediately.
