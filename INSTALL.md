# Installing WebTerminal on several devices

This guide sets up the full multi-device installation: the macOS app, a
headless Linux server, your own hub (the "collecting terminal") as a
Cloudflare Worker, devices linked to that hub, and conductor sessions that
start work on a paired device.

Parts of the interface are German only for now (the menu bar, the settings
navigation, the hub screens). Labels are quoted as they appear, with a
translation in parentheses. The rest of the interface follows the browser
language; switch it under ⚙.

## 1. How the parts fit

```
 Phone / laptop (browser)
        │  https://<ui-host>   Cloudflare Access, then the hub passkey
        ▼
 ┌──────────────────────────────────────────────┐
 │ Cloudflare Worker (the hub)                  │
 │ One Durable Object: devices, token hashes,   │
 │ the hub passkey. Stores no terminal data.    │
 └──────────────────────────────────────────────┘
        ▲  wss://<device-host>/v1/node   (outbound only, bearer token)
        │
 ┌──────┴─────────────┬───────────────────────────┐
 │ macOS app          │ Linux server              │
 │ server on          │ node server.js under      │
 │ 127.0.0.1:<port>,  │ systemd, tmux, SQLite     │
 │ tmux bundled       │                           │
 └────────────────────┴───────────────────────────┘
        ▲                         ▲
        └── signed conductor jobs (key derived at pairing);
            the hub only carries them
```

- **Device**: one WebTerminal installation with its own database, its own
  login and its own tmux sessions. Every device works without a hub too.
- **Hub**: the same web interface, built in hub mode, showing the sessions of
  every linked device. Each device opens one outbound WebSocket to the hub. No
  device needs an open port, a fixed IP or a tunnel.
- **Two hostnames** for the Worker: the UI host for people (behind Cloudflare
  Access) and the device host, which answers only `POST /v1/enroll` and
  `GET /v1/node` (nothing in front of it).
- **Conductor**: a session on device A that starts and steers child sessions,
  also on device B once A and B are paired.

### What needs access to the private source

The application source lives in a private repository. Only the macOS app is
public, here. Ask the owner of this repository for read access if you need the
source.

| Part | Comes from | Needs the private source? |
|---|---|---|
| Install and update the macOS app | [Releases](https://github.com/SaschaHenning/web-terminal-releases/releases) of this repository | no |
| Link the macOS app to a hub (in its settings) | the app | no |
| Install and update a Linux server | `git clone` of the source; its updater runs `git fetch` | yes, permanent read access |
| Run a device from the Docker image | private container registry, login required | yes |
| Build and deploy your own hub | the `hub/` folder plus the root build script `build:hub` | yes, a fork with your hostnames |
| Link a device from the command line (`npm run hub:join`) | script in a source checkout | yes (the settings page works without) |

For a Linux server the updater's `git fetch` has to run without a prompt: use a
read-only deploy key or a fine-grained token with "Contents: read".

### Your own hub, or someone else's?

A hub knows exactly one person: one passkey, one registration. If you link
your device to someone else's hub, that person can type into your sessions and
you cannot see theirs. If everything should be yours, run your own hub
(section 3). This guide assumes you do.

## 2. macOS app

### Install

1. Download the newest `WebTerminal-<version>-arm64-mac.dmg` from
   [Releases](https://github.com/SaschaHenning/web-terminal-releases/releases).
   Apple Silicon only. No GitHub account needed.
2. Open the `.dmg` and drag **WebTerminal** into Applications.
3. Launch it. The app is signed with a Developer ID and notarized by Apple, so
   macOS opens it without the "Open Anyway" detour.
4. The icon appears in the menu bar. On the first launch the app opens your
   browser by itself; later use "WebTerminal öffnen" (open WebTerminal) in the
   menu.
5. Sign in with "Continue without passkey (localhost)". On the Mac that is
   enough, and it creates the first user, which the hub needs later
   (section 5).

The app bundles its own Node runtime and tmux; Homebrew is not required. It
listens on `127.0.0.1` only, on a free port it picks on the first launch. The
first time a session touches Desktop, Documents or Downloads, macOS asks once
per folder; the grant survives updates. The `.dmg` also carries
`ERSTE-SCHRITTE.html`, a short first-steps page in German.

### Where the data lives

Everything under `~/Library/Application Support/WebTerminal/`:

| Path | Content |
|---|---|
| `settings.json` | port, LAN access, localhost login, automatic updates |
| `session-secret` | cookie signing key (mode 0600) |
| `data/webterminal.db` | the database |
| `data/wt-hub.json` | hub token, written when you link the Mac (mode 0600) |
| `artifacts/` | files sessions leave for you |

Logs: `~/Library/Logs/WebTerminal/`. Deleting the app leaves this folder in
place.

### Menu bar

"Server stoppen" / "Server starten" (stop / start server), "Server-Log
anzeigen" (show server log), "Netzwerkzugriff (LAN)" (LAN access; off means
`127.0.0.1` only), "Autostart bei Anmeldung" (start at login), "Nach Updates
suchen …" (check for updates), "Automatisch aktualisieren" (update
automatically).

