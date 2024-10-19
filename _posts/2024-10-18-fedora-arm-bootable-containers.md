---
layout: post
title: Bootable Containers on ARM platforms
---

Part of my work at Red Hat involves occasionally bringing up ARM platforms.

Even though I'm mainly working on display drivers these days, I've always had a particular interest of mine in those platforms. I did so countless times, with both tailor-made systems and general purpose distributions.

Nowadays though, I would obviously do it with Fedora. And doing so scratched an old itch of mine.

# The problem

ARM platforms are in a weird place, and have been for a while. Historically speaking, their primary audience were vertically integrated devices, for which the manufacturer control both the hardware and the software. A typical example would be an embedded device.

Thus, over the years, a tremendous effort went into creating tools to help create that kind of systems ([Buildroot](https://buildroot.org/), [Yocto](https://www.yoctoproject.org/), [OpenWRT](https://openwrt.org/), etc.), and they all work great. They allow to create a deploy a highly customized system for a supported platform in a matter of minutes, without any kind of modification. [Android](https://www.android.com/) and [ChromeOS](https://www.chromium.org/chromium-os/) devices also belong in that category, and so does any device where the manufacturer does not expect the user to change the software stack for something else.

On the flip side, for a decade or more, some manufacturers designed ARM platforms for a more generic usage, including desktops. Such manufacturers are [BeagleBoard](https://www.beagleboard.org/), [RaspberryPi](https://www.raspberrypi.com/), [Pine64](https://pine64.org/) or [Libre Computer](https://libre.computer/) for example. They have been pretty successful doing so, so much that users expect the vendor to offer a generic distribution for their Single-Board Computer (SBC) market.

Nowadays, the release of ARM laptops such as the MacBooks or the Qualcomm-based ThinkPads blur the lines even more. Indeed, the system decision is in the users hands now, just like x86.

Besides, the SBCs being cheap and somewhat underpowered create a user incentive to create dedicated systems depending on the device itself but also its expected usage.

This led to the creation of dedicated distributions ([Armbian](https://www.armbian.com/), [RaspberryPi OS](https://www.raspberrypi.com/software/), etc.) that allow to customize the distribution after its installation. Unfortunately, most of these distributions are Debian or Ubuntu derivatives. And I believe Fedora is a much better distribution for these kind of platforms, if only for having more up-to-date packages.

Some tools were also created to automate the customization process ([debos](https://github.com/go-debos/debos), [ubuntu-image](https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/ubuntu-image.html), etc.). Unfortunately, all these tools use a custom syntax with a restricted set of commands.

I believe that we can do better.

# Working towards a solution

If we take a step back and try to identify what the users are expecting, we have, in no particular order:

- the support of their platform of choice;
- the ability to create an image for their platform easily;
- and the ability to customize their system before the installation.

The first item is a big sticking point for upstream distributions. And I believe the root cause is a cultural mismatch. Distributions have been generally concerned about x86, and PowerPC to some extent. In these two architectures, there's a firmware flashed somewhere that will deal with the early boot and loading Linux. ARM does not. To help, ARM released a few standards mandating a similar setup. Unfortunately, these standards are optional. As such, significant part of the ecosystem does not follow them.

Thus distributions, if they want to support any ARM platform, have to deal with compiling and shipping arbitrary firmware. To make this more interesting, every platform has different bootloader location requirements. That location might depend on the usage too. There might be other, and sometimes weird, requirements too. Some platforms need an [MBR](https://en.wikipedia.org/wiki/Master_boot_record) Partition Table that nobody really wants to use anymore for example. Others need a particular file-system geometry. This is arguably pretty complex, and understandably really rubs people used to x86 the wrong way.

Thus, distributions do not really want to deal with it. The logical suite is that they come up with policies such as only supporting standard-compliant ARM boards. This is totally reasonable, but excludes many platforms out of these distributions communities. Combine this with the lack of general tools to create and distribute variants, and you're in a pretty rough spot if you use one of these platforms.

Distributions also traditionally tend towards installers. And for a good reason: it works great on a typical x86 computer. On ARM platforms though, it does not work that well. One of the main reason is still the bootloader situation: you would need to have an installer image you support, which does not scale at all. Now, it does work for some platforms such as [Asahi](https://asahilinux.org/), and that is precisely because it does not have to scale. The lineup is pretty limited, and they all come with a bootloader in a flash memory that will do the early initialization.

As we have already established, SBC-like platforms do not follow that pattern. Using an installer also breaks the usual process on those platforms where you would prepare and flash the system from your main machine. This also is not practical for manufacturers: running an installer in an assembly line takes time, and time is money. Thus, their preferred solution is to write an OS image on the storage device either by the supplier directly, or during assembly.

In parallel, I switched to [Fedora Silverblue](https://fedoraproject.org/fr/atomic-desktops/silverblue/) a bit more than a year ago. Silverblue is the atomic desktop variant of Fedora: you get reliable, atomic updates. One of the key feature of Silverblue in my eyes is that you can easily create and distribute a custom image. The community then used it to develop opinionated versions such as systems [with KDE instead of GNOME](https://fedoraproject.org/atomic-desktops/kinoite/), [for Steam Deck-like devices](https://bazzite.gg/), [targeted towards developers](https://projectbluefin.io/), and so on.

Silverblue does so by publishing bootable containers, i.e. containers with an entire, functional system instead of the smallest needed to run an application.

Using containers also enable the distributions to focus on what they care about, and let third-parties change and distribute variants easily. Containers are also created using a common syntax nowadays. They are also integrated into many workflows already, and wide range of tools is available now (registries, CI/CD, etc.).

I think this approach would suit our situation really well too. Distributions could care only about the standard-compliant platforms, while enabling third-parties to offer variants supporting others. And allowing users to further customize their system if they see fit.

# Putting it all together

If we want to use those containers on ARM though, we need a tool to take a container and put it into a storage device. Thus, it would need to deal with all the bootloader constraints those platforms have.

A plausible list of features for such a tool to be complete would be:

- Ability to run on a separate machine / architecture than the one we create a system for;
- Support for any distribution;
- Support for [GUID Partition Table (GPT)](https://en.wikipedia.org/wiki/GUID_Partition_Table) and MBR partition tables;
- Support to place bootloaders:

  - In a boot partition;
  - At a fixed offset on the device (and possibly many times);
  - As an MBR payload.

The first location is pretty easy to support, since it would be a file in the container already. We might have extra requirements such as partition flags or particular geometry requirements for the filesystem, but it should not be too hard.

The last two are harder to achieve though. Indeed, the offset the platform expect the bootloader to be at might conflict with a GPT. Some (e.g. Allwinner) can work by specifying less GPT partitions entries than standard. Some require the bootloader to be in the second block and prevent the use of GPT entirely. And some platforms (e.g. Amlogic) require an MBR payload, while GPT does not have a counterpart.

# Usability

Now that we have established the technical constraints, let's talk about usability. Ideally, we should strive to make it as easy to use as possible.

We found that the partition layout is platform-specific, and that we expect to have a container for each platform or board. This means that we can encode the partition table along with the container. Thus, by default, whatever tool we end up with should produce a bootable system, without any user intervention. And since we expect users to produce variants of these containers, the variants should work equally well.

Of course, some users will want to customize that partition table too, so we should make it easy to change the defaults.

# A possible solution

I've spent some time recently playing around with those ideas. I've created a Fedora-based container for systems compatible for the [SystemReady IR](https://www.arm.com/architecture/system-architectures/systemready-certification-program/ir) specification, and another one for the [TI SK-AM62](https://www.ti.com/tool/SK-AM62) evaluation board.

[Buildah](https://buildah.io/) allows to create containers for other architectures, meaning you can create an arm or arm64 container from an x86 machine. Thus, we have the cross-platform feature done.

The SystemReady-IR specification requires GPT. In contrast, the TI AM62 platform can boot either with a bootloader at a raw offset, or with an MBR partition table and a FAT partition. At the platform level, a pin selects the behaviour. Depending on the board design, a switch can control that pin, or the board will force its level. The TI SK-AM62 uses a switch, so either way would have been fine. Another popular board, the [BeaglePlay](https://www.beagleboard.org/boards/beagleplay), forces it to use the FAT partition, so it looks more practical.

Altogether, we already cover pretty different use cases and a decent part of our problem.

To go with these containers, I created a tool called [ocibootstrap](https://github.com/mripard/ocibootstrap) that will turn such a container into a disk image.

To carry the partition table requirements along the container, I chose to use container labels. They are essentially metadata distributed along the containers. A nice thing is that if you create a new container from a base one, the new container will keep the base container labels. This allow to keep the partition table definition even if the user makes modifications.

ocibootstrap already works well. I plan on supporting the [Amlogic A311D](https://www.amlogic.com/#Products/407/index.html), [Rockchip RK3588](http://www.rock-chips.com/a/en/products/RK35_Series/2022/0926/1660.html) and [NXP i.MX8M Plus](https://www.nxp.com/products/processors-and-microcontrollers/arm-processors/i-mx-applications-processors/i-mx-8-applications-processors/i-mx-8m-plus-arm-cortex-a53-machine-learning-vision-multimedia-and-industrial-iot:IMX8MPLUS) next. They cover more spectrum of the problem, and when they are all supported, we should support most setups. Eventually, I'd like for ocibootstrap to share code or merge with [bootc](https://github.com/containers/bootc), and especially [bootc-image-builder](https://github.com/osbuild/bootc-image-builder).

And feel free to reach out if you're interested, or for any feedback.
