# Kontext für eine KI: Starship-Lieferroboter in CARLA (Stand 28.10.2026)

> **An die KI, die das liest:** Dieser Ordner fasst ein langes Gespräch zwischen dem Nutzer und einer KI
> (Claude Code) zusammen. Lies zuerst diese Datei, dann `01_Erklaerungen_aus_dem_Chat.md`. Die übrigen Ordner sind
> Nachschlagematerial. Antworte auf **Deutsch** und erkläre einfach: Der Nutzer ist Einsteiger in Unreal Engine,
> CARLA, Blender und Git.

## 1. Worum es geht

F&E-Projektarbeit an der **Hochschule Karlsruhe (HKA), Institut für Energieeffiziente Mobilität (IEEM)**, SS 2026:
„Implementierung von Last-Mile-Delivery-Robotern in CARLA“ (CARLA = Open-Source-Fahrsimulator auf Basis von Unreal Engine).

- **Autoren:** Anvin Lee (der Nutzer, zuständig für den Starship) und Ricardo Rauch (Clevon; hat außerdem den
  Starship nach CARLA 0.10 / Unreal Engine 5 portiert).
- **Betreuung:** Prof. Dr.-Ing. Reiner Kriesten; Koordinator Prof. Dr. rer. nat. Andreas Helfrich-Schkarbanenko;
  Betreuer am Institut Sebastian Gahm M.Sc.
- **Abgabe des Berichts:** 28.10.2026. Die Abgabefassung ist **Bericht V3-7** (`02_Bericht/`).

Zwei Roboter wurden als fahrbare CARLA-Fahrzeuge mit Sensorik und ROS-2-Anbindung umgesetzt:

| Fahrzeug | CARLA-ID | Kurz |
|---|---|---|
| **Starship-Gehwegroboter** im eigenen **HKA-Design** (anthrazit, rotes „HKA“ seitlich; aus Urheberrechtsgründen nicht im Originaldesign) | `vehicle.starship.bot` | 6 Räder, alle angetrieben, 35 kg, ca. 6 km/h, 26 Sensoren, Bordstein-Assistent |
| **Clevon 1** mit CargoBox | `vehicle.clevon.1` | Straßenroboter, 4 Räder, 450 kg, 6-kW-Ersatzantrieb, Licht, Ladeklappe |

## 2. Zwei Entwicklungsstände

| | **UE4-Stand** (Anvin, Windows) | **UE5-Stand** (Ricardo, Linux), **maßgeblich für den Bericht** |
|---|---|---|
| Software | CARLA 0.9.16 (Source-Build), Unreal Engine 4.26.2 | CARLA 0.10.0, Unreal Engine 5.5.4, ROS 2 Jazzy, Ubuntu 24.04 |
| Physik | PhysX: nur 4 Räder simuliert, Mittelräder drehen nur optisch mit | Chaos Vehicles: alle 6 Räder simuliert (Basisklasse `BaseVehiclePawnNW`) |
| Antrieb | nur Vorderräder | alle 6 Räder (Allrad: vorne je 25 %, Mitte/hinten je 12,5 %), Motor 3 Nm, Achsübersetzung 4,0, max. 720 U/min |
| Lenkung | Vorderräder bis 30° (wie ein Auto) | gleich. **Keine Panzerlenkung** wie beim Original, weil CARLAs Steuerbefehl nur einen Lenkwert kennt |
| Federung | 2/2 cm, Eigenfrequenz 9 Hz | Einfederweg 18 cm, Ausfederweg 4 cm, Federrate 14,3 N/cm, Dämpfung 0,7 |
| Rad-Boden-Abfrage | – | **Spherecast** (seit 21.10.; vorher Raycast) |
| Bordstein | – | C++-Komponente `UStarshipCurbAssistComponent` |
| ROS 2 | keine | CARLAs eingebautes ROS 2 (`--ros2`) plus eigener Ergänzungsknoten; alle 26 Sensoren unter `/carla/starship/…` |
| Assets | fertige `.uasset` im Repo | keine `.uasset` im Repo; Neuaufbau per Skript (`scripts/unreal_editor/ue5/build_all.sh`) |

Gemeinsam: 35 kg (25 kg leer + 9,1 kg Zuladung), Radradius 9 cm, Schwerpunkt um 15 cm abgesenkt, Höchstgeschwindigkeit
gemessen 6,3 km/h (Original: 6 km/h).

**Sensoren (26):**
- 9 RGB-Kameras: Anordnung nach Starship-Patent US20180253107A1, 4 Bilder/s
- 3 ToF-Sensoren als CARLA-Tiefenkamera: 320 × 240 (Chip ESPROS epc660), 5 m
- 4 Radare: Vorbild TI IWR6843AOP, 60° × 20°, 20 m
- 8 Ultraschall als CARLA-Hindernissensor: 3 m, +4° nach oben geneigt
- GNSS und IMU

Die Anzahl stammt aus MIT Technology Review (2018). Montagepunkte von ToF, Radar und Ultraschall sind nicht
veröffentlicht und begründet angenommen.

