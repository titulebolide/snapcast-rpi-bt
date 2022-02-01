# Turn your Raspberry Pi into a Bluetooth Speaker

The best rpi to be used is the Raspberry Pi 3 Model A+

Using the default `pi` user as an example.

## Setting up a Snapcast server
The snapcast server will be both the snapcast central server and a client to have it playing music as well. It will retreive its audio source from bluetooth like so:

```
device > bluetooth > pulseaudio > alsa loopback sink > alsa loopback source > snapcast server > network > {snapcast clients}

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

### Snapcast configuration
Clone snapcast and build it with the guide here : https://github.com/badaix/snapcast/blob/master/doc/build.md#linux-native
Setup the alsa loopback and make it visible to pulseaudio:
- Add the line `snd-aloop` at the end of `/etc/modules`
- Add the lines in `/etc/pulse/default.pa`
    - `load-module module-alsa-sink device=hw:0,0`
    - `set-default-sink alsa_output.hw_0_0`
- Change the very last line of `/etc/default/snapclient` to:
    - `SNAPCLIENT_OPTS="--host 127.0.0.1 -s Headphones"`


### Bluetooth configuration
The name of the bluetooth device can be changed by changing `/etc/hostname` and `/etc/hosts`.

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

### Debugging
The routing of the sound can be sometime quite complicated, it can be debugged with:
- `aplay -l`
- `pactl list`
- `pactl info`
- `journalctl -f -u snapclient`
- `journalctl -f -u snapserver`

## Setting up a Snapcast client
These are the very same steps given by the snapcast guide to install the client : https://github.com/badaix/snapcast/blob/master/doc/build.md#linux-native