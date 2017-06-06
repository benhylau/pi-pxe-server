# pi-pxe-server

![Screenshot](screenshot.png?raw=true)

## _What is this?_

These steps address a very specific use case I have—to configure a Raspberry Pi as a PXE server, pushing out a custom Arc-themed Debian distro containing the Arduino IDE, Inkscape, and Firefox, to a bunch of PCs in a classroom without touching their hard disks. Everything is loaded into and running from RAM, so the Raspberry Pi only needs to be wired with an ethernet cable during PXE boot. Each client has no locally persisted storage as the hard disk is not mounted, any project that needs to be saved during the workshop session are to be saved to a networked drive, and we get a fresh environment with each reboot.

If this happens to be your very specific use case as well, then you're in luck. There are three main parts:

* Building the custom Debian Live image (you need a machine with Debian Jessie, I just spun up a Digital Ocean Droplet for the hour)
* Building an [iPXE](http://ipxe.org) boot binary to chain-load the Debian Live image for clients that support UEFI-only (no support for legacy BIOS)
* Configuring a Raspberry Pi (I used a Pi 3 so it can get Internet with the on-board WiFi)

Compiling the custom Debian Live image takes about 15 minutes, and the iPXE boot binary takes about a minute. Booting the PC client with a Raspberry Pi 3 as PXE server (from Power-on to Desktop) takes about 100 seconds. The steps are easily adaptable to other single-board computers, and transfer of the 560 MB squashfs (squashed file system) is many times faster on a board with gigabit ethernet, making the overall PC client boot just under a minute. The PXE server works for all PC clients, regardless of BIOS or UEFI bootloaders, but does not boot Apple computers.

## Make a custom Debian Live image

1. Get a root shell and run the commands:

    ```
    # apt-get update
    # apt-get install -y live-build git
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
    # lb config --distribution stretch --archive-areas "main contrib non-free" --apt-recommends false
    ```

1. Copy theme files for [LightDM](https://wiki.debian.org/LightDM) and [Xfce](https://wiki.debian.org/Xfce):

    ```
    # git clone https://github.com/benhylau/pi-pxe-server.git ~/pi-pxe-server
    # cp -r ~/pi-pxe-server/config/includes.chroot/* config/includes.chroot/
    ```

1. Download the [Numix Circle](https://github.com/numixproject/numix-icon-theme-circle) icon package:

    ```
    # git clone https://github.com/numixproject/numix-icon-theme-circle.git ~/numix-icon-theme-circle
    # mkdir -p config/includes.chroot/usr/share/icons
    # cp -r ~/numix-icon-theme-circle/Numix-Circle config/includes.chroot/usr/share/icons/
    ```

1. The [Arduino IDE](http://playground.arduino.cc/Linux/Debian) requires the user to be in the `dialout` group.

    ```
    # mkdir -p config/includes.chroot/etc/live
    # echo 'LIVE_USER_DEFAULT_GROUPS="audio cdrom dip floppy video plugdev netdev powerdev scanner bluetooth dialout root"' > config/includes.chroot/etc/live/config.conf
    ```

1. Select the recommended and custom packages I need, then build the custom Debian Live image:

    ```
    # echo "live-tools user-setup sudo eject" > config/package-lists/recommends.list.chroot
    # echo "firmware-iwlwifi cifs-utils gvfs-backends network-manager-gnome" > config/package-lists/network.list.chroot
    # echo "task-xfce-desktop arc-theme numix-icon-theme" > config/package-lists/desktop.list.chroot
    # echo "firefox-esr inkscape arduino" > config/package-lists/applications.list.chroot
    # lb build
    ```

1. Copy **live-image-amd64.hybrid.iso** to your local machine

The default username is **user** with password **live**. Before building the next image, run `lb clean` or `lb clean --purge`. See the [Debian Live Manual](https://debian-live.alioth.debian.org/live-manual/stable/manual/html/live-manual.en.html) for details.

## Make a custom iPXE boot binary

1. On the same machine you built the custom Debian Live image:

    ```
    # apt-get install -y isolinux
    # git clone git://git.ipxe.org/ipxe.git ~/ipxe
    # cd ~/ipxe/src
    ```

1. Create **chain.ipxe** with the following content, with appropriate timezone:

    ```
    #!ipxe

    dhcp
    initrd tftp://192.168.0.2/initrd.img
    chain tftp://192.168.0.2/vmlinuz initrd=initrd.img boot=live components timezone=America/Toronto hooks=http://192.168.0.2/create-autostart fetch=http://192.168.0.2/live/filesystem.squashfs
    ```

1. Build the iPXE boot binary:

    ```
    # make bin-x86_64-efi/ipxe.efi EMBED=chain.ipxe
    ```

1. Copy **ipxe.efi** from **bin-x86_64-efi/** to your local machine:

## Set up DHCP and TFTP on a Raspberry Pi

1. Get a fresh install of [Raspbian Jessie Lite](https://www.raspberrypi.org/downloads/raspbian/) with SSH enabled, then copy your Debian Live **live-image-amd64.hybrid.iso** and iPXE boot binary **ipxe.efi**:

    ```
    $ scp /local/path/to/live-image-amd64.hybrid.iso pi@raspberrypi.local:/home/pi/
    $ scp /local/path/to/ipxe.efi pi@raspberrypi.local:/home/pi/
    ```

1. SSH to the Raspberry Pi and get a root shell:

    ```
    $ ssh pi@raspberrypi.local
    pi@raspberrypi:~ $ sudo -i
    ```

1. Get [dnsmasq](https://packages.debian.org/stretch/dnsmasq) for DHCP and TFPT, [apache2](https://packages.debian.org/stretch/apache2) for HTTP, [pxelinux](https://packages.debian.org/stretch/pxelinux) and [syslinux-common](https://packages.debian.org/stretch/syslinux-common) for PXE booting:

    ```
    # apt-get update
    # apt-get install -y dnsmasq apache2 pxelinux syslinux-common
    ```

1. To assign IP addresses and serve TFTP over the `eth0` wired network interface, append the following to the bottom of **/etc/dnsmasq.conf**:

    ```
    # PXE server configurations
    interface=eth0
    dhcp-range=192.168.0.3,192.168.0.253,255.255.255.0,1h
    dhcp-match=set:x86_BIOS,option:client-arch,0
    dhcp-match=set:x86_UEFI,option:client-arch,6
    dhcp-match=set:x64_UEFI,option:client-arch,7
    dhcp-match=set:x64_UEFI,option:client-arch,9
    dhcp-boot=tag:x86_BIOS,bios/pxelinux.0
    dhcp-boot=tag:x86_UEFI,uefi/ipxe.efi
    dhcp-boot=tag:x64_UEFI,uefi/ipxe.efi
    enable-tftp
    tftp-root=/srv/tftp
    ```

1. Configure TFTP directories serving clients with BIOS and UEFI boot loaders:

    ```
    # mkdir -p /srv/tftp/bios/pxelinux.cfg
    # ln -s /usr/lib/PXELINUX/pxelinux.0 /srv/tftp/bios/pxelinux.0
    # ln -s /usr/lib/syslinux/modules/bios/ldlinux.c32 /srv/tftp/bios/ldlinux.c32
    # mkdir -p /srv/tftp/uefi
    # cp /home/pi/ipxe.efi /srv/tftp/uefi/
    ```

1. Add a default entry to **/srv/tftp/bios/pxelinux.cfg/default** like this, with appropriate timezone:

    ```
    DEFAULT live

    LABEL live
    kernel tftp://192.168.0.2/vmlinuz
    append initrd=tftp://192.168.0.2/initrd.img boot=live components timezone=America/Toronto hooks=http://192.168.0.2/create-autostart fetch=http://192.168.0.2/live/filesystem.squashfs
    ```

1. Copy the Debian Live `.iso` file to the Raspberry Pi, then mount to extract files into the TFTP and HTTP directories:

    ```
    # mkdir -p /media/cdrom
    # mount -o loop /home/pi/live-image-amd64.hybrid.iso /media/cdrom
    # cp /media/cdrom/live/vmlinuz /srv/tftp/
    # cp /media/cdrom/live/initrd.img /srv/tftp/
    # mkdir -p /var/www/html/live
    # cp /media/cdrom/live/filesystem.* /var/www/html/live/
    # umount /media/cdrom
    ```

1. I need to automatically run a script when the live user auto-login to the live system. This is accomplished by inserting an autostart entry **/etc/xdg/autostart/autostart.desktop** that runs **/opt/autostart** from the live filesystem. This is accomplished by creating the following **/var/www/html/create-autostart** script, served over HTTP, and executed at boot time as a [live-config](https://packages.debian.org/stretch/live-config) hook:

    ```
    #!/bin/bash

    # Set these according to your needs
    WIFI_SSID="MY_WIFI_SSID"
    WIFI_PASSWORD="MY_WIFI_PASSWORD"
    SMB_SERVER_IP="MY_SMB_SERVER_IP"
    SMB_WORKGROUP="MY_SMB_WORKGROUP"
    SMB_USER="MY_SMB_USER"
    SMB_PASSWORD="MY_SMB_PASSWORD"
    SMB_REMOTE_PATH="/MY/SMB/REMOTE/PATH"
    SMB_LOCAL_MOUNT="/MY/SMB/LOCAL/MOUNT"

    # Create autostart entry
    mkdir -p /etc/xdg/autostart
    {
      echo "[Desktop Entry]"
      echo "Version=1.0"
      echo "Type=Application"
      echo "Name=Autostart"
      echo "Exec=/opt/autostart"
      echo "Terminal=false"
      echo "StartupNotify=false"
    } > /etc/xdg/autostart/autostart.desktop

    # Create autostart script
    {
      echo "#!/bin/bash"
      echo ""
      echo "# Set user directory ownership"
      echo "sudo chown -R user:user /home/user"
      echo ""
      echo "# Disable sleeps and suspensions"
      echo "sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target"
      echo ""
      echo "# Connect to WiFi"
      echo "sleep 10"
      echo "nmcli device wifi connect \"${WIFI_SSID}\" password \"${WIFI_PASSWORD}\""
      echo "nmcli device disconnect \$(nmcli device status | grep ethernet | awk '{print \$1}')"
      echo ""
      echo "# Mount network file system"
      echo "sudo mkdir -p ${SMB_LOCAL_MOUNT}"
      echo "echo \"//${SMB_SERVER_IP}${SMB_REMOTE_PATH} ${SMB_LOCAL_MOUNT} cifs credentials=/home/user/.smbcredentials,uid=1000,iocharset=utf8 0 0\" | sudo tee --append /etc/fstab"
      echo "sudo mount -a"
    } > /opt/autostart
    chmod +x /opt/autostart

    # Ignore laptop lid close
    {
      echo "HandleLidSwitch=ignore"
      echo "HandleLidSwitchDocked=ignore"
    } >> /etc/systemd/logind.conf

    # Disable light-locker
    echo "Hidden=true" >> /etc/xdg/autostart/light-locker.desktop

    # Configure SMB credentials
    {
      echo "workgroup=${SMB_WORKGROUP}"
      echo "username=${SMB_USER}"
      echo "password=${SMB_PASSWORD}"
    } > /home/user/.smbcredentials
    chmod 600 ~/.smbcredentials

    # Configure Firefox
    {
      echo "// Custom Firefox config"
      echo "pref(\"general.config.filename\", \"mozilla.cfg\");"
      echo "pref(\"general.config.obscure_value\", 0);"
    } > /usr/lib/firefox-esr/defaults/pref/autoconfig.js
    {
      echo "// Custom Firefox config"
      echo "lockPref(\"app.update.enabled\", false);"
      echo "lockPref(\"app.update.auto\", false);"
      echo "lockPref(\"app.update.mode\", 0);"
      echo "lockPref(\"app.update.service.enabled\", false);"
      echo "clearPref(\"extensions.lastAppVersion\");"
      echo "pref(\"browser.rights.3.shown\", true);"
      echo "pref(\"browser.startup.homepage_override.mstone\",\"ignore\");"
      echo "defaultPref(\"browser.startup.homepage\",\"data:text/plain,browser.startup.homepage=https://www.google.com\");"
      echo "lockPref(\"plugins.hide_infobar_for_outdated_plugin\", true);"
      echo "clearPref(\"plugins.update.url\");"
      echo "lockPref(\"datareporting.healthreport.service.enabled\", false);"
      echo "lockPref(\"datareporting.policy.dataSubmissionEnabled\", false);"
      echo "lockPref(\"toolkit.crashreporter.enabled\", false);"
      echo "Components.classes[\"@mozilla.org/toolkit/crash-reporter;1\"].getService(Components.interfaces.nsICrashReporter).submitReports = false;"
    } > /usr/lib/firefox-esr/mozilla.cfg
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

1. Reboot your Raspberry Pi, configure your PC client for PXE booting, then connect them with an ethernet cable

## BONUS: Build a portable Operating System Dispenser

![Operating System Dispenser](osd.jpg?raw=true)

This particular one uses an [ODROID C2 running Armbian](https://www.armbian.com/odroid-c2/) and a [D-Link GO-SW-5G gigabit switch](http://ca.dlink.com/products/connect/5-port-gigabit-easy-desktop-switch/) that draws 1 A at 5 V, so it's powered by the USB port of the C2 directly with a custom power cable. With gigabit ethernet, the 560 MB **filesystem.squashfs** gets transferred in seconds, and a PC client boots just under a minute. The 5-port switch allows multiple PC clients to be booted simultaneously.

Laser cut templates are include in the **osd** directory. Red lines are engravings and black lines cut through on 3mm acrylic. All mount holes fit M3 screws, but for the Raspberry Pi use M2.5 screws for mounting the board. Both the small and large cases should work with the Raspberry Pi 2 and 3, as well as the ODROID C2. The large case adds additional room for the D-Link switch and a bottom layer to hide all the extra cables.
