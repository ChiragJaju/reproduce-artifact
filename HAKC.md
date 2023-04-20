# Preparation
## Build the latest QEMU (v8.0.0-rc3) for MTE support
1. Download and build the latest QEMU: https://www.qemu.org/download/#source. During configuration, add the `--enable-slirp` flag. `slirp` has been the default networking backend for QEMU but it was removed since QEMU 7.2 release. In the end, remember to `make install` to avoid the 'failed to find romfile "efi-virtio.rom"' error. 
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

# Building the kernel
Follow the instructions in https://github.com/mit-ll/HAKC/tree/main/PMC-Pass

# Emulating with QEMU
- `mkdir linux-envs; cd linux-envs`
- Download a raspberry pi image (20230102_raspi_4_bullseye.img.xz) from https://raspi.debian.net/tested-images/. Copy to the /linux-envs directory.
- `dd if=/dev/null of=disk.img bs=1M seek=10240`
- `xzcat 20230102_raspi_4_bullseye.img.xz | dd of=disk.img conv=notrunc status=progress`
- `sudo partx -a -v disk.img `  
    -  This will show that it is using a loop device, such as /dev/loop21. That's the device we will continue to use, so replace accordingly.
- `mkdir host-mount` 
- `sudo mount /dev/loop21p1 host-mount`
- `cp host-mount/initrd.img* .`
- `cp host-mount/vmlinuz-5.10* .`
- `qemu-system-aarch64 -M virt,mte=on -m 4096 -cpu max -drive format=raw,file=disk.img -nographic -append "root=/dev/vda2 net.ifaces=0 rootwait" -initrd initrd.img-5.10.0-20-arm64 -kernel vmlinuz-5.10.0-20-arm64` 
    - The initial run of QEMU resizes the root filesystem partition to use the full remaining space in the disk image, and is necessary to install new modules. The username is `root` with no password.
    - If there is an error on raising the network interface (e.g., `systemctl status networking.service` tells me `ifup: failed to bring up eth0`), go to /etc/network/interfaces.d and replace `eth0` with whatever you see when you do `ip a` (e.g., enp0s1). Once done, `poweroff` and start QEMU again (`reboot` doesn't seem to work).
    - `apt update && apt install apache2`
- `poweroff`
- `sudo umount host-mount`
- `sudo mount /dev/loop21p2 host-mount`
    - If mount is getting these errors "mount: wrong fs type, bad option, bad superblock on /dev/loop21p2, missing codepage or helper program, or other error" and in `dmesg`, "EXT4-fs (loop21p2): bad geometry: block count 2490368 exceeds size of device (508928 blocks)", reformat the partition.
- 

 
