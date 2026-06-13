# website bei hosttech hosten & deployen (plesk + git)

**wann anwenden:** eine (statische oder php-) website soll bei hosttech laufen, der
code liegt auf github und wird über plesk deployt.

## voraussetzungen
- hosttech-account mit zugang zu plesk
- domain bei hosttech gekauft
- code in einem github-repo
- aktuell für statische seiten / php — **kein** build-schritt auf dem server (siehe stolperfallen)

## schritte
1. **domain als subscription anlegen**
   in plesk eine neue subscription für die domain erstellen. die domain selbst
   ist bei hosttech gekauft, in plesk wird sie über die subscription verwaltet.
2. **git mit der domain verbinden**
   in der subscription unter "Git" das github-repo verbinden (repo-url + branch).
   git pullt direkt in den webspace-root der domain.
3. **deployen per pull**
   bei jedem update im plesk-git-panel auf **"Pull"** klicken. damit wird der
   aktuelle stand vom repo in den webspace geholt. der pull ist manuell — es läuft
   keine automatische deploy-action.
4. **ssl einrichten**
   unter der domain → "SSL/TLS Certificates" ein kostenloses let's-encrypt-zertifikat
   ausstellen. das ist ein manueller schritt direkt in plesk nach dem domain-anlegen.

## stolperfallen
- **code mit build-schritt geht nicht direkt.** es gab schon ein projekt, das
  hosttech so nicht deployen konnte. shared hosting führt keinen build aus
  (kein `npm run build` o.ä.). wenn ein projekt einen build braucht, muss das
  fertige ergebnis lokal gebaut und ins repo committed werden — dann pullt plesk
  nur die fertigen dateien. reine statische/php-seiten sind unproblematisch.
- **pull nicht vergessen.** ein push auf github deployt nichts von selbst — der
  "Pull"-button in plesk muss manuell geklickt werden.

## referenzen
- siehe `TEMPLATE.md` für den aufbau dieser anleitung
