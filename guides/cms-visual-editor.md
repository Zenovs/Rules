# CMS + Visueller Editor für statische Websites

**Keywords:** cms, visual-editor, inline-editor, statische website, multi-page, json, php-api, node-express, partials, data-bind, seed-fallback, cache-busting, htaccess, plesk, mjml, mail-vorlagen

**Wann diese Regel gilt:** Immer wenn eine neue **statische Multi-Page-Website** (HTML/CSS/Vanilla-JS, ohne Build-Step) mit einem **leichtgewichtigen CMS** und einem **visuellen Klick-Editor** aufgebaut werden soll – so wie bei „Taxi Fredy". Diese Regel ist die verbindliche Bauanleitung. Halte dich an Struktur, Verträge und die Abnahme-Checkliste am Ende.

---

## 0. Was am Ende funktionieren muss (Definition of Done)

1. **Statische Site** mit mehreren Seiten, gemeinsame Bausteine (Nav/Footer/Formular/Kontakt) als **Partials**.
2. **CMS unter `/admin`**: Login, Dashboard, Einstellungen, je Inhaltsseite ein schema-getriebener Editor. Alle Texte/Bilder editierbar.
3. **Visueller Editor** auf der echten Website (`?edit`): Klick-to-edit für Texte, Klick auf ein Bild → Hochladen ODER Auswahl aus einer Mediathek, Werkzeugleiste mit **Seiten-Dropdown** und **Speichern**.
4. **Zwei API-Backends**: PHP (Produktion/Plesk) **und** Node/Express (lokal) lesen/schreiben dieselben JSON-Dateien.
5. **Selbstheilung**: fehlt eine Datendatei, greift ein **Seed-Fallback**.
6. **Cache-Busting** von Anfang an (öffentliche **und** Admin-Assets) + `.htaccess`.
7. Optional: **Mail-Vorlagen** (MJML) im CMS editierbar + **Web-Vorschau**.

---

## 1. Ordnerstruktur (verbindlich)

```
/
├─ index.html                     # Startseite
├─ <seite>/index.html             # jede Unterseite als Ordner + index.html (saubere URLs)
├─ partials/                      # gemeinsame Bausteine (kein <head>, nur Fragment)
│   ├─ nav.html  footer.html
│   ├─ kontaktformular.html  kontakt-info.html  services.html …
├─ assets/
│   ├─ css/ base.css components.css  pages/<seite>.css  inline-editor.css
│   └─ js/  cms-loader.js nav.js  <seite>.js  inline-editor.js  form.js …
├─ data/                          # CMS-Inhalte (JSON) – im Build getrackt
│   ├─ site.json home.json …      # je Bereich eine Datei
│   └─ _seed/*.json               # 1:1-Kopie als Fallback/Auslieferungsstand
├─ admin/
│   ├─ index.html                 # Login
│   ├─ dashboard.html
│   ├─ pages/<bereich>.html       # je Datei ein Editor (Admin.page{…})
│   ├─ js/admin.js  css/admin.css
│   └─ server/                    # Node/Express (lokal): server.js, routes/, middleware/
├─ api/index.php                  # Produktions-API (Plesk)
├─ mail-templates/*.mjml          # optional
├─ mail-vorschau/index.html       # optional (E-Mail-Vorschau)
└─ .htaccess                      # Cache-Steuerung
```

Regel: **eine Unterseite = ein Ordner mit `index.html`**. Assets immer als `assets/…` referenzieren (relativ), aufgelöst über ein dynamisches `<base href>` (siehe §9).

---

## 2. Content-Binding – die Verträge

Der Kern: HTML enthält den **Auslieferungstext als Fallback** und ein Attribut, das sagt, aus welcher JSON-Stelle der Wert kommt. `cms-loader.js` hydratisiert nach dem Laden.

| Attribut | Bedeutung | Beispiel |
| --- | --- | --- |
| `data-include="name"` | Partial aus `partials/name.html` einfügen | `<div data-include="nav"></div>` |
| `data-bind="datei:pfad"` | Text/Bild aus `data/datei.json` an `pfad` | `data-bind="home:hero.title"`, `data-bind="services:0.image"` |
| `data-site="pfad"` | globale Werte aus `data/site.json` | `data-site="phone"`, `data-site="bookingUrl"` |
| `data-href="tel\|mailto"` | editierbarer Kontaktlink (sichtbarer Text = Wert) | `<a data-site="phone" data-href="tel">…</a>` |
| `data-render="typ"` | Sammlung rendern (news/faq/team/history/steps/offers…) | `<div data-render="faq" data-service="0"></div>` |

