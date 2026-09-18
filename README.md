# Sunshine on Omarchy 4.x

Setup guide for [Sunshine](https://app.lizardbyte.dev/Sunshine) (Moonlight streaming host) on
[Omarchy](https://omarchy.org) 4.x. Verified 2026-09-17 against Omarchy 4.0.3.

**Status:** installer script automating this is planned next. PRs welcome, especially results
on real GPU hardware.

## Install

```bash
omarchy install service sunshine
```

Installs the package, udev rule, and capabilities; opens firewall ports for LAN + Tailscale;
adds a Hyprland autostart entry; enables the systemd user service.

Remove with `omarchy remove service sunshine`.

## Fix input (mouse/keyboard)

Not handled by the installer. Without it, video works but input silently doesn't:

```bash
sudo usermod -aG input "$USER"
```

Log out/in (or reboot) to apply.

## Set admin credentials

```bash
sunshine --creds <username> <password>
```

Restart the service/app after — doesn't hot-reload. Or use the web UI wizard at
`https://<host>:47990` on first visit.

## If it crashes on first boot

Cold boot can race the compositor and hit the service's restart limit
(`journalctl --user | grep sunshine` shows `core-dump` → `start-limit-hit`):

```bash
systemctl --user reset-failed app-dev.lizardbyte.app.Sunshine.service
systemctl --user restart sunshine
```

Hasn't recurred on a warm desktop.

## Notes

- **Encoder:** tries nvenc → vulkan → vaapi → falls back to software (libx264). No GPU
  passthrough (e.g. a VM) always lands on software — check
  `~/.config/sunshine/sunshine.log` for `Found H.264 encoder:` to confirm which one you got.
- **Disk encryption:** can't unlock LUKS remotely — leave the machine running, or drop
  encryption if you need remote power-on access.
- **Pairing:** Sunshine advertises over Avahi/mDNS, so Moonlight should auto-discover it on
  the same LAN.
