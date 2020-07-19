# Chromium OS Archero Developer Guide

This guide will walk you through the processing of replacing Chromium OS ARC++ with Anbox customized by FydeOS. Since Archero is still in an early stage of development, any contribution is welcome. If you have questions with this guide, just [file an issue](https://github.com/FydeOS/chromium_os-archero-developer-guide/issues/new).

These related projects are the followings:

- [anbox_fydeos](https://github.com/FydeOS/anbox_fydeos): fork of [anbox/anbox](https://github.com/anbox/anbox).
- [anbox_platform_manifests](https://github.com/FydeOS/anbox_platform_manifests): fork of [anbox/platform_manifests](https://github.com/anbox/platform_manifests), Android manifest for building [AOSP](https://source.android.com/).
- [chromiumos_anbox_manifest](https://github.com/FydeOS/chromiumos_anbox_manifest): fork of [chromiumos/manifest](https://chromium.googlesource.com/chromiumos/manifest/), Chromium OS manifest for building Chromium OS.
- [overlay-amd64-anbox-generic](https://github.com/FydeOS/overlay-amd64-anbox-generic): board extended on `amd64-generic` with Anbox's dependencies.
- [platform_frameworks_base](https://github.com/FydeOS/platform_frameworks_base): fork of [anbox/plaform_framework_base](https://github.com/anbox/platform_frameworks_base), base components of AOSP framework.
- [project-archero](https://github.com/FydeOS/project-archero): solution to replace Google's ARC++ based on Anbox.

## 0. Hardware requirements

We will use a server as an example because we need to download and build lots of code.

- OS: Ubuntu 18.04
- CPU: 4 cores
- Memory: 32G
- Disk: 500G

## 1. Build Anbox

Firstly, check whether your JDK version is 1.8.x, if not, install JDK version 1.8:

```bash
# Check JDK version
$ java -version
openjdk version "1.8.0_252"
OpenJDK Runtime Environment (build 1.8.0_252-8u252-b09-1~18.04-b09)
OpenJDK 64-Bit Server VM (build 25.252-b09, mixed mode)

# If the version is already 1.8.x, just skip installing and switching JDK
$ sudo apt update
$ sudo apt-get install openjdk-8-jdk openjdk-8-jre

# Select version 1.8
$ sudo update-alternatives --config java
```

Next, ready to download and build Anbox:

```bash
$ mkdir $HOME/anbox-work
$ cd $HOME/anbox-work
$ repo init -u https://github.com/FydeOS/anbox_platform_manifests.git -b anbox
$ repo sync -j4
$ . build/envsetup.sh
$ lunch anbox_x86_64-userdebug
$ make -j4
```

Start to build Android image:

```bash
# These two packages are required
$ sudo apt-get install -y attr squashfs-tools
$ cd $HOME/anbox-work/vendor/anbox
$ scripts/create-package.sh \
    $PWD/../../out/target/product/x86_64/ramdisk.img \
    $PWD/../../out/target/product/x86_64/system.img
```

When the script is done, there should be a file named `android.img` in the current directory.

## 2. Build Chromium OS

Download Chromium OS source code:

```bash
$ mkdir $HOME/chromiumos
$ cd $HOME/chromiumos
$ repo init -u https://github.com/FydeOS/chromiumos_anbox_manifest.git --repo-url https://chromium.googlesource.com/external/repo.git -b release-R80-12739.B
$ repo sync -j4
```

Replace the Android image of Chromium OS with `android.img` which is generated in the last step:

```bash
$ cd $HOME
$ cp anbox-work/vendor/anbox/android.img chromiumos/src/overlays/project-archero/app-emulation/anbox/files/android_amd64.img
$ cd chromiumos
$ repo sync -j4
```

That's all for Chromium OS.

## 3. Build Chromium

The works for building Chromium are more complicated and time-consuming (~3 hours or longer depending on the network and hardware).

Firstly, get Chromium source code and its dependencies:

```bash
$ mkdir ~/chromium && cd ~/chromium
$ fetch --nohooks chromium
$ cd src
$ git checkout tags/80.0.3987.165
$ ./build/install-build-deps.sh
$ gclient runhooks
```

Next, edit `.gclient` located in `chromium` directory and add a line:

```
target_os = ['chromeos']
```

Now `cros_sdk` is ready to build Chromium:

```bash
$ cd $HOME
$ ./chromiumos/chromite/bin/cros_sdk --nouse-image --chrome_root ./chromium
```

After `cros_sdk` is executed, edit `../third_party/chromiumos-overlay/eclass/cros-board.eclass` to insert `amd64-anbox-generic` into `ALL_BOARDS` configuration, like:

```
ALL_BOARDS=(
    amd64-anbox-generic
    acorn
    amd64-corei7
    ...
)
```

Finally, start to build Chromium and cross your fingers:

```bash
(sdk)$ export BOARD=amd64-anbox-generic
(sdk)$ setup_board --board=${BOARD}
(sdk)$ CHROME_ORIGIN=LOCAL_SOURCE ./build_packages --board=${BOARD} --nowithautotest
(sdk)$ ./build_image --board=${BOARD} --noenable_rootfs_verification --adjust_part="STATE:+4G" test
# Start VM
(sdk)$ cros_vm --start --qemu-args "-vnc :1 -machine pc-i440fx-2.3" --image-path /mnt/host/source/src/build/images/amd64-anbox-generic/latest/chromiumos_test_image.bin
```

## 4. Viewing VM in VNC

You can download [VNC Viewer](https://www.realvnc.com/en/connect/download/viewer/) on your local dev machine and access **127.0.0.1:5901** to view VM. Note that "Options > Picture quality" needs to be set to **high** for this to work.

After VM is connected successfully, you will see the welcome screen of Chromium OS. Connect to a network and log-in to the system. Open the shell by pressing `Ctrl` + `Alt` + `T` in Chromium, and launch Anbox with commands (admin password is `test0000`):

```bash
crosh> shell
chronos@localhost / $ sudo XDG_RUNTIME_DIR=/run/chrome anbox-session
```

In several seconds, Anbox will start.

![anbox-window](https://raw.githubusercontent.com/FydeOS/chromium_os-archero-developer-guide/master/screenshots/anbox-window.png)
