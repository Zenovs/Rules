# interne app auf dem gemeinsamen server anlegen

**wann anwenden:** eine neue kleine interne app soll auf den gemeinsamen
linux-server (`<SERVER>`, `<SERVER-IP>`, ubuntu 24.04) — **ohne** die bestehenden
apps zu stören, und der mac-client wird wie gewohnt per browser/portal installiert.

> **für coding-agents & repos:** diese datei beschreibt, wie eine neue interne app
> auf den geteilten server kommt und wie der mac-client per portal installiert
> wird. halte dich an diese konventionen — dann läuft alles nebeneinander.

die box hostet **mehrere kleine interne apps gleichzeitig**. es gibt **ein
gemeinsames portal** (eine webseite mit einer kachel pro app, erreichbar unter
der nackten server-ip) und **dünne mac-clients**, die direkt mit ihrer app reden.
reverse-proxy ist **ein einziges, geteiltes caddy**.

## voraussetzungen
- ssh-zugang zum server
- code in einem github-repo
- freier tcp-port aus der port-registry (§3)
- node 22, pnpm 11, caddy auf dem server (vorhanden); **kein** postgres/docker per default

## 1. das mentale modell

```
                 http://<SERVER-IP>/            ← gemeinsames PORTAL (Caddy :80)
                 ┌───────────────────────────┐    eine Kachel pro App + Download-Befehl
 Browser  ─────▶ │  Interne Apps             │
                 │  [App-A] [App-B] [ … ]     │
                 └───────────────────────────┘
                         │ curl|bash install-client.sh
                         ▼
 Mac-Client (dünne .app) ───────▶  http://<SERVER-IP>:<APP-PORT>   (direkt, ohne Caddy)
                                   ┌──────────────────────────────┐
                                   │ systemd-Dienst je App         │
                                   │ next start -H 0.0.0.0 -p PORT │
                                   └──────────────────────────────┘
```

**zwei wege, eine box:**
- **browser → portal (port 80):** download-hub. caddy liefert die portalseite und
  die installer unter `/downloads/`.
- **client → app (eigener port):** der installierte mac-client spricht die app
  direkt unter `http://<SERVER-IP>:<APP-PORT>` an. **kein** caddy dazwischen.

## 2. die goldenen regeln (das „worauf achten")

1. **eigener port, an `0.0.0.0` gebunden.** wähle einen freien tcp-port und trage
   ihn in die port-registry ein. niemals einen belegten port nehmen.
2. **eigenes verzeichnis** `/opt/<app>/` und **eigener system-user** `<app>`.
3. **eigener systemd-dienst** `<app>.service` (+ optional `<app>-scheduler`).
   eindeutiger name, kollidiert nicht mit anderen.
4. **eigene daten, eigene secrets.** sqlite-datei oder eigene postgres-db + eigener
   db-user. `.env` nie committen.
5. **das portal und caddy sind geteilt.** du *ergänzt* dort eine kachel bzw. einen
   `/downloads/`-pfad — du ersetzt nichts.
6. **der client wird per `curl | bash` installiert**, nicht per browser-download
   (umgeht gatekeeper, siehe §6).
7. **self-hosted actions-runner** deployt per push (`git reset --hard origin/main`).
   der server folgt strikt dem main-branch.

## 3. port-registry (stand pflegen!)

| port | belegt durch | zweck |
|------|--------------|-------|
| 22   | system       | SSH |
| 53   | system       | DNS |
| 80   | Caddy        | **gemeinsames Portal** + `/downloads/*` |
| 443  | Caddy        | TLS (optional, interne Hostnamen) |
| 631  | CUPS         | Druck |
| 2019 | Caddy        | Caddy-Admin |
| 4577 | <APP-A>      | App-A (`next start`) |
| 4578 | <APP-B>      | App-B (`next start`) |
| 45xx | *frei*       | nächste App hier eintragen |

> **freien port finden:** `ss -tlnH | awk '{print $4}' | sed -E 's/.*:([0-9]+)$/\1/' | sort -n | uniq`
> konvention: bleib im bereich **4577+** und zähl hoch.