Pfad-Syntax für `setPath/getPath`: Punkt-Notation, Zahlen = Array-Index (`hero.title`, `0.cardTitle`, `history.entries.1.text`).

Für `<img>`: `data-bind` setzt `src`. Für Sammlungen: HTML enthält einen leeren Container mit `data-render` (+ optional `data-variant`), JS baut die Items aus dem JSON.

**Wichtig:** Jeder editierbare Inhalt braucht ein `data-bind`/`data-site`. Hardcodierter Text ist später **nicht** im CMS/Visual-Editor bearbeitbar (häufiger Fehler → siehe Checkliste).

---

## 3. Kern-Skripte & ihre Verträge

### `assets/js/cms-loader.js` (exportiert `window.CMS`)
- `includePartials()` – ersetzt alle `[data-include]` durch `partials/<name>.html`; **`fetch(..., {cache:'no-cache'})`**.
- `fetchData(key)` – `fetch('data/'+key+'.json')`, **bei Fehler Fallback** `fetch('data/_seed/'+key+'.json')`; beides `cache:'no-cache'`.
- `bindSite()` – alle `[data-site]` aus `site.json`.
- `bindContent()` – alle `[data-bind]` aus der jeweiligen Datei.
- `renderDynamic()` – verteilt auf `renderNews/Faq/Team/History/Steps/Offers…` (per `data-render`).
- `applyBinding/setBoundValue` – setzt `textContent` bzw. `img.src`.

### `assets/js/nav.js`
Orchestriert beim `DOMContentLoaded`:
```
await CMS.includePartials(); CMS.setYear(); await CMS.bindSite();
CMS.renderDynamic(); await CMS.bindContent(); CMS.initReveal();
document.dispatchEvent(new Event('partials:loaded'));   // Formulare in Partials aktivieren
// danach: initScrollShadow, initMobileMenu, initLangSwitch, setActiveLink, maybeInitInlineEditor
```
`maybeInitInlineEditor()`: nur wenn `/[?&]edit(=|&|$)/.test(location.search)` → lädt `inline-editor.css/js` (mit `?v=`) und ruft nach `load` `window.InlineEditor.init()`.

### `assets/js/inline-editor.js` (exportiert `window.InlineEditor = { init }`)
- `authed()` → `fetch('/api/auth/check')`; ohne Login Hinweis + Weiterleitung zu `/admin/`.
- Editierbar = `[data-bind], [data-site]`; `<a>` **ohne** `data-href` überspringen (Buttons/URLs nicht als Text editieren).
- Texte → `contenteditable` + Speicherung `on blur`.
- Bilder → Klick öffnet einen Mediathek-Dialog (`.ie-media`): entweder „Bild hochladen" (Upload via `POST /api/upload`) oder ein Raster vorhandener Bilder (`GET /api/media`) zum Auswählen. Auswahl/Upload setzt `img.src` + merkt die Änderung. Schliessen per ✕ / Klick daneben / Esc.
- **Werkzeugleiste** (`.ie-bar`, fixiert unten): Label · **Seiten-Dropdown** (Navigation im Edit-Modus) · Änderungszähler · „Beenden" · „Speichern".
- `preserveEditLinks()` – hängt `?edit` an **interne** Links (gleicher Origin, kein `target=_blank`, keine reinen Anker), damit man im Edit-Modus navigieren kann.
- `saveAll()` – pro geänderte Datei: `load → setPath je Änderung → POST /api/data/:file`.

### `api/index.php` (Produktion) und `admin/server` (lokal) – identische Verträge
- **Whitelist** `ALLOWED_FILES` – **jede** Datendatei muss drinstehen (in **beiden** Backends!).
- `GET /api/data/:file` – **öffentlich**, liefert `data/<file>.json`, **Fallback** auf `data/_seed/<file>.json`.
- `POST /api/data/:file` – **nur mit Auth** (Session), schreibt JSON.
- `POST /api/upload` – nur mit Auth, speichert Bild, gibt Pfad zurück.
- `GET /api/media` – öffentlich, listet rekursiv alle Bilder unter `assets/img/` (Uploads zuerst, dann neueste), Rückgabe `{ ok:true, images:[pfad,…] }`. In beiden Backends. Uploads landen in `assets/img/uploads/`.
- `GET /api/auth/check` – `{authenticated:true|false}`; Login/Logout per Session-Cookie.

