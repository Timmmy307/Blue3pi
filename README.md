# Blue3pi
mp3 player for rpi0 v2 with waveshare screen v4 paper white

---

## 1. Installer script (run once on the Pi)

Save this as `install_mp3_player.sh` on the Pi and run:

```bash
chmod +x install_mp3_player.sh
sudo ./install_mp3_player.sh
```

```bash name=install_mp3_player.sh
#!/usr/bin/env bash
set -e

APP_DIR="/opt/eink-mp3-player"
SERVICE_NAME="eink-mp3-player.service"
MUSIC_MOUNT="/mnt/music"
MUSIC_DIR="${MUSIC_MOUNT}/mp3"

echo "=== eInk Bluetooth MP3 Player Installer (64-bit, Bookworm) ==="

if [[ $EUID -ne 0 ]]; then
  echo "Please run as root: sudo ./install_mp3_player.sh"
  exit 1
fi

echo "[1/10] Updating apt and installing packages..."
apt-get update
apt-get install -y \
  python3 python3-pip python3-venv \
  git \
  bluez pipewire-audio wireplumber \
  alsa-utils \
  mpg123 \
  python3-pil \
  python3-spidev \
  python3-gpiozero \
  unzip

echo "[2/10] Enabling SPI for e-paper..."
raspi-config nonint do_spi 0 || true

echo "[3/10] Configuring /boot/firmware/config.txt for e-paper and sound..."

CONFIG_FILE="/boot/firmware/config.txt"
if [ ! -f "$CONFIG_FILE" ]; then
  CONFIG_FILE="/boot/config.txt"
fi

ensure_line() {
  local line="$1"
  grep -qF "$line" "$CONFIG_FILE" || echo "$line" >> "$CONFIG_FILE"
}

ensure_line "dtparam=spi=on"
ensure_line "dtparam=audio=on"

echo "[4/10] Configuring Bluetooth (BlueZ + PipeWire)..."
systemctl enable bluetooth.service
systemctl start bluetooth.service

# Enable PipeWire audio for the default user (uid 1000)
# On Bookworm, PipeWire is already enabled by default for desktop;
# for Lite, we ensure pipewire and wireplumber user services are enabled.
DEFAULT_USER="$(getent passwd 1000 | cut -d: -f1 || true)"
if [ -n "$DEFAULT_USER" ]; then
  sudo -u "$DEFAULT_USER" systemctl --user enable pipewire.service pipewire-pulse.service wireplumber.service || true
  sudo -u "$DEFAULT_USER" systemctl --user start pipewire.service pipewire-pulse.service wireplumber.service || true
fi

echo "[5/10] Creating application directory at ${APP_DIR}..."
rm -rf "$APP_DIR"
mkdir -p "$APP_DIR"

echo "[6/10] Writing application files..."

cat > "${APP_DIR}/config.py" << 'EOF'
MUSIC_ROOT = "/mnt/music/mp3"
LAST_DEVICE_FILE = "/opt/eink-mp3-player/last_bt_device.txt"

# For Waveshare 4.2" v4 (commonly 400x300). Adjust if your exact model differs.
SCREEN_WIDTH = 400
SCREEN_HEIGHT = 300

REFRESH_INTERVAL_SEC = 1.0
EOF

cat > "${APP_DIR}/mp3_library.py" << 'EOF'
import os
from dataclasses import dataclass
from typing import List

from mutagen import File as MutaFile

from config import MUSIC_ROOT

@dataclass
class Track:
    path: str
    title: str
    duration: int  # seconds

def _safe_title(path: str) -> str:
    base = os.path.basename(path)
    name, _ = os.path.splitext(base)
    return name

def _get_duration(path: str) -> int:
    try:
        audio = MutaFile(path)
        if audio and audio.info and hasattr(audio.info, "length"):
            return int(audio.info.length)
    except Exception:
        pass
    return 0

def scan_tracks() -> List[Track]:
    tracks: List[Track] = []
    if not os.path.isdir(MUSIC_ROOT):
        return tracks
    for root, _, files in os.walk(MUSIC_ROOT):
        for f in sorted(files):
            if not f.lower().endswith(".mp3"):
                continue
            path = os.path.join(root, f)
            title = _safe_title(path)
            duration = _get_duration(path)
            tracks.append(Track(path=path, title=title, duration=duration))
    return tracks
EOF

cat > "${APP_DIR}/bluetooth_manager.py" << 'EOF'
import subprocess
import time
import os
from typing import Optional

from config import LAST_DEVICE_FILE

def _run(cmd: str) -> str:
    return subprocess.check_output(cmd, shell=True, text=True).strip()

def get_paired_devices() -> list[tuple[str, str]]:
    """
    Returns list of (mac, name)
    """
    try:
        out = _run("bluetoothctl paired-devices")
    except subprocess.CalledProcessError:
        return []
    res = []
    for line in out.splitlines():
        parts = line.split()
        if len(parts) >= 3 and parts[0] == "Device":
            mac = parts[1]
            name = " ".join(parts[2:])
            res.append((mac, name))
    return res

def connect_device(mac: str) -> bool:
    try:
        _run(f"bluetoothctl connect {mac}")
        time.sleep(3)
        return True
    except subprocess.CalledProcessError:
        return False

def disconnect_device(mac: str) -> None:
    try:
        _run(f"bluetoothctl disconnect {mac}")
    except subprocess.CalledProcessError:
        pass

def make_pairable() -> None:
    cmds = [
        "bluetoothctl power on",
        "bluetoothctl agent NoInputNoOutput",
        "bluetoothctl default-agent",
        "bluetoothctl pairable on",
        "bluetoothctl discoverable on",
    ]
    for cmd in cmds:
        try:
            _run(cmd)
        except subprocess.CalledProcessError:
            pass

def save_last_device(mac: str) -> None:
    with open(LAST_DEVICE_FILE, "w") as f:
        f.write(mac.strip())

def load_last_device() -> Optional[str]:
    if not os.path.isfile(LAST_DEVICE_FILE):
        return None
    try:
        with open(LAST_DEVICE_FILE) as f:
            mac = f.read().strip()
        return mac or None
    except Exception:
        return None

def auto_connect() -> Optional[str]:
    """
    Try last device, then all paired. Returns connected MAC or None.
    """
    make_pairable()

    last = load_last_device()
    if last:
        if connect_device(last):
            return last

    for mac, _ in get_paired_devices():
        if connect_device(mac):
            save_last_device(mac)
            return mac

    return None

def scan_and_pair(timeout: int = 30) -> Optional[str]:
    """
    Simple scanning/pairing: scan for devices, pair and connect to
    the first *new* device which succeeds.
    """
    make_pairable()
    try:
        _run("bluetoothctl scan on &")
        seen = set(mac for mac, _ in get_paired_devices())
        start = time.time()
        while time.time() - start < timeout:
            time.sleep(5)
            out = _run("bluetoothctl devices")
            for line in out.splitlines():
                parts = line.split()
                if len(parts) >= 3 and parts[0] == "Device":
                    mac = parts[1]
                    if mac in seen:
                        continue
                    try:
                        _run(f"bluetoothctl pair {mac}")
                        _run(f"bluetoothctl trust {mac}")
                        if connect_device(mac):
                            save_last_device(mac)
                            return mac
                    except subprocess.CalledProcessError:
                        continue
        return None
    finally:
        try:
            _run("bluetoothctl scan off")
        except subprocess.CalledProcessError:
            pass
EOF

cat > "${APP_DIR}/display.py" << 'EOF'
import time

from PIL import Image, ImageDraw, ImageFont

from config import SCREEN_WIDTH, SCREEN_HEIGHT

try:
    from waveshare_epd import epd4in2
except ImportError:
    epd4in2 = None

class Display:
    def __init__(self):
        if epd4in2 is None:
            raise RuntimeError(
                "waveshare_epd.epd4in2 not installed. "
                "Install Waveshare e-Paper python library first."
            )
        self.epd = epd4in2.EPD()
        self.epd.Init()
        self.epd.Clear()
        self.image = Image.new("1", (SCREEN_WIDTH, SCREEN_HEIGHT), 255)
        self.draw = ImageDraw.Draw(self.image)
        try:
            self.font_big = ImageFont.truetype(
                "/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf", 20
            )
            self.font_small = ImageFont.truetype(
                "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf", 14
            )
        except Exception:
            self.font_big = ImageFont.load_default()
            self.font_small = ImageFont.load_default()

    def clear(self):
        self.image = Image.new("1", (SCREEN_WIDTH, SCREEN_HEIGHT), 255)
        self.draw = ImageDraw.Draw(self.image)

    def render_status(
        self,
        title: str,
        elapsed: int,
        duration: int,
        bt_status: str,
        info: str,
    ):
        """
        title: track title or message
        elapsed/duration: seconds
        bt_status: 'BT: ...'
        info: short message
        """
        self.clear()

        # Title
        max_title_chars = 26
        if len(title) > max_title_chars:
            title = title[: max_title_chars - 3] + "..."
        self.draw.text((10, 10), title, font=self.font_big, fill=0)

        # Time
        def fmt_time(s: int) -> str:
            if s <= 0:
                return "--:--"
            m, s = divmod(s, 60)
            return f"{m:02}:{s:02}"

        time_str = f"{fmt_time(elapsed)} / {fmt_time(duration)}"
        self.draw.text((10, 50), time_str, font=self.font_small, fill=0)

        # Progress bar
        if duration > 0 and elapsed >= 0:
            bar_x = 10
            bar_y = 70
            bar_w = SCREEN_WIDTH - 20
            bar_h = 10
            self.draw.rectangle((bar_x, bar_y, bar_x + bar_w, bar_y + bar_h), outline=0, width=1)
            ratio = max(0.0, min(1.0, elapsed / duration))
            fill_w = int(bar_w * ratio)
            self.draw.rectangle((bar_x, bar_y, bar_x + fill_w, bar_y + bar_h), fill=0)

        # BT status
        self.draw.text((10, 100), bt_status, font=self.font_small, fill=0)

        # Info
        self.draw.text((10, 120), info, font=self.font_small, fill=0)

        # Bottom help
        bottom_text = "Kbd: S=scan  N=next  P=prev  Space=pause"
        self.draw.text((10, SCREEN_HEIGHT - 20), bottom_text, font=self.font_small, fill=0)

        self.epd.display(self.epd.getbuffer(self.image))
EOF

cat > "${APP_DIR}/player.py" << 'EOF'
import os
import subprocess
import sys
import time
import threading
import queue
import termios
import tty
import select
import signal

from config import REFRESH_INTERVAL_SEC
from mp3_library import scan_tracks, Track
from bluetooth_manager import auto_connect, scan_and_pair
from display import Display

class HeadlessKeyListener(threading.Thread):
    """
    Listens to keyboard input (USB keyboard) on the console.
    If no TTY, it does nothing.
    """
    def __init__(self, event_queue: "queue.Queue[str]"):
        super().__init__(daemon=True)
        self.event_queue = event_queue
        self.running = True

    def run(self):
        if not sys.stdin.isatty():
            return

        old_settings = termios.tcgetattr(sys.stdin)
        try:
            tty.setcbreak(sys.stdin.fileno())
            while self.running:
                r, _, _ = select.select([sys.stdin], [], [], 0.1)
                if sys.stdin in r:
                    ch = sys.stdin.read(1)
                    self.event_queue.put(ch)
        finally:
            termios.tcsetattr(sys.stdin, termios.TCSADRAIN, old_settings)

    def stop(self):
        self.running = False

class MP3Player:
    def __init__(self):
        self.tracks: list[Track] = scan_tracks()
        self.index = 0
        self.proc: subprocess.Popen | None = None
        self.start_time = 0.0
        self.paused = False
        self.pause_start = 0.0
        self.accumulated_pause = 0.0

    def has_tracks(self) -> bool:
        return len(self.tracks) > 0

    def current_track(self) -> Track | None:
        if not self.tracks:
            return None
        return self.tracks[self.index]

    def _stop_mpg123(self):
        if self.proc and self.proc.poll() is None:
            self.proc.terminate()
            try:
                self.proc.wait(timeout=2)
            except subprocess.TimeoutExpired:
                self.proc.kill()
        self.proc = None

    def play_current(self):
        track = self.current_track()
        if not track:
            return
        self._stop_mpg123()
        self.start_time = time.time()
        self.paused = False
        self.accumulated_pause = 0.0
        # Use default ALSA/PipeWire sink; BlueZ/PipeWire will route audio to BT
        self.proc = subprocess.Popen(
            ["mpg123", "-q", track.path],
            stdout=subprocess.DEVNULL,
            stderr=subprocess.DEVNULL,
        )

    def next_track(self):
        if not self.tracks:
            return
        self.index = (self.index + 1) % len(self.tracks)
        self.play_current()

    def prev_track(self):
        if not self.tracks:
            return
        self.index = (self.index - 1) % len(self.tracks)
        self.play_current()

    def toggle_pause(self):
        if not self.proc:
            return
        if not self.paused:
            self.proc.send_signal(signal.SIGSTOP)
            self.paused = True
            self.pause_start = time.time()
        else:
            self.proc.send_signal(signal.SIGCONT)
            self.paused = False
            self.accumulated_pause += time.time() - self.pause_start

    def is_playing(self) -> bool:
        return self.proc is not None and self.proc.poll() is None and not self.paused

    def elapsed(self) -> int:
        if not self.proc:
            return 0
        if self.start_time == 0:
            return 0
        if self.paused:
            return int(self.pause_start - self.start_time - self.accumulated_pause)
        return int(time.time() - self.start_time - self.accumulated_pause)

def main():
    disp = Display()
    player = MP3Player()

    if not player.has_tracks():
        disp.render_status(
            title="No MP3 files",
            elapsed=0,
            duration=0,
            bt_status="BT: not connected",
            info="Put MP3s into /mnt/music/mp3 and reboot",
        )
        time.sleep(10)
        return

    bt_mac = auto_connect()
    bt_status = "BT: connected" if bt_mac else "BT: none (kbd S=scan)"

    player.play_current()

    event_q: "queue.Queue[str]" = queue.Queue()
    key_listener = HeadlessKeyListener(event_q)
    key_listener.start()

    info_msg = ""

    try:
        while True:
            # Keyboard events
            while True:
                try:
                    ch = event_q.get_nowait()
                except queue.Empty:
                    break
                if ch in ("n", "N"):
                    player.next_track()
                    info_msg = "Next track"
                elif ch in ("p", "P"):
                    player.prev_track()
                    info_msg = "Previous track"
                elif ch == " ":
                    player.toggle_pause()
                    info_msg = "Play/Pause"
                elif ch in ("s", "S"):
                    info_msg = "Scanning for BT devices..."
                    disp.render_status(
                        title=player.current_track().title if player.current_track() else "Scanning...",
                        elapsed=player.elapsed(),
                        duration=player.current_track().duration if player.current_track() else 0,
                        bt_status="BT: scanning",
                        info=info_msg,
                    )
                    mac = scan_and_pair()
                    if mac:
                        bt_mac = mac
                        bt_status = "BT: connected"
                        info_msg = f"Connected to {mac}"
                    else:
                        bt_status = "BT: none"
                        info_msg = "No BT device found"
                elif ch in ("q", "Q"):
                    return

            # Track finished -> next
            if player.proc and player.proc.poll() is not None:
                player.next_track()

            track = player.current_track()
            if track:
                title = track.title
                elapsed = player.elapsed()
                duration = track.duration
            else:
                title = "No track"
                elapsed = 0
                duration = 0

            if bt_mac:
                bt_status = f"BT: {bt_mac}"
            else:
                bt_status = "BT: none"

            disp.render_status(
                title=title,
                elapsed=elapsed,
                duration=duration,
                bt_status=bt_status,
                info=info_msg,
            )
            info_msg = ""

            time.sleep(REFRESH_INTERVAL_SEC)
    finally:
        key_listener.stop()

if __name__ == "__main__":
    main()
EOF

echo "[7/10] Creating Python virtual environment and installing Python deps..."
cd "$APP_DIR"
python3 -m venv venv
# shellcheck disable=SC1091
source venv/bin/activate

pip install --no-cache-dir mutagen

echo "[8/10] Creating music directory and fstab hint..."
mkdir -p "$MUSIC_DIR"

FSTAB_HINT_FILE="${APP_DIR}/FSTAB_HINT.txt"
cat > "$FSTAB_HINT_FILE" << 'EOF'
To make MP3s easily drag-and-drop editable:

1. Create a separate partition on the SD card (e.g. /dev/mmcblk0p3) formatted as exFAT or FAT32.
2. Add a line to /etc/fstab (adjust device accordingly):

   /dev/mmcblk0p3  /mnt/music  exfat  defaults,uid=1000,gid=1000  0  0

3. mkdir -p /mnt/music
4. sudo mount -a

Then put your MP3 files in /mnt/music/mp3. You can safely power off the Pi,
remove the SD card, and copy/delete MP3s on another machine.
EOF

echo "[9/10] Creating systemd service..."

DEFAULT_USER="$(getent passwd 1000 | cut -d: -f1 || echo pi)"
DEFAULT_UID="$(id -u "$DEFAULT_USER" 2>/dev/null || echo 1000)"

cat > "/etc/systemd/system/${SERVICE_NAME}" << EOF
[Unit]
Description=eInk Bluetooth MP3 Player
After=multi-user.target bluetooth.service

[Service]
Type=simple
User=${DEFAULT_USER}
Group=${DEFAULT_USER}
WorkingDirectory=${APP_DIR}
Environment="XDG_RUNTIME_DIR=/run/user/${DEFAULT_UID}"
Environment="PIPEWIRE_RUNTIME_DIR=/run/user/${DEFAULT_UID}"
ExecStart=${APP_DIR}/venv/bin/python ${APP_DIR}/player.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable "${SERVICE_NAME}"

echo "[10/10] Done."
echo "Next steps:"
echo "1. Install Waveshare e-Paper python library (see README step below)."
echo "2. Optionally edit /etc/fstab to mount a dedicated music partition at ${MUSIC_MOUNT}."
echo "3. Copy MP3 files into ${MUSIC_DIR}."
echo "4. Reboot: sudo reboot"
echo "After boot, the player will auto-start."
```

