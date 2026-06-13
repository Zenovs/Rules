# rules

zentraler ort für wiederverwendbare regeln und anleitungen. statt prozeduren in
jedem projekt neu zu erklären, liegen sie hier einmal sauber dokumentiert und
werden aus anderen projekten referenziert.

das repo ist **öffentlich** — keine secrets, keys, internen adressen oder
kundendaten. echte werte werden durch platzhalter ersetzt (`<DOMAIN>`, `<KUNDE>`,
`<SERVER>`, `<USER>`).

## aufbau

- `guides/` → schritt-für-schritt anleitungen (nach `TEMPLATE.md` aufgebaut)
- `rules/` → kürzere regeln und konventionen
- `TEMPLATE.md` → vorlage für neue anleitungen

## index

### guides
_noch keine einträge_

### rules
_noch keine einträge_

## wie verweise ich aus anderen projekten darauf

jede datei ist direkt per raw-github-url erreichbar:

```
https://raw.githubusercontent.com/Zenovs/Rules/main/<pfad>
```

beispiel:

```
https://raw.githubusercontent.com/Zenovs/Rules/main/guides/hosttech-deployment.md
```

in einer projekt-`CLAUDE.md` oder `AGENTS.md` referenzierst du das so:

```markdown
## externe regeln
folge den anleitungen aus dem zentralen rules-repo:
- deployment bei hosttech:
  https://raw.githubusercontent.com/Zenovs/Rules/main/guides/hosttech-deployment.md

bei einer aufgabe, die zu einer dieser anleitungen passt, lies sie zuerst per
raw-url und halte dich daran.
```
