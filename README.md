# WoW private server setup files

Prepared configuration for running a private World of Warcraft server on a
Windows host with Docker Desktop. Nothing here is connected to the rest of this
repository; the files live here only so they survive and stay versioned.

Prepared on 28.08.2026.

## Which stack to use

Three projects were evaluated. The deciding factor is the **game client build**,
because every server rejects a client whose build number it does not accept.

| Project | Client required | What it gives you |
| --- | --- | --- |
| [mserajnik/tortoise-deploy](https://github.com/mserajnik/tortoise-deploy) | Turtle WoW `1.18.1` build `7272` | Turtle WoW content. Best-maintained of the three. |
| [kasperfriend/tortoise-docker](https://github.com/kasperfriend/tortoise-docker) | Turtle WoW `1.18.1` build `7272` | Turtle WoW content **plus playerbots** (AI players). |
| [mserajnik/vmangos-deploy](https://github.com/mserajnik/vmangos-deploy) | Vanilla `1.12.1` build `5875` | Blizzlike vanilla. No Turtle content. |

The client build is not a matter of preference. `Penqle/tortoise-wow`
(`src/realmd/RealmList.cpp`) accepts only build `7272` and above; a `5875`
client is refused at the auth challenge before the password is ever checked.
Verify a client's build in the bottom corner of its login screen, or via
`WoW.exe` → Properties → Details.

## Choices baked into these files

Common to all three: Windows host, realm reachable over the **LAN**, no
optional services enabled, everything else left at the documented defaults.
The realm address is set to `192.168.178.28`, the host's IPv4 on its FRITZ!Box
network. Give that machine a DHCP reservation in the router, or the address
will eventually change and every client will fail at the realm screen. Note
that the host also runs a NordVPN tunnel (`NordLynx`, `10.5.0.2`) — that is not
the LAN address and must not be used here.

### `tortoise-deploy/compose.yaml`

Differs from the upstream `compose.yaml.example` by **one line** (the realm
address). Uses the `stable` images. Time zone left at the shipped `Etc/UTC`.
Ports: realmd `3724`, mangosd `8085`.

### `vmangos-deploy/compose.yaml`

Differs from upstream by **four lines**: three `TZ` values set to
`Europe/Luxembourg`, plus the realm address. Uses the `5875` images, matching a
vanilla `1.12.1` client. Warden and Anticheat left disabled as shipped.
Ports: realmd `3724`, mangosd `8085`.

### `tortoise-docker/env.template`

Copy to `.env` in a clone of that repository. Playerbots enabled with 10 random
bots, the count the upstream README recommends for a first start. **The two database passwords are placeholders** — replace both with
strong values before first start, and keep them out of version control (the
upstream `.gitignore` already excludes `.env`). Use alphanumeric characters
only: the repository's `.cmd` helper scripts interpolate the password unquoted
into `mariadb -p%DBPASS%`, so punctuation breaks them.
Ports: realmd `3724`, mangosd **`8090`**.

## Known issues in kasperfriend/tortoise-docker

Found by reading the repository on 28.08.2026. None are blocking, but all four
will cost time if hit unaware.

1. **`GAME_BIND_IP` is dead configuration.** It appears only in `.env.example`
   and is referenced nowhere in `docker-compose.yml`. Setting it to `127.0.0.1`
   does not restrict anything; the game ports publish on all interfaces
   regardless, on every network the machine joins. The database port is
   correctly bound to localhost only.
2. **`repair-db.cmd` hardcodes `set DBPASS=root`** and never reads `.env`,
   unlike `backup-server.cmd` and `update-server.cmd`. With a strong root
   password it fails with access denied. Fix by replacing that line with the
   same `findstr /b "DB_ROOT_PASSWORD=" .env` loop the other scripts use.
3. **`config/` is rewritten on every start.** `render-config.sh` runs `sed -i`
   over the tracked config files, so the working tree goes dirty from the first
   boot. Run `git checkout -- config` before pulling repository updates.
   `update-server.cmd` only pulls images, so it is unaffected.
4. **Published images contain no client-data extractors.** CI builds with
   `USE_EXTRACTORS=OFF`, so `mapextractor`, `vmapextractor`, `vmap_assembler`
   and `MoveMapGen` are absent. Extract with
   `ghcr.io/mserajnik/tortoise-server:stable extract-client-data` instead (same
   client generation) and copy the resulting `dbc`, `maps`, `vmaps`, `mmaps`
   into `data/`, or rebuild locally with `USE_EXTRACTORS=ON` and
   `EXTRACTORS_ONLY=ON`.

Bot counts in `.env` always override `config/aiplayerbot.conf`, because
`render-config.sh` rewrites `AiPlayerbot.Enabled`, `MinRandomBots` and
`MaxRandomBots` from the environment on every start. Edit `.env` only.

## Setup outline (Windows)

Full step-by-step version with exact commands: [INSTALL.md](INSTALL.md).

1. Install Docker Desktop (WSL2 backend) and confirm `docker version` and
   `docker compose version` both respond.
2. Clone the chosen repository and copy its example files into place.
3. Drop in the prepared file from this folder and substitute the LAN IP.
4. Allow the two TCP ports inbound on the **Private** profile only —
   `3724` plus `8085` (mserajnik stacks) or `8090` (tortoise-docker).
5. Extract the client data. This runs for hours and must finish before the
   first server start.
6. `docker compose up -d`, then follow `docker compose logs -f mangosd` and
   wait for the ready line — `World server is up and running!` on Tortoise,
   `World initialized.` on VMaNGOS. Never interrupt the first start: the
   initial database creation is not resumable.
7. Create an account by attaching to the `mangosd` container, then detach with
   <kbd>Ctrl</kbd>+<kbd>P</kbd> <kbd>Ctrl</kbd>+<kbd>Q</kbd>. Keep the account
   you play on at GM level `0`; higher levels make characters invulnerable.
8. Point each client's `realmlist.wtf` at the LAN IP.

## Security notes

Neither mserajnik stack publishes a database port at all, and tortoise-docker
binds MariaDB to `127.0.0.1` only. The default database credentials are
therefore reachable only from the containers themselves. That stops being true
the moment phpMyAdmin is enabled or a port is forwarded — do not expose the
database, and do not port-forward the game ports to the internet without
deciding how to secure them first. Backups contain account password hashes.
