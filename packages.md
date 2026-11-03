## packages and hdx

- what is hdx

hdx is the package manager of HDXI. it is a set of shell scripts, and it works the way a BSD ports system does: there are no prebuilt packages. every program beyond the base tarball is a port, a small recipe that tells hdx how to download, build and install one piece of software from source.

when you install something, hdx finds the port, downloads the upstream source, checks it against a pinned SHA256, builds it on your own machine and installs it, by default under /usr/local. nothing is hidden: the recipes and hdx itself are plain scripts you can read.

the hdx system needs an internet connection, because ports download and build their sources.

- what is a port

a port is a folder in the ports tree, named class-name (eg. util-pfetch). it holds:

    make.sh        the recipe: download, verify, build, install
    deps           ports it needs, one per line (optional)
    info           description, upstream, license
    .multibin      extra binaries it installs, one per line (optional)
    uninstall.sh   removes libraries, config and links too (optional)
    warning        a note shown before the build, used by skunkports (optional)

- classes

every port has a class, and a dash before its name (eg. util-pfetch). current classes:

base: the system itself (init, kernel, bootloader). protected from removal.

util: small tools and utilities.

devel: build tools and libraries for development.

misc: everything else.

more will appear as the tree grows. "skp" is not a class: it is reserved for skunkports, see below.

- the ports tree

the ports tree is called hdxp. it lives at /hdxp as an ordinary git clone. update the list of ports with

    hdx u

if /hdxp does not exist yet, hdx u clones it. to update and install in one go:

    hdx u i util-pfetch

or the shorthand

    hdx ui util-pfetch

- install

    hdx i class-name

another example

    hdx i util-pfetch

several at once works too

    hdx i util-pfetch devel-meson

before building anything, hdx works out what the install needs and shows you:

    hdx: the following ports will be installed:
      devel-meson
      util-pfetch
    hdx: 2 port(s) in total
    hdx: already satisfied: devel-python3
    proceed? [y/N]

the list is in build order: dependencies come first. ports that are already installed are not on the list, and are shown as satisfied instead. if a port is already installed and you named it, hdx says so and does nothing. nothing is built until you answer y. to skip the question, for scripts and the like:

    hdx i -y util-pfetch

or set HDX_YES=1. without a terminal and without -y, hdx answers no.

to rebuild a port that is already installed:

    hdx i -f util-pfetch

the list marks it as a rebuild.

- list and inspect

    hdx l

lists installed ports.

    hdx info util-pfetch

shows a port's info file.

- remove

    hdx r util-pfetch

hdx shows the binaries it found, and prints the port's uninstall script if it has one, then asks before doing anything. ports can ship an uninstall.sh that removes libraries, config files, and links as well. if that script fails, nothing else is removed. ports in the base class are protected:

    hdx r -f base-limine

forces it. think twice: removing your init, kernel, or bootloader can leave the machine unbootable. your own files are left alone: bootloader config (/boot/limine.conf), services in /etc/sv, and scripts you added under /etc/rc.d.

- skunkports

skunkports are the extremely heavy or experimental ports: things that take forever to build, pull in huge dependency trees, or barely fit a small system (graphics stacks, GPU drivers, desktop environments). they are unsupported and may break, so they live in their own tree, hdxp-skp, instead of the main one. hdx is the same, the name just gets a prefix:

    hdx skp
    hdx i skp/devel-mesa

hdx skp clones the skunkport tree to /hdxp/skp the first time (and updates it after that). once it is there, hdx u keeps it up to date along with the main tree.

a skunkport can ship a warning file. hdx prints it in the install list, before you answer the question, so you see why it is a skunkport (build time, no GPU driver, replaces files from a normal port) before anything starts. a skunkport may need normal ports, and hdx builds those first. a normal port never needs a skunkport.

skunkports are listed, inspected and removed with the same prefix:

    hdx l
    hdx info skp/devel-mesa
    hdx r skp/devel-mesa

- when a build fails

hdx stops, marks nothing as installed, and leaves the source in /hdxp/src/class-name-src/ (/hdxp/src/skp-class-name-src/ for a skunkport). cd there, fix the problem (a missing tool, an edited config.h), and re-run the build by hand or run `hdx i class-name` again.

- ports that touch /boot

base-kernel and base-limine refuse to run unless your EFI system partition is mounted at /boot, so they never write a kernel or bootloader to the wrong place. if you really want a kernel in a plain /boot directory on the root filesystem, run `HDX_BOOT_ANYWHERE=1 hdx i base-kernel`.

- amd graphics

base-kernel builds the amdgpu driver in for AMD Stoney Ridge graphics (the a6-9220e and similar) when its firmware is already installed. install the firmware first, then build the kernel:

    hdx i misc-amdgpu-firmware
    hdx i -f base-kernel

without the firmware base-kernel builds without amdgpu, and the screen keeps working through simpledrm.

- where things live

    /hdxp/pkgs/        port recipes
    /hdxp/skp/         skunkport recipes (after hdx skp)
    /hdxp/src/         downloaded and built sources
    /hdxp/installed/   one marker file per installed port (skp-class-name for skunkports)
