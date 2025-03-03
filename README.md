# Qemu-Simulation-CXL
If you are exposed to CXL-related tests in actual work, but do not have actual CXL2.0 equipment, then we can use QEMU to simulate the construction of CXL2.0 equipment. The following is the construction process, I hope it will be helpful to your work.
QEMU currently can emulate the following CXL 2.0 compliant CXL system components (Qemu CXL doc):

CXL Host Bridge (CXL HB): equivalent to PCIe host bridge.
CXL Root Ports (CXL RP): serves the same purpose as a PCIe Root Port. There are a number of CXL specific Designated Vendor Specific Extended Capabilities (DVSEC) in PCIe Configuration Space and associated component register access via PCI bars.
CXL Switch: has a similar architecture to those in PCIe, with a single upstream port, internal PCI bus and multiple downstream ports.
CXL Type 3 memory devices as memory expansion: the device can act as a system RAM or Dax device. Currently, volatile and non-volatile memory emulation has been merged to mainstream. CXL 3.0 introduces a new CXL memory device that implements dynamic capacity-DCD (dynamic capacity device). The support of DCD emulation in QEMU has been posted to the mailing list and will be merged soon.
CXL Fixed Memory Windows (CFMW): A CFMW consists of a particular range of Host Physical Address space which is routed to particular CXL Host Bridges. At time of generic software initialization it will have a particularly interleaving configuration and associated Quality of Service Throttling Group (QTG). This information is available to system software, when making decisions about how to configure interleave across available CXL memory devices. It is provide as CFMW Structures (CFMWS) in the CXL Early Discovery Table, an ACPI table.
Setup CXL Testing/Development environment with QEMU Emulation
To test a CXL device with QEMU emulation, we need to have the following prerequisites:

A QEMU emulator either compiled from source code or preinstalled with CXL emulation support;
A Kernel image with CXL support (compiled in or as modules);
A file system that serves as the root fs for booting the guest VM.
Install prerequisite packages
My Ubuntu:

root@root1-virtual-machine:~# lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 20.04.6 LTS
Release:        20.04
Codename:       focal

## Dependency packages
```
gcc version： 9.4.0
ninja version： 1.10.0 
meson version： 0.53.2
python verson： python3.8.10
```
Building QEMU and the Linux kernel rely on some preinstalled packages.

sudo apt-get install libglib2.0-dev libgcrypt20-dev zlib1g-dev \
    autoconf automake libtool bison flex libpixman-1-dev  zstd\
    make ninja-build libncurses-dev libelf-dev libssl-dev debootstrap \
    libcap-ng-dev libattr1-dev libslirp-dev libslirp0
Just in case, install more.

sudo apt-get install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc flex libelf-dev bison
Build QEMU from source code
It is recommended to build the QEMU from source code for two reason:

The pre-compiled binary can be old and lack the latest features that are supported by QEMU;
Building QEMU from source code allows us to customize QEMU based on our needs, including development debugging, applying specific patches to test some un-merged features, or modifying QEMU to test some ideas or fixes, etc.
Steps to build QEMU emulator:
We can download QEMU source code from different sources, for example:

Mainstream QEMU: https://github.com/qemu/qemu
QEMU CXL Maintainer Jonathan's local tree for To-be-Merged patches: https://gitlab.com/jic23/qemu
Fan Ni's local github tree for Latest DCD emulation: https://github.com/moking/qemu/tree/dcd-v6
Below we will use DCD emulation setup as an example.

Step 1: download QEMU source code

git clone https://github.com/moking/qemu.git

Step 2: configure QEMU

For example, configure QEMU with x86_64 CPU architecture and debug support:

root@root1-virtual-machine:~#./configure --target-list=x86_64-softmmu --enable-debug
Step 3: Compile QEMU

make -j$(nproc)
or

make -j32 
or

make -j16

make install
If compile succeed, a new QEMU binary will be generated under build directory or in /usr/bin path:

root@root1-virtual-machine:~# qemu-
qemu-edid            qemu-ga              qemu-img             qemu-io              qemu-nbd             qemu-pr-helper       qemu-storage-daemon  qemu-system-x86_64

