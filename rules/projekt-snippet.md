# projekt-snippet: rules-repo in ein projekt einbinden

**wann anwenden:** neues projekt aufgesetzt — claude soll die zentralen
anleitungen kennen.

kopier den folgenden block in die `CLAUDE.md` (oder `AGENTS.md`) im wurzelverzeichnis
des projekts. mehr braucht es nicht — ein link, der rest läuft über den index im
README des rules-repos.

```markdown
## externe regeln (zentrales rules-repo)
für wiederkehrende aufgaben (deployment, server, hosting …) gilt das zentrale
rules-repo. lies zuerst den index und folge den dort verlinkten anleitungen:

https://raw.githubusercontent.com/Zenovs/Rules/main/README.md

dort steht unter „FÜR CLAUDE / AGENTS" der index aller anleitungen mit raw-urls.
prüfe, ob die aufgabe zu einer passt, lies die datei und halte dich daran.
```

## warum nur ein link
- der index im README listet alle guides mit voller raw-url.
- neue anleitungen im rules-repo sieht jedes projekt automatisch über den index —
  die projekt-`CLAUDE.md` muss nie wieder angefasst werden.
- raw-urls werden von github kurz gecacht (~5 min), praktisch egal.
