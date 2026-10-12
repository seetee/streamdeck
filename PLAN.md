# Plan

## 1. Minecraft toggle (`scripts/minecraft/`)

The Minecraft server runs in a Docker container on a Proxmox host on the LAN.
The scripts reach it over SSH and use the `docker` CLI.

- **Status:** `ssh $MC_SSH_USER@$MC_SSH_HOST docker inspect -f '{{.State.Running}}' $MC_CONTAINER`
- **Press:** `docker start` or `docker stop`, depending on status
- **Button:** a different icon or label for up and down

### To research

- [ ] How StreamController can poll a script and change a button's icon or label. Is the built-in OS plugin enough, or is a community plugin needed?
- [ ] Does the Flatpak sandbox need `flatpak-spawn --host` to reach `ssh` and `~/.ssh`?

### Open questions

- [ ] Docker host: its hostname/IP, the SSH user, and the container name
- [ ] What does "up" mean: the container is running, or port 25565 answers? Startup is slow, so maybe a third "starting" state is needed.
- [ ] Can the SSH user run `docker` without sudo (is it in the `docker` group)?
- [ ] Which deck gets which buttons? (Mini vs. the big deck)
- [ ] Should "stop" need a confirmation, for example a long press?

## Ideas

- …
