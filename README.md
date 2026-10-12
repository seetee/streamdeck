# streamdeck

Small scripts bound to buttons on an Elgato Stream Deck and a Stream Deck Mini.

## Requirements

- Linux with [StreamController](https://github.com/StreamController/StreamController) (Flatpak: `com.core447.StreamController`)
- `bash`, `ssh`
- Key-based SSH access to the hosts the scripts control

## Layout

```
scripts/<name>/   one folder per button script
config.env        local settings (hosts, users, names). Git-ignored.
```

## Setup

```sh
cp config.example.env config.env
$EDITOR config.env
```

## Scripts

| Script | What it does |
|--------|--------------|
| `scripts/minecraft/` | Shows whether the Minecraft server is up. A press starts or stops it. *(planned)* |

See [PLAN.md](PLAN.md) for the roadmap and open questions.
