# pihole-home-network-monitor

Network-wide ad/tracker blocking (Pi-hole) plus optional home network
monitoring, all as Docker containers defined in one `docker-compose.yml`.

Built to run first on a regular laptop (Intel or Apple Silicon Mac, Windows,
Linux — anything with Docker), then move as-is to a **Raspberry Pi Zero 2 W**
so it can run 24/7 on almost no power. Same repo, same commands, on either
machine.

## What's in here

| Service | What it does | Always on, or optional? |
|---|---|---|
| **Pi-hole** | DNS server that blocks ads/trackers for every device on your network | Always on (core) |
| **NetAlertX** | Watches your LAN and alerts you when a new or unrecognized device joins | Optional (`monitor` profile) |
| **Uptime Kuma** | Dashboard that pings your devices/services and alerts you if one goes down | Optional (`monitor` profile) |

## 1. Check you have Docker

Open **Terminal** on your Mac (Cmd+Space, type "Terminal", hit Enter) and run:

```bash
docker --version
docker compose version
```

- **If both print a version number** — you're set, skip to step 2.
- **If you get "command not found"** — install [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop/), open it once so it finishes setup (you'll see a whale icon in your menu bar), then re-run the two commands above to confirm.

## 2. Clone this repo

```bash
git clone https://github.com/rishvaiyer/pihole-home-network-monitor.git
cd pihole-home-network-monitor
```

## 3. Set your password

Copy the example env file and edit it:

```bash
cp .env.example .env
open -e .env
```

That opens `.env` in TextEdit. At minimum, change `PIHOLE_PASSWORD` to
something real — this protects the admin page that can see every device and
domain on your network. Set `TZ` to your timezone too (list of names is
linked inside the file). Save and close.

## 4. Start Pi-hole

```bash
docker compose up -d
```

`-d` runs it in the background. First run pulls the image, so it may take a
minute. Check it's healthy:

```bash
docker compose ps
```

Then open **http://localhost:8080/admin** in your browser and log in with
the password you set.

## 5. Point your network at it (this is what actually blocks ads)

Right now Pi-hole is running but nothing is using it yet. Two options:

- **Test on one device first (recommended to start):** go into your phone or
  laptop's Wi-Fi settings → DNS → set it manually to your Mac's local IP
  address (find it with `ipconfig getifaddr en0` on the Mac). Browse for a
  bit and confirm ads shrink and http://localhost:8080/admin shows queries
  coming in.
- **Cover your whole house:** log into your router's admin page and set its
  DNS server to your Mac's IP address instead. Every device on the network
  now goes through Pi-hole automatically. (This is also the step you'll
  repeat once Pi-hole is running on the Pi instead of the Mac — just swap in
  the Pi's IP.)

**Heads up:** while Pi-hole is only running on your laptop, ad-blocking stops
whenever the laptop is off or asleep. That's exactly why the Pi Zero 2 W
(small, silent, sips power, can run untouched 24/7) is the intended long-term
home — the Mac is just for building and testing it first.

## 6. (Optional) Turn on the monitor layer

NetAlertX (new-device alerts) and Uptime Kuma (uptime dashboard) are defined
but off by default so the first run stays simple. Turn them on with:

```bash
docker compose --profile monitor up -d
```

- NetAlertX: **http://localhost:20211**
- Uptime Kuma: **http://localhost:3001** (it'll walk you through creating an
  admin account and adding your first monitors — e.g. your router's IP, this
  Pi-hole box, your ISP, any NAS or smart-home hub you have)

**Important limitation on the Mac:** NetAlertX detects devices by watching
raw ARP traffic on your local network, which requires "host networking" —
something Docker Desktop on macOS only partially supports. It'll start fine
and the web UI will work, but device discovery will likely be incomplete or
empty until this moves to the Pi, where host networking works natively on
Linux. Uptime Kuma has no such limitation and works fully on the Mac today.

## 7. Moving to the Raspberry Pi Zero 2 W later

That's the whole point of this repo being portable — the process is
identical, just on the Pi instead of the Mac:

1. Flash Raspberry Pi OS Lite (64-bit) to your SD card with [Raspberry Pi
   Imager](https://www.raspberrypi.com/software/) — use its gear icon to
   pre-configure Wi-Fi, hostname, and SSH before first boot so you never need
   a monitor/keyboard on the Pi itself.
2. SSH into it, then install Docker with the official convenience script:
   ```bash
   curl -fsSL https://get.docker.com | sh
   sudo usermod -aG docker $USER
   ```
   Log out and back in for the group change to apply.
3. Clone this same repo and repeat steps 3–6 above, on the Pi.
4. Update your router's DNS setting to the **Pi's** IP address instead of
   the Mac's. Give the Pi a static/reserved IP in your router's DHCP
   settings first, so it doesn't change on reboot and silently break DNS
   for the house.
5. Stop Pi-hole on the Mac (`docker compose down`) once the Pi is confirmed
   working, so you don't have two DNS servers disagreeing.

The Pi Zero 2 W is enough for Pi-hole and NetAlertX comfortably. If you add
heavier stuff later (see below), watch its 512MB of RAM — keep the stack
lean, or step up to a Pi 4/5 for the extra services.

## Everyday commands

```bash
docker compose ps                 # what's running
docker compose logs -f pihole     # tail Pi-hole's logs
docker compose down               # stop everything (data is kept)
docker compose pull && docker compose up -d   # update to latest images
```

Your data (Pi-hole's block lists/query history, NetAlertX's device
inventory, Uptime Kuma's monitors) lives in `./data/`, outside the
containers. Back it up by copying that folder; wiping and re-running
`docker compose up -d` never loses it as long as `./data/` survives.



## About the "check for cameras / anti-tracking / Flipper Zero" ask

Being straight about what this kind of software stack can and can't do:

- **Hidden cameras / physical bugs**: no Docker container can detect these —
  they need physical tools (RF/lens detectors) or a manual sweep. What this
  stack *can* do is flag anything on your Wi-Fi calling out to the internet:
  NetAlertX will show you every device's MAC address, and MAC prefixes
  (OUIs) often reveal the manufacturer — e.g. addresses starting with
  certain blocks belong to Hikvision or Dahua, common cheap-camera brands.
  If NetAlertX shows a device you don't recognize with a camera-brand OUI,
  that's your lead to go find it physically. A phone app like
  [Fing](https://www.fing.com/) does the same MAC/vendor lookup if you want
  a quick check before this stack is even running.
- **Anti-tracking**: this is exactly what Pi-hole plus the extra blocklists
  above already gets you — it blocks known ad/tracking/telemetry domains
  network-wide, no per-device software needed.
- **Flipper Zero or other hacked hardware nearby**: a Flipper only shows up
  on your *network* monitor if it's actually joined your Wi-Fi or is
  emulating a device on it — in which case NetAlertX would flag it as an
  unrecognized new device like anything else. Its more common uses (RFID,
  sub-GHz, BLE) don't touch your Wi-Fi at all, so they're invisible to this
  stack by nature — that needs a dedicated RF/BLE scanner, not a Docker
  container.

The realistic, buildable version of "who/what is on my network" is the
NetAlertX layer above — that's the part worth turning on.
