# Arrow

A home music system built around a Synology NAS, Raspberry Pi, and ESP32. The NAS serves as the music library backend, the Pi handles audio playback, and the ESP32 provides physical controls and NFC-based playback triggering.

## Architecture

```
[Synology NAS]  ──── Navidrome (music server)
      │
      │  Subsonic API
      │
[Raspberry Pi]  ──── Mopidy (music player) ──── i2s Audio HAT ──── speakers
      │
      │  REST API
      │
   [ESP32]  ──── NFC reader, buttons, display
```

## Components

### NAS — Navidrome

Navidrome runs as a music server on the Synology NAS, exposing your library via the Subsonic API.

- Install guide: https://mariushosting.com/how-to-install-navidrome-on-your-synology-nas/

Once running, Navidrome is reachable at `http://<your-nas-ip>:4533`.

---

### Raspberry Pi — [`pi/piserver`](pi/piserver)

The Pi runs Mopidy as a music player daemon and connects to Navidrome on the NAS via the Subidy (Subsonic) plugin. Audio is output through a Waveshare WM8960 i2s Audio HAT.

#### Audio HAT Setup

- HAT: [Waveshare WM8960 Audio HAT](https://www.waveshare.com/wiki/WM8960_Audio_HAT)
- Set the default audio device:
  ```sh
  sudo raspi-config
  ```
- Verify audio output:
  ```sh
  aplay /usr/share/sounds/alsa/Front_Center.wav
  aplay -l  # list available devices
  ```

#### Mopidy Installation

Install guide: https://docs.mopidy.com/stable/installation/debian/#debian-install

```sh
sudo apt install python3-pip
```

Install GStreamer plugins required for AAC playback:

```sh
sudo apt install gstreamer1.0-plugins-bad gstreamer1.0-libav
```

Configure ALSA audio output in `/etc/mopidy/mopidy.conf`:

```ini
[audio]
output = alsasink
```

#### Mopidy-Subidy (Navidrome/Subsonic Plugin)

```sh
sudo python3 -m pip install Mopidy-Subidy --break-system-packages
```

> **Note:** `--break-system-packages` is required — the plugin won't install otherwise.

Add to `/etc/mopidy/mopidy.conf`:

```ini
[subidy]
url = http://<your-nas-ip>:4533
username = your_navidrome_username
password = your_navidrome_password
```

#### MPD Interface

Mopidy exposes an MPD-compatible interface for control over the network.

Install the extension:

```sh
sudo python3 -m pip install Mopidy-MPD --break-system-packages
```

Add to `/etc/mopidy/mopidy.conf`:

```ini
[mpd]
enabled = true
hostname = ::
port = 6600
```

Install `mpc` for command-line control:

```sh
sudo apt install mpc
```

Common `mpc` commands:

```sh
mpc ls "Subsonic/Artists"                                              # list artists
mpc ls "Subsonic/Artists/Air"                                          # list albums for an artist
mpc add "Subsonic/Artists/Air/The Virgin Suicides: Original Motion Picture Score"  # queue an album
mpc play                                                               # start playback
```

#### Iris (Web Client)

A browser-based UI for Mopidy:

```sh
sudo python3 -m pip install Mopidy-Iris --break-system-packages
```

#### Running as a Service

```sh
sudo systemctl enable mopidy
sudo systemctl restart mopidy
sudo journalctl -u mopidy -n 50  # view logs
```

#### piserver (REST API)

The `piserver` submodule is a REST API that the ESP32 calls directly to control playback. It translates incoming HTTP requests into MPD commands.

```sh
sudo apt install python3-full
python3 -m venv /home/pi/piserver/venv
```

See [`pi/piserver`](pi/piserver) for endpoint docs and setup.

---

### ESP32 — [`esp32/arrow-controller`](esp32/arrow-controller)

The ESP32 acts as a physical control surface and NFC trigger, communicating with the Pi over HTTP via the piserver REST API.

- **NFC reader** — tap a card to trigger playback of a linked album or playlist
- **Physical buttons** — play/pause, volume up/down
- **Display** — shows current track info

The ESP32 communicates with the Pi by calling the `piserver` REST API directly over the local network.

See [`esp32/arrow-controller`](esp32/arrow-controller) for firmware setup and flashing instructions.

---

### Stereo Control

The system includes IR blaster integration for controlling a physical stereo receiver:

- **Power** — on/off toggling, with a photoresistor to detect current power state
- **Volume** — IR volume commands
- **Input select** — switch the receiver's input source