---

## 4. Admin-Panel (`admin/js/admin.js`)

- **Schema-getrieben:** jede `admin/pages/<x>.html` ruft `Admin.page({ key, file, type:'object'|'collection', title, subtitle, schema })`.
- **Feldtypen:** `text`, `textarea`, `image`, `group`, `list`, `objlist` (Sammlung mit `labelKey`/`fields`), `json`, `checkbox`. Gruppen dürfen Sammlungen enthalten.
- **Navigation (WICHTIG):** Bei vielen Seiten läuft eine flache Topbar über. Deshalb:
  - **Feste Links:** `Dashboard`, `Einstellungen`.
  - **Alle Inhaltsseiten in ein Dropdown** „Seite bearbeiten" (aktive Seite hervorheben). Selektion → `location.href`.
  - Exportiere trotzdem die **Gesamtliste** `Admin.NAV` (das Dashboard baut daraus seine Kacheln).
- Topbar-Aktionen: „✏️ Visuell bearbeiten" (`/?edit`, neuer Tab), „Website ansehen", „Logout".

---

## 5. Seed-Fallback (Selbstheilung, 3 Ebenen)

`data/_seed/*.json` ist eine 1:1-Kopie des Auslieferungsstands. Fehlt `data/<x>.json` (z. B. nach einem harten Deploy oder wenn Datendateien nicht mitgezogen werden), liefert der Fallback trotzdem Inhalt. **Implementiere den Fallback an allen drei Stellen:** `api/index.php`, `admin/server`, `cms-loader.js`.

---

## 6. Cache-Busting + `.htaccess` (PFLICHT – der wichtigste Stolperstein)

Ohne das erscheinen Änderungen nach dem Deploy **nicht** (Browser liefert alte CSS/JS aus dem Cache).

1. **Versions-Query an ALLE CSS/JS** – öffentlich **und** im Admin:
   `assets/css/components.css?v=<TOKEN>`, `admin/js/admin.js?v=<TOKEN>`.
   Token z. B. `JJJJMMTT` + Buchstabe. **Bei jeder Asset-Änderung Token hochzählen.** Auch die dynamisch von `nav.js` injizierten `inline-editor.*` versionieren.
2. **`.htaccess`** im Root:
```apache
<IfModule mod_headers.c>
  <FilesMatch "\.(html|json|css|js)$">
    Header set Cache-Control "no-cache, must-revalidate"
  </FilesMatch>
</IfModule>
<IfModule mod_expires.c>
  ExpiresActive On
  ExpiresByType text/html "access plus 0 seconds"
  ExpiresByType application/json "access plus 0 seconds"
  ExpiresByType text/css "access plus 0 seconds"
  ExpiresByType application/javascript "access plus 0 seconds"
</IfModule>
```
3. Partials/JSON immer mit `cache:'no-cache'` fetchen (revalidieren).
4. Nach dem ersten Deploy ggf. **einmal** den Browser-Cache leeren / privaten Tab nutzen (bereits gecachte HTML zeigt sonst noch alte Asset-Links).

---

## 7. Mobile-Umsetzung (Desktop nicht verändern)

- **Viewport-Meta** auf jeder Seite: `<meta name="viewport" content="width=device-width, initial-scale=1.0">` (fehlt es, greifen `≤`-Breakpoints nicht).
- Alle Mobile-Anpassungen **ausschließlich** in `@media`-Queries (Desktop bleibt unberührt). Breakpoints konsistent (z. B. Nav `≤992px`, Inhalte `≤768px`).

---

## 8. Mail-Vorlagen (optional, aber empfohlen)

