# Raspberry Pi display kiosk

Deploy a Pi as a Chromium kiosk that auto-registers with the Displays system and shows the assigned playlist.

**Server (beta):** `https://app-beta.josh.me.uk`  
**Client script:** [`scripts/kiosk.sh`](../scripts/kiosk.sh)

---

## What you need

- Raspberry Pi (3/4/5 preferred; Pi 2 works but is slow)
- SD card, power supply, network (Ethernet preferred)
- Monitor (HDMI, HDMI→VGA, or the **official 7" DSI touchscreen**)
- Access to staff UI → **Displays**

---

## 1. Flash the OS

1. Use [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
2. Choose **Raspberry Pi OS with desktop** (32-bit on Pi 2 / older boards).
3. Set hostname, user (`pi`), Wi‑Fi (optional), and enable SSH.
4. Boot, connect network, confirm you can SSH in.

---

## 2. Desktop auto-login

The kiosk needs a graphical session (not console-only).

```bash
sudo raspi-config
```

**System Options → Boot / Auto Login → Desktop Autologin**

If the desktop packages are missing:

```bash
sudo apt update
sudo apt install -y -o Dpkg::Options::="--force-overwrite" raspberrypi-ui-mods openbox
sudo reboot
```

Confirm after reboot (SSH):

```bash
who
# expect: pi ... (:0) or seat0
```

---

## 3. Install packages

```bash
sudo apt update
sudo apt install -y chromium unclutter unclutter-xfixes x11-xserver-utils xinput
```

`unclutter-xfixes` hides the cursor more reliably on touchscreens (including hide-on-touch). `xinput` is used so touch tracking matches screen rotation.

Remove the keyring prompt (it blocks the kiosk):

```bash
sudo apt remove -y gnome-keyring
mkdir -p ~/.config/autostart
for src in /etc/xdg/autostart/gnome-keyring-*.desktop; do
  [ -f "$src" ] || continue
  printf '%s\n' '[Desktop Entry]' 'Hidden=true' > ~/.config/autostart/"$(basename "$src")"
done
```

---

## 4. Install the kiosk script

From a machine that has the repo (or copy the file another way):

```bash
scp /var/www/app.jb/beta/scripts/kiosk.sh pi@PI_IP:~/kiosk.sh
```

On the Pi:

```bash
chmod +x ~/kiosk.sh
```

Default server is beta. To point elsewhere:

```bash
# example: edit the SERVER default, or export before launch
export KIOSK_SERVER=https://app-beta.josh.me.uk
```

---

## 5. Autostart on login

```bash
mkdir -p ~/.config/autostart
cat > ~/.config/autostart/kiosk.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=Kiosk
Exec=/home/pi/kiosk.sh
X-GNOME-Autostart-enabled=true
EOF
```

If the desktop user is not `pi`, change the `Exec=` path.

---

## 6a. Official 7" DSI touchscreen

The ribbon-cable official panel shows up as **`DSI-1`** (800×480). `kiosk.sh` prefers DSI over HDMI when both are connected, rotates with `xrandr`, remaps touch via Coordinate Transformation Matrix, and reports the **xrandr mode** (not Chromium’s CSS viewport) to the Displays admin page as screen size.

### Packages (extra)

```bash
sudo apt install -y unclutter-xfixes xinput
```

Then re-copy the latest [`scripts/kiosk.sh`](../scripts/kiosk.sh).

### Confirm the panel is seen

```bash
export DISPLAY=:0 XAUTHORITY=/home/pi/.Xauthority
xrandr | grep connected
# expect: DSI-1 connected …
xinput list --name-only
# expect a touchscreen / ft5x / goodix / raspberrypi-ts style device
```

### Orientation in admin

Set **Landscape** or **Portrait** to match how content should appear (not a raw xrandr flag). `kiosk.sh` detects whether the panel is landscape-native (e.g. HDMI 1920×1080) or portrait-native (e.g. DSI 720×1280) and maps to the correct `xrandr --rotate`:

| Admin setting | Landscape-native panel | Portrait-native panel |
|---|---|---|
| Landscape | `normal` | `left` (90°) |
| Portrait (one way up) | `left` | `normal` |
| Portrait (other way up) | `right` | `inverted` (180°) |

Within ~60s the Pi also updates the touch `xinput` matrix. If the image is upright but taps are wrong, try the other Portrait option, or report the device name from `xinput list`.

### Cursor hiding

`kiosk.sh` starts `unclutter-xfixes` with `--timeout 0 --hide-on-touch --start-hidden`. If a pointer still appears:

```bash
sudo apt install -y unclutter-xfixes
pkill -f chromium; pkill -f kiosk; ~/kiosk.sh
```

### Do not use legacy `display_rotate` / `lcd_rotate`

On Bookworm+ with `vc4-kms-v3d`, those `config.txt` options are ignored for the desktop. Use admin orientation + `kiosk.sh` (or Screen Configuration) instead.

### Optional: force X11 if the Pi boots Wayland

The kiosk script expects X11 (`xrandr` / `xinput`). Under Wayland you will see `XWAYLAND0` and `xrandr --rotate` fails with `BadMatch`. Screen size is still reported via DRM (`/sys/class/drm/.../modes`), but orientation from the admin UI will not apply until you switch:

```bash
sudo raspi-config
# Advanced Options → Wayland → X11
sudo reboot
```

Confirm after reboot: `echo $XDG_SESSION_TYPE` should print `x11`.

---

## 6. HDMI → VGA (if needed)

Many old VGA monitors need firmware HDMI tweaks. Edit:

```bash
sudo nano /boot/firmware/config.txt
# older images: /boot/config.txt
```

Recommended for Pi + HDMI→VGA:

```ini
# Prefer FKMS so hdmi_* options apply (especially Pi 2)
dtoverlay=vc4-fkms-v3d
# disable_fw_kms_setup=1   # keep this commented out / removed

[all]
hdmi_force_hotplug=1
hdmi_drive=2
hdmi_group=2
hdmi_mode=47
config_hdmi_boost=7
```

Notes:

- `hdmi_mode=47` = 1440×900. Match your panel, or use custom CVT if the monitor says “Input Not Supported”:

```ini
hdmi_group=2
hdmi_mode=87
hdmi_cvt=1440 900 60 5 0 0 1
```

- Then `sudo reboot` and confirm the desktop is visible on the monitor before relying on the kiosk.

---

## 7. First boot / register the display

1. Reboot the Pi with monitor attached.
2. Chromium should open the player URL after connect.
3. In staff UI → **Displays**, find **New Display** (MAC like `b8:27:eb:…_1`).
4. Edit: set **name**, **playlist**, **orientation**:
   - **Landscape**
   - **Portrait (rotate left)** / **Portrait (rotate right)** — applied by `kiosk.sh` via `xrandr`
5. Save. Within about a minute the Pi re-reads connect and applies orientation / URL changes.

Manual test from SSH (only after desktop is up):

```bash
export DISPLAY=:0 XAUTHORITY=/home/pi/.Xauthority
~/kiosk.sh
```

---

## 8. Day-to-day controls

In **Displays**:

| Action | Effect |
|--------|--------|
| Refresh | Soft remount of playlist content (player polls ~5s) |
| Reload | Full browser reload |
| Orientation | Pi applies `xrandr` on next connect poll (~60s) |

Player URL (for debugging in a normal browser): copy from the display edit dialog (`/d/display/…`).

---

## Troubleshooting

| Symptom | Check |
|---------|--------|
| Text console only | Desktop autologin; `ps aux \| grep -E 'Xorg\|openbox'` |
| `chromium-browser: command not found` | Use `chromium` (script already does) |
| Black screen / “Input Not Supported” | HDMI→VGA config above; try `hdmi_mode=16` (1024×768) |
| Touch taps miss after portrait | Re-copy latest `kiosk.sh`; confirm `xinput` installed; try the other portrait direction |
| Cursor still visible on 7" | Install `unclutter-xfixes`; restart kiosk |
| Keyring password dialog | Remove `gnome-keyring` (step 3) |
| White screen / old session | `pkill -f chromium`; clear `~/.config/chromium-kiosk/Singleton*` |
| Display not in admin list | `curl` connect API (below); check MAC/IP |
| Refresh/reload do nothing | Hard-reload the player once so it has current JS; wait ≤5s |

Connect smoke test:

```bash
MAC=$(cat /sys/class/net/eth0/address)   # or wlan0
IP=$(hostname -I | awk '{print $1}')
curl -sS "https://app-beta.josh.me.uk/api/dashboards/connect?ip=${IP}&mac=${MAC}&display=1"
```

Expect JSON with `"url":"https://…/d/display/…"` and `"orientation":"…"`.

---

## Checklist (new Pi)

- [ ] OS with desktop + autologin  
- [ ] `chromium`, `unclutter` installed; keyring removed  
- [ ] `~/kiosk.sh` from repo, executable  
- [ ] `~/.config/autostart/kiosk.desktop`  
- [ ] HDMI/VGA picture OK before kiosk  
- [ ] Display appears in admin; name + playlist + orientation set  
- [ ] Reboot test: board comes up without SSH  
