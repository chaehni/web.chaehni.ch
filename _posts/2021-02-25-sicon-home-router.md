---
title: "Building a SCION enabled Home Router"
excerpt: How to bring SCION into your home!
toc: true
header:
  overlay_image: "/assets/images/scion-router/router-teaser.jpg"
  overlay_filter: 0.5
  teaser: "/assets/images/scion-router/router-teaser.jpg"
gallery:
  - url: /assets/images/scion-router/luci.png
    image_path: /assets/images/scion-router/luci.png
    alt: LuCI router interface
    title: LuCI router interface
gallery2:
  - url: /assets/images/scion-router/luci_scion_setup.png
    image_path: /assets/images/scion-router/luci_scion_setup.png
    alt: SCION setup interface
    title: SCION setup interface
  - url: /assets/images/scion-router/luci_scion_sig.png
    image_path: /assets/images/scion-router/luci_scion_sig.png
    alt: SIG setup interface
    title: SIG setup interface
categories:
  - Networks
techs:
  - openwrt:
    name: OpenWrt
    url: https://openwrt.org/
    image: "/assets/images/tech/openwrt.png"
  - scion:
    name: SCION
    url: https://www.scion-architecture.net/
    image: "/assets/images/tech/scion.png"
  - musl:
    name: "musl libc"
    url: https://www.musl-libc.org/
    image: "/assets/images/tech/musl.png"
  - golang:
    name: Go
    url: https://golang.org/
    image: /assets/images/tech/golang.png
  - lua:
    name: Lua
    url: http://www.lua.org/
    image: /assets/images/tech/lua.png
tags:
  - Hardware
  - SCION
  - OpenWRT
---

