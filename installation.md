# installation

installation is relatively simple, but it is manual. you partition a disk, unpack a tarball, clone the ports tree, and then build your own kernel and bootloader with hdx.

the tarball contains a base system: musl, busybox, a build toolchain, the situation init, and hdx. it does not contain the /hdxp ports tree, which you clone from github during installation (step 4). it does not contain a kernel or a bootloader either. those are ports, so you choose how to build them.

this procedure has so far been tested in qemu with UEFI firmware only. real hardware may need adjustments.

## what you need

* any linux live system (or the HDXI installation media) with: tar, a GPT-capable partitioning tool (sfdisk, sgdisk, parted, cfdisk, or gparted), mkfs.vfat, mkfs.ext4, blkid, and chroot. live systems differ, so check what yours has: `command -v sfdisk sgdisk parted cfdisk`.
* a UEFI machine. legacy BIOS is not supported by `base-limine` yet.
* secure boot turned off. the bootloader is built from source and is not signed.
* an internet connection. the ports tree is cloned from github, hdx downloads sources, and the kernel source is about 150 MB.
* patience. building the kernel takes a while.

## 1 partition the disk

use cfdisk, gparted, anything. you need a GPT disk with two partitions:

* an EFI system partition, 512 MB, formatted FAT32 with the label HDXIBOOT. this will be mounted at /boot, and it holds your kernel and bootloader.
* a root partition, ext4, using the rest of the disk.

example with sfdisk (replace /dev/sdX with your disk, this destroys its contents):

    printf 'label: gpt\nsize=512M, type=U\ntype=L\n' | sfdisk /dev/sdX

or with sgdisk:

    sgdisk --zap-all /dev/sdX
    sgdisk -n 1:0:+512M -t 1:EF00 -c 1:HDXI-ESP /dev/sdX
    sgdisk -n 2:0:0 -t 2:8300 -c 2:hdxi-root /dev/sdX

or with parted:

    parted -s /dev/sdX mklabel gpt mkpart ESP fat32 1MiB 513MiB set 1 esp on mkpart root ext4 513MiB 100%

then make the kernel re-read the table and create the filesystems:

    partprobe /dev/sdX
    mkfs.vfat -F 32 -n HDXIBOOT /dev/sdX1
    mkfs.ext4 /dev/sdX2

check with `lsblk -f /dev/sdX` before going on: a vfat partition labelled HDXIBOOT, then an ext4 one. if the old partitions still show up, or partprobe says some partitions "are in use", the kernel is still using the old table. do not format anything yet: formatting would write to the old partition locations, not the new ones. some live systems turn on swap or mount internal partitions by themselves, so release them first (`swapoff -a`, then `umount /dev/sdX*`) and run partprobe again. if that does not work, reboot the live system. the new table is already on the disk, so after the reboot you only need to check it with lsblk, not repartition.

on nvme disks the partitions are named like /dev/nvme0n1p1 and /dev/nvme0n1p2, and on some others like /dev/mmcblk0p1.

## 2 unpack the base system

mount the root partition and extract the tarball onto it. the tarball includes an empty /boot directory, so mount the ESP there afterwards:

    mount /dev/sdX2 /mnt
    tar -xzf hdxi-VERSION-x86_64.tar.gz -C /mnt --numeric-owner
    mount /dev/sdX1 /mnt/boot

extract as root, and keep --numeric-owner, so file ownership is preserved. if your tar rejects that option (busybox tar may), leave it out: the tarball already stores numeric ownership.

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

## 4 get the ports tree

the tarball does not contain /hdxp. clone the ports tree into it from inside the chroot (the dns server you wrote in step 3 is what lets git reach github), then create the two folders hdx keeps its local state in. sources and install records are not part of the repository:

    git clone https://github.com/daveydavy333/hdxp /hdxp
    mkdir -p /hdxp/src /hdxp/installed
    cp /usr/share/hdxi/installed/* /hdxp/installed/

the last command restores the install records for the ports that ship in the base system (situation and pfetch), so `hdx l` lists them.

if git says /hdxp already exists and is not empty, you are using an older tarball that still shipped /hdxp. in that case you can skip this step.

## 5 build the kernel

    hdx i base-kernel

this downloads the linux source, verifies its checksum, configures it, and builds it. it installs /boot/vmlinuz and a versioned copy, plus the modules in /lib/modules. the config is minimal and builds in the root filesystem driver, virtio and framebuffer console support, so no initramfs is needed and you get a text console on the screen.

if you want to change the kernel configuration, see the info file: `hdx info base-kernel`.

## 6 install the bootloader

! the EFI system partition must be mounted at /boot first (step 2).

    hdx i base-limine

this builds limine for UEFI and puts it at /boot/EFI/BOOT/BOOTX64.EFI.

## 7 write the bootloader config

limine needs to know which kernel to start and where the root partition is. the root partition is identified by its PARTUUID, a 36 character id. do not type it by hand. one wrong character gives a kernel panic that says almost nothing: `Cannot open root device ""` with an empty name, and an empty list of partitions. let the shell fill it in.

from the live system (outside the chroot, with the root partition mounted at /mnt and the ESP at /mnt/boot):

    PARTUUID=$(blkid -s PARTUUID -o value /dev/sdX2)
    echo "$PARTUUID"

if that prints nothing, `sgdisk -i 2 /dev/sdX` shows the same id as "Partition unique GUID", and you can set PARTUUID= to it by copy and paste. then write the config:

    printf 'timeout: 3\n\n/HDXI\n    protocol: linux\n    path: boot():/vmlinuz\n    cmdline: root=PARTUUID=%s rootwait rw init=/sbin/init console=tty0\n' "$PARTUUID" > /mnt/boot/limine.conf
    cat /mnt/boot/limine.conf

the file should read:

    timeout: 3

    /HDXI
        protocol: linux
        path: boot():/vmlinuz
        cmdline: root=PARTUUID=YOUR-PARTUUID rootwait rw init=/sbin/init console=tty0

if the boot panics with an empty root device name, the root= option did not reach the kernel: check the cmdline line for typos. at limine's menu you can press `e` on the entry to see and edit it for one boot.

`rootwait` makes the kernel wait for the root disk to appear. real hardware (usb sticks, emmc, some sata controllers) brings its disks up after the kernel starts looking for root, and without rootwait the boot panics. there is no initramfs to wait for you. virtual machines do not need it, but it does no harm.

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
