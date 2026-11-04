

<img width="600" height="240" alt="Untitled45_20261007220815" src="https://github.com/user-attachments/assets/e61c83f2-2439-4eff-9a29-3059aa098d5b" />


The standard for pure computing.

HDXI is an independent linux distribution built for user sovereignty, transparency, and speed.

HDXI stands for High Definition Computing Environment.

---

## architecture

HDXI rejects the bloated infrastructure of mainstream operating systems. it strips away tracking, heavy databases, and binary package managers in favor of a clean, text-based ports filesystem.

* userland: musl libc and busybox utilities. the base also ships a build toolchain (gcc, make, binutils, git, curl), because every port is built from source.
* kernel: the mainline linux LTS kernel, built from kernel.org sources by the `base-kernel` port. it is not baked into the base system, so you can configure and rebuild it yourself.
* init: the base tarball ships no init, you pick one after unpacking it: runit (`base-runit`) or situation, a small shell init and service manager (`base-situation`). both use the shared boot scripts from `base-bootscripts`.
* bootloader: limine, installed by the `base-limine` port (UEFI only for now). a `base-grub` port is planned.
* directory layout: all ports, sources, and install records live transparently inside a single root directory named /hdxp.
* desktop: there is no fixed desktop. you build the one you want out of ports, and the port list is still small, so right now the options are the dwl compositor on wlroots, the foot terminal and firefox, drawn with software rendering by default (hardware rendering is optional, see skunkports below). they are all ports (`util-dwl`, `util-foot`, `misc-firefox`), and more ports are welcome (see contribute.md). firefox is the one exception to building from source: it is too heavy to compile here, so `misc-firefox` installs a prebuilt bundle.

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

before building anything, hdx prints the plan: every port it is about to install, dependencies first, with (rebuild) and (skunkport) tags and any warning a port carries. it then asks `proceed? [y/N]` once. `-y` (or HDX_YES=1) skips the question, and `-f` rebuilds a port that is already installed. `hdx u` updates the ports tree, and `hdx u i port` or `hdx ui` updates and then installs.

source is kept in /hdxp/src/class-name-src/. if a build fails, nothing is marked as installed, and the source stays right there: cd into it, edit files like config.h, and debug or re-compile by hand.

when a build succeeds, hdx writes a marker file to /hdxp/installed/class-name. that is the entire install database: one empty-ish text file per port.

### removing

running `hdx r class-name` looks in /bin, /usr/bin, and /usr/local/bin for the port's primary binary (the part after the class) and any names listed in .multibin. if the port has an uninstall.sh, hdx also runs it, to remove everything else the port installed (libraries, config files, links). hdx shows you exactly what it found and prints the uninstall script before asking for confirmation, so you always see what will run as root.

it never deletes busybox applets, and it warns you if another installed port depends on the one you are removing. if an uninstall script fails, hdx stops and removes nothing else.

ports in the `base` class (init, kernel, bootloader) are protected. removing one needs `hdx r -f`.

known limit: only ports that ship an uninstall.sh clean up beyond their binaries. a library port with no uninstall.sh can only be dropped from the install records.

### skunkports

skunkports (skp) are a separate, unsupported tier for ports that are very heavy to build or experimental, the kind of thing that takes hours to compile or arguably shouldn't be ported at all. they live in their own repository, hdxp-skp, which `hdx skp` clones to /hdxp/skp, and they install with the skp/ prefix: `hdx i skp/devel-mesa`.

* a skunkport may depend on normal ports and on other skunkports. normal ports never depend on skunkports.
* a skunkport can ship a `warning` file, which hdx prints in the install plan.
* once /hdxp/skp exists, `hdx u` updates it along with the main tree.

today there are two: `devel-mesa` (mesa 26.1.6 with softpipe and virgl, no llvm) and `devel-wlroots-gles` (wlroots with the GLES2 renderer; it replaces the files of `devel-wlroots`, and `hdx i -f devel-wlroots` goes back).

---

## status

what works today, tested in qemu with UEFI firmware and on two real laptops (a lenovo ideapad slim 1, and an hp elitebook with nvme storage and an intel wifi card):

* boot: UEFI firmware, then limine, then the HDXI kernel, then your init (runit or situation), then a login prompt.
* dual boot: HDXI can share an EFI system partition with another OS. on the elitebook, limine shows a menu with HDXI and an existing arch linux install (chainloaded), and both boot.
* storage: root on emmc (ideapad), nvme (elitebook) and virtio (qemu). the drivers are built into the kernel, so there is no initramfs.
* networking: dhcp through the net service, which works under both inits. wifi works on both laptops through the ports `devel-libnl` and `util-wpa_supplicant`, plus a firmware port for the card: `misc-ath10k-firmware` for the ideapad's atheros, `misc-iwlwifi-firmware` for the elitebook's intel ax201. the firmware is built into the kernel, so install the firmware port before `base-kernel`.
* boot checks: the EFI system partition (FAT) is checked at boot before it is mounted. the root filesystem relies on ext4's journal replay.
* desktop: dwl, foot and firefox run on both laptops and in qemu. install them with `hdx i util-dwl util-foot misc-dejavu-fonts misc-firefox`; hdx builds their dependencies first.
* graphics acceleration (experimental): the ideapad's amd gpu (stoney ridge) uses the amdgpu driver, which is built into the kernel when the `misc-amdgpu-firmware` port is installed before `base-kernel`. the elitebook's intel gpu (tiger lake) uses i915, which is always built in; install `misc-i915-firmware` before `base-kernel` too, or the gpu runs without runtime power management. on either laptop, install `skp/devel-wlroots-gles` (it builds against `skp/devel-mesa`) and start dwl with `LD_LIBRARY_PATH=/usr/lib:/lib WLR_RENDERER=gles2`, which loads the mesa libraries that come with `misc-firefox`. on both laptops it is smoother than software rendering. in qemu software rendering is faster, so it stays the default.
* ports: base-runit, base-situation, base-kernel, base-limine, util-pfetch, plus the wifi and desktop ports above and the libraries they need. the full list is in the hdxp repository.

not done yet:

* BIOS (legacy) boot, and secure boot. the bootloader is built from source and unsigned.
* wifi at boot. for now wpa_supplicant is started by hand or from a login script.
* graphics acceleration on other gpus (only the ideapad's amd gpu and the elitebook's intel gpu are tested), and it still needs the environment variables above set by hand.
* the built-in touchpads of both laptops. they need more kernel options. usb mice work, but dwl only sees devices that are plugged in before it starts, because there is no udev daemon.
* testing on more hardware than two laptops.

see installation.md to install, packages.md to use ports, and contribute.md to write one.