The goal of this project is to build my own home router to access the Internet.
In a second step, I then want to enable
[SCION](https://scion-architecture.net) support on the router by installing the
SCION stack. The result will be a home router that can natively speak the SCION protocol
and even translate between legacy IP and SCION using the
[SCION IP Gateway (SIG)](https://docs.scionlab.org/content/apps/remote_sig.html). Of
course, regular IP traffic should not be affected and work just fine in parallel
to SCION.

## Why build your own home router?
Reading the title of this post you might ask yourself: "Why build your own
router when my ISP ships a perfectly capable router already?"
The truth is that ISPs, like all companies, operate on a budget. It's in their
interest to sell as many Internet connections as possible for as little cost as
possible. ISP provided routers tend to be bad in a couple of ways:

- ISP routers tend to be configured for their convenience and not for security
  in order to cut down on tech support.
- Often, these devices are locked down in one way or the other. Certain features may not be available
  and cannot be added by the end user. I think that users should have full
  control over the devices they run in their own network.
- Updates to ISP provided routers are a rare thing (if they happen at all).
  Something as critical as your router, protecting your home network from the
  Internet, should always operate with the latest security patches applied.
- Potentially millions of people run the exact same router with the exact same
  standard configuration, making these routers a prime target for any attacker.


## Overview
With the "Why?" out of the way, I'd now like to provide an overview of my envisioned router.

The figure below shows an abstraction of the desired outcome. The router
contains an IP and a SCION stack which are connected to their respective
upstream networks. Between the two stacks sits the SIG which translates between
IP and SCION. Clients can connect to the router via Ethernet or Wi-Fi.
Independent of the connection, clients can make use of both network stacks.

![Usecase Overview](/assets/images/scion-router/usecase_overview.png "Router Overview")

It is clear, that IP packets from my client devices will use the IP stack on the router
while SCION packets will use the SCION stack. However, with the SIG I have the
possibility to translate certain IP connections to SCION connections. This is
useful to make applications that are completely oblivious of SCION benefit from
SCION features such as path selection, multi-path communication and hidden
paths. The downside is that on the other end another SIG is needed
to translate the SCION packets back to IP. This means that I need prior knowledge
about which IP addresses sit behind a SIG and can therefore be accessed via
SCION. I need to manually configure these address on my home router,
such that it knows to send corresponding packets via the SIG, translating the
connections to SCION. The goal is to make this configuration easily
accessible via the router's web interface.

A few words about residential SCION connectivity: Because most ISPs, like mine, don't yet
offer a native SCION connection, I will connect my router to the
[SCIONLab](https://www.scionlab.org/) network. SCIONLab is a global research
network, open to anyone who who wants to try out SCION. Once my ISP offers
access to the SCION production network I'll be able to easily switch over to
that network.
{: .notice--info}

## The Hardware
For the hardware, I chose the APU platform from [PC
Engines](https://www.pcengines.ch/) which is a widely adopted server platform
for low power, wireless networking and embedded applications based on the x86
architecture. All the parts required for this build are listed below.

| Part         | Description                                                                                                    |
| ------------ | -------------------------------------------------------------------------------------------------------------- |
| apu2e4       | APU.2E4 system board (GX-412TC quad core / 4GB / 3 Intel GigE / 2 miniPCI express / mSATA / USB / RTC battery) |
| case1d2bluu  | Enclosure for alix2 / apu1 / apu2 (3 LAN, blue, USB)                                                           |
| ac12veur3    | AC adapter with euro plug                                                                                      |
| msata16g     | 16GB mSATA SSD module                                                                                          |
| wle600vx     | Compex WLE600VX 802.11ac miniPCI express wireless card                                                         |
| 2 x antsmadb | Antenna                                                                                                        |
| 2 x pigsma   | IPEX to reverse SMA pigtail cable                                                                              |
| usbcom1a     | USB to DB9F serial adapter                                                                                     |

### Assembling the Router
For the basic assembly of the APU I followed this video:

<p>
<iframe width="560" height="315" src="https://www.youtube.com/embed/ft_Ic2ZdLHw" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</p>

In addition to the standard setup, of course I also wanted my router to be Wi-Fi capable!
For that reason I installed the miniPCI wireless card and the antennas.

{% comment %}
(image of router internals)
{% endcomment %}

This is what my fully assembled router looks like:
![Full assembled SCION Router](/assets/images/scion-router/router.jpg "Fully assembled SCION Router")

## The Software
Of course, hardware alone doesn't make a router. Luckily, there are excellent
open-source router platforms available. One of them is OpenWRT.

### OpenWRT
[OpenWRT](https://openwrt.org/) is a full Linux-based operating system
designed for wireless routers. It offers a writable file system and it even
includes a package manager which makes the OS highly extensible. OpenWRT has
wide support for different hardware, including our
[APU2](https://openwrt.org/toh/pcengines/apu2) board.

OpenWRT ships with a web UI called LuCI. LuCI is your typical router's user
interface which you can access over the web (usually `192.168.1.1`).
However, like everything else about OpenWRT it is very feature rich and highly customizable.

### Accessing the board
Since the APU2 board does not have a display output and SSH is only available
once the OS is installed, we need to rely on the DB9 serial port to establish a
connection. There are several serial communication programs available. I used
`picocom`. The following command connects to the board
over the USB-to-serial cable plugged into my computer and the APU:

`picocom -b 115200 /dev/ttyUSB0`

115200 is the baud rate (or symbol rate) of the APU.

### Flashing the OS
At the time of writing, the most recent OpenWRT version is ```19.06.4``` and can
be downloaded from
[here](https://downloads.openwrt.org/releases/18.06.4/targets/x86/64/).
I'll be using the Ext4 file system instead of the squashfs because I'd like to
have a fully writable file system and space limitations are not a concern thanks
to the large 16GB SSD installed in the board. The OS comes with two partitions, the boot partition and
the root file system partition. I used to following steps to flash the OS to the
board and expand the root file system to us the full size of the SDD (instead of
an SD card, one can also use a USB thumb drive):

* Copy the downloaded image onto to an SD card: `gzip -dc openwrt-19.07.7-x86-64-combined-ext4.img.gz | sudo dd status=progress bs=8M of=/dev/sdX`
* Boot the board from SD card
* Clone the SD onto the SSD drive: `dd bs=8M if=/dev/mmcblk0 of=/dev/sda`
* The root partition now has the same size (256M) as the SD. We need to increase it:
    * `fdisk /dev/sda` (fdisk needs to be installed)
    * `d 2` to delete the root file system partition on the SSD
    * `n 2` to create a new partition, when asked for input accept standard values
    * `p` to see the new partition layout, confirm the second partition has increased in size
    * `w` to write the changes to disk
* Copy over the root file system from the SD card: `dd bs=8M if=/dev/mmcblk0p2 of=/dev/sda2`
* Expand the file system to make use of the entire partition:
    * `e2fsck -f /dev/sda2` (accept eventual repairs)
    * `resize2fs /dev/sda2` (resize2fs needs to be installed)
* remove the SD card and boot from SSD, confirm the root partition `/dev/root` has increased in size: `df -h`

Success! The router can now be accessed via a browser at `192.168.1.1`.

{% include gallery id="gallery" %}

## Make it speak SCION!

So far, I have a fully functional and highly configurable router, where I
control the hard- and software. This was the first goal of the project. The next
step is now to install the SION stack on the router. The plan is to run
a full SCION autonomous system (AS) inside the router. For this, I will install
the `SCION Control Service`, `SCION Border Router`, `SCION Dispatcher`, `SCION Daemon`
and the `SCION IP Gateway`.

### Compiling the SCION binaries

The services listed above are written in Google's [Go](https://golang.org/)
programming language and are targeting Debian x86 systems. However, some
of the services call functions from the C standard library behind the scenes.
Since Debian uses the GNU C Library (glibc), the services are dynamically
linking against that specific libc implementation. Usually this is not a
problem. However, OpenWRT uses musl-libc which means that
the services need to be compiled from source against musl-libc.
Luckily, it is easy to adapt the usual `go build` command to accept a C compiler
other than the default.
The following commands build the binaries using `musl-gcc` which is a wrapper for the
`gcc` toolchain that targets musl-libc instead of glibc:

```
$ CC=/usr/local/musl/bin/musl-gcc go build --ldflags '-linkmode external'
$ CC=/usr/local/musl/bin/musl-gcc go build --ldflags '-linkmode external -extldflags "-static"'
```
The first command builds dynamically linked binaries whereas the second command
builds statically linked binaries.

### Adding a SCION tab to the LuCI interface

With the back end ready to to speak SCION, the only thing missing now is a new
tab in the LuCI user interface to configure the SCION configuration.

A brief explanation on how to add a new tab to the LuCI interface can be found
[here](https://openwrt.org/docs/guide-developer/luci). Essentially, it requires
a web view written in HTML together with a back end written in Lua. I added a SCION tab with
two sub-tabs, one for the SCION configuration and one for the SIG configuration.

This is the result:
{% include gallery id="gallery2" %}

I can upload any valid SCION configuration and then use the `Start`, `Restart` and
`Stop` buttons to control the SCION installation through the convenience of
the LuCI interface. Additionally, SIG rules can be defined and the SIG can be
started and stopped independently of SCION.

## Testing

Coming soon

{% comment %}
### Bandwidth measurements

#### IP
(add speedtest.net result image from gigabit lan)
![Dowload and upload speed](/assets/images/scion-router/speedtest.png "Dowload
and upload speed")

#### SCION

### Native SCION

### SIG
{% endcomment %}

## Presentation

The SCION home router has been showcased at the SCION Day on November 6, 2019.
Find the slides and video here: [SCION Day presentation](https://video.ethz.ch/events/2019/scion/61dd8a87-3894-489d-99d3-77ca66c5ad38.html).