## 4. verzeichnis-layout pro app

```
/opt/<app>/
  app/          ← das Repo (Next.js), folgt origin/main; hier liegt .env
  downloads/    ← herunterladbare Clients: <App>-Mac.zip, install-client.sh, version.json
  deploy.sh     ← vom Actions-Runner ausgeführt
```

dazu **ein** gemeinsames portal (app-neutral). heute liegt es unter
`/opt/<app-a>/portal/index.html`; künftige apps fügen dort eine kachel hinzu
(oder es wandert nach `/opt/portal/`, siehe §7).

## 5. systemd-dienst (vorlage)

`/etc/systemd/system/<app>.service`:

```ini
[Unit]
Description=<App> (interner Server)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=<app>
Group=<app>
WorkingDirectory=/opt/<app>/app
Environment=NODE_ENV=production
Environment=HOME=/home/<app>
EnvironmentFile=/opt/<app>/app/.env
# An ALLE LAN-Interfaces binden, eigener Port:
ExecStart=/usr/bin/corepack pnpm exec next start -H 0.0.0.0 -p <APP-PORT>
Restart=always
RestartSec=3
NoNewPrivileges=true
ProtectSystem=full

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now <app>.service
```

## 6. client-verteilung — der „kein-gatekeeper"-trick

der browser markiert downloads mit dem quarantäne-flag → macos zeigt „… kann nicht
geöffnet werden". **`curl` setzt dieses flag nicht.** deshalb installiert man den
client per terminal-einzeiler, den das portal anzeigt:

```bash
curl -fsSL http://<SERVER-IP>/downloads/install-client.sh | APP_HOST=<SERVER-IP> bash
```

`install-client.sh` (pro app, in `/opt/<app>/downloads/`):

1. lädt `<App>-Mac.zip` (eine gezippte `.app`) per `curl`,
2. entpackt mit `ditto` nach `/Applications` (oder `~/Applications` ohne admin),
3. `xattr -dr com.apple.quarantine` (zur sicherheit),
4. schreibt die server-url `http://<SERVER-IP>:<APP-PORT>` in
   `~/.<app>-client/server.txt` (der client liest das beim start),
5. `open` die app.

dazu `version.json` für update-checks: `{ "version": N, "url": "/downloads/<App>-Mac.zip" }`.

> **der client selbst** ist eine **dünne native macos-webview-`.app`** (lädt nur
> `http://<SERVER-IP>:<APP-PORT>`), nicht electron/tauri — darum ist das zip klein
> (~1 MB) und der download schnell. wer einen tauri-client hat: gleiches schema
> (`.app` zippen, `install-client.sh` anpassen), nur grösser.

## 7. caddy (geteilt) — portal + downloads

`/etc/caddy/Caddyfile` (eine datei für die ganze box). der `:80`-block liefert das
portal und **pro app** einen `/downloads/`-zweig. **ergänzen, nicht ersetzen:**

```caddy
:80 {
    # je App ein eigener Download-Pfad:
    handle_path /downloads/app-a/*  { root * /opt/<app-a>/downloads; file_server }
    handle_path /downloads/app-b/*  { root * /opt/<app-b>/downloads; file_server }
    # Portal-Startseite (Kachel-Hub für alle Apps):
    handle { root * /opt/portal; file_server }
}
```

> **heutiger stand:** <APP-A> nutzt `/downloads/*` direkt aus `/opt/<app-a>/downloads`
> und das portal aus `/opt/<app-a>/portal`. beim aufnehmen weiterer apps entweder
> pro-app-pfade wie oben einführen **oder** die app liefert ihre downloads über
> ihren **eigenen port** aus (siehe <APP-B>: `http://<SERVER-IP>:4578/downloads/...`
> direkt aus der next.js-app — dann ist gar kein caddy-eingriff nötig). letzteres
> ist am konfliktärmsten.

