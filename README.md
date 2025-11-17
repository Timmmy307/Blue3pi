# Blue3pi
mp3 player for rpi0 v2 with waveshare screen v4 paper white
# .

1. Installer script (run once on the Pi)
Save this as install_mp3_player.sh on the Pi and run:

bash
chmod +x install_mp3_player.sh
sudo ./install_mp3_player.sh

install_mp3_player.sh
v2
#!/usr/bin/env bash
set -e

APP_DIR="/opt/eink-mp3-player"
SERVICE_NAME="eink-mp3-player.service"
MUSIC_MOUNT="/mnt/music"
2. Waveshare e‑paper library (64‑bit works fine)
On the Pi (after network is set up):

bash
cd /opt
sudo git clone https://github.com/waveshareteam/e-Paper.git
cd e-Paper/RaspberryPi_JetsonNano/python
sudo python3 setup.py install
If your 4.2" v4 has a different module name than epd4in2, open display.py and change:

Python
from waveshare_epd import epd4in2
to whatever module/class the Waveshare example for your exact screen uses.

3. How to install on Raspberry Pi OS Lite 64‑bit
Flash Raspberry Pi OS Lite 64‑bit (Bookworm) to SD using Raspberry Pi Imager.
Boot the Pi Zero 2 W, set up Wi‑Fi/SSH.
Copy install_mp3_player.sh to the Pi (e.g. scp or wget).
Run:
bash
chmod +x install_mp3_player.sh
sudo ./install_mp3_player.sh
Install Waveshare library (commands above).
Put some .mp3 files into /mnt/music/mp3 (you can power off, remove SD, and copy from a PC if you like).
Reboot:
bash
sudo reboot
4. How to use it
On boot, the service eink-mp3-player.service starts.
It:
Scans /mnt/music/mp3 for MP3s.
Tries to auto‑connect to the last Bluetooth device, then any paired device.
Starts playing tracks in order.
E‑paper screen shows:
Title
elapsed / total time
Progress bar
Bluetooth status (MAC or “none”)
Short info messages
Keyboard controls (USB keyboard):

S – scan & pair/connect to a Bluetooth audio device
N – next track
P – previous track
Space – play/pause
Q – quit (systemd will restart the service)
If you want, I can next add a separate file to listen for headset AVRCP buttons (play/pause/next/prev) so you don’t need a keyboard for normal use.