### Updates

- The app reads the releases of this repository without a token: 30 seconds
  after launch, then once a day. A new version shows in the menu as "Update auf
  v<version> installieren …" and in the web interface.
- With "Automatisch aktualisieren" on, it checks every 15 minutes and installs
  as soon as no session is working. It restarts its server; tmux sessions keep
  running. The menu says why it is waiting ("Wartet: …").
- Before swapping, the app verifies the signature and the signing team. If an
  update brings a different tmux version, it warns when running sessions would
  become unreachable.
- From the hub: Einstellungen → Geräte (settings → devices), the Mac's row,
  "Prüfen" (check) or "Aktualisieren …" (update). The app picks the request up
  within about 10 seconds and follows the same rules as automatic updates.

Check the installed version:

```bash
/usr/libexec/PlistBuddy -c 'Print CFBundleShortVersionString' /Applications/WebTerminal.app/Contents/Info.plist
```

## 3. Your own hub (Cloudflare Worker)

Needs the private source (section 1).

### 3.1 Prerequisites

- A Cloudflare account with your own domain as a zone in it.
- Cloudflare Zero Trust (Access) on the same account; it gives you a team
  domain `https://<team>.cloudflareaccess.com`.
- A fork of the source repository, Node.js 22. `wrangler` is a dependency of
  the `hub/` package.
- The hub is built for the Workers Free plan: one SQLite-backed Durable Object,
  hibernating WebSockets, and an API socket instead of one request per call.

### 3.2 Pick the hostnames

Two new names in your zone, without existing DNS records, for example:

| Role | Example | In front |
|---|---|---|
| UI host | `t.example.org` | Cloudflare Access |
| Device host | `t-node.example.org` | nothing |
| `RP_ID` (passkey) | `example.org` | a registrable parent of the UI host |

With `RP_ID` on the parent domain, a passkey also works on other hosts under
it, for example a Linux server at `terminal.example.org` that sets the same
`RP_ID`.

### 3.3 Change the code

The hostnames are fixed in the source. In your fork:

| File | What |
|---|---|
| `hub/wrangler.jsonc` | `account_id` (your account, shown by `npx wrangler whoami`), the two `routes` (`pattern` = UI host and device host, keep `custom_domain: true`), `vars.TEAM_DOMAIN`, `vars.POLICY_AUD` (AUD tag, 3.4) |
| `hub/src/config.ts` | `HUB_HOST`, `NODE_HOST`, `RP_ID`, optionally `RP_NAME`. `ROOM_NAME` can stay: a new account starts empty |
| `frontend/src/hub/lib/enroll.ts` | `NODE_HOST`: shown in the "Gerät hinzufügen" (add device) dialog as the `hub:join` line |
| `hubEnroll.js` (optional) | `DEFAULT_HUB`: pre-fill of the hub address on devices that run from your fork |
| `scripts/hub-join.mjs` (optional) | `USAGE`: help text only |

Keep `"workers_dev": false` and `"preview_urls": false`: a `workers.dev` or
preview URL would serve the interface without Access. Keep
`"run_worker_first": true` too, or static files would skip the Worker's Access
check.

The test suites name the original hostnames (for example
`hub/test/helpers.ts` and `hub/test/access.test.ts`); those cases fail until
you change them as well.

### 3.4 Cloudflare Access in front of the UI host

In the Zero Trust dashboard (Cloudflare may rename things):

1. Access → Applications → a new self-hosted application for the **UI host**,
   whole domain, no path.
