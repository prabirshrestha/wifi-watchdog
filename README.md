# wifi-watchdog

Keeps a NetworkManager Wi-Fi connection up, also at the login screen, so SSH and RDP over Wi-Fi keep working. For Linux systems with NetworkManager older than 1.58.

Useful for Wi-Fi-only machines that you reach over the network, for example:

- NVIDIA DGX Spark (DGX OS 7, NetworkManager 1.46)

## Problem

NetworkManager before 1.58 asks for a new Wi-Fi password after one failed WPA handshake. It does not try the saved password again. The GDM login screen runs also when no monitor is connected. It takes the request and waits for a person, so the Wi-Fi stays down until someone logs in or toggles Wi-Fi.

A handshake timeout at boot is enough to cause this. The log shows:

```
wlP9s9: CTRL-EVENT-DISCONNECTED bssid=... reason=15
device (wlP9s9): Activation: (wifi) disconnected during association, asking for new key
device (wlP9s9): state change: config -> need-auth
```

The upstream fix is [NetworkManager](https://gitlab.freedesktop.org/NetworkManager/NetworkManager) commit [`746a5902a`](https://gitlab.freedesktop.org/NetworkManager/NetworkManager/-/commit/746a5902ad85ec0611a3e6ebfd7b68b45621a40b) ("wifi: use authentication retry mechanism", [MR 2308](https://gitlab.freedesktop.org/NetworkManager/NetworkManager/-/merge_requests/2308), closes [issue 1316](https://gitlab.freedesktop.org/NetworkManager/NetworkManager/-/issues/1316)), first in 1.58.0. It is not backported:

| System | NetworkManager | Fixed |
|---|---|---|
| Ubuntu 24.04 (also DGX OS 7) | 1.46.0 | No |
| Ubuntu 26.04 (also DGX OS 8) | 1.54.3 | No |

## Fix

A systemd timer checks the connection every minute. If the connection is not active, it runs `nmcli connection up <connection>`, which uses the saved password and replaces the stuck request. If that fails, it tries again next minute. It does nothing while the connection is up. If you turn Wi-Fi off on purpose, it turns the connection back on within a minute; run `uninstall` first.

## Install

Download the script:

```bash
sudo curl -fsSL https://raw.githubusercontent.com/prabirshrestha/wifi-watchdog/main/wifi-watchdog -o /usr/local/bin/wifi-watchdog
sudo chmod +x /usr/local/bin/wifi-watchdog
```

Or clone the repo to read it first, and copy it:

```bash
git clone https://github.com/prabirshrestha/wifi-watchdog.git
sudo install -m 755 wifi-watchdog/wifi-watchdog /usr/local/bin/
```

The timer runs the script from the path you start `install` with, so keep it there.

Show the saved Wi-Fi connections:

```bash
wifi-watchdog list
```

```
NAME                             ACTIVE  WATCHED  READY BEFORE LOGIN
MyWifi                           yes     no       yes
```

If your Wi-Fi is not in the list, connect to it first. Run this on the machine itself (monitor and keyboard, or a KVM), because it has no network yet. `--ask` asks for the password, so it is not in your shell history. A connection made with `sudo nmcli` is saved for all users:

```bash
nmcli device wifi list
sudo nmcli --ask device wifi connect MyWifi
```

If `READY BEFORE LOGIN` is `no`, the password is not saved for all users (for example, it is only in your user keyring). Save it for all users. The command asks for the password, so it is not in your shell history:

```bash
sudo wifi-watchdog set-password MyWifi
```

Start watching the connection. With no name, it uses the active Wi-Fi connection:

```bash
sudo wifi-watchdog install MyWifi
```

## Commands

| Command | What it does |
|---|---|
| `list` | Show saved Wi-Fi connections |
| `set-password [CONNECTION]` | Save the Wi-Fi password for all users. Asks for it. |
| `install [CONNECTION]` | Start watching the connection |
| `status` | Show the timer and recent log lines |
| `update` | Download the newest version and keep the settings |
| `uninstall` | Stop watching and remove all files, including the script |
| `version` | Print the version |

Logs: `journalctl -t wifi-watchdog`

Uninstall when your system has NetworkManager 1.58 or later (`NetworkManager --version`):

```bash
sudo wifi-watchdog uninstall
```

## Installed files

| File | Purpose |
|---|---|
| `/usr/local/bin/wifi-watchdog` | The script (you copy it) |
| `/etc/default/wifi-watchdog` | Connection name |
| `/etc/systemd/system/wifi-watchdog.service` | Runs `wifi-watchdog check` |
| `/etc/systemd/system/wifi-watchdog.timer` | Runs the check 90 s after boot, then every 60 s |

## Releasing

Change `VERSION` in the script on every change. `update` compares it with the installed version.
