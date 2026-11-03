# contributing to HDXP

contributing a port to HDXP is relatively simple, as all build scripts are simply manual installations.

## specifics

the name of a port has to be class-name, classes are mentioned in packages.md. for example, devel-meson, util-pfetch, base-limine. names are lowercase letters, digits, dashes, and underscores. hdx rejects anything else, so a name can never point outside the ports tree. "skp" is reserved for skunkports and cannot be a class.

the part after the class is the port's primary binary name. hdx uses it to check whether a dependency is already present and to find the binary on removal. if your port installs a binary with a different name, list it in .multibin.

the contents of a port folder should have 2-6 files.

- make.sh
- info* (can be .md, txt, whatever.)
- deps (only if there are dependencies)
- .multibin (only if your port builds multiple binaries)
- uninstall.sh (only if your port installs more than binaries)
- warning (only for skunkports, or a port that replaces files of another port)

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

hdx stops with an error if dependencies form a cycle. before building, hdx shows the user the whole list of ports it is about to build, dependencies first, and asks once. so list real dependencies only: everything in deps is something the user will see and have to build.

a normal port must never list a skunkport. a skunkport may list normal ports, and other skunkports in the form skp/class-name.

---
### example - deps

devel-meson
util-fetch

---
### what is .multibin?
.multibin is another text file, case sensitive, named ".multibin". only add it when a port installs multiple binaries, or one whose name differs from the part after the class. it uses the same one-name-per-line syntax, and lets hdx find and remove every binary when a port is removed. hdx looks in /bin, /usr/bin, and /usr/local/bin. hdx also accepts the names .multbin and _multibin, but .multibin is the one to use.

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
### what is warning?
warning is optional, case sensitive, named "warning". it is plain text. hdx prints it inside the install list, before the user answers the question, so it is the place to say what is unusual about your port: it takes hours to build, it needs a lot of memory, it has no GPU driver, it replaces files that another port installed. skunkports should always have one. keep it short and honest.

---
### example - warning

SKUNKPORT: experimental. takes a long build and renders in software only.
unsupported; may break.

---
### what is make.sh?
make.sh is the main part of a port, case sensitive, named "make.sh". it is a stupidly simple shell script which automates a manual installation, eg. by cloning the source and making it. hdx runs it with `sh -e`, from the port's folder, when installing a package. any failing command stops the build, and the port is not marked installed.

hdx gives your script a few environment variables:

- HDX_SRC: where your source should go, /hdxp/src/class-name-src. use it instead of hardcoding the path. for a skunkport it is /hdxp/src/skp-class-name-src.
- HDX_PKG: the port name, eg. util-pfetch. for a skunkport it is skp-class-name.
- HDXP: the root of the ports tree, normally /hdxp.

rules of thumb:

- keep the source under $HDX_SRC, so people can debug a failed build there.
- make it safe to run again. a failed build leaves the clone behind, so guard the clone: `[ -d "$SRC/.git" ] || git clone ...`
- pin what you build. use a release tag, a commit hash, or a tarball with a sha256 checksum. a port that follows "latest" changes under people's feet.
- verify downloads: `echo "HASH  file" | sha256sum -c -`
- never ask questions. the script must run unattended. the user has already been asked once, before the build started.
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
? warning

## skunkports

skunkports are for ports that are extremely heavy or experimental: huge builds, huge dependency trees, software that barely fits a small musl system. the idea is that they get a home without being promised as supported. they live in their own repo, hdxp-skp, cloned to /hdxp/skp by `hdx skp`, and the user installs them with a prefix:

    hdx i skp/devel-mesa

a skunkport is written exactly like a normal port, with a few differences:

- the folder is class-name inside the hdxp-skp repo, with the same files. the class and name rules are the same.
- it should have a warning file.
- it may depend on normal ports, and on other skunkports (skp/class-name). a normal port never depends on a skunkport.
- install under /usr/local, like normal ports. if you replace files that a normal port installs, say so in warning and say how to go back.
- hdx names it skp-class-name internally: HDX_PKG is skp-class-name, the install marker is /hdxp/installed/skp-class-name and the source goes in /hdxp/src/skp-class-name-src.
- if your port is mostly a prebuilt bundle or something that cannot be built from source, say so in warning too.

if you are not sure whether a port is a normal port or a skunkport, ask yourself: would a normal user be surprised by the build time, the size, or the way it can break the system? if yes, it is a skunkport.

## testing a port

you do not need a real install to try a port. point hdx at a scratch tree:

    export HDXP=$HOME/hdxp-test
    mkdir -p $HDXP/pkgs/util-hello
    echo 'echo building hello' > $HDXP/pkgs/util-hello/make.sh
    hdx i util-hello
    hdx l

hdx asks proceed? [y/N] before it builds. in scripts use `hdx i -y util-hello` or HDX_YES=1.

to try a skunkport the same way, point HDX_SKP at a scratch folder with a .git folder in it (hdx treats that as the cloned tree):

    export HDX_SKP=$HOME/skp-test
    mkdir -p $HDX_SKP/.git $HDX_SKP/devel-hello
    echo 'echo building skunk hello' > $HDX_SKP/devel-hello/make.sh
    hdx i skp/devel-hello

for anything that installs into system directories, test inside a qemu image or a chroot, not on your own machine.