2. One allow policy for your own identity only, for example your email address.
3. Copy the application's "Application Audience (AUD) Tag" into
   `vars.POLICY_AUD` and your team domain into `vars.TEAM_DOMAIN`.
4. Put **no** Access application in front of the **device host**, and make sure
   no wildcard application covers it: devices authenticate with a bearer token,
   not with Access.
5. Optional, only to install the hub as an app on an iPhone: a second
   application on the UI host with a bypass policy for exactly the paths in
   `INSTALL_FILES` (`hub/src/index.ts`): `/manifest.webmanifest`,
   `/apple-touch-icon.png`, `/icon-192.png`, `/icon-512.png`, `/favicon.svg`,
   `/favicon-32.png`.

The Worker verifies the Access JWT again itself. An empty `POLICY_AUD`, or a
`TEAM_DOMAIN` that does not start with `https://`, refuses every request to the
UI host (503 `misconfigured`).

### 3.5 Build and deploy

From the root of your fork:

```bash
npm ci                     # root: build:hub needs Vite and the frontend
cd hub
npm ci
npx wrangler login
npx wrangler whoami        # account id for account_id
```

Enter the account id, hostnames and Access values (3.3, 3.4), then:

```bash
npm run deploy             # = npm --prefix .. run build:hub && wrangler deploy
```

This builds the interface in hub mode into `hub/dist` and creates the Worker
with both custom domains. Cloudflare creates the DNS records and certificates.

### 3.6 Setup code and first passkey

```bash
openssl rand -base64 32              # keep the value in your password manager
npx wrangler secret put SETUP_CODE   # prompts for the value
```

`SETUP_CODE` is the only lock on the first passkey for anyone who gets past
Access. A value shorter than 16 characters counts as unset; without it,
registration answers 503 `setup_disabled`.

Open `https://<ui-host>`: first the Access login, then the hub asks for the
"Einrichtungscode" (setup code) and creates its passkey. The first verified
passkey closes registration. After ten wrong codes within ten minutes the hub
refuses every attempt (429), the right code included, until the oldest wrong
attempt is ten minutes old.

A lost hub passkey cannot be reset (a reset needs a login). The way out is a
new `ROOM_NAME` in `hub/src/config.ts` and a deploy; the hub then starts empty
and every device joins again.

### 3.7 Update the hub

Pull the new source into your fork (`git fetch upstream` and merge), then:

```bash
npm ci && (cd hub && npm ci && npm run deploy)
```

When a feature spans hub and devices, deploy the Worker first, then update the
devices (section 6.6). Devices reconnect on their own after a deploy.

## 4. Linux server

Needs the private source (section 1). A known-good setup: Ubuntu 24.04, Node 22,
tmux 3.4. Two ways: a systemd user unit written by the install script
(simpler, recommended), or a hardened system unit (4.5).

### 4.1 Packages

```bash
sudo apt-get update
sudo apt-get install -y git tmux build-essential python3 curl
```

Plus Node.js 22. The unit later runs exactly the `node` binary that ran the
install script; a Node from `nvm` breaks the unit once that version is removed.
`build-essential` and `python3` are only needed when npm finds no prebuilt
binary for `better-sqlite3` or `node-pty`.

### 4.2 A dedicated deploy clone

The service never runs from a checkout where agents create branches: the
updater aborts on any uncommitted change.

```bash
git clone <source-repository-url> ~/.local/share/web-terminal/app
mkdir -p ~/.local/share/web-terminal/data
cd ~/.local/share/web-terminal/app
```

The updater follows the clone's `origin` (`WEBTERMINAL_UPDATE_BRANCH`, default
`main`). With a fork, `origin` is your fork.

### 4.3 `.env` before the first start

```bash
npm run setup:native
```

This installs the npm packages, checks tmux, builds the frontend and creates
`.env` with a random `SESSION_SECRET`. Then add to `.env`:

```sh
# Required for the hub link: absolute, outside the clone.
WEBTERMINAL_DB_PATH=/home/<user>/.local/share/web-terminal/data/webterminal.db

# Only when the server gets NO https address (reached only through an SSH
# tunnel and the hub): otherwise NODE_ENV=production marks the cookie Secure
# and signing in over http://localhost fails without an error message.
SESSION_COOKIE_SECURE=0

# Only with your own https address behind a proxy or tunnel:
# RP_ID=example.org
```

