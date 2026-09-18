# Sunshine on Omarchy 4.x — remote desktop streaming

A tested, corrected setup guide for running [Sunshine](https://app.lizardbyte.dev/Sunshine)
(self-hosted Moonlight game/desktop streaming host) on [Omarchy](https://omarchy.org) Linux.
Most guides floating around (including the one this started from) predate Omarchy 4.0 and
describe a Docker setup that's no longer the right approach. Corrections and PRs welcome —
especially reports from real (non-VM) hardware with a GPU, and other Omarchy/Sunshine versions.

**Status:** an installer script that automates the manual steps below is planned next.

Verified live on 2026-09-17 against Omarchy 4.0.3 (VM, no GPU passthrough — see the encoder
note below). **Supersedes the old Docker-based guide below the fold** — Omarchy 4.x ships
Sunshine as a first-party package with a one-command installer. The original guide (from
https://github.com/basecamp/omarchy/discussions/179, written for pre-4.0 Omarchy) was Docker-only
and is obsolete on 4.x; kept at the bottom only in case you're on an older install or plain Arch.

## The whole install, in one command

```bash
omarchy install service sunshine
```

This single script (`/usr/bin/omarchy-install-service-sunshine`, needs sudo) does everything:
- Installs `sunshine` from Omarchy's own pacman repo (`Repository: omarchy`, not AUR) — package
  already ships the uinput udev rule (`60-sunshine.rules`) and has `cap_sys_admin,cap_sys_nice`
  capabilities baked into the binary via `setcap`, so no Docker, no manual udev rule, no
  `--cap-add` needed.
- Opens firewall ports (`47984,47989,48010` TCP + `5353,47998,47999,48000,48002,48010` UDP) via
  `ufw`, scoped to RFC1918 private ranges plus `tailscale0` specifically — not exposed publicly.
- Installs a "Sunshine Admin" launcher app that opens the web UI in a chromeless window.
- Adds `o.launch_on_start("sunshine")` to `~/.config/hypr/autostart.lua` so it starts with your
  Hyprland session.
- Runs `systemctl --user enable --now sunshine` — resolves via `Alias=sunshine.service` to
  `/usr/lib/systemd/user/app-dev.lizardbyte.app.Sunshine.service`, which has
  `Restart=on-failure` (5 restarts / 500s) and a 5s `ExecStartPre` delay so it doesn't race the
  compositor coming up.

To remove: `omarchy remove service sunshine` (closes the same ports, uninstalls cleanly).

## Gotcha — input group is NOT handled by the installer

The installer does not add you to the `input` group. `/dev/uinput` is `root:input 0660`, so
without this, video capture works fine but mouse/keyboard from the Moonlight client silently
won't:

```bash
sudo usermod -aG input "$USER"
```

Log out/in (or reboot) for it to take effect — group membership doesn't apply retroactively to
an existing session.

## Gotcha — the systemd service and the Hyprland autostart can race on first boot

Observed directly: on a cold boot the aliased `sunshine.service` hit its restart limit and
core-dumped 5 times in under a minute (`journalctl --user | grep sunshine` showed
`Failed with result 'core-dump'` → `'start-limit-hit'`), because it started fighting for
GPU/Wayland resources before the desktop had settled — even with the 5s startup delay. If you
hit this: `systemctl --user reset-failed app-dev.lizardbyte.app.Sunshine.service` then either
`systemctl --user restart sunshine` or just relaunch from the app menu once the desktop is
idle. It has not recurred on a warm desktop.

## Encoder fallback — expect software (libx264) unless you have real GPU passthrough

Sunshine tries `nvenc` → `vulkan` → `vaapi` → software in order and logs (harmlessly) every
failure along the way. On a VM with no GPU passthrough (no `/dev/dri/renderD128`, only a
display-only `card1` node from virtio-gpu) it always lands on `libx264` software encoding —
works, but costs CPU instead of using the GPU. If your Omarchy box has a real/passed-through
GPU, check `~/.config/sunshine/sunshine.log` for `Found H.264 encoder:` to see which one it
actually picked. **If you test this on real hardware, please open an issue/PR with what you
saw** — that's the main gap in this guide right now.

## First run — set admin credentials

Either through the web UI wizard at `https://<host>:47990` on first visit, or non-interactively:

```bash
sunshine --creds <username> <password>
```

Takes effect on the next Sunshine restart (it doesn't hot-reload), so restart the service/app
after setting it. Then pair from Moonlight as normal — Sunshine also advertises itself over
Avahi/mDNS under the machine's hostname, so it should show up in Moonlight's auto-discovery on
the same LAN without typing an IP.

## Known limitation — full-disk encryption (unchanged from the original guide)

You cannot reach the pre-boot LUKS password screen remotely — the system needs the password
typed locally before it boots far enough for Sunshine to run. No workaround short of removing
disk encryption or leaving the machine powered on so you never hit that screen.

## Contributing

Issues and PRs welcome — especially: results on real GPU hardware (AMD/Intel/Nvidia), other
Omarchy versions, and anything that doesn't match once the planned installer script lands.

---

# Old guide (pre-4.0 / non-Omarchy Arch, Docker-based) — kept for reference only

Original source: https://github.com/basecamp/omarchy/discussions/179. Don't use this on
Omarchy 4.x — use `omarchy install service sunshine` above instead.

## 1. Host prep — input group + uinput permissions

```bash
sudo usermod -aG input "$USER"
sudo tee /etc/udev/rules.d/85-sunshine-input.rules >/dev/null <<'EOF'
KERNEL=="uinput", SUBSYSTEM=="misc", OPTIONS+="static_node=uinput", TAG+="uaccess", GROUP="input", MODE="0660"
EOF
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Log out/in once so the `input` group membership takes effect.

## 2. Directories + compose file

```bash
mkdir -p ~/docker/sunshine/{config,assets}
```

`~/docker/sunshine/compose.yml`:

```yaml
name: sunshine

services:
  sunshine:
    container_name: sunshine
    image: lizardbyte/sunshine:v2025.1027.181930-archlinux
    restart: unless-stopped
    user: "0:0"
    privileged: true
    ipc: host

    environment:
      TZ: Etc/UTC

    devices:
      - /dev/dri:/dev/dri
      - /dev/uinput:/dev/uinput

    volumes:
      - ./config:/home/lizard/.config/sunshine
      - ./assets:/home/lizard/.local/share/sunshine/assets

    ports:
      - "47984-47990:47984-47990/tcp"
      - "48010:48010/tcp"
      - "47998-48000:47998-48000/udp"
```

## 3. Start it

```bash
cd ~/docker/sunshine
docker compose up -d
```

## 4. Start on boot

`/etc/systemd/system/sunshine-compose.service` (replace `youruser`):

```ini
[Unit]
Description=Sunshine stack managed by Docker Compose
Requires=docker.service
After=docker.service network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/youruser/docker/sunshine
ExecStart=/usr/bin/docker compose up -d --remove-orphans
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=120
TimeoutStopSec=120

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now sunshine-compose.service
```

## 5. Firewall (if using ufw)

```bash
sudo ufw allow in on tailscale0 to any port 47984:47990 proto tcp
sudo ufw allow in on tailscale0 to any port 48010 proto tcp
sudo ufw allow in on tailscale0 to any port 47998:48000 proto udp
```

## 6. Pair

Open `https://<host>:47990`, set Sunshine admin credentials, then pair from Moonlight.