Build Kernel with CXL support enabled
You have two options here: Note: build cxl drivers as modules and load/unload them on demand._ Note: build CXL drivers directly into the kernel, making them always available without the need for manual loading or unloading.

Step 1: download Linux Kernel source code

Linux source code can be downloaded from different sources:

https://git.kernel.org/
CXL related development repository: https://git.kernel.org/pub/scm/linux/kernel/git/cxl/cxl.git/?h=fixes
Kernel with DCD drivers:https://github.com/weiny2/linux-kernel/tree/dcd-2024-03-24
Below we will use DCD kernel code as an example.
code path: https://github.com/weiny2/linux-kernel/tree/dcd-2024-03-24
git clone https://github.com/weiny2/linux-kernel.git
Step 2: configure kernel

After we downloaded the source code, we need to configure the kernel features we want to pick up in the following compile step.

make menuconfig
or,

make kconfig
After the kernel is configured, a .config file will be generated under the root directory of linux kernel source code.

Note: You can use my .config directly, in case you meet any problems while execute the command below to bootup qemu/kernel With my config, all CXL are built to kernel bzImage, there is no need to load modules after QEMU kernel is loaded.

put .config to folder below:
kz@kz-HP-EliteBook:~/cxl/linux-kernel-dcd-2024-03-24$ ls .config -lh
-rw-rw-r-- 1 kz kz 279K  3-р сар  1 23:02 .config
If you want to understand the change I made, please diff two files by

