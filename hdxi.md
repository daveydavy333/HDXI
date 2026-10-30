# HDXI

The standard for pure computing.

HDXI is an independent linux distribution built for user sovereignty, transparency, and speed.

HDXI stands for High Definition Computing Environment.

---

## architecture

HDXI rejects the bloated infrastructure of mainstream operating systems. it strips away tracking, heavy databases, and binary package managers in favor of a clean, text-based ports filesystem.

* userland: musl libc and busybox utilities. the base also ships a build toolchain (gcc, make, binutils, git, curl), because every port is built from source.
* kernel: the mainline linux LTS kernel, built from kernel.org sources by the `base-kernel` port. it is not baked into the base system, so you can configure and rebuild it yourself.
* init: situation, a small shell init and service manager (`base-situation`). it ships with the base system.
* bootloader: limine, installed by the `base-limine` port (UEFI only for now). a `base-grub` port is planned.
* directory layout: all ports, sources, and install records live transparently inside a single root directory named /hdxp.
* desktop: the dwl compositor on wlroots, the foot terminal and firefox, drawn with software rendering. they are all ports (`util-dwl`, `util-foot`, `misc-firefox`). firefox is the one exception to building from source: it is too heavy to compile here, so `misc-firefox` installs a prebuilt bundle.

---

## HDXP (high definition executable ports)

package management on HDXI is driven by HDXP, a decentralized ecosystem that treats code compilation as an open playground rather than a hidden background task.

the system is managed with a lightweight, stupidly simple shell script called hdx (high definition execution).

### port directory structure

every port lives in /hdxp/pkgs/class-name/ (eg. /hdxp/pkgs/util-pfetch/) and contains plain-text files:

* make.sh: a transparent shell script that fetches the source and builds and installs it.
* info: description and metadata (any file starting with "info" works).
* deps: a flat list of dependencies, one per line. omitted if there are none.
* .multibin: optional. lists every binary a port installs, for tracking and removal.

### installing

running `hdx i class-name` reads the port's deps file. any dependency that is missing is built first, recursively, before the port itself. a dependency counts as present if hdx installed it, or if its binary is already on the system path. circular dependencies stop with an error naming the cycle.

source is kept in /hdxp/src/class-name-src/. if a build fails, nothing is marked as installed, and the source stays right there: cd into it, edit files like config.h, and debug or re-compile by hand.

when a build succeeds, hdx writes a marker file to /hdxp/installed/class-name. that is the entire install database: one empty-ish text file per port.

### removing

running `hdx r class-name` looks in /bin, /usr/bin, and /usr/local/bin for the port's primary binary (the part after the class) and any names listed in .multibin. if the port has an uninstall.sh, hdx also runs it, to remove everything else the port installed (libraries, config files, links). hdx shows you exactly what it found and prints the uninstall script before asking for confirmation, so you always see what will run as root.

it never deletes busybox applets, and it warns you if another installed port depends on the one you are removing. if an uninstall script fails, hdx stops and removes nothing else.

ports in the `base` class (init, kernel, bootloader) are protected. removing one needs `hdx r -f`.

known limit: only ports that ship an uninstall.sh clean up beyond their binaries. a library port with no uninstall.sh can only be dropped from the install records.

---

## status

what works today, tested in qemu with UEFI firmware and on a real laptop (a lenovo ideapad slim 1):

* boot: UEFI firmware, then limine, then the HDXI kernel, then situation, then a login prompt.
* networking: dhcp through a situation service. wifi works on the laptop's atheros card through the ports `devel-libnl`, `util-wpa_supplicant` and `misc-ath10k-firmware`.
* boot checks: the EFI system partition (FAT) is checked at boot before it is mounted. the root filesystem relies on ext4's journal replay.
* desktop: dwl, foot and firefox run on the laptop and in qemu. install them with `hdx i util-dwl util-foot misc-dejavu-fonts misc-firefox`; hdx builds their dependencies first.
* ports: base-situation, base-kernel, base-limine, util-pfetch, plus the wifi and desktop ports above and the libraries they need. the full list is in the hdxp repository.

not done yet:

* BIOS (legacy) boot, and secure boot. the bootloader is built from source and unsigned.
* wifi at boot. for now wpa_supplicant is started by hand or from a login script.
* hardware graphics acceleration. rendering is in software and there is no mesa port yet.
* the laptop's built-in touchpad. it needs more kernel options. usb mice work, but dwl only sees devices that are plugged in before it starts, because there is no udev daemon.
* testing on more hardware than one laptop.

see installation.md to install, packages.md to use ports, and contribute.md to write one.