**Bordstein-Assistent:** 15-cm-Bordstein hoch 10 von 12, runter 8 von 8. Diese Werte wurden noch mit **Raycast**
gemessen; eine Messung mit Spherecast steht aus. Wichtig zum Verständnis siehe `01_Erklaerungen_aus_dem_Chat.md`:
Der Assistent nutzt **keinen** Sensor des Bots, und die Hubkraft ist eine vereinfachte äußere Kraft.

## 3. Repository

- **Maßgeblich:** GitLab `https://git.ieem-ka.de/rari1014/last_mile-delivery-carla`, Branch **`clevon-starship-carla-0.10`**,
  Commit `0197d69` (23.10.2026). Enthält Bericht V3-7, Video-Tutorial und Startanleitung (große Dateien über Git LFS).
- **GitHub-Spiegel** `https://github.com/rari1014/last_mile-delivery-carla`:
  - Branch `clevon-starship-carla-0.10` ist ein älterer Stand (`2c63fb7`), ohne Bericht und Video.
  - Branch `merge/clevon-starship-ein-repo` ist Anvins eigene Zusammenführung mit `DATEIUEBERSICHT.md` (Sinn und
    Funktion jeder Datei) und Ordner-READMEs; nicht in Ricardos Stand übernommen.
- Ursprüngliches Starship-Repo: `https://github.com/rari1014/starship-carla` (Branch `feat/carla-0.10-ue5` = UE5).
- **Ordner im Repo:**
  - `00_Abgabe/`: Bericht V3-7, Video
  - `01_Clevon1/`
  - `02_starship_emodell/`: Starship, mit `docs/IMPLEMENTIERUNG_CARLA_0.10.md` als Hauptprotokoll
  - `Anleitung_Bots_starten.pdf`
- Lokal beim Nutzer: `C:\Users\letie\Downloads\repos\last_mile-delivery-carla` (GitLab-Stand ausgecheckt).

## 4. Inhalt dieses Ordners

| Datei/Ordner | Inhalt |
|---|---|
| `00_LIES_MICH_ZUERST.md` | diese Datei |
| `01_Erklaerungen_aus_dem_Chat.md` | **die Fragen des Nutzers und die erarbeiteten Erklärungen** (Bordstein-Assistent, Federrate, Bodenechos, Git-Begriffe …) |
| `02_Bericht/` | Abgabebericht V3-7 (PDF und Volltext als `.txt`), Startanleitung (PDF und Text) |
| `03_Repo_Doku/` | wichtigste Dokumente aus dem Repo: Übersichten, Starship-Protokoll, Sensorik, ROS 2, Fahrzeugwerte, Clevon-READMEs |
| `04_Bilder/` | HKA-Modell, Skelett, Sensorbeschriftung, Sensoranordnung, Starship in CARLA 0.10 |
| `05_Kapitel5_Entwurf_Claude/` | Ein Entwurf für Kapitel 5 (LaTeX), bevor bekannt war, dass V3-7 schon ein Starship-Kapitel hat. **Nur Zusatzmaterial**: Belege (Patente, epc660), Sensor-Abbildung. |
| `06_Eigene_Dokumente/` | Anvins frühere Dokumente zum UE4-Stand: Bericht mit Laien-Erklärung aller Probleme und Lösungen, Tutorial für Laien |
| `09_Chatverlauf_komplett_lesbar.md` | ganzer Verlauf (nur Texte), zum gezielten Nachschlagen |

## 5. Wichtige Entscheidungen und Vorgaben des Nutzers

- Starship im **eigenen HKA-Design**, nicht im Originaldesign (Urheberrecht).
- **Lenkung wie ein Auto statt Panzerlenkung**, begründet mit CARLAs Fahrzeugschnittstelle und Einfachheit.
- Im Bericht durften der **Clevon-Teil und die allgemeinen Teile nicht verändert** werden.
- Zwei Skripte (`starship_import_hka.py`, `starship_dump_values.py`) sind für den Start nicht nötig, bleiben aber als
  Nachweis im Repo.
- Ins Repo nur nach ausdrücklicher Zustimmung pushen; der Branch `clevon-starship-carla-0.10` gehört Ricardo.

## 6. Offene Punkte

- Bordsteinmessung mit Spherecast nachholen (am Linux-PC: `scripts/carla/starship_curb_test.py --assist --label spherecast`).
- Im Bericht wird der Bordstein-Assistent als „Strahlabfrage“ beschrieben. Klarstellen würde, dass er **keinen** der
  simulierten Sensoren nutzt (der Nutzer wollte das nur verstehen, keine Änderung).
- Abschnitt 3.1 des Berichts spricht von „Radnabenmotoren“. Das Patent US20180194411A1 nennt mindestens 4 Motoren
  (je Vorderrad einen, je Seite einen für Mittel- und Hinterrad), die in den Rädern sitzen *können*.
- ROS 2: kein Not-Halt bei ausbleibenden Fahrbefehlen; ToF wird als `bgra8` statt in Metern gesendet.
- Frage des Betreuers zu Fußgänger-Trajektorien und N-Rad-Fahrzeugen: in V3-7, Abschnitt 5.5 „Machbarkeit weiterer
  Funktionen“ beantwortet (nicht umgesetzt, nur als machbar begründet).
