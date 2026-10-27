---
layout: post
title: "Computing in the Shadow of AI: HomeServer - Part 2 - Jellyfin Media Server"
date: 2026-10-21 
comments: false
categories: [homeserver, virtualization]
tags: [homeserver, proxmox, virtualization, jellyfin, media server, streaming, jekyll]
image: //assets/images/BP-HomeServer-02-01.png
description: "Computing in the Shadow of AI: Installing Jellyfin Media server in a LXC container and successfully streaming to my TV"
---

Welcome to **Computing in the shadow of AI**, where I try re-purposing, upgrading, optimizing and fixing older hardware because I am too broke to afford the current hardware prices.

## The Plan
Today I want to install **Jellyfin** on my homeserver. [Jellyfin](https://jellyfin.org/) is a free and open source media server that organizes, manages and streams digital media files to devices within my network. It also has a bunch of great client apps for client devices like smartphones, tablets, smart TVs and the web browser.

Here is a quick snapshot of where we left off last time:
![Current HomeServer State](/assets/images/BP-HomeServer-02-01.png)

These are the steps I will follow to setup Jellyfin:
1. Set up an **LXC Container** with **Debian**
2. Passthrough **iGPU** and storage location
3. Install & Configure **Jellyfin**
4. Upload some files
5. Test it by watching some media on my TV

## LXC container creation
The first step is to create the **LXC container** in which **Jellyfin** will be running. The first thing is to download the **Debian** container template. To do this click on one of your **Storages**, navigate to **CT Templates** and click on **Templates**. Then just search for the desired **OS** template and click download:

![Downloading Debian Template](/assets/images/BP-HomeServer-02-02.png)

Once we have that we can start creating the container. Right click on the **Node** the container should be created on and select **Create CT**. A screen will appear where you can configure some general stuff about the container, such as the hostname and a root password:

(I turned **Nesting** off, because from what I can tell you only need it when trying to run **Docker** or similar in the container. As I am not planning to do that I can disable it for a small optimization)

![Creating an LXC Container - General](/assets/images/BP-HomeServer-02-03.png)

Then select the **OS Template** that got downloaded in the beginning:

![Creating an LXC Container - Template](/assets/images/BP-HomeServer-02-04.png)

Next allocate some disk space for the container on one of the storages. I chose to give it **20GB** as this is only for the **Jellyfin** installation itself. The actual media files will be stored somewhere else:

![Creating an LXC Container - Disks](/assets/images/BP-HomeServer-02-05.png)

Assign it some **CPU cores**. 2 should be enough here, since the **iGPU** will also do some heavy lifting:

![Creating an LXC Container - CPU](/assets/images/BP-HomeServer-02-06.png)

And some **memory**, I chose **2GB**:

![Creating an LXC Container - Memory](/assets/images/BP-HomeServer-02-07.png)

Setting up the **network**: After finishing the container setup, I chose an IP for the container and reserved it within my router's **DHCP** with the **MAC address** of the virtual network switch (The **MAC address** is in the **Network** tab of the container). In this step I set the **IPv4** and **IPv6** addresses to **static** and left them empty. If they are left on **DHCP** they will conflict with my address binding and I won't be able to reach the container via my chosen IP: 

![Creating an LXC Container - Network](/assets/images/BP-HomeServer-02-08.png)

I skipped over the **DNS** section, since I used all default options. Then review your choices and hit finish. Proxmox will then create your **LXC container**:

![Creating an LXC Container - Finish](/assets/images/BP-HomeServer-02-09.png)

## iGPU driver installation
I want to use the **iGPU** of my system with **Jellyfin** to use video acceleration. This will result in massive **CPU** performance savings, since en/decoding will now be done by the **GPU** instead. Before I can passthrough the **iGPU** I have to first install the **drivers** for it on my **Proxmox host**.

To be able to install the necessary proprietary intel drivers, I had do add the **contrib, non-free and non-free-firmware** repositories. To do that I had to add the following to my **/etc/apt/sources.list** file:
```bash
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
deb http://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
```

And then run update to update my package list in **apt**:
```bash
apt update
```

Now I could install the required **drivers**, **firmware** and **info utility**:
```sh
apt install -y intel-media-va-driver-non-free vainfo intel-gpu-tools
```

Before continuing I used the `vainfo` (Video Acceleration info utility) command to check that the [VA-API](https://en.wikipedia.org/wiki/Video_Acceleration_API) is communicating with my iGPU correctly:
![vainfo shows that iGPU is communicating with va-api](/assets/images/BP-HomeServer-02-11.png)

Now that I could verify that it works, lets pass it through to the container.

## iGPU and storage pass through
Passing through any device is extremely convenient on my Proxmox version. I simply have to got to my **container** and then to the **resources** menu. From there simply click **Add > Device Passthrough**

I passed both the **/dev/dri/renderD128** and **/dev/dri/card0** with the **GID (Group ID)** of my render usergroup:
![iGPU Passthrough with advanced options setting the GID](/assets/images/BP-HomeServer-02-12.png)

You can find out the **GID** of your render group with this command:
```bash
getent group render
```

By executing the `vainfo` command on the container shell I can verify that the passthrough was successful:
![vainfo on container shows successful passthrough](/assets/images/BP-HomeServer-02-14.png)

Similarly to the **iGPU**, I also passed a storage location to the container. This is where I will be storing all my media files that **Jellyfin** will stream later. Simply click **Add > Mount Point**. Since I do not have a **NAS** yet (why? see my introductory statement), for now this is just a folder on the **HDD** in my server.

Make sure that the jellyfin user has read/write permissions in your mount point in the container, otherwise it won't be able to access the media files! 

## Installing Jellyfin
Now it is time to start the container for the first time and get to installing **Jellyfin**.

First step is always to **update && upgrade** the system with:
```bash
apt update && apt upgrade -y
```

Then I installed `vainfo` on the container as well to verify that the **iGPU** pass through was successful (see previous image for the output):
```bash
apt install vainfo -y
vainfo
```

Installed **Jellyfin** via their official **install script** with **curl**:
```bash
apt install curl -y
curl https://repo.jellyfin.org/install-debuntu.sh | bash
```

Then enabled the **service** in systemctl:
```bash
systemctl enable --now jellyfin
```

Added **Jellyfin** to the render and video groups, so that it can access the **iGPU** we passed through earlier:
```bash
usermod -aG render jellyfin
usermod -aG video jellyfin
```

Then restarted the **Jellyfin** **service** to enable it to use the new user groups:
```bash
systemctl restart jellyfin
```

And then navigate to **http://YOUR_JELLYFIN_IP:8096** and you will be greeted with this screen:
![Jellyfin initial setup screen](/assets/images/BP-HomeServer-02-13.png)

## Configuring Jellyfin
At this point Jellyfin is running on my server. I did it yay!
For the last part I will show some important configuration that I did.

Go to your **dashboard** and navigate to **Libraries**. Click on **Add Media Library**, name your library and point to the folder (or subfolder) that was mounted earlier:
![Jellyfin add media library](/assets/images/BP-HomeServer-02-15.png)

Afterwards don't forget to click **Scan All Libraries**, so that the content in the folder gets recognized by Jellyfin.

And lastly turn on **transcoding** with the iGPU we passed through earlier. Simply go to **Playback > Transcoding** and set the Hardware Acceleration to **Intel Quicksync (QSV)** and the **QSV Device** to **/dev/dri/renderD128**:
![Jellyfin configure transcoding](/assets/images/BP-HomeServer-02-16.png)

Also do not forget to create some user(s). Avoid using the admin user for normal use.

## Testing and conclusion
To test I uploaded an Anime that I owned to the Jellyfin directory with the [SCP](https://en.wikipedia.org/wiki/Secure_copy_protocol) command like this:
```bash
scp -r /path/to/local/folder root@<your-proxmox-ip>:/path/to/jellyfin/folder
```

And after installing the Jellyfin app to my FireTV Stick and logging in, here is the result in all its glory:
![Streaming anime to my TV](/assets/images/BP-HomeServer-02-17.png)

With this I can say goodbye to any streaming service and instead focus on building my own physical/digital media library. I am very happy with the results, even though I suspect I will be spending a lot of money on storage in the future.

The next project I want to realize on this server is to set up a **VM** running **[Immich](https://immich.app)** to be able to backup and organize all of my vacation photos. Stay tuned!
