# Plan

## 1. Minecraft power button (`scripts/minecraft/`)

A key on the **Stream Deck Mini** on **narrator** shows the Minecraft server's state. A press turns the whole Minecraft guest (**potre**) on, or stops Minecraft cleanly and shuts potre down.

```
Köksserver (Proxmox, also NetBird + Pi-hole)
└── potre (Ubuntu Server guest, Minecraft only)
    └── Docker
        └── mc_potre (itzg/minecraft-server:java8, Forge 1.12.2, Project Ozone 3)
```

### Design

```
narrator: Stream Deck Mini ─► StreamController (Flatpak) ─► scripts/minecraft/mc status|toggle
                                                             ├─► Proxmox API (curl): is potre on? start / shutdown
                                                             └─► ssh potre-button ─► mc-guard: status / start / stop
```

- **Two levels, two tools.** When potre is off, nothing on it can answer, so its power goes through the Proxmox API. Anything inside potre goes over SSH to `mc-guard`.
- **Proxmox API token, not SSH to Proxmox.** The token only gets `VM.Audit` + `VM.PowerMgmt` on potre's ID. It can see potre and turn it on or off, and nothing else on the hypervisor. SSH to Proxmox would mean root on the host that also runs Pi-hole and NetBird.
- **`mc-guard`** on potre is the only thing the button's SSH key may run (`restrict,command=` in `authorized_keys`):
  - `status`: prints `DOWN`, `STARTING`, `UP <players>` or `ERR`, using `docker inspect` and `mc-health`
  - `start`: `docker start mc_potre`
  - `stop`: refuses while `mc-health` reports `online>0`; otherwise `docker stop mc_potre` (waits up to 150 s)
- **`mc`** on narrator has two commands:
  - `status`: potre off → `OFF`; potre on but SSH not answering → `BOOT`; otherwise the output of `mc-guard status`. Proxmox not answering → `?`.
  - `toggle`:

    | State | Press does |
    |---|---|
    | `OFF` | start potre over the API. Minecraft comes up by itself (`restart: always`). |
    | `DOWN` | `mc-guard start` |
    | `STARTING`, `UP 0`, `ERR` | `mc-guard stop`, and if that succeeds, shut potre down over the API |
    | `UP 1+` | nothing (`mc-guard stop` refuses) |
    | `BOOT`, `?` | nothing (shortcut: a hung potre has to be stopped in the Proxmox UI) |

- **Minecraft is stopped *before* potre shuts down.** It doesn't rely on the shutdown chain (Proxmox → systemd → dockerd), whose timeouts I haven't checked.
- **`restart: always` instead of `unless-stopped`.** The button stops the container with `docker stop`, and `unless-stopped` would then keep it down on the next boot. With `always`, it comes back whenever potre boots.
- **One key, two actions** in StreamController, both OS plugin → *Run Command*:
  - A: `~/documents/streamdeck/scripts/minecraft/mc status`, auto-run every 10 s, output shown as the label.
  - B: `…/mc toggle`, no auto-run.
  - A press runs both, so the label refreshes right away.

### Verified facts (2026-10-12)

