# cardano-node-setup-riscv64
How to build, setup and run Cardano Node and tools on RISC-V.

These are the steps how I got a working binary running on a HiFive Unmatched

# Prerequisites

## HiFive Unmatched


## Something that can run QEMU

## OS


### Installing Ubuntu 21.04

Installing Ubuntu [source](https://blogjawn.stufftoread.com/install-ubuntu-on-hifive-unmatched.html)


#### Preparing the SD Card


```bash
wget https://cdimage.ubuntu.com/releases/21.04/release/ubuntu-21.04-preinstalled-server-riscv64+unmatched.img.xz 
unxz ubuntu-21.04-preinstalled-server-riscv64+unmatched.img.xz
```

Flashing the Image Via Command Line
To flash the image to the SD card via the command line, run

```bash
dd if=</path/to/image.img> of=/dev/mmcblk0 bs=1M status=progress
```

Booting for the First Time

Connect to the serial console using the supplied USB cable.
```bash
sudo screen /dev/ttyUSB1 115200
```

Installing Ubuntu to an NVMe drive

```bash
wget http://cdimage.ubuntu.com/ubuntu/releases/21.04/release/ubuntu-21.04-preinstalled-server-riscv64+unmatched.img.xz 
unxz /ubuntu-21.04-preinstalled-server-riscv64+unmatched.img.xz
```

Make sure the NVMe drive is present by running

```bash
ls -l /dev/nvme*
```

```bash
sudo dd if=/ubuntu-21.04-preinstalled-server-riscv64+unmatched.img of=/dev/nvme0n1 bs=1M status=progress
sudo mount /dev/nvme0n1p1 /mnt
sudo chroot /mnt
U_BOOT_ROOT="root=/dev/nvme0n1p1"
u-boot-update
exit
reboot
```


# GHC compiler

```bash

git clone git@github.com:RV64-Stake-Pool/ghc.git
cd ghc
git checkout ghc-8.10.4-release-risc5-patches
```

# cabal-install

# Building



# Running

Follow the instructions available add/replace the RTS options for `cardanode-node` with these values `+RTS -N3 --disable-delayed-os-memory-return -I0.3 -Iw600 -A32m -F1.5 -H3000M -T -S -RTS`.

## Download pre-compiled binaries

Cabal and GHC are only needed for development/compiling new releases.

* [Patched GHC 8.10.4](https://ipfs.io/ipfs/QmXSSxRZK9mPvgLxR5vuKm5gPBiy6NmKTcpUk5K4FXx1Tm?filename=ghc-8.10.4-patched.tar.gz)
* [Cabal 3.4.0.0](https://ipfs.io/ipfs/QmT94BAFioQ5XL9Foytk2QHHLtpfD2kUQ8Y958cg4Dc2Ki?filename=cabal-3.4.0.0-riscv64.tgz)
* [libsodium](https://ipfs.io/ipfs/QmXLkhEfs5fHyows7xZARiNxGMSpRP3uj25SQL5Kpk3srN?filename=libsodium.tgz)
* [Cardano Node 1.28.0](https://ipfs.io/ipfs/QmUxY1VFhWPpTsK8eGfBrHQTW6EqfgfAJxYgSFK1hTyFh9?filename=cardano-1.28.0.tgz)

