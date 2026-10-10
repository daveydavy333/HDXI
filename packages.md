## installing packages

installation of packages is straightforward.

hdx manages installation itself. the hdx system needs an internet connection, because ports download and build their sources.

- classes

every port has a class, and a dash before its name (eg. util-pfetch). current classes:

base: the system itself (init, kernel, bootloader). protected from removal.

util: small tools and utilities.

devel: build tools and libraries for development.

misc: everything else.

more will appear as the tree grows.

- install

    hdx i class-name

another example

    hdx i util-pfetch

several at once works too

    hdx i util-pfetch devel-meson

if a port has dependencies, hdx builds the missing ones first. if a port is already installed, hdx says so and does nothing. to rebuild it anyway:

    hdx i -f util-pfetch

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

- when a build fails

hdx stops, marks nothing as installed, and leaves the source in /hdxp/src/class-name-src/. cd there, fix the problem (a missing tool, an edited config.h), and re-run the build by hand or run `hdx i class-name` again.

- ports that touch /boot

base-kernel and base-limine refuse to run unless your EFI system partition is mounted at /boot, so they never write a kernel or bootloader to the wrong place. if you really want a kernel in a plain /boot directory on the root filesystem, run `HDX_BOOT_ANYWHERE=1 hdx i base-kernel`.

- where things live

    /hdxp/pkgs/        port recipes
    /hdxp/src/         downloaded and built sources
    /hdxp/installed/   one marker file per installed port
