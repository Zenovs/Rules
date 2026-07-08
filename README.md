# rules

zentraler ort für wiederverwendbare regeln und anleitungen. statt prozeduren in
jedem projekt neu zu erklären, liegen sie hier einmal sauber dokumentiert und
werden aus anderen projekten referenziert.

das repo ist **öffentlich** — keine secrets, keys, internen adressen oder
kundendaten. echte werte werden durch platzhalter ersetzt (`<DOMAIN>`, `<KUNDE>`,
`<SERVER>`, `<USER>`).

---

## FÜR CLAUDE / AGENTS — bitte zuerst lesen

du bist auf dieses repo verwiesen worden. es enthält die verbindlichen
anleitungen für wiederkehrende aufgaben. **vorgehen:**

1. lies die index-liste unten.
2. prüfe, ob die aktuelle aufgabe zu einer anleitung passt (titel + stichworte).
3. falls ja: hol die passende(n) datei(en) per raw-url (WebFetch) und halte dich
   daran, bevor du anfängst.
4. im zweifel: lies die anleitung, die am ehesten passt — lieber einmal zu viel.

### guides (vollständige raw-urls)
- **website bei hosttech hosten & deployen** — domain, plesk-subscription, git-pull, ssl
  https://raw.githubusercontent.com/Zenovs/Rules/main/guides/hosttech-deployment.md
- **interne app auf dem gemeinsamen server anlegen** — port, systemd, caddy-portal, mac-client, deploy
  https://raw.githubusercontent.com/Zenovs/Rules/main/guides/interne-app-auf-shared-server.md

### rules (vollständige raw-urls)
- **projekt-snippet: rules-repo in ein projekt einbinden** — kopiervorlage für die projekt-CLAUDE.md
  https://raw.githubusercontent.com/Zenovs/Rules/main/rules/projekt-snippet.md
- **CMS-inhalte vom git-deploy trennen** — data-dateien nicht ins repo, sonst überschreibt der pull die online-inhalte
  https://raw.githubusercontent.com/Zenovs/Rules/main/rules/cms-inhalte-vom-deploy-trennen.md

---

## aufbau

- `guides/` → schritt-für-schritt anleitungen (nach `TEMPLATE.md` aufgebaut)
- `rules/` → kürzere regeln und konventionen
- `TEMPLATE.md` → vorlage für neue anleitungen

## index (lesbare links)

### guides
- [website bei hosttech hosten & deployen](guides/hosttech-deployment.md) — domain, plesk-subscription, git-pull, ssl
- [interne app auf dem gemeinsamen server anlegen](guides/interne-app-auf-shared-server.md) — port, systemd, caddy-portal, mac-client, deploy

### rules
- [projekt-snippet: rules-repo in ein projekt einbinden](rules/projekt-snippet.md) — kopiervorlage für die projekt-CLAUDE.md
- [CMS-inhalte vom git-deploy trennen](rules/cms-inhalte-vom-deploy-trennen.md) — data-dateien nicht ins repo, sonst überschreibt der pull die online-inhalte

## wie verweise ich aus anderen projekten darauf

ziel: du musst claude in einem neuen projekt **nur einmal auf dieses repo
verweisen** — den rest findet er über den index oben selbst.

füg diesen block in die `CLAUDE.md` (oder `AGENTS.md`) des projekts ein. fertige
kopiervorlage: [rules/projekt-snippet.md](rules/projekt-snippet.md).

```markdown
## externe regeln (zentrales rules-repo)
für wiederkehrende aufgaben (deployment, server, hosting …) gilt das zentrale
rules-repo. lies zuerst den index und folge den dort verlinkten anleitungen:

https://raw.githubusercontent.com/Zenovs/Rules/main/README.md

dort steht unter „FÜR CLAUDE / AGENTS" der index aller anleitungen mit raw-urls.
prüfe, ob die aufgabe zu einer passt, lies die datei und halte dich daran.
```

das ist alles — ein link. neue anleitungen, die ich hier hinzufüge, sieht jedes
projekt automatisch über den index, ohne dass du die projekt-`CLAUDE.md` änderst.
