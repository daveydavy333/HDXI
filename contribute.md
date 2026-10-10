# contributing to HDXP

contributing a port to HDXP is relatively simple, as all build scripts are simply manual installations.

## specifics

the name of a port has to be class-name, classes are mentioned in packages.md. for example, devel-meson, util-pfetch, base-limine. names are lowercase letters, digits, dashes, and underscores. hdx rejects anything else, so a name can never point outside the ports tree.

the part after the class is the port's primary binary name. hdx uses it to check whether a dependency is already present and to find the binary on removal. if your port installs a binary with a different name, list it in .multibin.

the contents of a port folder should have 2-5 files.

- make.sh
- info* (can be .md, txt, whatever.)
- deps (only if there are dependencies)
- .multibin (only if your port builds multiple binaries)
- uninstall.sh (only if your port installs more than binaries)

### what is info?
info is a document describing specifics or information about your port. it contains things like maintainer, an about section, upstream, and license. this file is up to you. `hdx info` prints any file in the port folder whose name starts with "info".

---
### example - info

pfetch is a pretty system information tool written in POSIX sh
upstream: https://github.com/Un1q32/pfetch (maintained fork of dylanaraps/pfetch)
license: MIT
ported to hdxp by: googlebutt984

---
### what is deps?
deps is a simple text file, case sensitive, simply named "deps". it contains dependencies, one name per line. hdx uses it to build missing dependencies before your port, automatically. blank lines and lines starting with # are ignored. do not include a deps file if the port has no dependencies.

hdx stops with an error if dependencies form a cycle.

---
### example - deps

devel-meson
util-fetch

---
### what is .multibin?
.multibin is another text file, case sensitive, named ".multibin". only add it when a port installs multiple binaries, or one whose name differs from the part after the class. it uses the same one-name-per-line syntax, and lets hdx find and remove every binary when a port is removed. hdx looks in /bin, /usr/bin, and /usr/local/bin.

---
### example - .multibin

bin1
bin2
bin3

---
### what is uninstall.sh?
uninstall.sh is optional, case sensitive, named "uninstall.sh". hdx already removes the binaries it can find (the primary name and everything in .multibin). use uninstall.sh for everything else your port put on the system: libraries, headers, config files, symlinks, directories. hdx prints the whole script before asking the user to confirm, then runs it as root with `sh -e`, from the port's folder. HDX_PKG and HDXP are set. it runs before the binaries are deleted, and if it fails, nothing else is removed and the port stays marked installed.

rules of thumb:

- you are already root, so do not use sudo or doas.
- only remove what your port installed. never delete a file the user may have written, like their bootloader config.
- check before deleting something shared: `cmp -s` a file against your source copy, or `readlink` a symlink, before removing it.
- never build a path from a variable that could be empty. `rm -rf "/lib/modules/$VER"` with an empty VER is a disaster, so refuse to continue if it is empty.
- keep versions in one place. the kernel port reads VER out of its own make.sh with sed, so the two files cannot disagree.
- end the script on a command that succeeds. with `sh -e`, a final `[ x ] && y` that is false counts as a failure.

---
### example - uninstall.sh

    rm -f /lib/situation.sh
    if [ "$(readlink /sbin/init)" = ../bin/situation ]; then
        rm -f /sbin/init
    fi

---
### what is make.sh?
make.sh is the main part of a port, case sensitive, named "make.sh". it is a stupidly simple shell script which automates a manual installation, eg. by cloning the source and making it. hdx runs it with `sh -e`, from the port's folder, when installing a package. any failing command stops the build, and the port is not marked installed.

hdx gives your script a few environment variables:

- HDX_SRC: where your source should go, /hdxp/src/class-name-src. use it instead of hardcoding the path.
- HDX_PKG: the port name, eg. util-pfetch.
- HDXP: the root of the ports tree, normally /hdxp.

rules of thumb:

- keep the source under $HDX_SRC, so people can debug a failed build there.
- make it safe to run again. a failed build leaves the clone behind, so guard the clone: `[ -d "$SRC/.git" ] || git clone ...`
- pin what you build. use a release tag, a commit hash, or a tarball with a sha256 checksum. a port that follows "latest" changes under people's feet.
- verify downloads: `echo "HASH  file" | sha256sum -c -`
- never ask questions. the script must run unattended.
- installing into system directories needs root. the usual pattern: `if [ "$(id -u)" -eq 0 ]; then SU=""; else SU=doas; fi` and then `$SU make install`.
- ports that write to /boot (kernels, bootloaders) should check that it is mounted: `mountpoint -q /boot || exit 1`, so nothing lands on the wrong partition.

---
### example - make.sh

    SRC="${HDX_SRC:-/hdxp/src/util-pfetch-src}"
    [ -d "$SRC/.git" ] || git clone https://github.com/Un1q32/pfetch "$SRC"
    cd "$SRC"
    if [ "$(id -u)" -eq 0 ]; then SU=""; else SU=doas; fi
    $SU install -Dm755 pfetch /usr/local/bin/pfetch

!!! this is a primitive example of a build script, and it follows the default branch. a real port should pin a tag or commit.

---

in the end, a port named util-pfetch should be..

/hdxp/pkgs/util-pfetch/

inside..

make.sh
info
? deps
? .multibin
? uninstall.sh

## testing a port

you do not need a real install to try a port. point hdx at a scratch tree:

    export HDXP=$HOME/hdxp-test
    mkdir -p $HDXP/pkgs/util-hello
    echo 'echo building hello' > $HDXP/pkgs/util-hello/make.sh
    hdx i util-hello
    hdx l

for anything that installs into system directories, test inside a qemu image or a chroot, not on your own machine.