Why `WEBTERMINAL_DB_PATH` must be set before the service installer runs: the
hub link starts only when the server receives the variable at start (from the
unit; `.env` alone is not enough). The installer copies the `.env` values into
the unit. Without the entry it writes the variable only while no database
exists yet; if you ran `npm start` first, you get a hint instead.

### 4.4 Service as a user unit

```bash
npm run service:install -- --enable
systemctl --user cat web-terminal.service | grep -E 'WEBTERMINAL_DB_PATH|RESTART_METHOD|KillMode'
curl -s http://127.0.0.1:7681/v1/health
```

This writes `~/.config/systemd/user/web-terminal.service` with
`KillMode=process` (tmux survives a server restart), `Restart=on-failure`,
`NODE_ENV=production`, `WEBTERMINAL_RESTART_METHOD=systemd-user` and every
value from `.env`. It runs `loginctl enable-linger <user>` so the service
starts after a reboot without a login; if that fails, run it yourself.

After changing `.env`: run the installer again, then
`systemctl --user restart web-terminal.service`.

### 4.5 Variant: a hardened system unit

```ini
# /etc/systemd/system/web-terminal.service
[Unit]
Description=WebTerminal
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=<user>
Group=<user>
WorkingDirectory=/home/<user>/.local/share/web-terminal/app
EnvironmentFile=/home/<user>/.local/share/web-terminal/app/.env
Environment=PORT=7681
Environment=NODE_ENV=production
Environment=TMUX_PATH=/usr/bin/tmux
Environment=WEBTERMINAL_DB_PATH=/home/<user>/.local/share/web-terminal/data/webterminal.db
Environment=WEBTERMINAL_RESTART_METHOD=systemd-self-exit
Environment=WEBTERMINAL_SERVICE_NAME=web-terminal.service
ExecStart=/usr/bin/node server.js
LimitNOFILE=8192
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=false
ReadWritePaths=/home/<user>/.local/share/web-terminal /tmp

[Install]
WantedBy=multi-user.target
```

The source ships drop-ins under `scripts/systemd/`; link them rather than copy:

| Drop-in | Purpose |
|---|---|
| `web-terminal-self-exit-restart.conf` | `Restart=always`, `KillMode=process`: the in-app updater restarts the service without `sudo` under `NoNewPrivileges` |
| `web-terminal-priority.conf` | `Nice=-10`, `CPUWeight=500`: the server keeps answering under heavy agent load |
| `web-terminal-claude-profiles.conf.in` | template for write access to the Claude profile folders under `ProtectSystem=strict`; render it, never link it |

```bash
sudo mkdir -p /etc/systemd/system/web-terminal.service.d
sudo ln -sf /home/<user>/.local/share/web-terminal/app/scripts/systemd/web-terminal-self-exit-restart.conf \
  /etc/systemd/system/web-terminal.service.d/self-exit-restart.conf
sudo ln -sf /home/<user>/.local/share/web-terminal/app/scripts/systemd/web-terminal-priority.conf \
  /etc/systemd/system/web-terminal.service.d/priority.conf
sudo systemctl daemon-reload
sudo systemctl enable --now web-terminal.service
curl -s http://127.0.0.1:7681/v1/health
```

Rendering the profile drop-in, and why `%h` points to `/root` in a system unit,
is described in `docs/update-deployment.md` of the source.

### 4.6 First sign-in through an SSH tunnel

Without an https address of its own, the first sign-in goes through a tunnel.
The hub needs it: a request through the hub runs on the device as its first
signed-in user; without one the device answers every API request from the hub
with 401 (terminals still work).

```bash
ssh -L 7681:localhost:7681 <user>@<linux-server>
```

Open `http://localhost:7681` on your own computer and create a passkey. The
first passkey can only be created on a loopback name or under `RP_ID`;
`localhost` through the tunnel qualifies. It closes registration for anyone
else. If sign-in fails silently, `SESSION_COOKIE_SECURE=0` is missing (4.3).

With your own address (reverse proxy, or a Cloudflare Tunnel with Access in
front): set `RP_ID`, keep `WEBTERMINAL_TRUST_PROXY` at its default `1`, leave
`SESSION_COOKIE_SECURE` unset.

### 4.7 Updates

