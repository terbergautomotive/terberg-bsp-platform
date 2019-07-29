Toradex oe-core Setup with Mender
=================================

To simplify installation we provide a repo manifest which manages the different git repositories
and the used versions. (more on repo: http://code.google.com/p/git-repo/ )

Install the repo bootstrap binary:

    mkdir ~/bin
    PATH=~/bin:$PATH
    curl http://commondatastorage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
    chmod a+x ~/bin/repo

Create a directory for your oe-core setup to live in and change location to it:

    mkdir oe-core && cd oe-core

Fetch the manifest file

    repo init -u git@github.com:terbergautomotive/terberg-bsp-platform.git -b LinuxImageV2.8-terberg

Fetch meta information

    repo sync

Set path to templates:

    export TEMPLATECONF=${PWD}/layers/meta-mender-terberg/conf

Setup environment:

    source layers/openembedded-core/oe-init-build-env

To start build run:

    bitbake console-tdx-image

The interesting output is:

- tmp-glibc/deploy/images/colibri-imx7-emmc/Console-Image-colibri-imx7-emmc.sdimg.gz
    - used to provision eMMC with correct partition table and content
- tmp-glibc/deploy/images/colibri-imx7-emmc/u-boot.imx
    - U-boot binary which needs to be flashed separately on eMMC hardware partition
- tmp-glibc/deploy/images/colibri-imx7-emmc/Console-Image-colibri-imx7-emmc.mender
    - Mender artifact which is feed to the Mender client to perform an update. This is also the file you will upload to the Mender server.

Provisioning device
-------------------

Prerequisite:

- Colibri Evaluation V3.2B carrier board
- Colibri iMX7D 1GB V1.1A SOM inserted in above
- USB cable connected to console port (X27) and PC
- USB cable connected to OTG USB port (X29) and PC
- Ethernet cable connected to RJ45 (X17) on same network as PC
- [TFT server](https://help.ubuntu.com/community/TFTP) running on PC
- Copy `u-boot.imx` and `Console-Image-colibri-imx7-emmc.sdimg.gz` to your TFT server directory

Put SoM in recovery mode based on instruction found on [Toradex developer page](https://developer.toradex.com/knowledge-base/vfxx-recovery-mode#Colibri_Evaluation_Board_3132).

You should be able to see a new USB device enumerate on the PC if it was successful:

    $ lsusb | grep 15a2
    Bus 003 Device 011: ID 15a2:0076 Freescale Semiconductor, Inc.

Clone `flash-utils`:

    git clone https://github.com/terbergautomotive/imx7-flash-utils.git

Change directory to `imx7-flash-utils`

    cd imx7-flash-utils

Copy `u-boot.imx` which was part of the Yocto/OE-core build output to `imx7-flash-utils`

    cp ../oe-core/build/tmp-glibc/deploy/images/colibri-imx7-emmc/u-boot.imx .

Transfer `u-boot.imx` to RAM and execute it. NOTE! This requires you to halt the boot process in U-boot otherwise it will continue with the normal boot flow and requires a level of swiftness since the bootdelay is only 2 seconds:

    sudo ./imx_usb u-boot.imx

You should now have the U-boot prompt on your serial console:

    Colibri iMX7 #

The following commands are all executed from the U-boot prompt.

Setup convenience script to calculate number of blocks based on file size:

    setenv set_blkcnt 'setexpr blkcnt ${filesize} + 0x1ff && setexpr blkcnt ${blkcnt} / 0x200'

Setup TFT server IP address in U-boot:

    setenv serverip 10.245.33.120

Setup client IP address in U-boot:

    setenv ipaddr 10.245.33.128

Transfer U-boot using TFTP and write to eMMC:

    tftpboot ${loadaddr} u-boot.imx && run set_blkcnt && mmc dev 0 1 && mmc write ${loadaddr} 2 ${blkcnt}

Transfer disk image to RAM using TFTP:

    tftpboot ${loadaddr} Console-Image-colibri-imx7-emmc.sdimg.gz

Write disk image to eMMC:

    gzwrite mmc 0 $loadaddr 0x$filesize

That is it, you can now power-cycle or reset the device and you should have
a Mender compatible image running on your device with the Mender client.

Please read the official [Mender documentation](https://docs.mender.io/) to understand how to connect your device to different server types and additional generic information about Mender functionality.

