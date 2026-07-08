# CMS-inhalte vom git-deploy trennen (data-dateien nicht ins repo)

**wann anwenden:** eine website/app hat ein CMS, mit dem der kunde inhalte
**online** editiert, die als dateien auf dem server liegen (z.b. `data/*.json`),
und das projekt wird über git deployt (plesk-„pull", `git pull` o.ä.). **von
anfang an so aufsetzen** — nicht erst reparieren, wenn inhalte weg sind.

## das problem (einmal passiert, deshalb diese regel)
- der kunde editiert online → das schreibt in `data/*.json` **auf dem server**.
- liegen diese dateien auch im git-repo, setzt ein **pull** den server-stand hart
  auf den repo-stand.
- folge: **alle seit dem letzten repo-commit online gemachten CMS-änderungen
  werden überschrieben** — bei *jedem* pull, egal ob der push die `data/`-dateien
  berührt hat oder nicht.
- nicht der push zerstört die inhalte, sondern der pull danach.

## die regel
inhalte, die zur **laufzeit auf dem server** entstehen (CMS-daten, uploads,
user-generierte dateien), gehören **nicht** ins git-tracking. das repo enthält
den **code**, der server hält die **inhalte**. so kann ein code-deploy die
inhalte prinzipiell nie anfassen.

## einrichtung (option B — die saubere lösung)
1. **daten-verzeichnis in `.gitignore`** aufnehmen, z.b.:
   ```gitignore
   data/*.json
   # oder ganzes verzeichnis:
   uploads/
   ```
2. **aus dem tracking nehmen** (falls schon eingecheckt), ohne die dateien lokal
   zu löschen:
   ```bash
   git rm --cached data/*.json
   git commit -m "CMS-daten aus git-tracking nehmen (inhalte vom deploy trennen)"
   ```
3. **einmalig sicherstellen, dass die datenbasis auf dem server existiert.** ab
   jetzt lebt sie nur noch dort — ein pull fasst sie nie mehr an, der kunde kann
   jederzeit editieren, code-pushes überschreiben nichts.

## stolperfallen
- **trade-off: das repo hat keine kopie der inhalte mehr.** ein frisch
  aufgesetzter server braucht einmal **seed-daten** (leere oder beispiel-`data/`).
  seed-dateien mit anderem namen versionieren (z.b. `data.seed/` oder
  `data/*.example.json`), damit sie nicht mit den echten laufzeit-daten kollidieren.
- **backup nicht vergessen.** ohne repo-kopie ist der server die einzige quelle
  der inhalte → server-backup muss die `data/`-/`uploads/`-verzeichnisse abdecken.
- **option A ist die fehleranfällige alternative:** `data/*.json` im repo lassen
  und vor *jedem* pull die online geänderten dateien vom server ins repo
  zurückholen und committen. funktioniert, aber ein vergessener schritt = daten
  weg. deshalb: option B.

## referenzen
- [website bei hosttech hosten & deployen](../guides/hosttech-deployment.md) —
  dort passiert der pull, der bei falschem setup die inhalte überschreibt
- siehe `TEMPLATE.md` für den aufbau dieser anleitung
