+++
date = '2024-11-13T01:01:49-05:00'
draft = false 
title = 'Root of Un-Trust'
author = 'David Hillman'
+++
## Introduction

Recently, I received a couple of Kingdel branded mini PCs from a system integrator friend of mine as a favor for some pro bono work I did. The machines all came with an Intel quad core processor, 16GB of DDR3 RAM, dual Realtek NICS and were all silent. I thought they'd be perfect for a small Proxmox virtualization environment, especially since they all supported dual MSATA flash disks. This blog post will document my experience analyzing these devices to determine whether they are fit for production use in my lab.

## Teardown

I started my assessment of these tiny devices by booting up each one to see whether they worked. All devices booted up perfectly to Windows. Next, I opened up the back panel to check out the hardware. All of the devices came with a single Samsung 240GB MSATA disk and Intel wireless card. Next, I looked for any BIOS upgrades that might be available, and that's where the problems started. The manufacturer website (https://kingdel.com/cn) showed no BIOS images available. I then searched the BIOS maker website--American Megatrends--but came up empty handed. According to the website, Kingdel is based out of Shenzhen, China.

![Kingdel PC](/05_kingdel_pc.jpg 'Kingdel PC')

## Linux OS Installation

Next steps were to install the OS I had been running for a year on various desktops in my lab, Fedora Silverblue. I figured I could at least use the Privacy & Security tools to assess device security. Silverblue installation went off without a hitch. I was even able to install the usual suite of software I use on a Linux desktop: Firefox flatpack version, Inkscape, LibreOffice, Teams for Linux, Ungoogled Chromium, Visual Studio Code, etc. Everything worked just as expected.

## Firmware Assessment

Using the Privacy & Security tools, I was able to determine the device didn't have SecureBoot enabled. This is important to device and data security because Secure Boot uses cryptographic signatures to verify code integrity as the device boots. Two critical databases are involved in this process: an Allow DB (db) of approved components and a Disallow DB (dbx) of vulnerable or malicious components--firmware code, drivers, bootloaders, shims etc. A Key Exchange Key (KEK) protects these databases from being modified inappropriately. All of this is then verified by a Platform Key (PK), which serves as the root of trust for firmware updates. PK is not used as part of the boot process, but it is still pretty important. The dbx, db, and KEK are what's used to verify signatures for any objects loaded at boot time, including the operating system images. I was not surprised when the firmware tools revealed some outdated objects present on these systems. Even worse, the firmware was signed by a PK that was invalidated. 

![Device Checks](/00_device_checks_result.jpg 'Device Checks')

## Vulnerability Note VU#455367 - Insecure Platform Key (PK) used in UEFI System

According to the description for this vulnerability and the UEFI standard, there is to be a trust relationship using Public Key Infrastructure (PKI) between a platform owner, platform firmware, and the operating system. Central to everything is the Platform Key, which is designed to secure this trust connection between the platform owner and the platform firmware. The PKFail vulnerability highlighted a critical flaw in the entire UEFI ecosystem. The PK is supposed to originate from the Original Equipment Manufacturer (OEM) using a hardware security module (HSM) to proctect the key, but in practice software keys (softkeys) are hard-coded for ease of building and testing. One of these test keys did make its way onto production firmware like the one running on the Kingdel devices. This is a serious issue because UEFI firmware is pretty much invisible to Endpoint Detection and Response (EDR) tools, making it difficult to detect the use of compromised keys. Remote measurement could dynamically check the key database for integrity problems, but most UEFI implementations lack this capability.

## Assess Using fwupdmgr Tools

The fwupd project (https://github.com/fwupd/fwupd) aims to make firmware updates on Linux automatic, safe, and reliable. The tools can be installed using Snap or Flatpak if they are not already installed. Bleeding edge versions can be compiled, but should not be considered a replacement for the distro-provided version installed on the system. The tools are configured by default to download firmware from Linux Vendor Firmware Service (LVFS). The service is available to all OEMs and firmware authors who would like to make firmware available through Linux.

![AMI BIOS Do Not Use](/03_ami_bios_do_not_use.jpg 'AMI BIOS Do Not Use')

###Check Device Supported by fwupd

- fwupdmgr get-devices
This displays all devices detected by fwupd. [img]

- fwupdmgr refresh
Download the latest metadata from LVFS

- fwupdmgr update
Download and apply all updates for the system