---

## 2. Waveshare e‑paper library (64‑bit works fine)

On the Pi (after network is set up):

```bash
cd /opt
sudo git clone https://github.com/waveshareteam/e-Paper.git
cd e-Paper/RaspberryPi_JetsonNano/python
sudo python3 setup.py install
```

If your 4.2" v4 has a different module name than `epd4in2`, open `display.py` and change:

```python
from waveshare_epd import epd4in2
```

to whatever module/class the Waveshare example for your exact screen uses.

---

## 3. How to install on Raspberry Pi OS Lite 64‑bit

1. Flash **Raspberry Pi OS Lite 64‑bit (Bookworm)** to SD using Raspberry Pi Imager.
2. Boot the Pi Zero 2 W, set up Wi‑Fi/SSH.
3. Copy `install_mp3_player.sh` to the Pi (e.g. `scp` or `wget`).
4. Run:

```bash
chmod +x install_mp3_player.sh
sudo ./install_mp3_player.sh
```

5. Install Waveshare library (commands above).
6. Put some `.mp3` files into `/mnt/music/mp3` (you can power off, remove SD, and copy from a PC if you like).
7. Reboot:

```bash
sudo reboot
```

---

## 4. How to use it

- On boot, the service `eink-mp3-player.service` starts.
- It:
  - Scans `/mnt/music/mp3` for MP3s.
  - Tries to auto‑connect to the last Bluetooth device, then any paired device.
  - Starts playing tracks in order.
