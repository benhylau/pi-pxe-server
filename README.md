# pi-pxe-server

## _What is this?_

These steps address a very specific use case I have—to configure a Raspberry Pi as a PXE server, pushing out a custom Arc-themed Debian distro containing the Arduino IDE, Inkscape, and Firefox, to a bunch of PCs in a classroom without touching their hard disks. Everything is loaded into and running from RAM, so the Raspberry Pi only needs to be wired with an ethernet cable during PXE boot. Each client has no locally persisted storage as the hard disk is not mounted, any project that needs to be saved during the workshop session are to be saved to a networked drive, and we get a fresh environment with each reboot.

If this happens to be your very specific use case as well, then you're in luck. There are two main parts:

* Building the custom Debian image (you need a machine with Debian Jessie, I just spun up a Digital Ocean Droplet for the hour)
* Configuring a Raspberry Pi (I used a Pi 3 so it can get Internet with the on-board WiFi)

## Make A Custom Debian Live Image

1. Get a root shell and run the commands:

    ```
    # apt-get install -y live-build
    # mkdir -p debian-jessie-amd64/auto
    # cd debian-jessie-amd64/
    # cp /usr/share/doc/live-build/examples/auto/* auto/
    # rm auto/config
    ```

1. Create **auto/config** with the following content:

    ```
    #!/bin/sh

    lb config noauto \
    	--architectures amd64 \
    	--linux-flavours 686-pae \
    	"${@}"
    ```

1. Create the initial configuration tree without recommended packages:

    ```
    # lb config --apt-recommends false
    ```

1. The [Arduino IDE](http://playground.arduino.cc/Linux/Debian) requires the user to be in the `dialout` group.

    ```
    # mkdir -p config/includes.chroot/etc/live/config/
    # echo 'LIVE_USER_DEFAULT_GROUPS="audio cdrom dip floppy video plugdev netdev powerdev scanner bluetooth dialout root"' > config/includes.chroot/etc/live/config/user-setup.conf
    ```

1. Add the [Arc Theme](https://github.com/horst3180/arc-theme) Debian repository:

    ```
    # echo 'deb http://download.opensuse.org/repositories/home:/Horst3180/Debian_8.0/ /' > config/archives/arc-theme.list.chroot
    # wget -nv https://download.opensuse.org/repositories/home:Horst3180/Debian_8.0/Release.key -O config/archives/arc-theme.key.chroot
    ```

1. Remove hook that somehow would've caused the build to fail:

    ```
    # rm config/hooks/0400-update-apt-file-cache.hook.chroot
    ```

1. Select the recommended and custom packages I need, then build the live image:

    ```
    # echo "live-tools user-setup sudo eject" > config/package-lists/recommends.list.chroot
    # echo "task-xfce-desktop arc-theme iceweasel inkscape arduino" > config/package-lists/my.list.chroot
    # lb build
    ```

The default username is **user** with password **live**. Before building the next image, run `lb clean` or `lb clean --purge`. See the [Debian Live Manual](https://debian-live.alioth.debian.org/live-manual/stable/manual/html/live-manual.en.html) for details.

## Set Up DHCP and TFTP on a Raspberry Pi

1. Get **dnsmasq** for DHCP and TFPT, and **syslinux-common** for serving `.iso` images:

    ```
    $ sudo apt-get update
    $ sudo apt-get install -y dnsmasq syslinux-common
    ```

1. To assign IP addresses and serve TFTP over the `eth0` wired network interface, append the following to the bottom of **/etc/dnsmasq.conf**:

    ```
    # PXE server configurations
    interface=eth0
    dhcp-range=192.168.0.3,192.168.0.253,255.255.255.0,1h
    dhcp-boot=pxelinux.0,pxeserver,192.168.0.2
    pxe-service=x86PC, "Install Linux", pxelinux
    enable-tftp
    tftp-root=/srv/tftp
    ```

1. Configure the TFTP directory serving the images:

    ```
    $ sudo mkdir -p /srv/tftp/pxelinux.cfg
    $ sudo mkdir -p /srv/tftp/iso
    $ sudo cp /usr/lib/syslinux/memdisk /srv/tftp/
    $ sudo cp /usr/lib/syslinux/modules/bios/ldlinux.c32 /srv/tftp/
    $ sudo wget http://ftp.nl.debian.org/debian/dists/jessie/main/installer-amd64/current/images/netboot/pxelinux.0 -O /srv/tftp/pxelinux.0
    ```

1. Place your `.iso` live image file in **/srv/tftp/iso/** and add an entry to **/srv/tftp/pxelinux.cfg/default** like this:

    ```
    LABEL my_iso_name
    kernel memdisk
    append iso initrd=iso/my.iso
    ```

1. Change the `eth0` block in **/etc/network/interfaces** to use a static IP:

    ```
    auto eth0
    iface eth0 inet static
        address 192.168.0.2
        netmask 255.255.255.0
    ```

1. Since we are only assigning IP addresses on the `eth0` interface, you can optionally use the Raspberry Pi 3's on-board WiFi to connect your PXE server to your home wireless network:

    1. Add `auto wlan0` to the `wlan0` block in **/etc/network/interfaces**:

        ```
        auto wlan0
        allow-hotplug wlan0
        iface wlan0 inet manual
            wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
        ```

    1. Append your home network details to **/etc/wpa_supplicant/wpa_supplicant.conf** with something like:

        ```
        network={
            ssid="MY_SSID"
            psk="MY_PASSWORD"
        }
        ```

1. Reboot your Raspberry Pi and connect it to the PC client configured for PXE booting using an ethernet cable
