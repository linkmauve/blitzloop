# BlitzLoop on a Raspberry Pi

BlitzLoop can be installed on a Raspberry Pi, for what might possibly be the
world's smallest and cheapest full-featured karaoke box.

Normally, BlitzLoop runs on OpenGL on GLX on Xorg, and uses mpv with the
opengl-cb backend. The Raspberry Pi does not work efficiently with this setup.
However, BlitzLoop also supports a native Raspberry Pi backend, which uses EGL
directly without Xorg, and takes advantage of the hardware compositor to display
karaoke lyrics on a layer above the video playback layer, provided by mpv's rpi
backend. This guide focuses on this specific Raspberry Pi support.

## Prerequisites

* Raspberry Pi 3
* Arch Linux ARM

BlitzLoop has only been tested on a Raspberry Pi 3. Previous models may not be
powerful enough to work properly. Feel free to try.

## Installation

### Base OS

Follow the [Arch Linux ARM installation guide](https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-3)
for Raspberry Pi 3 (32bit version).

It is assumed that you are using the default `alarm` user to run BlitzLoop.
Additionally, you might want to change the password for `alarm` and/or `root`,
set up SSH keys, etc.

Make sure your Ethernet network is properly configured and you have internet
access (Arch Linux ARM uses DHCP by default), at least during installation.
We will be using WiFi as an AP for the client remote control devices to connect
to; it is possible to configure WiFi as a client to an existing network, but
that is beyond the scope of this guide.

This guide assumes that you have `sudo` installed and configured. If you don't,
run the commands prefixed with `sudo` as root (e.g. `su`).

### Set boot options

```shell
cat <<EOF | sudo tee /boot/config.txt > /dev/null
gpu_mem=256
disable_overscan=1
dtparam=audio=on
EOF
```

Reboot.

### Configure environment

Set the locale to something with UTF-8:

```shell
echo 'en_US.UTF-8 UTF-8' | sudo tee /etc/locale.gen
sudo locale-gen
echo 'LANG=en_US.UTF-8' | sudo tee /etc/locale.conf
```

Grant `alarm` audio access and real-time priority for JACK:

```shell
sudo usermod -aG audio alarm
echo 'alarm - rtprio 99' | sudo tee -a /etc/security/limits.conf
```

Log out and back in.

### Update system

```shell
sudo pacman -Syu
```

### Install dependencies

```shell
sudo pacman -S --needed base-devel jack ffms2 freetype2 python-numpy python-pillow python-opengl alsa-tools alsa-utils wget
```

We need to build mpv-rpi and ffmpeg-mmal from AUR. This will take a while:

```shell
sudo pacman -S --needed hardening-wrapper ladspa yasm python-docutils enca
mkdir -p ~/pkg; cd ~/pkg
wget https://aur.archlinux.org/cgit/aur.git/snapshot/ffmpeg-mmal.tar.gz
wget https://aur.archlinux.org/cgit/aur.git/snapshot/mpv-rpi.tar.gz
tar xvzf ffmpeg-mmal.tar.gz
tar xvzf mpv-rpi.tar.gz
cd ~/pkg/ffmpeg-mal
MAKEFLAGS="-j4" makepkg --skippgpcheck
sudo pacman -U ffmpeg-mmal-*.pkg.tar.xz
cd ~/pkg/mpv-rpi
MAKEFLAGS="-j4" makepkg --skippgpcheck
sudo pacman -U mpv-rpi-*.pkg.tar.xz
```

### Install and configure BlitzLoop

```shell
cd ~
python -m venv blitz --system-site-packages
source blitz/bin/activate
pip install 'git+git://github.com/marcan/blitzloop.git'
mkdir -p ~/.blitzloop
cat <<EOF >~/.blitzloop/cfg
display=rpi
fps=60
mpv-vo=rpi
EOF
```

### Get songs

Put your songs in `/home/alarm/.blitzloop/songs/` (create it first).

### Test out BlitzLoop

Run:

```shell
cd ~
source blitz/bin/activate
blitzloop
```

Connect to TCP port 10111 and check to see if everything works properly.
Note that you will not have any microphone support at this point, as the
Raspberry Pi has no audio input.

If you have no audio, you may need to run `amixer cset numid=3 2`.

## Systemd service

```shell
cat >~/startblitz.sh <<EOF
#!/bin/sh
cd "\$(dirname "\$0")"
amixer cset numid=3 2
source blitz/bin/activate
blitzloop --port=80
EOF
chmod +x ~/startblitz.sh

cat <<EOF | sudo tee /etc/systemd/system/blitzloop.service > /dev/null
[Unit]
Description=BlitzLoop

[Service]
Type=simple
User=alarm
Group=alarm
ExecStart=/home/alarm/startblitz.sh
AmbientCapabilities=CAP_NET_BIND_SERVICE
LimitRTPRIO=99
LimitMEMLOCK=1024000000

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable blitzloop
sudo systemctl start blitzloop
```

## WiFi setup

Set up a WiFi hotspot for tablets to connect to:

```shell
sudo pacman -S --needed dnsmasq hostapd iptables

cat <<EOF | sudo tee /etc/hostapd/hostapd.conf >/dev/null
interface=wlan0
ssid=BlitzLoop
hw_mode=g
channel=6
wmm_enabled=1
EOF
sudo systemctl enable hostapd
sudo systemctl start hostapd

cat <<EOF | sudo tee /etc/dnsmasq.conf >/dev/null
no-resolv
server=8.8.8.8
address=/blitz/192.168.42.1
address=/bl/192.168.42.1
address=/b/192.168.42.1
interface=wlan0
dhcp-range=192.168.42.100,192.168.42.200,12h
EOF

sudo systemctl enable dnsmasq
sudo systemctl start dnsmasq

cat <<EOF | sudo tee /etc/systemd/network/hostap.network >/dev/null
[Match]
Name=wlan0

[Network]
Address=192.168.42.1/24
IPMasquerade=yes
EOF

sudo systemctl restart systemd-networkd.service
```

Now you can connect to the `BlitzLoop` SSID and browse to http://b/.
