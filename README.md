# Using LineageOS with Jetson Nano Baseboard and Jetson TX2 NX

This guide provides instructions for building and running the [LineageOS](https://lineageos.org/) Android distribution on Antmicro's open hardware [Jetson Nano Baseboard](https://github.com/antmicro/jetson-nano-baseboard) with NVIDIA Jetson TX2 NX SoM.

![Photo of LineageOS running on Antmicro's Jetson Nano Baseboard](Android-on-JNB.png)

## Prepare the workspace

First, initialize the workspace and install the necessary dependencies.
If you don’t want to use Docker, you can omit the line with the `docker run` command.

```bash
mkdir sources && pushd sources
docker run -v `pwd`/:/build -it debian:buster /bin/bash
export DEBIAN_FRONTEND="noninteractive"
export LINEAGE_VERSION="20.0"
export LINEAGE_BUILD="/build"
apt -qqy update
apt -qqy install procps sudo wget bc bison build-essential ccache curl flex \
                g++-multilib gcc-multilib git git-lfs gnupg gperf \
                imagemagick lib32ncurses5-dev lib32readline-dev lib32z1-dev \
                libelf-dev liblz4-tool libncurses5 libncurses5-dev \
                libsdl1.2-dev libssl-dev libxml2 libxml2-utils lzop \
                pngcrush rsync schedtool squashfs-tools xsltproc zip \
                zlib1g-dev p7zip-full libwxgtk3.0-gtk3-dev zstd
ln -s $(which python3) /usr/bin/python
```

Download tools necessary for downloading and building the sources.

```bash
pushd $LINEAGE_BUILD
wget https://dl.google.com/android/repository/platform-tools-latest-linux.zip
unzip platform-tools-latest-linux.zip
mkdir -p ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
export PATH=$PATH:`pwd`/platform-tools:~/bin
```

## Download the sources

The AOSP tree sources are going to be initialized using the Android repo tool.
The version of LineageOS depends on the branch of the manifest file passed to the `repo init` command.
Available versions include:

* lineage-16.0
* lineage-17.1
* lineage-18.1
* lineage-19.1
* lineage-20.0

```bash
pushd $LINEAGE_BUILD
git config --global user.email "<email>"
git config --global user.name "<Name>"
repo init -u https://github.com/LineageOS/android.git -b lineage-20.0 --git-lfs
repo sync
```

### Apply a patch

When building version 19.1 or higher, LineageOS needs a patch to be applied for the build to succeed.
The patch fixes the a/b zip generation failure and should be applied in the `$LINEAGE_BUILD/build/make` directory.
The patch can be found [here](https://review.lineageos.org/c/LineageOS/android_build/+/357489).

## Build the sources

```bash
pushd $LINEAGE_BUILD
source build/envsetup.sh

# This command is going to fail, but is important to run, as it initializes the target quill directory
breakfast quill
pushd device/nvidia/quill
m otatools
./extract-files.sh
popd
breakfast quill
croot
brunch quill
```

When the build succeeds, artifacts should be produced in the `$OUT` directory, which defaults to `out/target/product/quill_tab`.
The LineageOS zip installer package should be called `lineage-$LINEAGE_VERSION-*-UNOFFICIAL-quill.zip`.

## Flash the device

The device should first be flashed with a LineageOS recovery image and then can be flashed with the prepared LineageOS image.
The flash package can be downloaded from [here](https://www.androidfilehost.com/?w=files&flid=328892).
It is important to download a flash package in the same version as the LineageOS image we want to flash.

The flashing script checks compatibility with the board and the module, and assumes that the board being flashed is the NVIDIA Jetson Xavier NX baseboard.
That compatibility check fails with Antmicro's Jetson Nano Baseboard, which is why we need to skip it.

```bash
pushd flash_package
sed -i "s/TARGET_CARRIER_ID=.*;/TARGET_CARRIER_ID=;/" flash.sh
popd
```

Now the script is ready to flash.
Connect your device to the host PC and put the device in recovery mode.
Next, flash the device using the `flash.sh` script.

```bash
pushd flash_package
sudo ./flash.sh
```

When the script flashes the recovery image successfully, a LineageOS recovery image menu should be displayed on the monitor connected through HDMI.
To flash the LineageOS image, choose the `Apply update` -> `Apply from ADB` option.
Then, sideload the image using Android Debug Bridge (ADB).

```bash
adb sideload <package.zip>
```

When the image flashes successfully, reboot the device by choosing `Reboot system now`.