**konfliktärmste variante (empfohlen für neue apps):** die app liefert
portal-kachelinhalt und installer über ihren **eigenen port** selbst aus. dann
braucht es am geteilten caddy **keine änderung** — die app ist unter
`http://<SERVER-IP>:<APP-PORT>/` vollständig (download-seite + `/downloads/*` + app).
das portal verlinkt nur dorthin.

### 7a. kachel ins gemeinsame portal eintragen (PFLICHT — sonst ist die app von der nackten ip nicht auffindbar)

die nackte ip `http://<SERVER-IP>/` (port 80) gehört caddy → dem portal-hub
(`/opt/<app-a>/portal/index.html`). dort liegt ein **„in vorbereitung"-platzhalter**
für die nächste app. **trage deine app dort ein** — sonst kommt man nur über
`http://<SERVER-IP>:<APP-PORT>` an sie.

> ⚠️ das ist eine **fremde, produktive datei**. **immer zuerst sichern, dann gezielt
> ersetzen**, nie blind überschreiben. (ein agent braucht hierfür die ausdrückliche
> freigabe des nutzers.)

```bash
sudo cp /opt/<app-a>/portal/index.html /opt/<app-a>/portal/index.html.bak-<app>
```

ersetze den platzhalter-kachelblock (`<div class="card soon"> … in Vorbereitung … </div>`)
durch eine echte kachel mit **installer-befehl + zip-link**. die adressen werden
**im browser aus dem host abgeleitet** und zeigen auf den **eigenen port** der app —
so bleibt der geteilte caddy unangetastet:

```html
<!-- App-Kachel (ersetzt den "card soon"-Platzhalter) -->
<div class="card">
  <div class="row">
    <div class="ico">🟢</div>
    <div>
      <p class="name"><App></p>
      <p class="desc">Kurzbeschreibung</p>
    </div>
  </div>
  <p class="lead">So installierst du <App> — <b>ohne Sicherheitswarnung:</b></p>
  <div class="cmd"><code id="appCmd">…</code><button class="copy" onclick="copyApp()">Kopieren</button></div>
  <details><summary>Lieber manuell als ZIP</summary>
    <a class="btn ghost" id="appZip" href="#" download>⬇︎ Mac-App (ZIP)</a></details>
</div>
```

und im vorhandenen `<script>` (nutzt die schon definierte `host`-variable und
`flash()` des portals):

```js
var appBase = "http://" + host + ":<APP-PORT>";
var appCmd  = "curl -fsSL " + appBase + "/downloads/install-client.sh | <APP>_URL=" + appBase + " bash";
document.getElementById("appCmd").textContent = appCmd;
document.getElementById("appZip").href = appBase + "/downloads/<App>-Mac.zip";
function copyApp(){ navigator.clipboard && navigator.clipboard.writeText(appCmd); flash(event.target); }
```

robust per skript (statt editor) — gezielte ersetzungen unveränderlicher
textstellen, nie regex über verschachtelte `<div>`:

```bash
python3 - /opt/<app-a>/portal/index.html <<'PY'
import sys; p=sys.argv[1]; h=open(p,encoding="utf-8").read()
reps=[('class="card soon"','class="card"'),
      ('<p class="name">Weitere App</p>','<p class="name"><App></p>'),
      ('<p class="desc">folgt</p>','<p class="desc">Kurzbeschreibung</p>'),
      ('<span class="tag">in Vorbereitung</span>','<!-- App-Kachel-Inhalt: lead + cmd + details/zip -->'),
      ('document.getElementById("srv").textContent = srv;',
       'document.getElementById("srv").textContent = srv;\n  /* App-JS: appBase/appCmd/appZip/copyApp hier */')]
for a,b in reps: h=h.replace(a,b,1)
open("/tmp/portal-new.html","w",encoding="utf-8").write(h)
PY
sudo cp /tmp/portal-new.html /opt/<app-a>/portal/index.html
# Prüfen + App-A intakt:
curl -s http://127.0.0.1/ | grep -c '<App>'          # >0
curl -s http://127.0.0.1/ | grep -c 'APP_A_HOST'     # noch 1
# Rückgängig:  sudo cp /opt/<app-a>/portal/index.html.bak-<app> /opt/<app-a>/portal/index.html
```