- E‑paper screen shows:
  - Title
  - `elapsed / total` time
  - Progress bar
  - Bluetooth status (MAC or “none”)
  - Short info messages

**Keyboard controls (USB keyboard):**

- `S` – scan & pair/connect to a Bluetooth audio device
- `N` – next track
- `P` – previous track
- `Space` – play/pause
- `Q` – quit (systemd will restart the service)



---


cd /opt
sudo rm -rf e-Paper
sudo git clone https://github.com/waveshareteam/e-Paper.git
cd e-Paper/RaspberryPi_JetsonNano/python/lib

# Install just the 'waveshare_epd' package with pip (no easy_install)
sudo python3 -m pip install .


---



You’re very close. The only thing that failed was that we tried to `pip install .` from the wrong subdirectory.

Here are the exact commands you need to run now, in order, with no extra explanation:

```bash
cd /opt/e-Paper/RaspberryPi_JetsonNano/python
ls
```

Check the output for a folder named `lib` or similar (for current repo it should be `lib`). Then run (assuming it is `lib`):

```bash
cd /opt/e-Paper/RaspberryPi_JetsonNano/python/lib
sudo /opt/eink-mp3-player/venv/bin/python -m pip install .
```

Then:

```bash
sudo mkdir -p /mnt/music/mp3
sudo reboot
```
