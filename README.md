# Build Embedded Linux Image for A5 Systems

## Info

The goal of this readme is to make the image creation process easy, even for those that have no clue what they're doing. The documentation I found for this was fine, but it lacked simple details for those who weren't super knowledgeable in linux. I was/am one of those people, so I'm making this for myself so I always have something to reference when making images. All I've done is remade the documentation with more explanation.

I'm doing this on a debian 10 vm. This vm's sole purpose is for making images for linux images for embedded devices, which means a lot of the configuration I'll be doing will be on the vm itself to make this easier. Just ignore these steps if you're not on a vm, but I wouldn't recommend building these images on anything but a vm. You can mess up your distro if you do this without caution.

## Prerequisites

### Download required packages:

```
sudo apt-get install libc6-armel-cross libc6-dev-armel-cross binutils-arm-linux-gnueabi libncurses5-dev dosfstools git multistrap qemu qemu-user-static binfmt-support dpkg-cross
```

### Organize a workspace:

```
cd ~
mkdir bootloader kernel linux deps
cd deps; mkdir arm armhf
cd ~
```

### Download Toolchains

There are two toolchains we will be using:
 - arm-linux-gnueabi 6.4.1
 - arm-linux-gnueabihf 6.4.1

arm-linux-gnueabihf is for hardware with a FPU, and arm-linux-gnueabi is for... hardware without it

I'm going to be downloading and installing the toolchains manually because I had issues with the toolchains available on the official repos.

##### arm-linux-gnueabihf download and install

```
cd ~/deps/armhf

wget -c https://releases.linaro.org/components/toolchain/binaries/6.4-2018.05/arm-linux-gnueabihf/gcc-linaro-6.4.1-2018.05-x86_64_arm-linux-gnueabihf.tar.xz

tar -xvf gcc-linaro-6.4.1-2018.05-x86_64_arm-linux-gnueabihf.tar.xz

(OPTIONAL): rm gcc-linaro-6.4.1-2018.05-x86_64_arm-linux-gnueabihf.tar.xz

# Export the toolchain for use
export CCARMHF=~/deps/armhf/gcc-linaro-6.4.1-2018.05-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-
```

##### arm-linux-gnueabi download and install

```
cd ~/deps/arm

wget -c https://releases.linaro.org/components/toolchain/binaries/6.4-2018.05/arm-linux-gnueabi/gcc-linaro-6.4.1-2018.05-x86_64_arm-linux-gnueabi.tar.xz

tar -xvf gcc-linaro-6.4.1-2018.05-x86_64_arm-linux-gnueabi.tar.xz

(OPTIONAL) rm gcc-linaro-6.4.1-2018.05-x86_64_arm-linux-gnueabi.tar.xz

# Export the toolchain for use
export CCARM=~/deps/arm/gcc-linaro-6.4.1-2018.05-x86_64_arm-linux-gnueabi/bin/arm-linux-gnueabi-
```

##### Export these toolchains on boot, because doing it every time is annoying

I did this in ~/.bashrc because everyone on the internet has a different opinion on where to actually put environment variables and I got sick of researching it.

```
echo "export CCARMHF=~/deps/armhf/gcc-linaro-6.4.1-2018.05-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-" >> ~/.bashrc

echo "export CCARM=~/deps/arm/gcc-linaro-6.4.1-2018.05-x86_64_arm-linux-gnueabi/bin/arm-linux-gnueabi-" >> ~/.bashrc
```

If you're following this guide in order, I would go ahead and pick which toolchain you're going to use and export it as CC. This way you can use these instructions without needing to change them. This may be a little convoluted, but it's how I do it to easily switch between scripts and instructions.

```
# if your target hardware has a FPU
export CC=${CCARMHF}
# otherwise, use CCARM
export CC=${CCARM}
```



## SD Prep

First, use ```lsblk``` to find what SD card is yours. Then export it for ease. Mine is /dev/sdb so:

```
export DISK=/dev/sdb
```

**WARNING** You're about to delete all the data on this SD card. Hope you're cool with that, boss. 

Also make sure you didn't accidentally select a disk that isn't your sd card.

Now use ```fdisk``` to make some partitions:

```
fdisk ${DISK}
```

I'm first going to include a section with the inputs and the explanations, then just a section with the inputs.

```fdisk``` Inputs + Explanation and Details
```
# delete all the partitions
# just keep entering 'd' til all the partitions are gone
d       # delete partition

# first we'll create the boot partition
n       # new partition

# hit enter to use default partition number. 
# If the default isn't 1, you messed up. Go back and delete partitions.
<enter>

# hit enter to use default first sector (2048)
<enter>

+128M # partition extending 128 MB

# Now we'll create the partition that will hold our root filesystem
n       # new partition

# hit enter to use the default partition number
# if this isn't 2 you messed up
<enter> 

# hit enter to use the default start sector
<enter>

# Here's where you can pick the size of your root filesystem. I'm arbitrarily using 4GB even though I have a 32GB SD card. This is because I plan on making my root filesystem + boot sectors read only. Then I'll make a third partition on the first boot of the device expanding to the size  it can hold. This is so that the image I create will only be the size of the bootloader + rootfs. It's also to secure the rootfs and bootloader partitions in case of a brown out or an improper shutdown. Failing to do so could corrupt the image on the SD card, causing you to have to reload the SD card with another image. This doesn't really happen in a normal linux distro for your desktop pc, but in the embedded world where your devices are being powered by batteries or by some sort of power source that isn't guaranteed to be on at all times, you've got to make sure you have some answer to immediate power outtages. This is my answer. It probably isn't the best one, but it's the one I use.

+4G

# If fdisk asks you to remove a previously existing signature, say yes.

w # finalize your changes by writing your changes to the disk
```

