# Step 1: Flash eMMC with etcher

Download etcher from this [page](https://www.balena.io/etcher/).

Download desired ubuntu minimal image from [here](https://wiki.odroid.com/odroid-xu4/os_images/linux/ubuntu_4.14/ubuntu_4.14).

For reference on how to flash the eMMC, see [odroid wiki guide](https://wiki.odroid.com/odroid-xu4/getting_started/os_installation_guide?redirect=1).

# Step 2: ssh into odroid 

On first boot, the odroid will run the root file system resizing process automatically. After resizing process, it will shutdown and the blue LED is off within 5~20 seconds.
Press power button again and the heartbeat blue LED keeps flashing.

Connect the odroid to the ethernet port of your router and check its IP address.

As the fresh install of Ubuntu Minimal includes only the root user (password: odroid), ssh into root@xxx.xxx.x.xxx where xxx.xxx.x.xxx is the IP address assigned to your odroid. 

Reference: [headless setup](https://wiki.odroid.com/odroid-xu4/application_note/software/headless_setup), [a better looking guide](https://github.com/AerialRobotics-IITK/Wiki/wiki/ODROID-XU4-Setup)

# Step 3: Create user odroid and change host name

### Add new user

In root, from the terminal, to create new user run this:

```
adduser odroid
```

add user to sudo group to grant sudo privileges. 

```
usermod -aG sudo odroid
```

### If there are multiple odroids connected in the same network, and the intention is to network over ROS. Then a unique hostname would be necessary. The default hostname is odroid.

Replace hostname with desired hostname

```
hostname <new_hostname>
```

Replace the old hostname with the new hostname in /etc/hostname

```
nano /etc/hostname
```

Rename all instances of the old hostname in /etc/hosts

```
nano /etc/hosts
```

Reference: [add new user](https://www.digitalocean.com/community/tutorials/how-to-add-and-delete-users-on-ubuntu-16-04), [host name guide](https://github.com/olinrobotics/Odroid_Setup/wiki/Changing-the-Hostname)

# Step 4: Update packages

### From this point onwards, always log into user: odroid

update packages by running the following code in terminal:
```
sudo apt update
sudo apt upgrade
sudo apt dist-upgrade
shutdown -r now
```
# Step 5: Setup WiFi and Odroid's WiFi module 5a

If you downloaded the image from odroid that uses the 4.14 Kernel, that's a chance that 5a might not work out of the box. This is because there are a few files missing. To solve this, it is necessary to rebuild and install the 4.14 Kernel natively. The following steps are taken verbatim from Odroid's kernel build guide. 

### NOTE! This is guide is for <u>NATIVE BUILD</u> ONLY - run it on the board

**Installing building tools**

You have to install some building tools and related packages.

```bash
sudo apt install git gcc g++ build-essential libssl-dev bc
```
Download and build the kernel source.

**Updating Kernel and DTB (Device Tree Blob)**

Please note that native kernel compile on ODROID-XU4 will take about 25 minutes.

```
git clone --depth 1 https://github.com/hardkernel/linux -b odroidxu4-4.14.y
cd linux
make odroidxu4_defconfig
make -j8
sudo make modules_install
sudo cp -f arch/arm/boot/zImage /media/boot
sudo cp -f arch/arm/boot/dts/exynos5422-odroidxu3.dtb /media/boot
sudo cp -f arch/arm/boot/dts/exynos5422-odroidxu4.dtb /media/boot
sudo cp -f arch/arm/boot/dts/exynos5422-odroidxu3-lite.dtb /media/boot
sync
```

**Updating root ramdisk (Optional)**
```
sudo cp .config /boot/config-`make kernelrelease`
sudo update-initramfs -c -k `make kernelrelease`
sudo mkimage -A arm -O linux -T ramdisk -C none -a 0 -e 0 -n uInitrd -d /boot/initrd.img-`make kernelrelease` /boot/uInitrd-`make kernelrelease`
sudo cp /boot/uInitrd-`make kernelrelease` /media/boot/uInitrd
sync
```

**Before you start with new Linux kernel v4.14**

You would check all necessary files are in place as below before reboot. The file size would differ.

```
ls -l /media/boot/
total 14756
-rwxr-xr-x 1 root root    9536 Oct 25 23:29 boot.ini
-rwxr-xr-x 1 root root     753 Aug 20 22:38 boot.ini.default
-rwxr-xr-x 1 root root   62565 Nov  2 01:24 exynos5422-odroidxu3.dtb
-rwxr-xr-x 1 root root   61814 Nov  2 01:24 exynos5422-odroidxu3-lite.dtb
-rwxr-xr-x 1 root root   62225 Nov  2 01:24 exynos5422-odroidxu4.dtb
-rwxr-xr-x 1 root root   61714 Oct 25 23:30 exynos5422-odroidxu4-kvm.dtb
-rwxr-xr-x 1 root root 9996513 Nov  2 01:27 uInitrd
-rwxr-xr-x 1 root root 4844744 Nov  2 01:24 zImage
```
```
sudo sync
sudo reboot
```
**Check WIFI module**

After restarting, plug in the WIFI module 5a and run the following command to see if it is recognized. 

The drivers should be built into the kernel. 

```bash
iwcofig
```

If the WIFI module is recognized, **log into root** and configure the Odroid to connect to WiFi on boot by writing the following configuration to `/etc/network/interfaces`. Make sure to change `YOUR-WIFI-SSID` and `YOUR-WIFI-PASSWORD` to your WiFi's ssid and and password respectively.

```bash
echo -e "\nallow-hotplug wlan0 \niface wlan0 inet dhcp \n\twpa-ssid YOUR-WIFI-SSID\n\twpa-psk YOUR-WIFI-PASSWORD" >> /etc/network/interfaces
```

Then reboot and the odroid should connect to the wifi on boot without any user logging in.

```bash
reboot
```

Once booted, log in and try to ping the outside world.

```bash
ping google.com
```

Reference: [Odroid Kernel Build Guide](https://wiki.odroid.com/odroid-xu4/software/building_kernel), [Olin Robotics Wifi Guide](https://github.com/olinrobotics/Odroid_Setup/wiki/Configuring-a-WiFi-Dongle)

# Step 6: Disable GUI if ubuntu mate was installed instead of ubuntu minimal

Since systemd, the OS always boots into a so-called "target". On GUI systems, the default boot target is `graphical.target`, whereas on systems without any DE or graphical environment, it is called `multi-user.target`.

The proper way to make the machine boot only into tty, without any GUI, is to switch the default target:

```bash
systemctl set-default multi-user.target
```

To check what is the current default target:

```bash
systemctl get-default
```

To make the system boot into the GUI, you can just revert it by switching back to `graphical.target`:

```
systemctl set-default graphical.target
```

Reference: [boot headless guide](https://superuser.com/questions/1450916/how-to-boot-debian-headless-without-gui)

# Step 7: Install OpenCV and ROS

Please follow the reference link below

Reference: [OPENCV/ROS setup guide for 16.04](https://github.com/chahatdeep/ubuntu-for-robotics/tree/master/OdroidXU4)

# Special Notes

### Changing console font size
Reference: [change font size](https://www.ostechnix.com/how-to-change-linux-console-font-type-and-size/)

# ~Step 5: Setup WiFi and Odroid's WiFi module 5a~ (depreciated)

### ~Automatically rebuilds and installs on kernel updates. DKMS is in official sources of Ubuntu, for installation do:

```
sudo apt-get install build-essential dkms git
```
### Clone the driver source from GitHub and the driver source must be copied to /usr/src/8812au-4.2.2
```
git clone https://github.com/gnab/rtl8812au
sudo cp -a rtl8812au /usr/src/8812au-4.2.2
```

### Then add it to DKMS:
```
sudo dkms add -m 8812au -v 4.2.2
sudo dkms build -m 8812au -v 4.2.2
sudo dkms install -m 8812au -v 4.2.2
```

Reference: [5a wifi driver](https://wiki.odroid.com/odroid-h2/application_note/howto_wifi_driver_rtl8812au)