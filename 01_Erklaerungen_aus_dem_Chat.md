# Erklärungen aus dem Chat

Diese Fragen hat der Nutzer gestellt, um seinen Bericht zu verstehen. Die Antworten sind geprüft (Quellcode, Repo-Doku,
Patente). Reihenfolge: zuerst Technik des Starship, dann Repo und Git.

---

## Starship-Technik

### Welcher Sensor wird für den Bordstein-Assistenten benutzt?
**Keiner.** Die C++-Komponente `UStarshipCurbAssistComponent` misst direkt in der 3D-Welt von Unreal mit sogenannten
**Line Traces** (`LineTraceSingleByObjectType` gegen statische Objekte):
1. Je Fahrzeugseite prüft ein waagerechter Strahl 5 cm über dem Boden 10–110 cm vor dem Vorderrad, ob dort eine Kante ist.
2. Strahlen von oben nach unten im 1-cm-Raster messen die Stufenhöhe (4–25 cm gelten als Bordstein) und wo die Oberseite eben ist.
3. Ein PD-Regler hebt dann die Räder nacheinander an (höchstens 150 N je Rad).

Die 26 Sensoren (Kameras, ToF, Ultraschall, Radar) laufen völlig getrennt davon und werden vom Assistenten nicht gelesen.
Die Messung ist ideal: kein Rauschen, keine Fehlmessungen. Die Python-Vorstufe nutzte `world.cast_ray()` und `world.ground_projection()`.

### „In echt gibt es solche Strahlen nicht, und das Anheben geht auch nicht magisch“
Richtig, beides ist eine Abkürzung der Simulation:
- **Erkennen:** Ein echter Roboter kennt die Kartengeometrie nicht. Er müsste den Bordstein mit seinen Sensoren
  erkennen, vermutlich mit den nach unten geneigten Kameras oder ToF-Sensoren.
- **Anheben:** Beim Original drehen Motoren Kipphebel, an denen die Räder sitzen (Patent US12005753B2). Das Rad drückt
  gegen den Boden, der Boden drückt zurück. Im Modell wirkt dagegen eine **äußere Kraft ohne Gegenpartner**
  (`AddForceAtLocation`), wie eine unsichtbare Hand.
- **Warum:** Chaos Vehicles kennt Räder nicht als eigene Körper mit Gelenken, nur als Strahl mit Feder am starren
  Fahrzeugkörper. Kipphebel lassen sich deshalb nicht einbauen.
- **Bewertung:** Das ist eine funktionale Nachbildung der Wirkung, kein Nachbau der Mechanik. Mögliche Ausbaustufen
  wären Erkennung über die ToF-Daten, aktive Federverstellung oder ein Mehrkörpermodell. Der Nutzer wollte das nur
  verstehen, nicht ändern.

### Woher kommen die 4 cm, und wofür wird die Federrate berechnet?
- **4 cm = Ausfederweg** (`suspension_max_drop`): So weit kann ein Rad aus der Ruhelage nach unten ausfedern. Das ist
  ein **gewählter** Wert, kein gemessener. Ricardo hat ihn bei der Bordstein-Abstimmung von 2 auf 4 cm erhöht, für
  eine weichere Federung und mehr Weg nach unten.
- **Formel:** k = Radlast / Ausfederweg = (35 kg · 9,81 m/s² / 6) / 4 cm = 57 N / 4 cm ≈ **14,3 N/cm**.
- **Zweck:** Ohne Last hängt ein Rad in Chaos ganz unten, 4 cm unter seiner modellierten Position. Mit Last federt es
  um Radlast ÷ k ein. Eine Vorspannung wertet Chaos in diesem Modus nicht aus. Damit der Bot in Ruhe genau auf der in
  Blender modellierten Höhe steht, muss die Einfederung genau 4 cm sein. Daraus folgt k.
- **Gegenbeispiel:** Die erste Umrechnung aus der UE4-Eigenfrequenz von 9 Hz ergab 186,5 N/cm. Die Feder war zu hart,
  und der Bot stand 1,7 cm zu hoch.
- **Eigenfrequenz** jetzt ca. 2,5 Hz: weicher auf Markierungen und kleinen Unebenheiten.
- **Einheit:** Unreal kennzeichnet das Feld als N/m, Chaos multipliziert den Wert intern aber mit 100. Der eingetragene
  Wert (14,3) entspricht deshalb N/cm.
- **18 cm Einfederweg:** Ein 15-cm-Bordstein drückt das Rad 15 cm hoch, es bleiben also 3 cm Reserve bis zum harten Anschlag.

### Was sind Bodenechos, und warum ist das Radar-Sichtfeld enger?
Bodenechos sind Radar-Treffer auf der Fahrbahn selbst statt auf Hindernissen. Das Radar sitzt nur 35 cm hoch. Bei
großem vertikalem Sichtfeld treffen viele Strahlen schräg auf den Asphalt. Mit 60° × 20° statt 120° (Referenz TI
IWR6843AOP) zeigen weniger Strahlen in den Boden. Die 1500 Strahlen/s verteilen sich auf ein kleineres Feld und
treffen öfter echte Objekte. Nachteil: Lücken an den Ecken, die die schrägen Ultraschallsensoren abdecken.

