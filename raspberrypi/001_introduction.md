resize partition:raspi-config --expand-rootfs  http://elinux.org/RPi_raspi-config

deb http://mirrors.aliyun.com/raspbian/raspbian/ jessie main contrib non-free rpi

https://www.pine64.org/


# how to install raspbian on raspberry

## download
https://www.raspberrypi.org/downloads/raspbian/

## install
https://www.raspberrypi.org/documentation/installation/installing-images/mac.md

    diskutil list
    diskutil unmountDisk /dev/disk2
    diskutil list
    sudo dd bs=1m if=/Users/wuyangchun/Downloads/2019-04-08-raspbian-stretch-lite.zip/2019-04-08-raspbian-stretch-lite.img of=/dev/rdisk2 conv=sync
    

## enbale sshd
touch /Volumes/boot/ssh

## login
pi/raspberry

## edit source list
https://mirrors.aliyun.com/raspbian/raspbian/

## setup wifi
pi@raspberrypi:~ $ sudo cat /etc/wpa_supplicant/wpa_supplicant.conf
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="CMCC-venus"
    psk="sa_1234567"
}

## add user
sudo adduser dev
sudo adduser dev sudo
sudo deluser pi

## Expand Filesystem
sudo raspi-config
advanced options -> Expand Filesystem

## support xfs, exfat
sudo apt-get install xfsprogs exfat-utils