> **zukunft sauberer:** portal aus `/opt/<app-a>` in ein **app-neutrales**
> `/opt/portal` ziehen, das jede app per kachel ergänzt — dann ist es keine fremde
> datei mehr. bis dahin gilt: sichern, gezielt ersetzen, prüfen.

## 8. datenhaltung

- **diese box hat kein postgres und kein docker.** standardannahme: **sqlite**.
  eine datei unter `/opt/<app>/app/data/<app>.db` (oder via prisma
  `provider = "sqlite"`). null zusätzliche dienste, null portkonflikte.
- **wenn doch postgres** nötig ist: `sudo apt install postgresql`, dann **eigene db
  + eigener db-user pro app** (`CREATE USER <app>; CREATE DATABASE <app> OWNER
  <app>;`). niemals die db einer anderen app mitbenutzen.
- secrets (`AUTH_SECRET`, key-encryption etc.) **pro app** in deren `.env`,
  generiert mit `openssl rand -base64 32`. nicht committen.

## 9. deploy-automatik

jede app bringt ein `deploy.sh` mit, das der **self-hosted github-actions-runner**
(oder ein cron/timer) bei jedem push ausführt:

```bash
cd /opt/<app>/app
git fetch --all --prune
git reset --hard origin/main          # Server folgt strikt origin/main
corepack pnpm install --frozen-lockfile
corepack pnpm build
sudo systemctl restart <app>
```

der runner braucht eine `sudo`-regel nur für `systemctl restart <app>`
(`/etc/sudoers.d/<app>`), sonst nichts.

## 10. checkliste für ein neues repo

- [ ] freien port aus der registry (§3) gewählt und dort eingetragen.
- [ ] app bindet an `0.0.0.0:<port>` (`next start -H 0.0.0.0 -p <port>`).
- [ ] `/opt/<app>/` + system-user `<app>` angelegt.
- [ ] `<app>.service` installiert, `enable --now`.
- [ ] datenhaltung gewählt (sqlite-datei in `/opt/<app>/app/data/` bevorzugt).
- [ ] `.env` mit eigenen secrets (nicht committed).
- [ ] download-seite + `/downloads/*` **über den eigenen app-port** ausgeliefert
      (konfliktfrei) **oder** pfad im geteilten caddy ergänzt.
- [ ] `install-client.sh` + `<App>-Mac.zip` + `version.json` in den downloads.
- [ ] **kachel im gemeinsamen portal ergänzt — mit backup (§7a):** installer-befehl
      + zip-link, host-abgeleitet auf den eigenen port. (PFLICHT, sonst nur über
      `:<port>` erreichbar.)
- [ ] `deploy.sh` + actions-runner/timer eingerichtet.
- [ ] geprüft: bestehende apps laufen unverändert weiter (`systemctl status`,
      ports, portal erreichbar).

## 11. schnell-referenz: ist-zustand der box

- host `<SERVER>` · `<SERVER-IP>` · ubuntu 24.04 · node 22 · pnpm 11 · caddy ·
  **kein** postgres/docker · self-hosted actions-runner (`<GITHUB-ORG>`).
- **<APP-A>:** `/opt/<app-a>/app`, port **4577**, `<app-a>.service`, portal +
  downloads unter `/opt/<app-a>/{portal,downloads}`, client = gezippte `.app` via
  `install-client.sh`.
- **<APP-B>:** `/opt/<app-b>/app`, port **4578**, `<app-b>.service`, **sqlite**
  (`apps/server/data/<app-b>.db`). portal + ui + downloads laufen über den eigenen
  port (`/`, `/app`, `/downloads/*`), client = dünne wkwebview-`.app`
  (`<APP-B>-macOS.zip`). kachel im gemeinsamen portal vorhanden (mit backup
  `index.html.bak-<app-b>`).

## 12. erprobtes deploy-rezept (so lief eine app auf die box)

