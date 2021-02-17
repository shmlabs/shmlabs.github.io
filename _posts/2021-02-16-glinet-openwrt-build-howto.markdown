---
layout: post
title:  "OpenWRT GL.iNet MT300N-V2 Firmware Build HOWTO [DRAFT]"
date:   2021-02-16 19:22:35 +0300
categories: jekyll update
---
This is a description on how to build your own router firmware.  You will need the following:

* Ubuntu 20.04 LTS 
* GL.iNet MT300N V2 Travel Router

## Preparing Your Build Environment

    cd ~

    sudo apt update
    sudo apt upgrade

    sudo apt install build-essentials libncurses5-dev gawk python git

    git clone https://git.openwrt.org/openwrt/openwrt.git

    cd openwrt

    git checkout v19.07.6

    ./scripts/feeds update -a
    ./scripts/feeds install -a

## Building Your Firmware

    cd ~/openwrt
    make menuconfig

* Set Target System → MediaTek Ralink MIPS
* Set Subtarget → MT76x8 based boards
* Set Target Profile → GL.iNet GL-MT300N V2

<!-- -->

    make

This could take awhile, so grab a cup of coffee and the netflix remote.  If there were no errors you can install the new firmware or continue with customization.

## Customization
In this section we are going to disable wireless, add the openvpn-openssl package, patch openvpn for obfuscation, and customize some configuration files for use with openvpn.

    cd ~/openwrt

    make menuconfig

Disable Wireless Deselect Kernel modules → Wireless Drivers → kmod-mt7603, kmod-mac80211, kmod-cfg80211
Enable OpenVPN Select Network → VPN → openvpn-openssl

Exit the menu when you are finished to save your new configuration.

## Patching OpenVPN to use Obfuscated Servers

    cd ~/openwrt

    wget https://github.com/Tunnelblick/Tunnelblick/archive/master.zip

    unzip master.zip

    cd Tunnelblick-master/third_party/sources/openvpn/openvpn-2.4.8/patches

    cp *xorpatch*.diff ~/openwrt/package/network/services/openvpn/patches

## Building Customized Firmware

    cd ~/openwrt
    make

## Installation

    scp ./bin/targets/ramips/mt76x8/openwrt-ramips-mt76x8-gl-mt300n-v2-squashfs-sysupgrade.bin root@192.168.8.1:/tmp

Log in to the device.

    ssh -l root 192.168.8.1

On the device, do the following:

    sysupgrade -n -v /tmp/openwrt-ramips-mt76x8-gl-mt300n-v2-squashfs-sysupgrade.bin 

## First Boot

Connect and set the root passwd.

    ssh -l root 192.168.8.1
    passwd

**System Setup** - Sets hostname and leds to show tunnel activity.

    uci set system.@system[0].hostname='OpenWrt'

    uci set system.led_wan.default='0'
    uci set system.led_wan.dev='tun0'
    uci set system.led_wan.mode='link tx'
    uci set system.led_wan.trigger='netdev'

    uci set system.led_wifi_led.default='0'
    uci set system.led_wifi_led.dev='tun0'
    uci set system.led_wifi_led.mode='link rx'
    uci set system.led_wifi_led.trigger='netdev'

    uci commit system


**Network Setup** - Sets LAN IP address, static DNS servers, and creates a tun0 interface for OpenVPN.

    uci set network.lan.ipaddr='192.168.8.1'

    uci set network.wan.peerdns='0'
    uci set network.wan.dns='8.8.8.8 8.8.4.4'

    uci set network.tunnel='interface'
    uci set network.tunnel.ifname='tun0'
    uci set network.tunnel.proto='none'

    uci commit network

**Firewall Setup** - Sets custom firewall routing and rules which will only forward traffic when the VPN is established.

    rm /etc/config/firewall
    touch /etc/config/firewall

    uci set firewall.defaults='defaults'
    uci set firewall.defaults.syn_flood='1'
    uci set firewall.defaults.input='ACCEPT'
    uci set firewall.defaults.output='ACCEPT'
    uci set firewall.defaults.forward='REJECT'

    uci add firewall zone
    uci set firewall.@zone[0].name='lan'
    uci set firewall.@zone[0].network='lan'
    uci set firewall.@zone[0].input='ACCEPT'
    uci set firewall.@zone[0].output='ACCEPT'
    uci set firewall.@zone[0].forward='REJECT'

    uci add firewall zone
    uci set firewall.@zone[1].name='wan'
    uci set firewall.@zone[1].network='wan'
    uci set firewall.@zone[1].input='ACCEPT'
    uci set firewall.@zone[1].output='ACCEPT'
    uci set firewall.@zone[1].forward='REJECT'

    uci add firewall zone
    uci set firewall.@zone[2].name='tunnel'
    uci set firewall.@zone[2].network='tunnel'
    uci set firewall.@zone[2].input='REJECT'
    uci set firewall.@zone[2].output='ACCEPT'
    uci set firewall.@zone[2].forward='REJECT'
    uci set firewall.@zone[2].masq='1'
    uci set firewall.@zone[2].mtu_fix='1'

    uci add firewall forwarding
    uci set firewall.@forwarding[0].src='lan'
    uci set firewall.@forwarding[0].dest='tunnel'

    uci add firewall rule
    uci set firewall.@rule[0].name='Allow-DHCP-Renew'
    uci set firewall.@rule[0].src='wan'
    uci set firewall.@rule[0].proto='udp'
    uci set firewall.@rule[0].dest_port='68'
    uci set firewall.@rule[0].target='ACCEPT'
    uci set firewall.@rule[0].family='ipv4'

    uci add firewall rule
    uci set firewall.@rule[1].name='Allow-Ping'
    uci set firewall.@rule[1].src='wan'
    uci set firewall.@rule[1].proto='icmp'
    uci set firewall.@rule[1].icmp_type='echo-request'
    uci set firewall.@rule[1].family='ipv4'
    uci set firewall.@rule[1].target='ACCEPT'

    uci commit firewall