```fdisk``` Inputs:

```
d       # delete partition
n       # new partition
<enter> # default partition no. 1
<enter> # default start sector
+128M   # 128MB boot partition size
n       # new partition
<enter> # default partition no. 2
<enter> # default start sector
+4G     # 4GB fs size
w       # finalize changes, write to disk
```

Make sure you created your partitions:

```
lsblk ${DISK}
```

If you don't see your partitions under your disk like this, you messed up:

```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sdb      8:16   1 29.7G  0 disk 
├─sdb1   8:17   1  128M  0 part 
└─sdb2   8:18   1    4G  0 part 
```

Now make your filesystems:

If your device enumerated as "/dev/mmcblkX", it means you're using a SD MMC (MultiMediaCard) port. To make your filesystems on it:
```
# Create bootloader fs
mkfs.vfat -F 16 -n BOOT ${DISK}p1

# Create rootfs
mkfs.ext4 -L rootfs ${DISK}p2
```

If your device enumerated as "/dev/sdX", it means you're using a USB -> SD MMC bridge of some kind, like a USB to SD Card adapter. To make your filesystems on it:
```
# Create bootloader fs
mkfs.vfat -F 16 -n BOOT ${DISK}1

# Create rootfs
mkfs.ext4 -L rootfs ${DISK}2
```

Now all that's left to do is mount your devices. 

They may automount on some systems. If they do, skip the mounting step and go to the step where we export them.

To find out where they're mounted, use this command:

```
lsblk ${DISK}
```

You should see something like this, with the mount points on the right:

```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sdb      8:16   1 29.7G  0 disk 
├─sdb1   8:17   1  128M  0 part /media/temp1
└─sdb2   8:18   1    4G  0 part /media/temp2
```

Now, for those that need to mount their drives (I would recommend unmounting from other locations and mounting to the ones that I suggest anyways):

```
# make our mount point directories
mkdir -p /mnt/boot/
mkdir -p /mnt/rootfs/

# for SD MMC devices:
mount ${DISK}p1 /mnt/boot/
mount ${DISK}p2 /mnt/rootfs/

# for USB -> SD MMC devices:
mount ${DISK}1 /mnt/boot/
mount ${DISK}2 /mnt/rootfs/
```

Now we're ready to start making the image

## Bootloader

We're going to use AT91Bootstrap as our bootloader. You can find more information on it here: http://www.at91.com/linux4sam/bin/view/Linux4SAM/AT91Bootstrap

```
cd ~/bootloader
# clone the at91bootstrap source repo
git clone https://github.com/linux4sam/at91bootstrap
cd at91bootstrap/
# checkout version 3.8.10 to a temporary branch
git checkout v3.8.10 -b tmp
cd at91bootstrap/
```

Configure and compile the bootloader

This is when things get slightly complicated if you're just following instructions without paying attention. I'm going to use the SAMA5D4 Xplained Ultra development board from Microchip as an example.

If we take a look at one of the configs for the bootloader, we'll see some basic options set:
```
more ~/bootloader/at91bootstrap/board/sama5d4_xplained/sama5d4_xplainedsd_uboot_secure_defconfig
```

You'll see simple things like HDMI, SDCARD, CPU Clock Freq... This is where your bootloader decides which drivers to load on boot.

Configure the bootloader
```
make ARCH=arm CROSS_COMPILE=${CC} distclean
# compile for the sama5d4_xplained -- boot using sd card
make ARCH=arm CROSS_COMPILE=${CC} sama5d4_xplainedsd_uboot_secure_defconfig
```

This next step is optional. This is if you want to configure your bootloader.

**NOTE** When performing this step, I had trouble in gnome-terminal. The UI was messed up, so I switched to XTerm.
```
make ARCH=arm CROSS_COMPILE=${CC} menuconfig
```

Now, compile the bootloader

```
make ARCH=arm CROSS_COMPILE=${CC}
```

## Kernel

I'm getting most of my information from here: http://www.at91.com/linux4sam/bin/view/Linux4SAM/Sama5d4XplainedMainPage#Build_Kernel_from_sources

Clone the linux-at91 git repo then add the linux4sam repo as a remote git repo. This way we can just pick which version of the kernel we want to use. I'm choosing to use version 4.14 because it's the one I've used before. Eventually I'll update to 4.19.

```
cd ~/kernel
git clone git://github.com/linux4sam/linux-at91.git
cd linux-at91/
git remote add linux4sam git://github.com/linux4sam/linux-at91.git
git remote update linux4sam
# list branches
git branch -r
# switch to 4.14
git checkout origin/linux-4.14-at91 -b linux-4.14-at91
```

Here's where we can configure the kernel. This is important. This is where you can enable or disable certain hardware. For example, if we want I2C enabled, we do it in the kernel config.

```
make ARCH=arm CROSS_COMPILE=${CC} sama5_defconfig 
# configure the kernel
make ARCH=arm CROSS_COMPILE=${CC} menuconfig
# compile the kernel
make ARCH=arm CROSS_COMPILE=${CC}
```


## Root Filesystem

## To Do:
 - Make a script for doing this automatically
 - Make a script for device tree configuration automatically