- The server checks at start and every 4 hours with `git fetch` against
  `origin/<branch>` and reads `VERSION` there. That needs the read access from
  section 1.
- Install: Settings → System → Updates & Versionen → "Update now", or in the hub
  under Einstellungen → Geräte → "Aktualisieren …". With sessions running,
  "Update now" takes the forced path: the server restarts, tmux keeps running,
  panes reconnect.
- From the console: `npm run update:install` in the deploy clone.

### 4.8 Emergency access without a browser

If Cloudflare is down and the server has no address of its own, there is no
browser path. Sessions run in tmux, each with its own socket:

```bash
ls "$XDG_RUNTIME_DIR/web-terminal/tmux"    # user unit
ls /tmp/web-terminal-tmux-$(id -u)         # system unit
tmux -S <folder>/wt-<id>.sock attach
```

`WEBTERMINAL_TMUX_SOCKET_DIR` overrides both folders. A plain `tmux ls` does not
show these sessions.

## 5. Link devices

Each device needs: a running server, one local sign-in (Mac: "Continue without
passkey (localhost)"; Linux: 4.6), and `WEBTERMINAL_DB_PATH` at start (the Mac
app sets it itself).

### 5.1 From the settings (Mac, and any device with a browser)

1. In the hub: Einstellungen → Geräte → "Gerät hinzufügen" (settings → devices
   → add device). The one-time code is valid for 10 minutes.
2. On the device: Settings → Zugang (access) → Sammel-Terminal (hub). Enter the
   code, a "Name im Sammel-Terminal" (name in the hub) if you like, and set
   **"Hub-Adresse" (hub address) to `https://<device-host>`**. The field is
   pre-filled with the original author's hub; replace it. Then "Verbinden"
   (connect).
3. The link starts at once, without a restart. The device writes
   `wt-hub.json` (mode 0600) next to its database.

The hub address is the device host, not the hub's web page. Entering the UI
host ends with a message that the hub redirected the request.

### 5.2 From the command line (server without a browser)

In the device's source checkout:

```bash
npm run hub:join -- https://<device-host> <code> [--name X] [--db path]
systemctl --user restart web-terminal.service   # or: sudo systemctl restart …
```

The script finds the database in this order: `--db` (relative to the folder
you typed `npm` in), `WEBTERMINAL_DB_PATH` from the environment (absolute only),
the installed launchd or systemd user unit, and on Linux the system unit with
its drop-ins. It refuses the checkout's default `data/webterminal.db`, because a
server on it never reads the token file. Without `--name` the name is the host
name without `.local`. It prints the file path, never the token.

### 5.3 Troubleshooting

| Message | Cause, fix |
|---|---|
| "the hub refused the code" (401) | code unknown, used, or older than 10 minutes: get a new one |
| `already_enrolled` (409) | the hub already lists this device: remove it in the hub under Geräte ("Entfernen"), then use the same code again within its 10 minutes |
| "redirected the request" | the UI host was entered instead of the device host |
| "no database path" | `WEBTERMINAL_DB_PATH` missing from the unit (4.3), or pass `--db` |
| `[hub] stopped` in the log after close code 4401 | token deleted or device removed in the hub: link again |
| `same hub token on two servers` | two servers share one `wt-hub.json` (copied database): stop one, link the other again |

A stopped link stays off until the next server start. Repairs happen on the
device itself: the hub can never call a device's hub-link or auth endpoints.

"Trennen" (disconnect) on the device's hub page stops the link and deletes
`wt-hub.json`. The hub keeps listing the device until you remove it there.

### 5.4 What does not work through the hub

| What | Instead |
|---|---|
| the device's passkeys, sign-in, changing its hub link | on the device |
| port preview and remote display (VNC) | on the device |
| uploads over 16 MiB, downloads over 32 MiB | on the device |

Terminals, new sessions, artifacts, uploads, the review inbox, history search,
mode, per-device settings (device picker at the top of the settings), pairing
and conductor grants all work through the hub.

## 6. Conductor sessions and pairing

### 6.1 What happens

A conductor session on device A starts and steers child sessions on device B.
For that, A holds a **pairing** with B and B a **permit** for A. Both derive
the same key from the pairing code and both node IDs and keep it in their own
database. Every job is signed with it; the hub carries jobs and stores none of
them. B checks every job against its permit: which subscriptions, which
folders, how many child sessions, until when.

