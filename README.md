# Turn Your Raspberry Pi Fleet Into a Bluetooth and Wifi Multi Room Audio System
This tutorial will show you how to setup your Raspberry Pis in order to build a multi room audio system working over the wifi network. The audio will be shared to this system through bluetooth

The best Rapsberry Pi to be used is the Raspberry Pi 3 Model A+, running Raspberry Pi OS Lite.
This document is using the default `pi` user as an example.

## Initial setup of Raspberry Pi OS Lite
Flash the OS to a SD card with balenaEtcher.
In order to run it headless (i.e. without any screen), place in the `boot` partition a file called `wpa_supplicant.conf` with the following data:
```
country=FR
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="WIFI_SSID"
    scan_ssid=1
    psk="WIFI_PASSWORD"
}
```
In addition to that, place a empty file called `ssh` in the very same partition, in order to enable ssh access

After the first boot, you will be able to access you raspberry pi via ssh:
```console
ssh pi@raspberrypi.lan
```
The default password is `raspberry`

## Setting up a Snapcast server
The snapcast server will be both the snapcast central server and a client to have it playing music as well. It will retreive its audio source from bluetooth like so:

```
Device > Bluetooth > Pulseaudio > Alsa Loopback sink > Alsa Loopback source > Snapcast server > Network > {snapcast clients}

network
├── > Snapcast Client 1 (the one ran on the same device than the server) > Alsa Headphone sink > Jack
├── > Snapcast Client 2 > Alsa Headphone sink > Jack
├── > Snapcast Client 3 > Alsa Headphone sink > Jack
└── > ...
```

### Dependancies
```console
sudo apt-get install pulseaudio pulseaudio-module-bluetooth
```

### General configuration
Edit `/etc/hostname` and `/etc/hosts` to replace `raspberrypi` by something unique, let's say for example `node_titulebolide`. It will set the DNS name of the server as well as its bluetooth device name.

### Snapcast configuration
Clone *snapcast* and build it with the guide here : https://github.com/badaix/snapcast/blob/master/doc/build.md#linux-native

Setup the alsa loopback and make it visible to pulseaudio:
- Add the line `snd-aloop` at the end of `/etc/modules`
- Add the lines in `/etc/pulse/default.pa`
    - `load-module module-alsa-sink device=hw:0,0`
    - `set-default-sink alsa_output.hw_0_0`
- Change the very last line of `/etc/default/snapclient` to:
    - `SNAPCLIENT_OPTS="--host 127.0.0.1 -s Headphones"`
- Edit the file `/etc/snapserver.conf` and change the `source` parameter to:
    - `source = alsa://?name=Snapserver&device=hw:0,1,0`

Reboot.

### Bluetooth configuration
```console
sudo hciconfig hci0 class 0x240408
bluetoothctl
```

The `bluetoothctl` environment will open. Paste:
```console
discoverable yes
pairable yes
```
Now take the device you want to pair and search for your rasperry pi and pair it, confirming the codes and connections both on the device and on the raspberry pi.

Still on the `bluetoothctl` environment, once the device is connected:
```console
trust
exit
```

Paste this to a new file named `pulseaudio.service` in `/etc/systemd/system` (using `nano /etc/systemd/system/pulseaudio.service`), as no pulseaudio service is created in Raspbian OS (note : the User=pi is necessary, using the user pulse don't work):
```systemd
[Unit]
Description=PulseAudio Deamon

[Install]
WantedBy=multi-user.target

[Service]
Type=simple
User=pi
Group=pi
ExecStart=/usr/bin/pulseaudio --realtime --exit-idle-time=-1
```

```console
sudo systemctl daemon-reload
sudo systemctl enable pulseaudio
sudo systemctl start pulseaudio
```

This should be good to go.

## Setting up a Snapcast client
These are the very same steps given by the snapcast guide to install the client : https://github.com/badaix/snapcast/blob/master/doc/build.md#linux-native
Afterwards you will just have to edit the file `/etc/default/snapclient` to:
`SNAPCLIENT_OPTS="--host <HOSTNAME>.lan -s Headphones"`. In our example `<HOSTNAME>` is `node_titulebolide`. `<HOSTNAME>.lan` can be replaced by the server's IP if it is static.

## Debugging the server
The routing of the sound can be sometime quite complicated, it can be debugged with:
- `aplay -l`
- `pactl list`
- `pactl info`
- `journalctl -f -u snapclient`
- `journalctl -f -u snapserver`