diff .config.old .config
Note: I didn't spend time to narrow down the minium change, but the workable change :)
To enable CXL related code support, we need to enable following configurations in .config file. (This way supports modules loading way by command modprobe in below sections, if you use my .config file, please ignore this step, jump to [Step 3: Compile Kernel](#Step 3: Compile Kernel))

root@root1-virtual-machine:~/linux-kernel-dcd-2024-03-24#  cat .config | egrep  "CXL|DAX|_ND_"
CONFIG_ARCH_WANT_OPTIMIZE_DAX_VMEMMAP=y
CONFIG_PCIEAER_CXL=y
CONFIG_CXL_BUS=m
CONFIG_CXL_PCI=m
CONFIG_CXL_MEM_RAW_COMMANDS=y
CONFIG_CXL_ACPI=m
CONFIG_CXL_PMEM=m
CONFIG_CXL_MEM=m
CONFIG_CXL_PORT=m
CONFIG_CXL_SUSPEND=y
CONFIG_CXL_REGION=y
CONFIG_CXL_REGION_INVALIDATION_TEST=y
CONFIG_CXL_PMU=m
CONFIG_ND_CLAIM=y
CONFIG_ND_BTT=m
CONFIG_ND_PFN=m
CONFIG_NVDIMM_DAX=y
CONFIG_DAX=y
CONFIG_DEV_DAX=m
CONFIG_DEV_DAX_PMEM=m
CONFIG_DEV_DAX_HMEM=m
CONFIG_DEV_DAX_CXL=m
CONFIG_DEV_DAX_HMEM_DEVICES=y
CONFIG_DEV_DAX_KMEM=m
CONFIG_FS_DAX=y
CONFIG_FS_DAX_PMD=y
CONFIG_FUSE_DAX=y


make -j$(nproc)
or

make -j32 
or

make -j16
After a successful compile, a new file vmlinux will be generated under kernel root directory. A compressed kernel image will also be available.

ls arch/x86_64/boot/bzImage -lh

e.g.
kroot@root1-virtual-machine:~/linux-kernel-dcd-2024-03-24# cd arch/x86_64/boot/
root@root1-virtual-machine:~/linux-kernel-dcd-2024-03-24/arch/x86_64/boot# ls
bzImage

Step 4: Install kernel modules This will install to host machine, not qemu, but it will be used by mounting host machine's directory /lib/modules to qemu's local folder.

sudo make modules_install
Creating Root File System for Guest VM
To create a disk image as root file system of the guest VM, we need to leverage the tools generated by compiling QEMU source code.

find . -name "qemu-img"

e.g.
root@root1-virtual-machine:~# find . -name "qemu-img"
./qemu-dcd-v6/build/qemu-bundle/usr/local/bin/qemu-img
./qemu-dcd-v6/build/qemu-img

Create a QEMU image with QEMU-image (e.g., 16G).
qemu-img create qemu.img 16G

e.g.
kroot@root1-virtual-machine:~#qemu-img create qemu.img 16G
Formatting 'qemu.img', fmt=raw size=17179869184
Create a file system for the image.
sudo mkfs.ext4 qemu.img

e.g.
root@root1-virtual-machine:~# sudo mkfs.ext4 qemu.img
mke2fs 1.46.5 (30-Dec-2021)
Discarding device blocks: done
Creating filesystem with 4194304 4k blocks and 1048576 inodes
Filesystem UUID: bd521f70-5c9b-4a33-bd51-f58ad962cc43
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done

root@root1-virtual-machine:~# 

Mount the file system to a directory.
mkdir mntdir
sudo mount -o loop qemu.img mntdir
Install the debian distribution to the file system.
sudo debootstrap --arch amd64 stable mntdir
or you want a specific version, use
sudo debootstrap --arch amd64 jammy mntdir

e.g.

root@root1-virtual-machine:~#   debootstrap --arch amd64 --no-check-gpg stable mntdir http://mirrors.aliyun.com/debian/
I: Keyring file not available at /usr/share/keyrings/debian-archive-keyring.gpg; switching to https mirror https://deb.debian.org/debian
I: Retrieving Packages
I: Validating Packages
I: Resolving dependencies of required packages...
I: Resolving dependencies of base packages...
I: Checking component main on https://deb.debian.org/debian...
I: Retrieving adduser 3.134
I: Validating adduser 3.134
I: Retrieving apt 2.6.1
I: Validating apt 2.6.1
...
I: Configuring nftables...
I: Configuring iproute2...
I: Configuring isc-dhcp-client...
I: Configuring ifupdown...
I: Configuring tasksel...
I: Configuring tasksel-data...
I: Configuring libc-bin...
I: Configuring ca-certificates...
I: Base system installed successfully.
kz@kz-HP-EliteBook:~/cxl/qemu-dcd-v6/build$
Setup host and guest VM directory sharing
echo "#! /bin/bash
mount -t 9p -o trans=virtio homeshare /home/kz
mount -t 9p -o trans=virtio modshare /lib/modules
" > ./rc.local
chmod a+x ./rc.local
sudo mv ./rc.local mntdir/etc/
sudo mkdir -p ./mntdir/home/kz
sudo mkdir -p ./mntdir/lib/modules/

e.g.

root@root1-virtual-machine:~#  echo "#! /bin/bash
mount -t 9p -o trans=virtio homeshare /home/kz
mount -t 9p -o trans=virtio modshare /lib/modules
" > ./rc.local

root@root1-virtual-machine:~#  chmod a+x ./rc.local

root@root1-virtual-machine:~#  sudo mv ./rc.local mntdir/etc/

root@root1-virtual-machine:~#  sudo mkdir -p ./mntdir/home/kz

root@root1-virtual-machine:~#  sudo mkdir -p ./mntdir/lib/modules/

root@root1-virtual-machine:~# 
Setup network for guest VM
Create a config.yaml file with following content under ./mntdir/etc/netplan.

network:
    version: 2
    renderer: networkd
    ethernets:
        enp0s2:
            dhcp4: true

e.g.

root@root1-virtual-machine:~#  sudo mkdir -p ./mntdir/etc/netplan/

root@root1-virtual-machine:~#  sudo vi -p ./mntdir/etc/netplan/config.yaml

root@root1-virtual-machine:~# ll ./mntdir/etc/netplan/config.yaml
-rw-r--r-- 1 root root 102  3-р сар  2 09:13 ./mntdir/etc/netplan/config.yaml
set the password up
sudo chroot mntdir and passwd

e.g.

root@root1-virtual-machine:~#  sudo chroot mntdir
root@kz-HP-EliteBook:/# passwd
New password:
Retype new password:
passwd: password updated successfully
root@kz-HP-EliteBook:/# exit

root@root1-virtual-machine:~# 
Umount mntdir
sudo umount mntdir

e.g.

root@root1-virtual-machine:~#  sudo umount mntdir
Bringing up the guest VM
Note:

You can also use start_cxl_qemu.sh and start_qemu_pmem.sh directly from git.
Username is root, password is what you set in step 7 above.
Exit qemu by ctrl+a, x
Example 1: boot up VM with a CXL persistent memory sized 512MiB, directly attached to the root port of a host bridge.

qemu-system-x86_64 \
    -s \
    -kernel /root/linux-kernel-dcd-2024-03-24/arch/x86_64/boot/bzImage \
    -append "root=/dev/sda rw console=ttyS0,115200 ignore_loglevel nokaslr \
             cxl_acpi.dyndbg=+fplm cxl_pci.dyndbg=+fplm cxl_core.dyndbg=+fplm \
             cxl_mem.dyndbg=+fplm cxl_pmem.dyndbg=+fplm cxl_port.dyndbg=+fplm \
             cxl_region.dyndbg=+fplm cxl_test.dyndbg=+fplm cxl_mock.dyndbg=+fplm \
             cxl_mock_mem.dyndbg=+fplm dax.dyndbg=+fplm dax_cxl.dyndbg=+fplm \
             device_dax.dyndbg=+fplm" \
    -smp 1 \
    -accel kvm \
    -serial mon:stdio \
    -nographic \
    -qmp tcp:localhost:4444,server,wait=off \
    -netdev user,id=network0,hostfwd=tcp::2024-:22 \
    -device virtio-net-pci,netdev=network0 \
    -monitor telnet:127.0.0.1:12345,server,nowait \
    -drive file=/root/qemu.img,index=0,media=disk,format=raw \
    -machine q35,cxl=on -m 8G,maxmem=32G,slots=8 \
    -virtfs local,path=/lib/modules,mount_tag=modshare,security_model=mapped \
    -virtfs local,path=/home/kz,mount_tag=homeshare,security_model=mapped \
    -object memory-backend-file,id=cxl-mem1,share=on,mem-path=/tmp/cxltest.raw,size=512M \
    -object memory-backend-file,id=cxl-lsa1,share=on,mem-path=/tmp/lsa.raw,size=512M \
    -device pxb-cxl,bus_nr=12,bus=pcie.0,id=cxl.1 \
    -device cxl-rp,port=0,bus=cxl.1,id=root_port13,chassis=0,slot=2 \
    -device cxl-type3,bus=root_port13,memdev=cxl-mem1,lsa=cxl-lsa1,id=cxl-pmem0 \
    -M cxl-fmw.0.targets.0=cxl.1,cxl-fmw.0.size=4G,cxl-fmw.0.interleave-granularity=8k

e.g.
root@root1-virtual-machine:~# qemu-system-x86_64 \
>     -s \
>     -kernel /root/linux-kernel-dcd-2024-03-24/arch/x86_64/boot/bzImage \
>     -append "root=/dev/sda rw console=ttyS0,115200 ignore_loglevel nokaslr \
>              cxl_acpi.dyndbg=+fplm cxl_pci.dyndbg=+fplm cxl_core.dyndbg=+fplm \
>              cxl_mem.dyndbg=+fplm cxl_pmem.dyndbg=+fplm cxl_port.dyndbg=+fplm \
>              cxl_region.dyndbg=+fplm cxl_test.dyndbg=+fplm cxl_mock.dyndbg=+fplm \
>              cxl_mock_mem.dyndbg=+fplm dax.dyndbg=+fplm dax_cxl.dyndbg=+fplm \
>              device_dax.dyndbg=+fplm" \
>     -smp 1 \
>     -accel kvm \
>     -serial mon:stdio \
>     -nographic \
>     -qmp tcp:localhost:4444,server,wait=off \
>     -netdev user,id=network0,hostfwd=tcp::2024-:22 \
>     -device virtio-net-pci,netdev=network0 \
>     -monitor telnet:127.0.0.1:12345,server,nowait \
>     -drive file=/root/qemu.img,index=0,media=disk,format=raw \
>     -machine q35,cxl=on -m 8G,maxmem=32G,slots=8 \
>     -virtfs local,path=/lib/modules,mount_tag=modshare,security_model=mapped \
>     -virtfs local,path=/home/kz,mount_tag=homeshare,security_model=mapped \
>     -object memory-backend-file,id=cxl-mem1,share=on,mem-path=/tmp/cxltest.raw,size=512M \
>     -object memory-backend-file,id=cxl-lsa1,share=on,mem-path=/tmp/lsa.raw,size=512M \
>     -device pxb-cxl,bus_nr=12,bus=pcie.0,id=cxl.1 \
>     -device cxl-rp,port=0,bus=cxl.1,id=root_port13,chassis=0,slot=2 \
>     -device cxl-type3,bus=root_port13,memdev=cxl-mem1,lsa=cxl-lsa1,id=cxl-pmem0 \
>     -M cxl-fmw.0.targets.0=cxl.1,cxl-fmw.0.size=4G,cxl-fmw.0.interleave-granularity=8k
SeaBIOS (version rel-1.16.3-0-ga6ed6b701f0a-prebuilt.qemu.org)



Booting from ROM..
[    0.000000] Linux version 6.8.0 (kz@kz-HP-EliteBook) (gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, GNU ld (GN5
[    0.000000] Command line: root=/dev/sda rw console=ttyS0,115200 ignore_loglevel nokaslr              cxl_acm
[    0.000000] KERNEL supported cpus:
[    0.000000]   Intel GenuineIntel
[    0.000000]   AMD AuthenticAMD
[    0.000000]   Hygon HygonGenuine
[    0.000000]   Centaur CentaurHauls
[    0.000000]   zhaoxin   Shanghai
[    0.000000] BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved

...

[    1.888950] cxl_core:devm_cxl_setup_hdm:159: cxl_port port1: HDM decoder registers not implemented
[    1.888951] cxl_port:cxl_switch_port_probe:84: cxl_port port1: Fallback to passthrough decoder
[    1.889009] cxl_core:add_hdm_decoder:37: cxl decoder1.0: Added to port port1
[    1.889010] cxl_core:cxl_bus_probe:2077: cxl_port port1: probe: 0
[    1.895118] cxl_core:cxl_bus_probe:2077: cxl_nvdimm_bridge nvdimm-bridge0: probe: 0
[    1.925898] [drm] Found bochs VGA, ID 0xb0c5.
[    1.926157] [drm] Framebuffer size 16384 kB @ 0xfd000000, mmio @ 0x80000000.
[    1.927458] [drm] Found EDID data blob.
[    1.927804] [drm] Initialized bochs-drm 1.0.0 20130925 for 0000:00:01.0 on minor 0
[    1.929917] ppdev: user-space parallel port driver
[    1.932479] fbcon: bochs-drmdrmfb (fb0) is primary device
[    1.936465] Console: switching to colour frame buffer device 160x50
[    1.937759] bochs-drm 0000:00:01.0: [drm] fb0: bochs-drmdrmfb frame buffer device
[    1.945678] Error: Driver 'pcspkr' is already registered, aborting...

Debian GNU/Linux 12 kz-HP-EliteBook ttyS0

kz-HP-EliteBook login: root
Password:
Linux kz-HP-EliteBook 6.8.0 #5 SMP PREEMPT_DYNAMIC Sat Mar  1 19:11:50 +08 2025 x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Mar  2 03:37:17 UTC 2025 on ttyS0
root@kz-HP-EliteBook:~#

Example 2: boot up VM with CXL DCD setup: the device is directly attached to the only root port of a host bridge. The device has two dynamic capacity regions, with each region being 2GiB in size.

qemu-system-x86_64 \
    -s \
    -kernel /root/linux-kernel-dcd-2024-03-24/arch/x86_64/boot/bzImage \
    -append "root=/dev/sda rw console=ttyS0,115200 ignore_loglevel nokaslr \
             cxl_acpi.dyndbg=+fplm cxl_pci.dyndbg=+fplm cxl_core.dyndbg=+fplm \
             cxl_mem.dyndbg=+fplm cxl_pmem.dyndbg=+fplm cxl_port.dyndbg=+fplm \
             cxl_region.dyndbg=+fplm cxl_test.dyndbg=+fplm cxl_mock.dyndbg=+fplm \
             cxl_mock_mem.dyndbg=+fplm dax.dyndbg=+fplm dax_cxl.dyndbg=+fplm \
             device_dax.dyndbg=+fplm" \
    -smp 1 \
    -accel kvm \
    -serial mon:stdio \
    -nographic \
    -qmp tcp:localhost:4444,server,wait=off \
    -netdev user,id=network0,hostfwd=tcp::2024-:22 \
    -device virtio-net-pci,netdev=network0 \
    -monitor telnet:127.0.0.1:12345,server,nowait \
    -drive file=/root/qemu.img,index=0,media=disk,format=raw \
    -machine q35,cxl=on -m 8G,maxmem=32G,slots=8 \
    -virtfs local,path=/lib/modules,mount_tag=modshare,security_model=mapped \
    -virtfs local,path=/home/kz,mount_tag=homeshare,security_model=mapped \
    -object memory-backend-file,id=cxl-mem1,share=on,mem-path=/tmp/cxltest.raw,size=512M \
    -object memory-backend-file,id=cxl-lsa1,share=on,mem-path=/tmp/lsa.raw,size=512M \
    -device pxb-cxl,bus_nr=12,bus=pcie.0,id=cxl.1 \
    -device cxl-rp,port=0,bus=cxl.1,id=root_port13,chassis=0,slot=2 \
    -device cxl-type3,bus=root_port13,memdev=cxl-mem1,lsa=cxl-lsa1,id=cxl-pmem0 \
    -M cxl-fmw.0.targets.0=cxl.1,cxl-fmw.0.size=4G,cxl-fmw.0.interleave-granularity=8k

e.g.
root@root1-virtual-machine:~# qemu-system-x86_64 \
>     -s \
>     -kernel /root/linux-kernel-dcd-2024-03-24/arch/x86_64/boot/bzImage \
>     -append "root=/dev/sda rw console=ttyS0,115200 ignore_loglevel nokaslr \
>              cxl_acpi.dyndbg=+fplm cxl_pci.dyndbg=+fplm cxl_core.dyndbg=+fplm \
>              cxl_mem.dyndbg=+fplm cxl_pmem.dyndbg=+fplm cxl_port.dyndbg=+fplm \
>              cxl_region.dyndbg=+fplm cxl_test.dyndbg=+fplm cxl_mock.dyndbg=+fplm \
>              cxl_mock_mem.dyndbg=+fplm dax.dyndbg=+fplm dax_cxl.dyndbg=+fplm \
>              device_dax.dyndbg=+fplm" \
>     -smp 1 \
>     -accel kvm \
>     -serial mon:stdio \
>     -nographic \
>     -qmp tcp:localhost:4444,server,wait=off \
>     -netdev user,id=network0,hostfwd=tcp::2024-:22 \
>     -device virtio-net-pci,netdev=network0 \
>     -monitor telnet:127.0.0.1:12345,server,nowait \
>     -drive file=/root/qemu.img,index=0,media=disk,format=raw \
>     -machine q35,cxl=on -m 8G,maxmem=32G,slots=8 \
>     -virtfs local,path=/lib/modules,mount_tag=modshare,security_model=mapped \
>     -virtfs local,path=/home/kz,mount_tag=homeshare,security_model=mapped \
>     -object memory-backend-file,id=cxl-mem1,share=on,mem-path=/tmp/cxltest.raw,size=512M \
>     -object memory-backend-file,id=cxl-lsa1,share=on,mem-path=/tmp/lsa.raw,size=512M \
>     -device pxb-cxl,bus_nr=12,bus=pcie.0,id=cxl.1 \
>     -device cxl-rp,port=0,bus=cxl.1,id=root_port13,chassis=0,slot=2 \
>     -device cxl-type3,bus=root_port13,memdev=cxl-mem1,lsa=cxl-lsa1,id=cxl-pmem0 \
>     -M cxl-fmw.0.targets.0=cxl.1,cxl-fmw.0.size=4G,cxl-fmw.0.interleave-granularity=8k
SeaBIOS (version rel-1.16.3-0-ga6ed6b701f0a-prebuilt.qemu.org)

Bringing up network
ip link set dev enp0s2 up
dhclient enp0s2

e.g.
ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp0s2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
root@root1-virtual-machine:~#
root@root1-virtual-machine:~# ip link set dev enp0s2 up
root@root1-virtual-machine:~# dhclient enp0s2
root@root1-virtual-machine:~#
root@root1-virtual-machine:~#
root@root1-virtual-machine:~# ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp0s2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s2
       valid_lft 86398sec preferred_lft 86398sec
    inet6 fec0::5054:ff:fe12:3456/64 scope site dynamic mngtmpaddr
       valid_lft 86378sec preferred_lft 14378sec
    inet6 fe80::5054:ff:fe12:3456/64 scope link
       valid_lft forever preferred_lft forever


List CXL device
apt install pciutils
lspci
lspci | grep CXL
lspci -vvv
lspci -vvv -s 0d:00.0

e.g.
```
root@root1-virtual-machine:~# lspci -tv
-+-[0000:00]-+-00.0  Intel Corporation 82G33/G31/P35/P31 Express DRAM Controller
 |           +-01.0  Device 1234:1111
 |           +-02.0  Red Hat, Inc. Virtio network device
 |           +-03.0  Red Hat, Inc. Virtio filesystem
 |           +-04.0  Red Hat, Inc. Virtio filesystem
 |           +-05.0  Red Hat, Inc. QEMU PCIe Expander bridge
 |           +-1f.0  Intel Corporation 82801IB (ICH9) LPC Interface Controller
 |           +-1f.2  Intel Corporation 82801IR/IO/IH (ICH9R/DO/DH) 6 port SATA Controller [AHCI mode]
 |           \-1f.3  Intel Corporation 82801I (ICH9 Family) SMBus Controller
 \-[0000:0c]---00.0-[0d]----00.0  Intel Corporation Device 0d93
root@root1-virtual-machine:~# lspci -s 0d:00.0 -vvv
0d:00.0 CXL: Intel Corporation Device 0d93 (rev 01) (prog-if 10 [CXL Memory Device (CXL 2.x)])
        Subsystem: Red Hat, Inc. Device 1100
        Physical Slot: 2
        Control: I/O+ Mem+ BusMaster- SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR+ FastB2B- DisINTx-
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Region 0: Memory at fe800000 (64-bit, non-prefetchable) [size=64K]
        Region 2: Memory at fe810000 (64-bit, non-prefetchable) [size=4K]
        Region 4: Memory at fe811000 (32-bit, non-prefetchable) [size=4K]
        Capabilities: [40] MSI-X: Enable- Count=6 Masked-
                Vector table: BAR=4 offset=00000000
                PBA: BAR=4 offset=00000800
        Capabilities: [80] Express (v2) Endpoint, MSI 00
                DevCap: MaxPayload 128 bytes, PhantFunc 0, Latency L0s <64ns, L1 <1us
                        ExtTag- AttnBtn- AttnInd- PwrInd- RBE+ FLReset- SlotPowerLimit 0W
                DevCtl: CorrErr+ NonFatalErr+ FatalErr+ UnsupReq+
                        RlxdOrd- ExtTag- PhantFunc- AuxPwr- NoSnoop-
                        MaxPayload 128 bytes, MaxReadReq 128 bytes
                DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
                LnkCap: Port #0, Speed 2.5GT/s, Width x1, ASPM L0s, Exit Latency L0s <64ns
                        ClockPM- Surprise- LLActRep- BwNot- ASPMOptComp-
                LnkCtl: ASPM Disabled; RCB 64 bytes, Disabled- CommClk-
                        ExtSynch- ClockPM- AutWidDis- BWInt- AutBWInt-
                LnkSta: Speed 2.5GT/s, Width x1
                        TrErr- Train- SlotClk- DLActive- BWMgmt- ABWMgmt-
                DevCap2: Completion Timeout: Not Supported, TimeoutDis- NROPrPrP- LTR-
                         10BitTagComp- 10BitTagReq- OBFF Not Supported, ExtFmt+ EETLPPrefix+, MaxEETLPPrefixes 4
                         EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
                         FRS- TPHComp- ExtTPHComp-
                         AtomicOpsCap: 32bit- 64bit- 128bitCAS-
                DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis- LTR- 10BitTagReq- OBFF Disabled,
                         AtomicOpsCtl: ReqEn-
                LnkCtl2: Target Link Speed: 2.5GT/s, EnterCompliance- SpeedDis-
                         Transmit Margin: Normal Operating Range, EnterModifiedCompliance- ComplianceSOS-
                         Compliance Preset/De-emphasis: -6dB de-emphasis, 0dB preshoot
                LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete- EqualizationPhase1-
                         EqualizationPhase2- EqualizationPhase3- LinkEqualizationRequest-
                         Retimer- 2Retimers- CrosslinkRes: unsupported
        Capabilities: [100 v1] Designated Vendor-Specific: Vendor=1e98 ID=0000 Rev=3 Len=60: CXL
                CXLCap: Cache- IO+ Mem+ Mem HW Init+ HDMCount 1 Viral-
                CXLCtl: Cache- IO+ Mem- Cache SF Cov 0 Cache SF Gran 0 Cache Clean- Viral-
                CXLSta: Viral-
                CXLSta2:        ResetComplete+ ResetError- PMComplete-
                Cache Size Not Reported
                Range1: 0000000000000000-000000001fffffff
                        Valid+ Active+ Type=CDAT Class=CDAT interleave=0 timeout=1s
                Range2: 0000000000000000-ffffffffffffffff
                        Valid- Active- Type=Volatile Class=DRAM interleave=0 timeout=1s
        Capabilities: [13c v1] Designated Vendor-Specific: Vendor=1e98 ID=0008 Rev=0 Len=36: CXL
                Block1: BIR: bar0, ID: component registers, offset: 0000000000000000
                Block2: BIR: bar2, ID: CXL device registers, offset: 0000000000000000
        Capabilities: [160 v1] Designated Vendor-Specific: Vendor=1e98 ID=0005 Rev=0 Len=16: CXL
                GPF Phase 2 Duration: 3s
                GPF Phase 2 Power: 51mW
        Capabilities: [170 v1] Designated Vendor-Specific: Vendor=1e98 ID=0007 Rev=2 Len=32: CXL
                FBCap:  Cache- IO+ Mem+ 68BFlit+ MltLogDev- 256BFlit- PBRFlit-
                FBCtl:  Cache- IO+ Mem- SynHdrByp- DrftBuf- 68BFlit- MltLogDev- RCD- Retimer1- Retimer2- 256BFlit- PBRFlit-
                FBSta:  Cache- IO+ Mem+ SynHdrByp- DrftBuf- 68BFlit+ MltLogDev- 256BFlit- PBRFlit-
                FBModTS:        Received FB Data: 0000ef
                FBCap2: NOPHint-
                FBCtl2: NOPHint-
                FBSta2: NOPHintInfo: 0
        Capabilities: [190 v1] Data Object Exchange
                DOECap: IntSup+
                        Interrupt Message Number 000
                DOECtl: IntEn-
                DOESta: Busy- IntSta- Error- ObjectReady-
        Capabilities: [200 v2] Advanced Error Reporting
                UESta:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
                UEMsk:  DLP- SDES- TLP- FCP- CmpltTO- CmpltAbrt- UnxCmplt- RxOF- MalfTLP- ECRC- UnsupReq- ACSViol-
                UESvrt: DLP+ SDES+ TLP- FCP+ CmpltTO- CmpltAbrt- UnxCmplt- RxOF+ MalfTLP+ ECRC- UnsupReq- ACSViol-
                CESta:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr-
                CEMsk:  RxErr- BadTLP- BadDLLP- Rollover- Timeout- AdvNonFatalErr+
                AERCap: First Error Pointer: 00, ECRCGenCap+ ECRCGenEn- ECRCChkCap+ ECRCChkEn-
                        MultHdrRecCap- MultHdrRecEn- TLPPfxPres- HdrLogCap-
                HeaderLog: 00000000 00000000 00000000 00000000
lspci: Unable to load libkmod resources: error -2
```
