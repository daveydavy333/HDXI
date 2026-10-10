# installation

installation is relatively simple, but it is manual. you partition a disk, unpack a tarball, and then build your own kernel and bootloader with hdx.

the tarball contains a base system: musl, busybox, a build toolchain, the situation init, hdx, and the /hdxp ports tree. it does not contain a kernel or a bootloader. those are ports, so you choose how to build them.

this procedure has so far been tested in qemu with UEFI firmware only. real hardware may need adjustments.

## what you need

* any linux live system (or the HDXI installation media) with: tar, a partitioning tool (cfdisk, gparted, sfdisk), mkfs.vfat, mkfs.ext4, and chroot.
* a UEFI machine. legacy BIOS is not supported by `base-limine` yet.
* secure boot turned off. the bootloader is built from source and is not signed.
* an internet connection. hdx downloads sources, and the kernel source is about 150 MB.
* patience. building the kernel takes a while.

## 1 partition the disk

use cfdisk, gparted, anything. you need a GPT disk with two partitions:

* an EFI system partition, 512 MB, formatted FAT32 with the label HDXIBOOT. this will be mounted at /boot, and it holds your kernel and bootloader.
* a root partition, ext4, using the rest of the disk.

example (replace /dev/sdX with your disk, this destroys its contents):

    printf 'label: gpt\nsize=512M, type=U\ntype=L\n' | sfdisk /dev/sdX
    mkfs.vfat -F 32 -n HDXIBOOT /dev/sdX1
    mkfs.ext4 /dev/sdX2

on nvme disks the partitions are named like /dev/nvme0n1p1 and /dev/nvme0n1p2, and on some others like /dev/mmcblk0p1.

## 2 unpack the base system

mount the root partition and extract the tarball onto it. the tarball includes an empty /boot directory, so mount the ESP there afterwards:

    mount /dev/sdX2 /mnt
    tar -xzf hdxi-VERSION-x86_64.tar.gz -C /mnt --numeric-owner
    mount /dev/sdX1 /mnt/boot

extract as root, and keep --numeric-owner, so file ownership is preserved.

## 3 configure the system

enter the new system. you need /proc and /dev inside it, and a dns server for downloads:

    mount -t proc proc /mnt/proc
    mount --rbind /dev /mnt/dev
    mount --make-rslave /mnt/dev
    echo "nameserver 1.1.1.1" > /mnt/etc/resolv.conf
    chroot /mnt /usr/bin/env -i HOME=/root TERM="$TERM" \
        PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin /bin/sh -l

then, inside:

    passwd root
    echo myhostname > /etc/hostname
    echo "LABEL=HDXIBOOT  /boot  vfat  rw,noatime  0 0" > /etc/fstab

other users, passwords, and so on can be set up with adduser, passwd, and similar.

at boot, situation does not read /etc/fstab by itself. the base ships a small boot script (/etc/rc.d/boot/05-mount.sh) that mounts everything in fstab, which is how /boot comes up.

the /etc/resolv.conf you just wrote is only for the install. at boot, the dhcp service (net) rewrites it.

## 4 build the kernel

    hdx i base-kernel

this downloads the linux source, verifies its checksum, configures it, and builds it. it installs /boot/vmlinuz and a versioned copy, plus the modules in /lib/modules. the config is minimal and builds in the root filesystem driver, virtio and framebuffer console support, so no initramfs is needed and you get a text console on the screen.

if you want to change the kernel configuration, see the info file: `hdx info base-kernel`.

## 5 install the bootloader

! the EFI system partition must be mounted at /boot first (step 2).

    hdx i base-limine

this builds limine for UEFI and puts it at /boot/EFI/BOOT/BOOTX64.EFI.

## 6 write the bootloader config

limine needs to know which kernel to start and where the root partition is. find your root partition's PARTUUID from the live system (outside the chroot):

    blkid /dev/sdX2

or `lsblk -o NAME,PARTUUID`. then, inside the chroot, create /boot/limine.conf:

    timeout: 3

    /HDXI
        protocol: linux
        path: boot():/vmlinuz
        cmdline: root=PARTUUID=YOUR-PARTUUID-HERE rw init=/sbin/init console=tty0

about the console= options: `console=tty0` sends kernel messages to the screen. `console=ttyS0` sends them to a serial port, which is handy in a virtual machine. you can give both, and the last one listed becomes the main console. either way, you get a login prompt on the screen, because the init starts a getty on tty1.

before you reboot, check that everything is where limine expects it:

    ls /boot                 # limine.conf, vmlinuz, EFI
    ls /boot/EFI/BOOT        # BOOTX64.EFI
    cat /boot/limine.conf    # the PARTUUID must match: blkid /dev/sdX2
    hdx l                    # base-kernel and base-limine should be listed

## installation should be finished at this point.

leave the chroot and unmount before rebooting:

    exit
    umount -R /mnt

reboot, and enjoy. after logging in, `hdx l` lists what is installed, `sctl list` shows services, and `hdx i class-name` installs anything else from the ports tree (see packages.md).
