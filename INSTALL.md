# Installation walkthrough (Windows)

Target: [kasperfriend/tortoise-docker](https://github.com/kasperfriend/tortoise-docker)
with playerbots, Turtle WoW client `1.18.1` build `7272`, realm reachable on the
LAN at `192.168.178.28`.

Total time is dominated by one step: extracting the client data runs for hours.
Everything else takes about 30 minutes.

## 0. Before you start

- **Use PowerShell, not Command Prompt.** Every command here is PowerShell:
  `$HOME`, `Get-Content`, backtick line continuation and `${PWD}` all fail in
  `cmd.exe`. Open it with <kbd>Win</kbd>+<kbd>X</kbd> → Terminal, and check that
  the prompt begins with `PS`.

- **Disk space.** Budget for the client itself plus roughly 10 GB or more of
  extracted map data, and the Docker images and database on top. Check the
  drive before starting rather than discovering it three hours in.
- **Memory.** Playerbots want real RAM. If the host is tight, cap WSL2 in
  `%UserProfile%\.wslconfig` rather than letting it swap.
- **Turtle WoW client.** Must be build `7272`. Verify in the bottom corner of
  the client's login screen, or via `WoW.exe` → Properties → Details.

## 1. Docker Desktop

Install from [docker.com](https://www.docker.com/products/docker-desktop/),
accepting the WSL2 backend. Launch it and wait for the whale icon to stop
animating, then confirm:

```powershell
docker version
docker compose version
```

Both must print a version. Docker Desktop has to be running for every command
below; nothing here works with it closed.

## 2. Clone the server repository

```powershell
cd $HOME
git clone https://github.com/kasperfriend/tortoise-docker
cd tortoise-docker
```

## 3. Put the prepared `.env` in place

Copy the prepared `.env` into this folder (`C:\Users\<you>\tortoise-docker\.env`).
It already carries the LAN address, 10 bots and strong database passwords.
Confirm it reads back correctly:

```powershell
Select-String -Path .env -Pattern 'REALM_ADDRESS|AI_M(IN|AX)_RANDOM_BOTS'
```

Expected: `REALM_ADDRESS=192.168.178.28`, and both bot counts at `10`.

Never commit this file. The repository's `.gitignore` already excludes it.

## 4. Fix `repair-db.cmd`

Shipped broken for anyone using a non-default root password — it hardcodes
`root` instead of reading `.env`. Fix it now, while you remember:

```powershell
(Get-Content repair-db.cmd) -replace '^set DBPASS=root$', 'for /f "tokens=1,2 delims==" %%A in (''findstr /b "DB_ROOT_PASSWORD=" .env 2^>nul'') do set DBPASS=%%B' | Set-Content repair-db.cmd
```

## 5. Extract the client data

The published images of this project contain no extractors, so use the tool
from mserajnik's Tortoise stack, which targets the same client generation.

```powershell
cd $HOME
git clone https://github.com/mserajnik/tortoise-deploy.git
cd tortoise-deploy
```

Copy the **entire contents** of your Turtle WoW client folder into
`storage\mangosd\client-data\`, so that `storage\mangosd\client-data\Data\`
exists — the extractor aborts without it. Then:

```powershell
docker run -i `
  -v "${PWD}/storage/mangosd/client-data:/opt/tortoise/storage/client-data" `
  -v "${PWD}/storage/mangosd/extracted-data:/opt/tortoise/storage/extracted-data" `
  --rm ghcr.io/mserajnik/tortoise-server:stable extract-client-data
```

This runs for hours. Scrolling notices and errors are normal as long as output
keeps moving. Leave the machine on.

When it finishes, copy the four result folders into the server's data folder:

```powershell
Copy-Item -Recurse storage\mangosd\extracted-data\dbc,`
  storage\mangosd\extracted-data\maps,`
  storage\mangosd\extracted-data\vmaps,`
  storage\mangosd\extracted-data\mmaps `
  -Destination $HOME\tortoise-docker\data\
```

Verify all four arrived:

```powershell
Get-ChildItem $HOME\tortoise-docker\data
```

## 6. Firewall

PowerShell **as Administrator**, once. Note the world port is `8090` here, not
the `8085` other Tortoise setups use:

```powershell
New-NetFirewallRule -DisplayName "Turtle realmd" -Direction Inbound -Protocol TCP -LocalPort 3724 -Action Allow -Profile Private
New-NetFirewallRule -DisplayName "Turtle mangosd" -Direction Inbound -Protocol TCP -LocalPort 8090 -Action Allow -Profile Private
```

Use the `Private` profile only. If Windows classifies the network as Public,
change the network's category instead of widening the rule.

## 7. First start

```powershell
cd $HOME\tortoise-docker
docker compose up -d
docker compose logs -f mangosd
```

The first start imports the world database and then builds playerbot caches and
travel data before the world opens — allow up to 20 minutes.
Wait for:

```text
World server is up and running
```

**Do not interrupt this.** The initial database import is not resumable; a
half-finished database has to be deleted and rebuilt.

If it looks stuck, check whether it is still working rather than assuming:

```powershell
docker compose logs --tail=20 mangosd
```

Repeated `INSERT` lines mean the import is still running. Wait.

> Bot count starts at `10`, which is what the upstream README recommends for a
> first start. To raise it later, edit both `AI_MIN_RANDOM_BOTS` and
> `AI_MAX_RANDOM_BOTS` in `.env` and run `docker compose up -d mangosd`. The
> server rebuilds bot data on that restart, but the world is already imported by
> then, so it is far quicker than the first boot.

## 8. Create your account

```powershell
.\create-account.cmd
```

Answer the prompts and leave the GM level at `0` for a character you intend to
play. The script verifies the account against the database afterwards; an empty
result means mangosd was not ready yet — wait for the ready line and retry.

## 9. Point the client at the server

In your Turtle WoW client folder, edit `realmlist.wtf` to exactly:

```text
set realmlist 192.168.178.28
```

Do the same on any other PC on the network that should connect. Log in with the
account you just created.

If a **different** PC cannot connect while the server machine can, suspect
NordVPN first: its kill switch and LAN-invisibility options block local traffic.
Disconnect the tunnel and retry before debugging anything else.

## 10. Routine operation

| Task | Command |
| --- | --- |
| Start | `.\start-server.cmd` or `docker compose up -d` |
| Stop (keeps all data) | `.\stop-server.cmd` or `docker compose down` |
| Watch the world server | `docker compose logs -f mangosd` |
| Check container state | `docker compose ps` |
| Back up the databases | `.\backup-server.cmd` |
| Update the server | `.\update-server.cmd` |
| Check/repair tables | `.\repair-db.cmd` |

Back up before every update — `update-server.cmd` offers to do it for you,
and the answer is always yes. This project tracks a fork that rebuilds daily
and its own README warns that things break, so weekly updates with a backup
first is a sensible rhythm.

Expect `git status` in the server folder to show modified files under `config/`
from the first start onward: the container rewrites those on every boot. That
is normal here. Run `git checkout -- config` before pulling repository updates.

Bot counts live in `.env` only. Editing `config/aiplayerbot.conf` by hand
accomplishes nothing, because the container overwrites those keys from the
environment on every start.