### 6.2 Prerequisites

- A and B are linked to the same hub and online.
- Both run a current version (hub first, 3.7).
- B has at least one subscription set up; a permit names subscriptions.
- Both clocks sync via NTP. More than 30 seconds ahead or about 15 seconds
  behind makes jobs fail (`clock_skew`, `expired`).
- Pairing, permits and conductor grants need an operator sign-in: a passkey on
  the device, or the hub. With "Continue without passkey" on the Mac the page
  is read-only. Doing everything from the hub is simplest.

### 6.3 Pair two devices

In the hub: settings, pick the device at the top, then Zugang → Geräte koppeln
(access → pair devices). Each device shows "This device: <name> · node ID" with
a copy button.

1. **On B** (where child sessions will run): "Other devices may start here" →
   "Permit a device". Node ID of A, subscriptions, folders, maximum child
   sessions, validity (1, 7 or 30 days) → "Permit and show code". B shows a
   26-character code, valid for 10 minutes and shown only this once, plus a
   check value.
2. **On A**: "This device may start there" → "Enter code". Node ID of B and the
   code → "Pair". A shows its check value; it must match the one on B.
3. **On A**, "Check": A sends a check job, B confirms it signed. Result:
   "Confirmed: both devices hold the same key."

A device that links to the hub again gets a new node ID, and its pairings and
permits become void: pair again.

### 6.4 Start a conductor session

In the session list, "+ Dirigent-Session" (phone: "+ Dirigent"). In the hub
the entry appears only when a device supports it through the hub; when several
do, the dialog asks which device runs the conductor.

In "New conductor session": name, subscription, start folder, then the grant:
subscriptions for child sessions, folders for child sessions, "Devices for
child sessions" (tick B), maximum child sessions at once, validity (1, 3, 7 or
14 days) → "Start conductor session". Later a grant can only be revoked or its
child limit changed; subscriptions, folders and devices are never added. A grant made before the pairing never gains that device: start a new
conductor session.

### 6.5 Inside the conductor session

`wt-conduct` is on the `PATH` of every session:

```bash
wt-conduct status                          # grant, devices, own child sessions
wt-conduct start --node <name|n_…> --abo <account-id> --dir <abs|~/…> --task-file <file|->
wt-conduct send <device>~<id> "<text>"
wt-conduct --help
```

Exit code 7 means the outcome on the other device is unknown; the job may have
run there. Never repeat it blindly: run `wt-conduct status` after the time it
names.

### 6.6 Order when setting up

1. Deploy the Worker (3.5 or 3.7).
2. Update and restart every device, so it announces conductor support to the
   hub.
3. Pair on both devices (6.3).
4. Start a new conductor session that names the paired device (6.4).

### 6.7 Ending it

Revoke a grant: the conductor session's ⋯ menu → "Conductor grant …" →
"Revoke grant". Revoke a permit on B, unpair on A. Running child sessions keep
running until they end.

## 7. Security in short

- Whoever controls the Worker can type into every session of every linked
  device and reads along. It stores nothing, but it sees everything that passes
  through, including conductor task texts. Hence: your own hub, Access in
  front, a passkey.
- A request through the hub never counts as local on a device: no localhost
  sign-in, no access to the device's auth and hub-link endpoints.
- Every process of the server's Unix user can read the hub token in
  `wt-hub.json` and the pairing key in the database, agents in sessions
  included. On B the permit still bounds what a job may do.
- The link starts only from a file owned by the server's user with mode 0600,
  and only with `WEBTERMINAL_DB_PATH` given at start, so a copied database does
  not take a device's place.

## 8. Checklist

- [ ] macOS app installed, signed in once with "Continue without passkey (localhost)"
- [ ] Fork with your hostnames (3.3), Access in front of the UI host, nothing in front of the device host
- [ ] `npm run deploy`, `SETUP_CODE` set, hub passkey created
- [ ] Linux server: deploy clone, `.env` with `WEBTERMINAL_DB_PATH`, `service:install`, `/v1/health` answers
- [ ] Linux server: passkey created through the SSH tunnel
- [ ] Both devices linked and shown online in the hub under Geräte
- [ ] Pairing confirmed, conductor session started with both devices, `wt-conduct status` lists them