vom mac aus, ohne github-push (rsync). setzt voraus: scoped sudo für den
deploy-user (`/etc/sudoers.d/<app>`: `NOPASSWD: /usr/bin/mkdir, /usr/bin/chown,
/usr/bin/cp, /usr/bin/mv, /usr/bin/tee, /usr/bin/ln, /usr/bin/systemctl`).

```bash
# 1. Zielverzeichnis (sudo)
ssh box 'sudo mkdir -p /opt/<app> && sudo chown -R $USER:$USER /opt/<app>'

# 2. Code rüberspielen (ohne node_modules/.next/.git/target/data/.env)
rsync -az --delete \
  --exclude .git --exclude node_modules --exclude .next --exclude target \
  --exclude 'apps/server/data' --exclude '*.log' --exclude .env \
  ./ box:/opt/<app>/app/

# 3. .env auf der Box (SQLite + Secrets)
ssh box 'cd /opt/<app>/app/apps/server && mkdir -p data && cat > .env <<EOF
DATABASE_URL="file:/opt/<app>/app/apps/server/data/<app>.db"
DOWNLOADS_DIR="/opt/<app>/app/apps/server/downloads"
AUTH_SECRET="$(openssl rand -base64 32)"
KEY_ENCRYPTION_SECRET="$(openssl rand -base64 32)"
EOF'

# 4. Install, DB, Build (auf der Box)
ssh box 'cd /opt/<app>/app && export COREPACK_ENABLE_DOWNLOAD_PROMPT=0 \
  && corepack pnpm install --frozen-lockfile \
  && corepack pnpm --filter @<scope>/server exec prisma generate \
  && corepack pnpm --filter @<scope>/server exec prisma migrate deploy \
  && corepack pnpm --filter @<scope>/server db:seed \
  && corepack pnpm --filter @<scope>/server build'

# 5. Dienst (sudo)
ssh box 'sudo cp /opt/<app>/app/deploy/systemd/<app>.service /etc/systemd/system/ \
  && sudo systemctl daemon-reload && sudo systemctl enable --now <app>'

# 6. Mac-Client bauen + ablegen (lokal, dann rsync) + Portal-Kachel (§7a)
bash client-mac/build.sh                       # -> apps/server/downloads/<App>-macOS.zip
rsync -az apps/server/downloads/ box:/opt/<app>/app/apps/server/downloads/

# 7. Prüfen
ssh box 'curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:<port>/ \
  && systemctl is-active <app> && ss -tlnH | grep :<port>'
```

> **mac-client ohne rust:** ein tauri-client liess sich wegen einer aktuellen
> `tauri-utils`/`time`-inkompatibilität nicht bauen. bewährt hat sich die dünne
> **wkwebview-`.app`** (swift, `swiftc`, universal arm64+x86_64) — sie lädt nur
> `http://<SERVER-IP>:<port>/app`, die ui kommt vom server. siehe `client-mac/`.

## stolperfallen
- **portal-datei ist fremd & produktiv.** vor jeder änderung sichern (`.bak-<app>`),
  gezielt ersetzen, danach prüfen dass die bestehenden apps + portal intakt sind.
  nie blind überschreiben, nie regex über verschachtelte `<div>`.
- **kachel vergessen = app unsichtbar.** ohne portal-kachel ist die app nur unter
  `http://<SERVER-IP>:<APP-PORT>` erreichbar, nicht von der nackten ip.
- **port doppelt vergeben.** immer erst freien port prüfen (`ss -tlnH`) und in die
  registry eintragen, sonst kollision mit einer laufenden app.
- **build auf shared box.** kein docker/postgres per default — sqlite ist die
  standardannahme. erst eine andere db einführen, wenn wirklich nötig (eigene db +
  user pro app).
- **tauri-client baut nicht** (tauri-utils/time): auf die dünne wkwebview-`.app`
  ausweichen.

## referenzen
- aufbau dieser anleitung: `TEMPLATE.md`
- verwandt: `guides/hosttech-deployment.md` (externes hosting statt eigener box)
