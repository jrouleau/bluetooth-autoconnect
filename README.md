# bluetooth-autoconnect

A linux command line tool to automatically connect to all paired and trusted bluetooth devices.

It can operate as a oneshot script or as a daemon where it waits for adapters to come online.

## Requirements

* Requires `python3`, `python-prctl`, `python-dbus`, and `bluez`

## Installation

If you have packaged this for another disto, please open an issue or PR so the instructions can be added to this list.

### Arch Linux

* Install AUR package from https://aur.archlinux.org/packages/bluetooth-autoconnect/
* Enable the service with `sudo systemctl enable bluetooth-autoconnect`

#### pulseaudio
* If you are using pulseaudio with a bluetooth headset or speaker, also enable the helper service with `systemctl --user enable pulseaudio-bluetooth-autoconnect`

### Manual

* Install the `bluetooth-autoconnect` script to somewhere in your `PATH`, such as `/usr/local/bin/`
* If you are using systemd, consider installing the `bluetooth-autoconnect.service` file to `/etc/systemd/system/` and modifying it to reflect the location of where you installed the script
  - Enable the service with `sudo systemctl enable bluetooth-autoconnect`

#### pulseaudio

If you are using a bluetooth headset with pulseaudio running as your user, then the above service will fail to connect to your headset on boot because pulseaudio won't have been started when to bluetooth service comes up.

* If you are using systemd, consider installing the `pulseaudio-bluetooth-autoconnect.service` file to `/etc/systemd/user/` or `~/.config/systemd/user/` and modifying it to reflect the location of where you installed the script
  - Enable the service with `systemctl --user enable pulseaudio-bluetooth-autoconnect`

## Usage

```sh
Usage: bluetooth-autoconnect [OPTIONS]...

Automatically connect to trusted bluetooth devices

OPTIONS:
  -d, --daemon      Monitor bluetooth adapters and automatically connect to
                    trusted devices when an adapter is powered on
  -h, --help        Print this help message
  -v, --verbose     Show more detailed log messages

```

Simply running the script will scan all powered on adapters and connect to any available devices
```sh
$ bluetooth-autoconnect
connecting to device /org/bluez/hci0/dev_XX_XX_XX_XX_XX_XX
successfully connected to device /org/bluez/hci0/dev_XX_XX_XX_XX_XX_XX
```

When operating in daemon mode with the `-d` flag, it will first connect to any available devices and then wait for new adapters to be connected/powered on. Sending the HUP signal to the process will force it to rescan all adapters.
```sh
$ bluetooth-autoconnect -d
connecting to device /org/bluez/hci0/dev_XX_XX_XX_XX_XX_XX
successfully connected to device /org/bluez/hci0/dev_XX_XX_XX_XX_XX_XX
[...]
connecting to device /org/bluez/hci1/dev_XX_XX_XX_XX_XX_XX
successfully connected to device /org/bluez/hci1/dev_XX_XX_XX_XX_XX_XX
```

## License

MIT License

Copyright (c) 2019 Jonathan Rouleau