### Spherecast statt Raycast (Änderung von Ricardo am 21.10.)
Das ist die Abfrage, ob ein Rad den Boden berührt.
- **Raycast:** Ein einzelner senkrechter Strahl unter der Radmitte. Das Rad tauchte damit in die Bordsteinfront ein
  und hakte ein.
- **Spherecast:** Eine Kugelabfrage in Radgröße. Sie erfasst die Kante schon mit der Radkontur, das Rad steigt auf.

Die frühere Rechnung, dass Spherecast an der Kante nach hinten drückt, galt ohne Bordstein-Assistent. Die
Bordstein-Ergebnisse 10/12 stammen noch von Raycast.

### Lenkung
Das Original lenkt wie ein Panzer: feste Radachsen, die Seiten drehen unterschiedlich schnell (Patent US20180194411A1).
Das Modell lenkt wie ein Auto über die Vorderräder (bis 30°, Ackermann). Grund: CARLAs Steuerbefehl kennt nur Gas,
Lenkung und Bremse, und die Fahrwerksmodelle setzen gelenkte Räder voraus. Folge: Der Bot kann nicht auf der Stelle
drehen, der Wenderadius beträgt 2,2–2,4 m.

### Frage des Betreuers (Fußgänger-Trajektorien, N-Räder)
Er wollte, dass untersucht und begründet wird, ob der Bot die Wege der CARLA-Fußgänger nutzen kann; umsetzen war nicht
gefordert.
- CARLA-Fußgänger laufen über ein Navigationsnetz mit eigenem KI-Controller, Fahrzeuge folgen Fahrspuren. Deshalb ist
  eine direkte Übernahme nicht möglich.
- Machbar wäre eine eigene Bahnführung über die Gehweg-Spuren der Karte.
- N-Räder: Chaos kann 6 Räder simulieren, PhysX unter UE4 nur 4.

Beantwortet in Bericht V3-7, Abschnitt 5.5.

---

## Repo und Git

### Was machen `starship_import_hka.py` und `starship_dump_values.py`? Kann man sie löschen?
- `starship_import_hka.py` (UE4): Importierte das fertige Blender-Modell `starship_hka_v5.fbx` in Unreal 4. Das
  HKA-Design selbst wurde in **Blender** modelliert, nicht mit diesem Skript.
- `starship_dump_values.py` (UE4): Liest alle Einstellungen des UE4-Fahrzeugs aus und speichert sie in
  `config/starship_values.json`. Es ändert nichts.
- Der Ordner `ue5/` enthält die Skripte, mit denen der Bot in UE5 Schritt für Schritt aufgebaut wird (`s01` … `s08`, `build_all.sh`).

Zum Starten werden beide nicht gebraucht. Löschen würde nichts kaputt machen, kein anderes Skript verweist darauf.
Empfehlung: drin lassen, als Nachweis für die Reproduzierbarkeit. Außerdem gehört der Branch Ricardo.

### Was ist `starship_values.json`?
Eine „Sicherungskopie der Einstellungen“ des fertigen UE4-Bots:
- Masse, Motor, Getriebe
- Räder, Federung, Bremsen
- Kollisionsbox
- Knochenpositionen
- Materialien

Ricardo hat daraus den Bot in UE5 neu aufgebaut, denn UE4-Dateien lassen sich in UE5 nicht öffnen. Die UE5-Werte
weichen teils bewusst ab (6 angetriebene Räder, Federung, Spherecast). Aktuelle Werte stehen in
`docs/IMPLEMENTIERUNG_CARLA_0.10.md`.

### Was ist `.gitattributes`?
Eine Textdatei, die Git sagt, wie es Dateitypen behandeln soll:
1. **Git LFS** für große Dateien (`.uasset`, `.blend`, `.fbx`, Bilder, PDF, Video): Im Repo liegt nur ein Platzhalter,
   die echte Datei liegt im LFS-Speicher. `git lfs pull` lädt sie. Die Abgabe-Dateien liegen nur im LFS-Speicher von
   GitLab, nicht auf GitHub.
2. **`binary`** und **`eol=lf`**: Binärdateien nicht als Text behandeln. Python- und Shell-Skripte bekommen immer
   Linux-Zeilenenden, sonst laufen `.sh`-Skripte auf Ricardos Linux-PC nicht.

### „Erstelle ein neues Verzeichnis“ und `.gitkeep`
- „Erstelle ein neues Verzeichnis“ ist die automatische Commit-Nachricht der Weboberfläche beim Anlegen eines Ordners.
  Es ist kein Prompt.
- `.gitkeep` ist eine leere Platzhalterdatei, weil Git leere Ordner nicht speichert.
- Auf `main` ist `02_starship_emodell` nur dieser Platzhalter; der Inhalt liegt auf den anderen Branches.

### Was hat die zusammengeführte Fassung gebracht?
Anvin wollte beide Bots in einem Repo und zu jeder Datei Sinn und Funktion. Claude hat das auf GitHub-Branch
`merge/clevon-starship-ein-repo` gemacht (`DATEIUEBERSICHT.md`, Ordner-READMEs). Parallel hat Ricardo die Bots selbst
zusammengeführt (GitLab, Branch `clevon-starship-carla-0.10`). Sein Stand ist maßgeblich, enthält aber die Dateiübersicht nicht.
