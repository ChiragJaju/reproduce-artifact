# Preparation
## Build the latest QEMU (v8.0.0-rc3) for MTE support
1. Download and build the latest QEMU: https://www.qemu.org/download/#source. During configuration, add the `--enable-slirp` flag. `slirp` has been the default networking backend for QEMU but it was removed since QEMU 7.2 release. In the end, remember to `make install` to avoid the 'failed to find romfile "efi-virtio.rom"' error. If you encounter a pixman-1 not found error, just lookup the package name for pixman ( for fedora its pixman-devel, for ubuntu its libpixman-1-dev ) and install it (eg. `sudo dnf install pixman-devel` ).
    ```
    sudo apt install libslirp-dev
    wget https://download.qemu.org/qemu-8.0.0-rc4.tar.xz
    tar xvJf qemu-8.0.0-rc4.tar.xz
    cd qemu-8.0.0-rc4
    ./configure --enable-slirp
    make
    make install 
    ```
2. If QEMU is not running the latest version we've just built, enter the /build directory and copy the QEMU builds to /usr/local/bin. 
    - `cd build`
    - `sudo cp aarch64-softmmu/qemu-system-aarch64 /usr/local/bin/qemu-system-aarch64`
    - `qemu-system-aarch64 --version`
    - If it's not showing the latest version still, either remove the `qemu-system-aarch64` in /usr/bin/ or replace the second command with `sudo cp aarch64-softmmu/qemu-system-aarch64 /usr/local/bin/qemu[VER]-system-aarch64`.

# Download the linux kernel
1. Download linux-6.3.6.tar.xz from here https://cdn.kernel.org/pub/linux/kernel/v6.x/
2. Run the tar command to extract the source code ` tar xvf linux-6.3.6.tar.xz `
3. Install the required packages ` sudo apt-get install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc flex libelf-dev bison `
4. ` cd linux-6.3.6 `
5. Copy the existing configuration file ` cp -v /boot/config-$(uname -r) .config `
6. To make changes in the config file run ` make menuconfig `
7. Then finally run ` make `  or to speed up compile run ` nproc ` to get number of cores and use -j flag ` make -j4 `. Use 2 if you have 4 cores and 4 if you have 6 cores to avoid crashing. gcc version used to compile here was 13.1.1 . If you face errors here and have gcc 11, download linux-5.10 series instead.


# Emulating with QEMU
1. `mkdir linux-envs; cd linux-envs`
2. Download a raspberry pi image (20230102_raspi_4_bullseye.img.xz) from https://raspi.debian.net/tested-images/. Copy to the /linux-envs directory.
3. `dd if=/dev/null of=disk.img bs=1M seek=10240`
4. `xzcat 20230102_raspi_4_bullseye.img.xz | dd of=disk.img conv=notrunc status=progress`
5. `sudo partx -a -v disk.img `  
    -  This will output a partition number and we will use that. eg loop0 will have two partitions loop0p1 and loop0p2 
6. `mkdir host-mount` 
7. `sudo mount /dev/loop0p1 host-mount`
8. `cp host-mount/initrd.img* .`
9. `cp host-mount/vmlinuz* .`
10. ( IMP ) Resize partition 2 to use all the remaining space in the disk image. This is important to prevent mounting error in partition 2 later on.
11. ` qemu-system-aarch64 -M virt,mte=on -m 4096 -cpu max -drive format=raw,file=disk.img -nographic -append "root=/dev/vda2 net.ifaces=0 rootwait" -initrd initrd.img-6.0.0-6-arm64 -kernel vmlinuz-6.0.0-6-arm64 `
    - The initial run of QEMU resizes the root filesystem partition to use the full remaining space in the disk image, and is necessary to install new modules. The username is `root` with no password.
    - If there is an error on raising the network interface (e.g., `systemctl status networking.service` tells me `ifup: failed to bring up eth0`), go to /etc/network/interfaces.d and replace `eth0` with whatever you see when you do `ip a` (e.g., enp0s1). Here you need to ` nano eth0 ` and replace all eth0 with enp0s1 then rename eth0 file to enp0s1. Once done, `poweroff` and start QEMU again (`reboot` doesn't seem to work).
    - `apt update && apt install apache2`
    - `poweroff`
12. `sudo umount host-mount`
13. `sudo mount /dev/loop0p2 host-mount`
    - If mount is getting these errors "mount: wrong fs type, bad option, bad superblock on /dev/loop21p2, missing codepage or helper program, or other error", and in `dmesg`, "EXT4-fs (loop21p2): bad geometry: block count 2490368 exceeds size of device (508928 blocks)", it is because the root partition is expanded beyond its allocated size. Remove the disk image, begin again from step 3 and remember to do step 10.
14. `pushd /home/chiragjaju/Downloads/linux-6.3.6; sudo make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- LLVM=1 CC=clang HOSTCC=clang INSTALL_MOD_PATH=/home/chiragjaju/Desktop/ubc/linux-envs/host-mount modules_install` 
    - Take note of the build number at the end: `5.10.24pmc-build` If you encounter a warning at the end its fine.
15. `popd`
16. `sudo umount host-mount`
17. `sudo mount /dev/loop0p1 host-mount`
18. ` qemu-system-aarch64 -M virt,mte=on -m 4096 -cpu max -drive format=raw,file=disk.img -nographic -append "root=/dev/vda2 net.ifaces=0 rootwait" -initrd initrd.img-6.0.0-6-arm64 -kernel vmlinuz-6.0.0-6-arm64 ` 
    - If there is a filesystem error "RASPIROOT: UNEXPECTED INCONSISTENCY", do `fsck -y /dev/vda2` and `exit` initramfs upon completion to resume QEMU operation.
    - `mkinitramfs -v -o /boot/firmware/initrd.*` [ there should be only one initrd* there]
19. Install gcc ` apt-get install gcc `
20. cd into home
21. Copy the program given here at the end https://docs.kernel.org/arm64/memory-tagging-extension.html and create a test.c 
22. Compile it with ` -march=armv8.5-a+memtag ` i.e. ` gcc test.c -march=armv8.5-a+memtag `
23. Run the program with ` ./a.out ` and if you see a Segmentation fault after " Expecting SIGSEGV " then MTE is working. 