- Vorlagen als **MJML** unter `mail-templates/`. Platzhalter in `[[doppelten_klammern]]` = `name`-Attribute der Formularfelder (`[[vorname]]`, `[[email]]`, `[[nachricht]]`, `[[seite]]` …).
- Ins CMS bringen über `data/mailtemplates.json` (`owner`/`user`: je `subject`, `preview`, `mjml`) + Admin-Editor (Textareas) + Whitelist.
- **Web-Vorschau** `mail-vorschau/index.html`: lädt die JSON, füllt Platzhalter mit Beispieldaten, rendert eine **MJML-Teilmenge** (`mj-section/column/text/image/button/divider/table`) zu HTML – kein externer Compiler nötig. Link „📧 Vorschau" im CMS.

---

## 9. Deployment & Umgebung

- **Plesk „Pull" = `git reset --hard`** → überschreibt getrackte Dateien. `data/*.json` während der Bauphase **tracken** (Seed sichert ab). Vor Go-Live entscheiden, ob Live-Daten entkoppelt werden.
- **`<base href>`** dynamisch setzen (Root vs. GitHub-Pages-Unterpfad):
```html
<script>(function(){var b="/";if(location.hostname.indexOf("github.io")>-1){var s=location.pathname.split("/").filter(Boolean)[0];if(s)b="/"+s+"/";}document.write('<base href="'+b+'">');})();</script>
```
- Git-Commits mit `Co-Authored-By`-Trailer; committen/pushen nur auf Ansage.

---

## 10. Empfohlene Umsetzungs-Reihenfolge

1. Grundgerüst + Partials (Nav/Footer) + `cms-loader.js`/`nav.js`, `data-include` steht.
2. `site.json` + `data-site`-Bindungen (Telefon, Mail, Social, bookingUrl …).
3. Je Seite: HTML mit Fallback-Text **und** `data-bind`; passende `data/<x>.json` + `_seed`.
4. API (PHP **und** Node) inkl. Whitelist + Auth + Upload + Seed-Fallback.
5. Admin-Panel: Login, Dashboard, je Datei ein `Admin.page`-Editor; Navigation = Dashboard + Einstellungen + **Seiten-Dropdown**.
6. Visueller Editor (`inline-editor.js/css`) + `maybeInitInlineEditor` + Toolbar mit Seiten-Nav + `preserveEditLinks`.
7. **Cache-Busting + `.htaccess`** (nicht vergessen!).
8. Optional Mail-Vorlagen + Vorschau.
9. Abnahme-Checkliste durchgehen.

---

## 11. Abnahme-Checkliste (alle Stolpersteine)

- [ ] Jeder editierbare Text/Bild hat `data-bind`/`data-site` (kein hardcodierter Inhalt, der später „nicht editierbar" ist).
- [ ] **Whitelist** enthält jede Datendatei – in **PHP UND Node**.
- [ ] **Seed-Fallback** in allen drei Ebenen (PHP, Node, cms-loader).
- [ ] **Cache-Busting** `?v=` auf allen CSS/JS – öffentlich **und** Admin; Token bei Änderungen erhöhen; `inline-editor.*` mitversioniert.
- [ ] `.htaccess` vorhanden (no-cache für html/json/css/js).
- [ ] Viewport-Meta auf jeder Seite; Mobile-Änderungen nur in `@media` (Desktop unverändert).
- [ ] Admin-Navigation: Dashboard + Einstellungen fix, restliche Seiten im **Dropdown** (keine überlaufende Leiste). `Admin.NAV` bleibt Gesamtliste fürs Dashboard.
- [ ] Visueller Editor: Toolbar mit **Seiten-Dropdown**; interne Links behalten `?edit`; `<a>` ohne `data-href` werden nicht als Text editierbar; externe/`target=_blank`/Anker ausgenommen.
- [ ] Endpoint `GET /api/media` vorhanden (PHP und Node); Klick auf ein Bild im Editor öffnet die Mediathek (Hochladen und Auswahl vorhandener Bilder); neue Uploads erscheinen darin.
- [ ] Partials/JSON laden mit `cache:'no-cache'`.
- [ ] Verifikation im Headless-Chrome: **JS-eingefügte Bilder** rendern in Screenshots kaputt/grau → über **Computed-Styles/DOM** (DevTools-Protokoll) prüfen, nicht nur per Bild.
- [ ] Nach Deploy einmal harter Reload / privater Tab (alte HTML kann noch alte Asset-Links cachen).
- [ ] Neue Datendatei? → JSON + `_seed` + Whitelist (PHP+Node) + Admin-Editor + Menüeintrag, alles zusammen.