**OpenVPN Client Setup** - You will need to add your client profile and credential here.

    cat <<EOF> /etc/openvpn/client.conf
    COPY_PASTE_CLIENT_PROFILE_HERE
    EOF

    chmod 600 /etc/openvpn/client.conf

    cat <<EOF> /etc/openvpn/client.auth
    OVPN_USERNAME
    OVPN_PASSWORD
    EOF

    chmod 600 /etc/openvpn/client.auth
    rm /etc/config/openvpn
    touch /etc/config/openvpn

    uci set openvpn.custom_config='custom_config'
    uci set openvpn.custom_config.enable='1'
    uci set openvpn.custom_config.config='/etc/openvpn/client.conf'

    uci commit openvpn

You must reboot the device for these changes to take effect.

    reboot

## Backup and Restore

**Backup**

    sysupgrade -b /tmp/backup-${HOSTNAME}-$(date +%F).tar.gz

**Restore**

    sysupgrade -r /tmp/backup-*.tar.gz

## Further Customization

Add the hotpug.d button script to the system.

    mkdir -p /etc/hotplug.d/button

    cat << "EOF" > /etc/hotplug.d/button/00-button

    source /lib/functions.sh

    do_button () {

        local button
        local action
        local handler
        local min
        local max

        config_get button "${1}" button
        config_get action "${1}" action
        config_get handler "${1}" handler
        config_get min "${1}" min
        config_get max "${1}" max

        [ "${ACTION}" = "${action}" -a "${BUTTON}" = "${button}" -a -n "${handler}" ] && {
            [ -z "${min}" -o -z "${max}" ] && eval ${handler}
            [ -n "${min}" -a -n "${max}" ] && {
                [ "${min}" -le "${SEEN}" -a "${max}" -ge "${SEEN}" ] && eval ${handler}
            }
        }
    }

    config_load system
    config_foreach do_button button

    EOF

    chmod 755 /etc/hotplug.d/button/00-button

create button action scripts to disable and enable openvpn.

    mkdir -p /etc/openvpn

    cat << "EOF" > /etc/openvpn/disable.sh
    /etc/init.d/openvpn stop
    uci set firewall.@zone[0].forward='ACCEPT'
    uci set firewall.@zone[1].input='REJECT'
    uci set firewall.@zone[1].masq='1'
    uci set firewall.@zone[1].mtu_fix='1'
    uci set firewall.@forwarding[0].dest='wan'
    /etc/init.d/firewall restart
    EOF

    chmod 750 /etc/openvpn/disable.sh
    cat << "EOF" > /etc/openvpn/enable.sh
    uci set firewall.@zone[0].forward='REJECT'
    uci set firewall.@zone[1].input='ACCEPT'
    uci set firewall.@zone[1].masq='0'
    uci set firewall.@zone[1].mtu_fix='0'
    uci set firewall.@forwarding[0].dest='tunnel'
    /etc/init.d/firewall restart
    /etc/init.d/openvpn start
    EOF

    chmod 750 /etc/openvpn/enable.sh

Set button handlers within the system settings.

    uci add system button
    uci set system.@button[0].button="BTN_0"
    uci set system.@button[0].action="pressed"
    uci set system.@button[0].handler="/etc/openvpn/enable.sh"

    uci add system button
    uci set system.@button[1].button="BTN_0"
    uci set system.@button[1].action="released"
    uci set system.@button[1].handler="/etc/openvpn/disable.sh"

    uci commit system

On boot, check to see what position the switch is in and run the appropriate script.

    cat <<"EOF"> /etc/rc.local

    # Put your custom commands here that should be executed once
    # the system init finished. By default this file does nothing.

    #######################################################
    # READ BTN_0 ("right") ON BOOT - TRIGGER BUTTON EVENT #
    #######################################################

    GPIO_FILE='/sys/kernel/debug/gpio'
    BUTTON='gpio-0'
    TEST=`/bin/cat "${GPIO_FILE}" | /bin/grep "${BUTTON}" | /bin/grep -c hi`

    if [ "$TEST" = 1 ]; then
       logger -t OPENVPN "${BUTTON} is returning HI - disable tunnel."
       /etc/openvpn/disable.sh 2>&1 | logger -t OPENVPN
    else
       logger -t OPENVPN "${BUTTON} is returning LO - enable tunnel."
       /etc/openvpn/enable.sh 2>&1 | logger -t OPENVPN
    fi

    #######################################################

    exit

    EOF


