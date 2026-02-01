# HT-HC01P + Raspberry Pi 4 + OpenMANET — Complete Guide

A single, start-to-finish guide: image the Pi, plug in the antenna, wire the Heltec HaLow breakout, boot, and enable the radio in the web dashboard.

---

## 1. Image the Raspberry Pi

Use an **OpenMANET** image built for **Raspberry Pi 4** with **MM6108 HaLow over SPI**.

- **Build target:** `ekh-bcm2711`
- **Device name:** RPI4-MM6108 (SPI)

**Get the image:**

- **Pre-built:** Check [OpenMANET firmware releases](https://github.com/OpenMANET/firmware/releases) for a file like `openmanet-*-RPI4-MM6108-SPI-sysupgrade.img.gz` (or similar for bcm27xx/bcm2711).
- **Build from source:**
  ```bash
  git clone https://github.com/OpenMANET/firmware.git
  cd firmware
  ./scripts/openmanet_setup.sh -i -b ekh-bcm2711
  make download
  make -j$(nproc)
  ```
  The image is in `bin/targets/bcm27xx/bcm2711/` (e.g. `openmanet-*-RPI4-MM6108-SPI-sysupgrade.img.gz`).

**Flash the SD card:**

1. Decompress if needed: `gunzip -k openmanet-*-RPI4-MM6108-SPI-sysupgrade.img.gz`
2. Write the `.img` to the SD card (replace `sdX` with your SD device, e.g. `sdb`):
   ```bash
   sudo dd if=openmanet-*-RPI4-MM6108-SPI-sysupgrade.img of=/dev/sdX bs=4M status=progress conv=fsync
   ```
   Or use Raspberry Pi Imager / balenaEtcher and choose the custom `.img` file.
3. Eject the SD card, insert it into the Pi. Do **not** power on yet.

---

## 2. Antenna — Required before wiring or power

**The antenna MUST be plugged into the HT-HC01P before you do any wiring or power on the board.**

Operating the module without an antenna can damage the radio. Plug the HaLow antenna into the antenna connector on the HT-HC01P Debug Board now, before the next step.

---

## 3. Wiring

Use the **HT-HC01P Debug Board** pinout from Heltec ([resource.heltec.cn/download/HT-HC01P](https://resource.heltec.cn/download/HT-HC01P)) to find MOSI, MISO, SCLK, CS, RST, IRQ, 3.3V, and GND on the breakout.

**Do all wiring with the Pi powered off and unplugged.**

| Pi physical pin | BCM GPIO | Signal   | Connect to        |
|-----------------|----------|----------|-------------------|
| 1               | —        | 3.3 V    | Debug Board 3V   |
| 6               | —        | GND      | Debug Board GND   |
| 19              | 10       | MOSI     | Debug Board MOSI  |
| 21              | 9        | MISO     | Debug Board MISO  |
| 23              | 11       | SCLK     | Debug Board SCLK  |
| 24              | 8        | CE0 (CS) | Debug Board CS    |
| 11              | 17       | RESET    | Debug Board RESET |
| 29              | 5        | INT      | Debug Board INT   |

OpenMANET’s device tree expects **RESET on GPIO 17 (pin 11)** and **IRQ on GPIO 5 (pin 29)**. Use these exact pins.

Double-check connections (especially MOSI/MISO and RESET/IRQ), then power on the Pi.

---

## 4. First boot and access

1. Power on the Pi with the OpenMANET SD card.
2. Connect to the Pi:
   -**Ethernet(Basic):** Plug Ethernet cable between laptop and Pi. OpenMANET Dashboard should load at 10.41.254.1 by default. SSH is also possible.
   - **Ethernet(DHCP):** Plug into the Pi’s Ethernet port; the Pi may get an IP via DHCP (e.g. 192.168.1.x). Find it from your router or with `arp -a` / a network scanner.
   - **On-board Wi‑Fi:** If enabled by default, connect to the OpenWrt Wi‑Fi network and use the IP shown in the device’s info (e.g. 192.168.1.1).
4. Open a browser and go to the Pi’s IP (e.g. `http://192.168.1.1`). Log in to LuCI (OpenWrt web interface).

---

## 5. Enable the HaLow radio in the dashboard

This is the main step to get HaLow working.

1. In LuCI go to **Network → Wireless**.
2. Find **radio2** (HaLow / Morse Micro SPI / MM6108). You may also see other radios (e.g. on-board Wi‑Fi or leftovers from earlier configs).
3. Make sure **radio2** is **Enabled**. If it is disabled, enable it.
4. Ensure **radio2** has at least one **wireless interface** (wifi-iface):
   - If there is already an interface on radio2 (e.g. an AP or mesh), check that it is enabled and has an SSID (and Mesh ID if using mesh). Edit if needed.
   - If there is no interface on radio2, add one:
     - Click **Add** (or **Add new interface**).
     - Set **Device** to the HaLow radio (radio2).
     - Set **Mode** to **Access Point** (or **Mesh** if you use mesh).
     - Set **SSID** (e.g. `HaLow-AP`). For mesh, set **Mesh ID**.
     - Set **Network** (e.g. `lan`).
     - Save.
5. Click **Save & Apply** at the bottom of the Wireless page.
6. Wait a few seconds. In **Overview** (or the wireless status section) you should see the HaLow radio listed (e.g. “Morse Micro SPI-MM601X 802.11ah”, channel, SSID, encryption). “Associations” may be 0 until a client connects or a mesh peer is in range.

Enabling the radio and having a wifi-iface on radio2 in the web dashboard is what actually brings the HaLow interface (morse0) into use.

---

## 6. Optional: morse0 still DOWN after reboot

If after a reboot the HaLow interface (`morse0`) is still DOWN, you can force it up.

**Option A — Hotplug script (runs when morse0 appears):**

```bash
mkdir -p /etc/hotplug.d/net
cat << 'EOF' > /etc/hotplug.d/net/20-morse0
#!/bin/sh
[ "$INTERFACE" != "morse0" ] || [ "$ACTION" != "add" ] && return 0
wifi reload
sleep 2
ip link set morse0 up 2>/dev/null || true
EOF
chmod +x /etc/hotplug.d/net/20-morse0
```

**Option B — Run once after boot (e.g. in `/etc/rc.local`):**

Add before `exit 0` (if present):

```bash
(sleep 6 && wifi reload && sleep 2 && ip link set morse0 up) &
```

---

## 7. Optional: Clean up extra radios

If **Network → Wireless** shows many radios from previous attempts, remove the ones that are not real hardware (duplicate or invalid entries). In LuCI, delete the extra devices/interfaces; keep **radio2** (HaLow) and the on-board Wi‑Fi if you use it. Then **Save & Apply**.

---

## Checklist

- [ ] Image SD card with OpenMANET ekh-bcm2711 (RPI4-MM6108-SPI).
- [ ] **Plug antenna into HT-HC01P before any wiring or power.**
- [ ] Wire: 3.3V, GND, SPI (pins 19, 21, 23, 24), RESET (pin 11), IRQ (pin 29).
- [ ] Power on Pi, connect to LuCI (Ethernet or on-board Wi‑Fi).
- [ ] **Network → Wireless:** Enable radio2 (HaLow), add or enable a wifi-iface on it, **Save & Apply**.
- [ ] Optional: Add hotplug or rc.local if morse0 stays DOWN after reboot.
- [ ] Optional: Remove extra/unused radios from the wireless config.

---

## Summary

The HT-HC01P is an MM6108 HaLow module on SPI; OpenMANET’s ekh-bcm2711 image supports it. After imaging, **always plug in the antenna first**, then wire power, SPI, RESET (pin 11), and IRQ (pin 29). The step that makes HaLow work is **enabling radio2 and its wifi-iface in the web dashboard** (Network → Wireless, Save & Apply). Extra radios can be removed; if morse0 is still DOWN after reboot, use the hotplug script or rc.local to bring it up.
