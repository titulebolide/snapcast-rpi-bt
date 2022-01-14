# Turn your Raspberry Pi into a Bluetooth Speaker

Using the default `pi` user ad an example.

The name of the bluetooth device can be changed by changing `/etc/hostname` and `/etc/hosts`.

```bash
sudo apt-get install pulseaudio pulseaudio-module-bluetooth # bluez bluez-tools
sudo hciconfig hci0 class 0x240408
bluetoothctl
```

The `bluetoothctl` environment will open. Paste:
```bash
discoverable yes
pairable yes
```
Now take the device you want to pair and search for your rasperry pi and pair it, confirming the codes and connections both on the device and on the raspberry pi.

Still on the `bluetoothctl` environment, once the device is connected:
```bash
trust
exit
```

Paste this to a new file named `pulseaudio.service` in `/etc/systemd/system` (using `nano /etc/systemd/system/pulseaudio.service`), as no pulseaudio service is created in Raspbian OS :
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

```bash
sudo systemctl daemon-reload
sudo systemctl enable pulseaudio
sudo systemctl start pulseaudio
```

This should be good to go.