- *Run Command* wraps commands in `flatpak-spawn --host` ([RunCommand.py](https://github.com/StreamController/OSPlugin/blob/main/actions/RunCommand/RunCommand.py)), so the script runs on narrator itself with its `ssh`, `curl` and `~/.ssh`.
- *Run Command* runs the same command on auto-run and on a press, hence two actions. Auto-run maximum is 60 s, and it sets only the label, not the image.
- The bug where auto-run stopped after a press ([#244](https://github.com/StreamController/StreamController/issues/244)) was closed 2025-06-26.
- Compose stores `stop_grace_period` in the container ([create.go](https://github.com/docker/compose/blob/main/pkg/compose/create.go)), so a plain `docker stop mc_potre` waits 150 s. Checked on potre: `StopTimeout=150`, `STOP_DURATION=120`.
- `mc-health` exists in the `java8` image and prints `online=N`. The healthcheck now uses it; `docker ps` shows `(healthy)`.
- `bore` orphan container removed. No ports are forwarded to the internet; friends reach 25565 only over NetBird. RCON is no longer published.
- Optional later: [Status Plugin](https://github.com/bgrande/StreamControllerStatusPlugin) (GPL-3.0) colours the key green or red. It doesn't use `flatpak-spawn`, so the command needs a `flatpak-spawn --host` prefix. It's a small project with 1 star.

### Problems and solutions

| Problem | Solution |
|---|---|
| No static IPs | DHCP reservations in the Google Home app (Wi-Fi → Network settings → Advanced networking → DHCP IP reservations, [guide](https://support.google.com/googlehome/answer/6274660?hl=en)), now needed for **both** Köksserver and potre. If a reservation doesn't stick, add `dhcp-identifier: mac` in potre's netplan. Use IPs in configs, not Pi-hole names. |
| Proxmox uses a self-signed TLS certificate, and its name/IP may not match | Pin the certificate's key: `curl -k --pinnedpubkey sha256//<hash>`. It still refuses any other server and survives IP changes. |
| The SSH key and API token have no passphrase (a button can't type one) | Both are scoped to the minimum: the key can only run `mc-guard`, and the token can only power potre. The token goes in `config.env` (git-ignored, `chmod 600`). |
| The `docker` group is root-equivalent | Enforced by `restrict,command="/usr/local/bin/mc-guard"`; the key can't get a shell. |
| ssh or curl hang when something is down | SSH: `BatchMode yes`, `ConnectTimeout 3`. curl: `--max-time 5`. |
| A stray tap kicks players | `mc-guard stop` refuses while players are online. Force stop by hand: `ssh potre docker stop mc_potre`. |
| A big modpack takes minutes to start | `start_period: 300s`; the label shows `BOOT` then `STARTING` until `mc-health` passes. |

### Steps

1. ~~**Kenneth:** answer the open questions.~~ Done 2026-10-12.
2. **Router:** DHCP reservations for Köksserver and potre.
3. ~~**potre:** in `docker-compose.yml`, change to `restart: always` and `start_period: 300s`, then run `docker compose up -d`.~~ Done and tested 2026-10-12.
4. **Proxmox:** create a role, a user and a token:
   ```sh
   pveum role add MCPower --privs "VM.Audit VM.PowerMgmt"
   pveum user add streamdeck@pve
   pveum acl modify /vms/<vmid> --users streamdeck@pve --roles MCPower
   pveum user token add streamdeck@pve button --privsep 0
   ```
5. **narrator:** generate the SSH key, add a `Host potre-button` entry to `~/.ssh/config`, and fill in `config.env` (Proxmox IP, node name, guest type, VMID, token, certificate pin).
6. **potre:** install `mc-guard` in `/usr/local/bin/`, and add the key to `authorized_keys` with the `restrict,command=…` prefix.
7. **Test from a terminal:** `mc status` in every state, `mc toggle` from `OFF` → `UP` → `OFF`, a refused stop with a player online, and `ssh potre-button ls` must be refused.
8. **StreamController:** on the Mini, add actions A and B and the icon.
9. **Repo:** add `mc`, `mc-guard`, the new `config.example.env` and the README section. `shellcheck` both scripts.

### Graphics

- **Avoid official Minecraft art.** It's Mojang/Microsoft IP and doesn't belong in a public repo.
- **[game-icons.net](https://game-icons.net)**: about 4,000 game icons (pickaxe, cube), CC BY 3.0. Credit the author in the README. **Recommended.**
- Alternatives: [Lucide pickaxe](https://lucide.dev/icons/pickaxe) (ISC), StreamController's store icon packs (Material, Tabler, Font Awesome, Bootstrap, Simple Icons), or your own 16×16 pixel-art block (CC0).
- The Mini's keys are small, so use one bold shape. The state is shown in the label.

### Open questions

1. ~~VM or LXC, ID?~~ **Answered:** potre is **qemu (VM) 102**. API path `/nodes/<node>/qemu/102/status/{current,start,shutdown}`. Other guests: 100, 101 (LXC, Nginx Proxy Manager), 103 (qemu blossom, the other containers); the token can't touch them.
   - API `shutdown` goes through the guest agent (installed).
2. ~~Node name?~~ **Answered:** `server` (`server.kitchen.local`, 192.168.42.201, from Proxmox's `/etc/hosts`). The first API call in step 7 confirms it.
   - The Proxmox installer sets a static IP, and `/etc/hosts` must match it, so Köksserver is most likely static already (check `/etc/network/interfaces`). Still reserve .201 in the router so DHCP never hands it to another device.
   - potre's QEMU guest agent is installed and running (its IP shows in the Proxmox summary), so the API `shutdown` goes through the agent.

All questions answered. Next: step 2 (DHCP reservations), then step 4 onward.

## Ideas

- …
