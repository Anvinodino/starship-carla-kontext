# Implementierung des Starship-Bots in CARLA 0.10 (UE 5.5): Protokoll

Dieses Dokument hält jeden Schritt der Portierung von `vehicle.starship.bot` aus CARLA 0.9.16 (UE 4.26, PhysX)
nach CARLA 0.10 (UE 5.5.4, Chaos) fest: was gemacht wurde, warum, und wie man es prüft.
Begonnen am 20.10.2026, Branch `feat/carla-0.10-ue5`.

## 0. Ausgangslage

| Punkt | Befund |
| --- | --- |
| CARLA-Quellen | `~/CarlaUE5` (Commit `1360bb9`), Projekt `~/CarlaUE5/Unreal/CarlaUnreal/CarlaUnreal.uproject` |
| Engine | `~/UnrealEngine5_carla`, UE 5.5.4 (CARLA-Fork) |
| Hardware | RTX 4070 Ti SUPER 16 GB VRAM, 30 GB RAM |
| Vorhandener Bot | Nur UE4-Assets (`unreal/ue4.26_vehicle3/`). UE4-`.uasset` lassen sich in UE5 nicht laden. Daher Neuaufbau aus `models/starship_hka_v5.fbx` und Werten aus `config/starship_values.json`. |
| Vorbild im UE5-Projekt | `Clevon1`: BP unter `/Game/Carla/Blueprints/Vehicles/Clevon1/`, Mesh/AnimBP unter `/Game/Carla/Static/Car/4Wheeled/Clevon1_CM/`, Parent `BaseVehiclePawn` |
| Mehr als 4 Räder | CARLA 0.10 hat das Blueprint `BaseVehiclePawnNW` (C++ `ACarlaWheeledVehicleNW`). Genutzt von `EuropeanHGV` und `CarlaCola`. Die C++-Klasse `UWheeledVehicleMovementComponentNW` gibt es auch, die Blueprints nutzen aber den normalen Chaos-Movement (siehe unten). |
| Registrierung | Nicht mehr über die VehicleFactory, sondern über `Content/Carla/Config/VehicleParameters.json` (Make/Model/Class/NumberOfWheels …). Clevon steht dort schon drin. |

## 1. Fernsteuerung des laufenden Editors

Damit jeder Schritt als Skript wiederholbar ist, laufen alle Editor-Aktionen als Python im laufenden Editor.

1. Im Editor: *Edit → Project Settings → Plugins → Python → Enable Remote Execution* (landet als
   `bRemoteExecution=True` in `Config/DefaultEngine.ini`).
2. **Linux-Stolperfalle:** Der Editor bindet seinen Multicast-Socket an `127.0.0.1:6766`. Der mitgelieferte
   Client (`remote_execution.py` aus dem PythonScriptPlugin) findet ihn deshalb nicht (`NO_EDITOR_FOUND`):
   Pakete an die Gruppe `239.0.0.1` kommen bei einem auf `127.0.0.1` gebundenen Socket nicht an.
   Lösung in [`scripts/unreal_editor/ue5/ue_remote.py`](../scripts/unreal_editor/ue5/ue_remote.py):
   Client-Socket auf `0.0.0.0` binden und jeden Ping zusätzlich per Unicast an `127.0.0.1:6766` schicken.
3. Aufruf: `python3 scripts/unreal_editor/ue5/ue_remote.py <skript.py>`. Das Skript läuft im Editor,
   dessen `print`-Ausgaben kommen zurück.

Prüfung: Ohne Argument gibt der Aufruf `hello from editor` und `SUCCESS` aus.

### Erkenntnisse aus den Referenzfahrzeugen (vor dem Bau ausgelesen)

| Frage | Ergebnis | Folge |
| --- | --- | --- |
| Welche Movement-Komponente nutzen 4- und 6-Rad-Fahrzeuge? | Beide `ChaosWheeledVehicleMovementComponent` (EuropeanHGV hat 6 Wheel Setups, Clevon 4) | 6 Räder gehen ohne C++-Änderung. Parent wird `BaseVehiclePawnNW` wie beim HGV. |
| Einheit von `SpringRate` | Tooltip sagt N/m, aber `ChaosVehicleWheel.h` rechnet `MToCm(SpringRate)`, also intern ×100 | Eingabewert ≈ Federsteifigkeit in **N/cm**. |
| Wozu dient `NumberOfWheels` in `VehicleParameters.json`? | Nur als Blueprint-Attribut `number_of_wheels` (`ActorBlueprintFunctionLibrary.cpp`) | Auf 6 setzen, ohne Einfluss auf die Physik. |

## 2. Schritt 1: Mesh importieren ([`s01_import_mesh.py`](../scripts/unreal_editor/ue5/s01_import_mesh.py))

- Quelle `models/starship_hka_v5.fbx` (cm, Wurzelknochen `VehicleBase`), Ziel `/Game/Carla/Static/Car/Starship_CM/`
  (gleiche Ablage wie Clevon1).
- Import als Skeletal Mesh, **neues** Skelett (das UE4-Skelett ist nicht übertragbar), Uniform Scale 1,
  ohne Animationen und Morph Targets, Materialien aus der FBX.
- Ergebnis: `SK_Starship`, `SK_Starship_Skeleton`, Materialien `dark_gray`, `black`, `hka_body`, `hka_red`, `hka_lid`.
- **Prüfung:** Bounds halbe Ausdehnung 28,24 × 27,71 × 32,85 cm, Mitte z = 32,85. Das ist identisch mit dem UE4-Stand
  in `config/starship_values.json`, also stimmt die Skalierung (kein Faktor-100-Fehler).
- Hinweis: Trotz `create_physics_asset = True` legt der Import in UE 5.5 kein Physics Asset an. Das übernimmt Schritt 2.

## 3. Schritt 2: Physics Asset ([`s02_physics_asset.py`](../scripts/unreal_editor/ue5/s02_physics_asset.py))

- `SkeletalMeshEditorSubsystem.create_physics_asset(mesh)` erzeugt `SK_Starship_PhysicsAsset` und weist es dem Mesh zu.
- Die Radknochen sind kleiner als die Mindestgröße für Bodies und werden dem Elternknochen zugeschlagen.
  Es entsteht also **genau ein Body auf `VehicleBase`**, das Löschen von Rad-Bodies wie in UE4 entfällt.
  Das Skript bricht ab, falls doch mehr Bodies entstehen.
- Python kommt an `SkeletalBodySetups` nicht heran. Der Body wird deshalb als Subobjekt
  `…SK_Starship_PhysicsAsset:SkeletalBodySetup_0` per `unreal.find_object` geholt.
- Geometrie ersetzt durch eine Box **55 × 45 × 33 cm, Mitte (0, 0, 23,45) cm** (Werte aus dem UE4-Stand):
  Unterkante 16 cm, Oberkante 49 cm über Boden, die Räder (Radius 9 cm) tragen das Fahrzeug.
- **Prüfung:** Ausgabe `nachher: VehicleBase [(55.0, 45.0, 33.0, z: 23.45)] sphyl 0 convex 0`.
- Hinweis: Beim Speichern meldet der Editor „Unable to Check Out From Revision Control“. Die Revisionskontrolle ist
  im Editor deaktiviert. Der unattended-Modus der Remote Execution bestätigt den Dialog selbst, die Dateien werden
  trotzdem gespeichert (Log: `LogSavePackage: Moving … .uasset`).

## 4. Schritt 3: Animations-Blueprint ([`s03_anim_bp.py`](../scripts/unreal_editor/ue5/s03_anim_bp.py))

- `AnimBP_EuropeanHGV` (6 Räder, Chaos-Wheel-Controller) nach `/Game/Carla/Static/Car/Starship_CM/AnimBP_Starship`
  dupliziert und `target_skeleton` auf `SK_Starship_Skeleton` umgestellt, danach kompiliert.
- Warum das reicht: Der Wheel Controller bekommt Radknochen, Drehung, Lenkwinkel und Federweg zur Laufzeit aus den
  Wheel Setups der Movement-Komponente und ist nicht an die HGV-Knochennamen gebunden.
- Der UE4-Trick mit 2 × `Copy Bone` (Mittelräder drehen nur optisch mit) entfällt, weil alle 6 Räder simuliert werden.
- **Prüfung:** Kompiliert ohne Fehler, Klasse `AnimBP_Starship_C` existiert.

## 5. Schritt 4: Rad-Blueprints ([`s04_wheels.py`](../scripts/unreal_editor/ue5/s04_wheels.py))

Sechs Blueprints `BP_Starship_{FLW,FRW,MLW,MRW,RLW,RRW}` unter `/Game/Carla/Blueprints/Vehicles/Starship/`,
Parent `ChaosVehicleWheel` (ersetzt `VehicleWheel` aus PhysX).

| Wert | Vorne | Mitte | Hinten | Herkunft |
| --- | --- | --- | --- | --- |
| Radius / Breite / Masse | 9 cm / 4 cm / 1 kg | gleich | gleich | UE4 |
| Achse (`axle_type`) | FRONT | REAR | REAR | neu |
| Angetrieben | ja | ja | ja | **neu**: 6 Radnabenmotoren wie beim echten Starship (UE4: nur Vorderräder) |
| Lenkwinkel | 30° | 0° | 0° | UE4 |
| Handbremse | nein | ja, 40 Nm | ja, 40 Nm | UE4 |
| Bremsmoment | 20 Nm | 20 Nm | 20 Nm | UE4 |
| Federweg hoch/runter | 2 / 2 cm | gleich | gleich | UE4 |
| Dämpfungsverhältnis | 0,8 | gleich | gleich | UE4 |
| `spring_rate` | 186,5 | gleich | gleich | umgerechnet, siehe unten |
| `spring_preload` | 20 | gleich | gleich | neu, kleiner als die Radlast von 57 N |
| Reibung, Schräglaufsteifigkeit | Chaos-Standard (2,0 / 1000) | gleich | gleich | zunächst unverändert, Feinabstimmung im Fahrtest |

**Umrechnung Federung:** UE4 nutzte Eigenfrequenz 9 Hz. Chaos will eine Federrate.
k = m · (2π f)² = 5,83 kg · (2π · 9 Hz)² = 18 650 N/m mit m = 35 kg / 6 Räder.
Weil Chaos intern `MToCm(SpringRate)` rechnet, wird **186,5** (N/cm) eingetragen.
Statische Einfederung: 57 N / 18 650 N/m ≈ 3 mm.

- **Stolperfalle (erst im Fahrtest aufgefallen, siehe Abschnitt 8):** Nach dem Setzen der Werte muss das Blueprint
  **noch einmal kompiliert** werden. Eine Blueprint-Klasse kopiert beim Erzeugen einer Instanz nur die Eigenschaften,
  die beim letzten Kompilieren vom Parent abwichen (`CustomPropertyListForPostConstruction`). Wurde nur vor dem
  Setzen kompiliert, zeigt das CDO zwar r = 9, jede neue Instanz hat aber wieder die Chaos-Standardwerte (r = 32 cm,
  Lenkwinkel 50°, nicht angetrieben). Das Skript erzeugt deshalb nach dem Speichern eine Test-Instanz per
  `unreal.new_object` und bricht ab, wenn deren Radius nicht stimmt.
- **Prüfung:** Die Ausgabe listet alle 6 Räder mit r = 9, b = 4, m = 1, Lenkung nur vorne, Handbremse nur Mitte und hinten,
  ohne Abbruch durch die Instanz-Prüfung.

## 6. Schritt 5: Drehmomentkurve und Fahrzeug-Blueprint ([`s05_vehicle_bp.py`](../scripts/unreal_editor/ue5/s05_vehicle_bp.py))

### Drehmomentkurve

- **Stolperfalle:** Mit leerer Drehmomentkurve schaltet Chaos die Mechanik-Simulation ab
  (`ChaosWheeledVehicleMovementComponent.cpp`: „has no torque curve defined, disabling mechanical simulation“).
  Der Bot würde dann nicht fahren.
- Die Kurvenpunkte lassen sich in Python nicht schreiben. Deshalb wird
  [`config/ue5/Curve_Starship_Torque.csv`](../config/ue5/Curve_Starship_Torque.csv) als `CurveFloat` importiert
  (`CSVImportFactory`, Typ `ECSV_CURVE_FLOAT`) und als `external_curve` eingehängt, so wie bei Clevon1.
- **Stolperfalle 2:** Die CSV darf **keine Kopfzeile** haben. UE liest jede Zeile als Punkt `x,y`, die Kopfzeile wurde
  zum Punkt (0, 0) und machte aus der flachen Kurve eine Rampe. Prüfung über `CurveFloat.get_float_value`:
  bei 0, 360 und 720 U/min jeweils 1,0.

### BP_Starship

`/Game/Carla/Blueprints/Vehicles/Starship/BP_Starship`, Parent `BaseVehiclePawnNW` (wie EuropeanHGV).

| Bereich | Wert | Bemerkung |
| --- | --- | --- |
| Mesh / AnimBP | `SK_Starship` / `AnimBP_Starship_C` | |
| Simulate Physics (Body Instance) | **an** | `BaseVehiclePawnNW` liefert *aus*, Clevon1 und HGV schalten es im eigenen BP ein. Ohne das bleibt der Bot eingefroren an der Spawnposition (Abschnitt 8). |
| Material-Overrides | Slot 2 `hka_body` → `dark_gray` | wie UE4, Korpus dunkelgrau |
| COM Nudge (Body Instance) | 0 / 0 / −15 cm | wie UE4 |
| Linear Damping | 0,01 | wie UE4 |
| Masse, Chassis B/H, Luftwiderstand | 35 kg, 50/50 cm, 0,3 | wie UE4 |
| Downforce | 0 | Chaos-Standard 0,3 ist für Autos gedacht |
| Wheel Setups | FLW, FRW, MLW, MRW, RLW, RRW → `Wheel_{Front,Middle,Rear}_{Left,Right}` | Index 0 muss ein lenkendes Rad sein: CARLA liest daraus den maximalen Lenkwinkel |
| Max RPM | 720 | wie UE4 |
| Max Torque | 3 Nm | neu, siehe Antriebsauslegung |
| Getriebe | 1 Gang, Übersetzung 1,0, Final 4,0, Rückwärts 1,0, Automatik | neu |
| Differential | `ALL_WHEEL_DRIVE`, `front_rear_split` 0,5 | Ursprünglich `UNDEFINED` (gleichmäßig 1/6 je Rad). **Am 20.10.2026 im Editor auf All Wheel Drive umgestellt**, siehe Abschnitt 10. Verteilung nach `axle_type` (`TransmissionUtility.cpp`): die 2 FRONT-Räder je 25 %, die 4 REAR-Räder (Mitte und hinten) je 12,5 %. |
| Lenkung | Ackermann, Standard-Lenkkurve | |

**Antriebsauslegung (Startwerte, im Fahrtest geprüft):**

- Höchstgeschwindigkeit: v = 720 U/min / 4 / 60 · 2π · 0,09 m = 1,70 m/s ≈ **6,1 km/h**.
- Zugkraft: 3 Nm · 4 · 0,9 / 0,09 m = 120 N. Bei 35 kg sind das rechnerisch bis 3,4 m/s².
  Das reicht für Bordsteinkanten und Steigungen, ohne dass die Räder ständig durchdrehen.

- **Prüfung:** Die Kontrollausgabe nach dem Speichern zeigt Mesh, AnimBP, COM −15, Masse 35, 6 Wheel Setups,
  720 U/min / 3 Nm / Kurve, Übersetzung [1.0] / Final 4.0 und Differential ALL_WHEEL_DRIVE / Split 0,5.

### Geerbte Komponenten

Von `BaseVehiclePawnNW` kommen `VehicleBounds` (Box, `NoCollision`), `CustomCollision`, `VelocityControl` und
`BaseMovementComponent`. `VehicleBounds` muss nicht mehr von Hand angepasst werden wie in UE4:
CARLA setzt die Box beim Spawnen selbst aus dem Mesh (`ACarlaWheeledVehicle::AdjustVehicleBounds`).

## 7. Schritt 6: Registrierung ([`s06_register.py`](../scripts/unreal_editor/ue5/s06_register.py))

- Läuft außerhalb des Editors: `python3 scripts/unreal_editor/ue5/s06_register.py`.
- Ergänzt in `~/CarlaUE5/Unreal/CarlaUnreal/Content/Carla/Config/VehicleParameters.json` den Eintrag
  Make `starship`, Model `bot` → ID **`vehicle.starship.bot`** (gleich wie in UE4, Clients bleiben kompatibel),
  `NumberOfWheels` 6, `BaseType` `robot`, `SpecialType` `delivery`, `HasLights` false.
- Vorher wird eine Sicherung `VehicleParameters.json.bak_<Zeitstempel>` angelegt. Mehrfaches Ausführen ändert nichts.
- CARLA liest die Datei bei jedem Play neu (`AVehicleActorFactory::GetDefinitions`), ein Neustart ist nicht nötig.
- **Prüfung:** Der `diff` gegen die Sicherung enthält nur den neuen Eintrag. Die JSON bleibt gültig.

## 8. Schritt 7: Fahrtest und Fehlersuche

Ablauf: Play im Editor starten ([`s07_play.py`](../scripts/unreal_editor/ue5/s07_play.py), beenden mit
[`s07_stop.py`](../scripts/unreal_editor/ue5/s07_stop.py)), dann den Abnahmetest
[`scripts/carla/starship_ue5_check.py`](../scripts/carla/starship_ue5_check.py) mit dem CARLA-0.10-Client ausführen.
Der Client ist das selbst gebaute Wheel `~/CarlaUE5/Build/PythonAPI/dist/carla-0.10.0-cp312-cp312-linux_x86_64.whl`,
installiert in einer eigenen venv (Python 3.12). Der Test prüft Blueprint-Attribute, Physikwerte im laufenden Spiel,
Ruhelage, Vollgas 8 s, Bremsen, Volllenkung rechts, Rückwärts und Handbremse.

### Lauf 1: Bot bleibt in der Luft stehen

- Symptom: Ruhelage 1,05 m über Boden, 0 km/h bei Vollgas. Clevon1 fährt im selben Test 14 m.
- Analyse im laufenden Spiel per Remote Execution: `VehicleMesh.is_simulating_physics()` = False.
  Vergleich der CDOs: Clevon1 und EuropeanHGV haben `body_instance.simulate_physics = True`, BP_Starship `False`
  (von `BaseVehiclePawnNW` geerbt).
- Fix in `s05_vehicle_bp.py`: `simulate_physics = True`.

### Lauf 2: Fährt, aber zu langsam und wackelig

- Symptom: 1 km/h Höchstgeschwindigkeit, Nick- und Rollwinkel bis 15°.
  `get_physics_control()` zeigt nur bei Rad 0 die richtigen Werte, Räder 1 bis 5 haben Radius 32 cm, 50° Lenkung
  und keinen Antrieb.
- Analyse: Die Rad-Instanzen haben die richtige Klasse (`BP_Starship_FRW_C` …), deren CDO zeigt r = 9.
  `unreal.new_object(BP_Starship_FRW_C)` liefert trotzdem r = 32, bei FLW dagegen r = 9.
  Unterschied: FLW wurde in einem früheren Lauf *nach* dem Setzen der Werte noch einmal kompiliert, die anderen nicht.
  Ursache ist die Post-Construction-Property-Liste der Blueprint-Klasse (Details in Schritt 4).
- Fix in `s04_wheels.py`: nach dem Setzen kompilieren, mit Instanz-Prüfung.

### Lauf 3: Abnahme bestanden

| Prüfung | Ergebnis | Soll |
| --- | --- | --- |
| Blueprint | `vehicle.starship.bot`, `number_of_wheels` 6, `base_type` robot, `special_type` delivery | ✓ |
| Physik im Spiel | 35 kg, 6 Räder r = 9 cm, Lenkwinkel [30, 30, 0, 0, 0, 0], alle angetrieben, 720 U/min, 3 Nm | ✓ |
| Bounding Box (halb) | 0,282 × 0,277 × 0,329 m, automatisch aus dem Mesh | ✓ |
| Ruhelage | steht auf den Rädern, 0 km/h | ✓ |
| Höchstgeschwindigkeit | **6,27 km/h**, 90 % nach 0,8 s | ~6 km/h, rechnerisch 6,1 |
| Nick-/Rollwinkel geradeaus | < 1° | stabil |
| Bremsweg aus ~6 km/h | 0,22 m | |
| Volllenkung bei 60 % Gas | Wenderadius ca. 2,2 bis 2,4 m, Rollwinkel < 3° | kippt nicht |
| Rückwärts | 6,1 km/h | ✓ |
| Handbremse | bremst auf 0 | ✓ |

Optische Kontrolle mit einer CARLA-Kamera (Town10HD): Alle 6 Räder stehen auf dem Boden, Korpus dunkelgrau wie im
UE4-Stand.

![Starship in CARLA 0.10](images/starship_ue5_seite_vorne.png)
![Starship in CARLA 0.10, Seitenansicht](images/starship_ue5_seite.png)

### Sensor-Ausstattung: Editor-Absturz wegen Grafikspeicher

- Versuch: alle 25 Sensoren aus `config/starship_sensors.json` anbringen (9 RGB-Kameras, 3 Tiefenkameras, 4 Radare,
  8 Hindernissensoren, GNSS, IMU) und Daten abwarten.
- Ergebnis: Der Editor stürzte beim Anlegen der dritten RGB-Kamera ab:
  `Fatal error: Out of memory on Vulkan; MemoryTypeIndex=4` (Log `Saved/Logs/CarlaUnreal.log`).
  Ursache ist der Grafikspeicher (16 GB): Town10HD in Epic-Qualität, der Editor-Viewport und jede Kamera mit eigenem
  Render-Target. Ein Fehler am Bot ist das nicht. Alle Bot-Assets waren vorher gespeichert (letzte Speicherung
  10:12, Absturz 10:13).
- **Offen:** Sensor-Test in einer leichteren Umgebung wiederholen, z. B. Town02_Opt, Qualität `Low`
  (`-quality-level=Low`), CARLA als eigenständiger Server statt Play im Editor, oder zuerst nur eine Kamera je Gruppe.

## 9. Ergebnis und Wiederholung

Angelegte Assets im CARLA-Projekt (`~/CarlaUE5/Unreal/CarlaUnreal/Content/`):

| Pfad | Inhalt |
| --- | --- |
| `Carla/Static/Car/Starship_CM/` | `SK_Starship`, `SK_Starship_Skeleton`, `SK_Starship_PhysicsAsset`, `AnimBP_Starship`, Materialien `dark_gray`, `black`, `hka_body`, `hka_red`, `hka_lid` |
| `Carla/Blueprints/Vehicles/Starship/` | `BP_Starship`, `BP_Starship_{FLW,FRW,MLW,MRW,RLW,RRW}`, `Curve_Starship_Torque` |
| `Carla/Config/VehicleParameters.json` | Eintrag `starship.bot` (Sicherung `VehicleParameters.json.bak_20261020_100231`) |

Komplett neu aufbauen (Editor offen, Remote Execution an, kein Play):

```bash
scripts/unreal_editor/ue5/build_all.sh
```

Danach Play starten und testen:

```bash
python3 scripts/unreal_editor/ue5/ue_remote.py scripts/unreal_editor/ue5/s07_play.py
python scripts/carla/starship_ue5_check.py
```

### Abweichungen gegenüber dem UE4-Stand (bewusst)

| Punkt | UE4 (PhysX) | CARLA 0.10 (Chaos) | Grund |
| --- | --- | --- | --- |
| Simulierte Räder | 4, Mittelräder per Copy Bone | 6 | Chaos kann N Räder. Entspricht dem echten Bot. |
| Antrieb | Vorderräder (Open Front Drive) | alle 6, All Wheel Drive (vorne je 25 %, Mitte/hinten je 12,5 %) | echter Starship hat 6 Radnabenmotoren, bessere Traktion an Kanten |
| Motor | Torque Curve 10 bis 15 Nm, Final 3,4 | flach 3 Nm, Final 4,0 | 6 statt 2 angetriebene Räder. Ziel weiter ~6 km/h. |
| Federung | Eigenfrequenz 9 Hz | Spring Rate 186,5 (aus 9 Hz umgerechnet) | Chaos hat kein Natural-Frequency-Feld |
| VehicleBounds | von Hand gesetzt | automatisch | CARLA 0.10 passt die Box beim Spawnen an |
| Tire Config | CommonTireConfig | entfällt, Chaos-Standardreibung | Chaos hat keine Tire Configs |

### Noch offen

- Sensor-Test (siehe oben).
- Feinabstimmung von Reibung und Schräglaufsteifigkeit, z. B. Bordsteinauffahrt und Steigung, sowie Vergleich mit
  Messwerten des echten Starship.
- `starship_drive.py` und `starship_sensors.py` sind für 0.9.16 geschrieben und noch nicht gegen 0.10 getestet.
  Beim Abnahmetest fiel auf: In 0.10 heißen die Radfelder anders (`wheel_radius` statt `radius`), und das
  Durchlaufen der Blueprint-Attribute mit `as_str()` brach mit `RuntimeError: std::exception` ab
  (Ursache nicht untersucht, `get_attribute` funktioniert).

## 10. Änderungen nach der ersten Abnahme

### 20.10.2026: Differential auf All Wheel Drive

- Im Editor an `BP_Starship` → Vehicle Movement → Differential Setup: `Differential Type` von `UNDEFINED` auf
  **`ALL_WHEEL_DRIVE`** umgestellt, `Front Rear Split` bleibt 0,5. Übernommen in `s05_vehicle_bp.py`, damit
  `build_all.sh` die Einstellung nicht zurücksetzt.
- Wirkung: Chaos teilt das Moment zuerst nach Achstyp (`FRONT` ↔ `REAR`) und dann auf die Räder dieser Achse.
  Weil Mittel- und Hinterräder beide `REAR` sind, bekommen die Vorderräder je 25 % und die übrigen vier je 12,5 %.
  Gleichmäßig je 1/6 wäre `front_rear_split` = 0,667.
- Fahreindruck: fährt gut auf ebener Strecke. Kommt nicht über Bordsteine, reagiert steif auf kleine Unebenheiten
  (Fahrbahnmarkierungen, Schlaglöcher) und bleibt teils stecken.

### 20.10.2026: Analyse Bordstein und Unebenheiten

**Messung der Umgebung** (Town10HD, 1150 Punkte an Gehwegkanten): Bordsteine sind fast überall **15 cm** hoch
(Median 14,8 cm), abgesenkte Stellen 0 cm, an 2 Stellen 23 cm. An der Unterkante hat der Bordstein eine ca. 2 cm
breite Fase.

**Basismessung** mit [`scripts/carla/starship_curb_test.py`](../scripts/carla/starship_curb_test.py)
(feste Stellen, senkrechte Anfahrt mit Vollgas, IMU für Stöße; Rohdaten
[`messungen/curb_results_basis.json`](messungen/curb_results_basis.json)):

| Fall | Ergebnis |
| --- | --- |
| hoch, 3 Stellen (13,6 bis 14,4 cm) | 3 × gescheitert, Bot steht 0,37 m vor der geschätzten Kante |
| runter, 2 Stellen | geschafft, aber Stoßspitze 37 bis 44 m/s² (≈ 4 g), Nickwinkel ca. 20° |

**Blockiermechanismus** (im laufenden Spiel ausgelesen):

- Die Kollisionsbox berührt den Bordstein nicht: Unterkante ca. 19 cm, Bordstein 15 cm.
- Die Mitte des Vorderrads steht 1 bis 2 cm vor der Bordsteinfront. Vorderräder tragen fast das ganze Gewicht,
  die Mittelräder hängen voll ausgefedert in der Luft.
- Chaos tastet jedes Rad mit einem senkrechten Strahl ab. Der Strahl startet `suspension_max_raise` (2 cm) über der
  Radmitte, also ca. 12 cm über der Fahrbahn und damit **unter** der Bordsteinoberkante (15 cm). Das Rad kann die
  Oberkante nie erfassen. In der Fase an der Unterkante trifft der Strahl dagegen schräge Flächen, und die Feder
  drückt entlang dieser Flächennormale nach hinten. Das wirkt wie eine Wand.

**Wie Chaos die Federkraft aufbringt** (`Chaos/Private/Chaos/PBDSuspensionConstraints.cpp`, UE 5.5):

1. Die Federkraft wirkt entlang der **Flächennormale** des Trefferpunkts, nicht entlang der Federachse.
   Schräge Treffer erzeugen also eine Kraft nach hinten.
2. Ist die Normale flacher als 45° und der Bot langsamer als 1 mph, wird sie durch die Senkrechte ersetzt
   (Anti-Rutsch-Mechanik). Bis 10 mph wird übergeblendet.
3. Wird die Feder über `max_raise` hinaus gestaucht, greift ein harter Anschlag entlang der Normale.
4. `spring_preload` wird in diesem Modus **nicht verwendet** (im Code auskommentiert). Die Ruhelage ergibt sich nur aus
   Federrate und Federweg: Einfederung aus voller Ausfederung = Radlast / k. Bei k = 186,5 N/cm sind das 3 mm von
   2 cm `max_drop`, der Bot fährt also ca. 1,7 cm höher als modelliert (passt zur gemessenen Ruhelage von +2 cm).
5. (Überholt, siehe Abschnitt 12.) `SHAPECAST` ist in UE 5.5 nicht implementiert und fällt auf Raycast zurück. `SPHERECAST` trifft die Bordsteinkante
   zuerst seitlich (Normale fast waagrecht). Das gibt eine große Kraft nach hinten (Rechnung: bei 8 cm Abstand ca.
   100 N je Vorderrad gegen 120 N Zugkraft gesamt). Deshalb wird **Raycast beibehalten**.

### Plan v1: Bordstein und weiche Federung ohne höhere Geschwindigkeit (umgesetzt, Ergebnis siehe unten)

| # | Änderung | Wert alt → neu | Begründung |
| --- | --- | --- | --- |
| 1 | `suspension_max_raise` (alle Räder) | 2 → **16 cm** | Strahl startet dann 26 cm über der Fahrbahn, über jedem 15-cm-Bordstein. Das Rad erfasst die flache Oberseite (Normale senkrecht, keine Kraft nach hinten) und kann 15 cm einfedern, ohne an den harten Anschlag zu kommen. Bildet die Kletterbewegung des echten Starship nach: Rad geht hoch, Aufbau kippt hinterher. |
| 2 | `spring_rate` | 186,5 → **28,6** (N/cm) | k = Radlast / `max_drop` = 57 N / 2 cm. Damit liegt die Ruhelage genau auf der modellierten Höhe, und die Eigenfrequenz sinkt von 9 auf 3,5 Hz (f = √(g/d)/2π). Das macht ihn auf Markierungen und Schlaglöchern weich. |
| 3 | `suspension_damping_ratio` | 0,8 → **0,7** | übliche Dämpfung für weiche Fahrwerke, verhindert Nachschwingen nach dem Bordstein |
| 4 | `spring_preload` | 20 → **0** | ohne Wirkung (siehe Punkt 4 oben), nur zur Klarheit |
| 5 | Kollisionsbox Unterkante | 16 → **19 cm** (Box 55 × 45 × 30, Mitte 25 cm über `VehicleBase`) | 4 cm Abstand über 15-cm-Bordsteinen, auch in dem Moment, in dem die Vorderräder schon oben sind und der Aufbau noch nicht. Die Oberkante bleibt bei 49 cm. |
| – | `max_rpm`, Übersetzung, `max_torque` | unverändert | Höchstgeschwindigkeit bleibt 6,3 km/h |
| – | Reibung (`friction_force_multiplier`) | unverändert | Traktion reicht: 2 × Radlast je Rad > Antriebskraft. Grip ist nicht der Engpass. |

Erwartete Nebenwirkungen: Beim Hochfahren springt das Rad sichtbar bis zu 15 cm nach oben, bevor sich der Aufbau
hebt. Hindernisse bis ca. 24 cm Höhe gelten für das Modell als befahrbar.

Rechnerische Prüfung der Box beim Überfahren (Aufbau starr, Federn im Gleichgewicht): Mittelräder an der Kante →
Box-Unterkante über der Kante ca. 23 cm; Hinterräder an der Kante → ca. 19 cm. Beides liegt über 15 cm.

**Abnahmekriterien** (gleicher Test, gleiche Stellen wie die Basismessung):

1. hoch: alle Stellen bestanden, je in höchstens 3 s
2. runter: alle Stellen bestanden, Stoßspitze mindestens halbiert (< 20 m/s²)
3. Abnahmetest `starship_ue5_check.py`: Höchstgeschwindigkeit 6,0 bis 6,5 km/h, Roll-/Nickwinkel geradeaus < 2°,
   Kurve ohne Umkippen
4. Ruhelage: Radknochen auf Fahrbahnhöhe (± 0,5 cm)

Nur falls Kriterium 1 an der Fase scheitert (Bot bleibt mit Radmitte an der Unterkante stehen), folgt als vorab
festgelegter zweiter Schritt mehr Drehmoment **nur bei niedriger Drehzahl**: Die Drehmomentkurve wird bei 0 U/min
angehoben und läuft bei 720 U/min auf den alten Wert aus. Die Höchstgeschwindigkeit hängt nur an `max_rpm` und der
Übersetzung und bleibt damit gleich.

### 20.10.2026: Umsetzung und Messreihe

| Lauf | Änderung | Bordstein hoch | Runter (Stoßspitze) | Geradeaus |
| --- | --- | --- | --- | --- |
| Basis | – | 0/3, Stopp bei −0,37 m | 2/2 (37 bis 44 m/s²) | 6,27 km/h |
| v1 | max_raise 16, k 28,6, Box-Unterkante 19 cm | 0/3, Stopp bei −0,37 m (Stelle 0: Vorderräder oben, Stopp bei −0,17 m) | 2/2 (35 bis 37 m/s²) | 6,29 km/h |
| v2 | max_drop 4, max_raise 18, k 14,3 | 0/3, Stopp bei −0,37 m | 2/2 (46 bis 47 m/s²) | 6,29 km/h, stabil |

Rohdaten: `messungen/curb_results_{basis,v1_federweg,v2_drop4}.json`.

**Widerlegte Annahmen:**

1. *Der Strahl startet unter der Bordsteinoberkante.* Das stimmt zwar, war aber nicht die alleinige Ursache.
   Mit 16 und 18 cm Federweg bleibt der Bot an derselben Stelle stehen.
2. *Die Kollisionsbox stößt an.* Widerlegt: Strahlen von unten gegen die Bot-Kollision zeigen die Unterseite bei
   19 cm. Der Bordstein ist 14,6 cm hoch und hat keine größere vereinfachte Kollisionsform (Messung mit
   `trace_complex` False und True identisch).
3. *Die Feder drückt an der geneigten Bordsteinfront (≈ 10°, Normale z = 0,18) nach hinten, proportional zur
   Federrate.* Widerlegt: Halbe Federrate ändert die Stoppposition nicht (±1 mm).

**Gesicherte Befunde:**

- Die Stoppposition ist an allen Stellen gleich: Vorderradmitte ca. 1 cm vor der Bordsteinfront. Der Anprall ist
  abrupt (1,5 → 0,13 m/s auf 7 cm). Im Stillstand stehen alle 6 Radstrahlen auf der Fahrbahn, die Räder drehen nicht.
- **Kräftegleichgewicht, kein starrer Anschlag:** Mit dreifachem Motormoment (9 Nm, nur am laufenden Test-Bot gesetzt)
  fährt der Bot weiter, und die Vorderräder steigen auf den Bordstein (Nickwinkel 22°).
- **Danach kippt er nach hinten über** (Nickwinkel 88°). Ursache: Liegen die Vorderräder auf dem Bordstein, drückt
  jede Vorderradfeder mit k · (max_drop + 15 cm) ≈ 270 N, zusammen mehr als das 1,5-Fache des Gesamtgewichts
  (343 N). Bei 38 cm Radstand und hohem Antriebsmoment hebt das die Front immer weiter an. Mit dem Aufbau neigen
  sich auch die Radstrahlen.
- Nebenbefund: Mit 30 % Gas fährt der Bot auf ebener Straße kaum an (≈ 1 cm/s). Bei niedriger Geschwindigkeit
  wirkt ein Widerstand in der Größenordnung des Antriebs. Wahrscheinlich hängt das mit dem Stopp am Bordstein zusammen.
  Noch nicht untersucht.

**Stand der Assets:** v2 (max_drop 4, max_raise 18, k 14,3, Box-Unterkante 19 cm). Normales Fahren unverändert
gut: 6,29 km/h, Nick-/Rollwinkel < 3°, Bremsweg 0,22 m. Bordsteine weiterhin nicht befahrbar.

### 20.10.2026: Variante A, Bordstein-Assistent (Nachbildung des Starship-Kipphebels)

**Hintergrund Original** ([Patent US12005753B2](https://patents.google.com/patent/US12005753B2/en)): Die Räder sitzen an
Kipphebeln (Bogie). Die Hebel drehen zuerst frei (ca. 25–45°), dann dreht sie ein Motor aktiv und hebt dabei vor allem
die Mittelräder an. Federn werden nicht genannt. Raddurchmesser „about 20 cm“ (Radius ≈ 10 cm, unser Modell 9 cm),
überwindbare Hindernisse 15–20 cm. Chaos kennt keine Hebel zwischen Rädern. Darum ersetzt ein Client-Skript den
Hebelmotor durch eine geregelte Hubkraft je Achse. **Fahrzeug-Assets und -Werte bleiben unverändert** (Stand v2).

Datei: [`scripts/carla/starship_curb_assist.py`](../scripts/carla/starship_curb_assist.py). Läuft allein neben jedem
Fahr-Client (`python starship_curb_assist.py`) oder als Modul (`CurbAssist(world, vehicle).step()`).
Im Bordstein-Test mit `--assist`.

| Baustein | Umsetzung |
| --- | --- |
| Erkennen | Ein waagrechter `cast_ray` 5 cm über dem Boden, 10 bis 110 cm vor der Vorderradmitte. Bei Treffer Höhe dahinter messen (Stufe 4–25 cm). Einmalig ein Profil im 1-cm-Raster: Ab wo ist die Oberseite eben? |
| Anheben | PD-Regler auf die Höhe des Radknochens: Soll = Oberkante + Radradius + 4 cm Ausfederung + 2 cm Luft. Kraft = 1500 N/m · Fehler − 250 N·s/m · Geschwindigkeit + 114 N (Gewichtsanteil einer Achse), begrenzt auf 0–300 N, als Impuls `add_impulse_at_location` je Regeltakt (max. 30 ms). |
| Reihenfolge | Erste Achse, die noch nicht über der Kante ist: **vorne** → nur Vorderachse (Aufbau kippt um die Hinterachse, geometrisch ≈ 30°). **Mitte/hinten** → diese Achse und alle dahinter gemeinsam (Aufbau richtet sich auf, die oberen Räder treiben). Freigabe 2 cm hinter Beginn der ebenen Oberseite. |
| Schutz | ab Erkennung höchstens 0,6 m/s (Bremsimpuls), Hubkraft vorne wird über 40° Nickwinkel zurückgenommen |

**Fehlersuche während der Umsetzung (Messungen gegen Unreal-Knochenpositionen per Remote Execution):**

1. **`carla.Transform.transform()` und `get_forward_vector()` haben im CARLA-0.10-Python-Client bei Neigung das
   falsche Vorzeichen.** Bei 15° Nase hoch steht in Unreal das Vorderrad 10 cm höher als das Hinterrad, der Client
   meldet es 10 cm tiefer. Die Fahrzeugposition selbst stimmt überein. Der Assistent rechnet Achspunkte deshalb
   selbst aus Gier- und Neigungswinkel (pitch > 0 = Nase hoch, wie in Unreal).
   Wegen dieses Fehlers sah es zunächst so aus, als würde `add_impulse_at_location` den Angriffspunkt spiegeln.
   Das ist **nicht** der Fall: Der Impuls wirkt am übergebenen Punkt.
2. Erste Version mit 11 `ground_projection`-Abfragen je Erkennung: zu langsam (Bot fuhr währenddessen 34 cm).
   Eine Regelpause von 100 ms ergab einen Einzelimpuls von 45 N·s, und der Bot überschlug sich.
   Behoben durch `cast_ray`-Erkennung und die Begrenzung der Taktzeit auf 30 ms.
3. Vorder- und Mittelachse gleichzeitig anzuheben gab 41° Nickwinkel. Behoben durch die Reihenfolge-Logik.
4. Feste Freigabe 6 cm: Vorderrad wieder an der Bordsteinfront. Feste Freigabe 12 cm: zu lange angehoben, 39°,
   Stillstand. Behoben durch die Freigabe aus dem gemessenen Profil.

**Messung** (gleiche Stellen wie Basis, Rohdaten `messungen/curb_results_a*_assist*.json`):

| Lauf | hoch Stelle 0 | hoch Stelle 1 | hoch Stelle 2 | runter |
| --- | --- | --- | --- | --- |
| Basis (ohne Assistent) | ✗ | ✗ | ✗ | 2/2 |
| a4, Durchlauf 1 | ✓ 2,6 s, 27° | ✓ 3,0 s, 32° | ✗ +0,06 m, Roll 17° | 2/2 |
| a4, Durchlauf 2 | ✗ −0,18 m | ✓ 2,7 s, 32° | ✗ +0,10 m, Roll 18° | 2/2 |

Zeiten inklusive 1,2 m Anlauf. Stoßspitze beim Hochfahren 14–17 m/s² (Basis-Runterfahren 37–44). Geradeausfahrt,
Höchstgeschwindigkeit und Assets unverändert.

**Bewertung gegen die Kriterien:** nicht erfüllt. Hoch gelingt in 3 von 6 Versuchen (vorher 0 von 9). Nickwinkel
27–32° statt < 25°. Das 25°-Kriterium ist für ein starres Modell mit 38 cm Radstand und 15-cm-Bordstein geometrisch
nicht erreichbar (mindestens ≈ 28°). Das Original umgeht das mit seinen Gelenkhebeln.

**Offene Ursachen und nächste Schritte:**

1. **Schräge Anfahrt / Rollen (Stelle 2, Roll 17–18°):** Kraft je Seite (linkes und rechtes Rad einer Achse) mit
   eigenem Kantenabstand statt in der Achsmitte.
2. **Streuung zwischen Durchläufen (Stelle 0):** Der Client läuft asynchron zum Server, die Regeltakte schwanken.
   Abhilfe: Test im synchronen Modus, um die Logik von der Latenz zu trennen. Für den Dauerbetrieb den Assistenten
   als Komponente im Fahrzeug (C++ im CARLA-Plugin, Takt = Physiktakt) umsetzen.

### 20.10.2026: Punkt 1, seitengetrenntes Anheben (Python-Assistent)

- Jede Seite (links/rechts, y = ∓25,6 cm) hat eine eigene Kantenerkennung, ein eigenes Profil und eine eigene
  Hebelphase. Die Kraft wirkt am einzelnen Rad: Vorsteuerung 57 N (1/6 Gewicht), PD 750 N/m / 125 N·s/m,
  höchstens 150 N je Rad.
- Rollschutz: Über 6° Rollwinkel wird die höhere Seite entlastet.
- Achspunkte mit der vollständigen Unreal-Rotationsmatrix (Pitch, Yaw, Roll). Geprüft gegen Unreal-Knochenpositionen
  bei 12° Pitch und 15° Roll: Abweichung ≤ 1,5 cm. Vorzeichen: Roll > 0 heißt rechte Seite tiefer.

| Lauf | hoch 0 | hoch 1 | hoch 2 | runter |
| --- | --- | --- | --- | --- |
| a5, Durchlauf 1 | ✓ 2,3 s, 27°, Roll 1° | ✓ 3,2 s, 32°, Roll 4° | ✗ −0,04 m, 34° | 2/2 |
| a5, Durchlauf 2 | ✓ 2,5 s, 27°, Roll 1° | ✓ 3,0 s, 32°, Roll 3° | ✓ 2,6 s, 33°, Roll 2° | 2/2 |

Hoch 5/6 (vorher 3/6). Rollwinkel 1–4° statt 17–18°. Der verbleibende Ausfall streut zwischen den Durchläufen
(Client-Takt) und ist der Anlass für Punkt 2.

### 20.10.2026: Punkt 2, Assistent als C++-Komponente im CARLA-Plugin

- Neue Klasse **`UStarshipCurbAssistComponent`** (`Plugins/Carla/Source/Carla/Vehicle/StarshipCurbAssistComponent.{h,cpp}`
  im CARLA-Checkout `~/CarlaUE5`, Kopie im Repo unter
  [`unreal/ue5_carla_plugin/`](../unreal/ue5_carla_plugin/Source/Carla/Vehicle/)). Gleiche Logik wie der
  Python-Assistent mit Punkt 1. Unterschiede:
  - Tick im Spieltakt **vor der Physik** (`TG_PrePhysics`), keine Client-Latenz.
  - Kraft per `AddForceAtLocation` je Frame statt Impulse, Bodenabfragen als direkte Line-Traces
    (statische Objekte, eigenes Fahrzeug ausgenommen).
  - Rotationen direkt aus Unreal (kein CARLA-Client-Vorzeichenproblem).
  - Alle Parameter als `UPROPERTY` im Details-Panel unter „Curb Assist“ einstellbar, `bEnabled` zum Abschalten,
    `bDebugLog` für Logausgaben (`LogCarla`).
- Build: Editor schließen, dann
  `~/UnrealEngine5_carla/Engine/Build/BatchFiles/Linux/Build.sh CarlaUnrealEditor Linux Development -project=$HOME/CarlaUE5/Unreal/CarlaUnreal/CarlaUnreal.uproject -game -buildscw`.
  Hinweis: `cmake --build Build --target carla-unreal-editor` bricht in dieser Shell ab, weil
  `CARLA_UNREAL_ENGINE_PATH` nicht gesetzt ist, und prüft außerdem fastdds neu. Der direkte UBT-Aufruf ist der
  Schritt, den das CMake-Ziel am Ende ausführt.
- An `BP_Starship` gehängt mit [`s08_curb_assist_component.py`](../scripts/unreal_editor/ue5/s08_curb_assist_component.py)
  (Komponente `CurbAssist`).

**Messung C++-Komponente, erste Fassung** (`messungen/curb_results_c1_cpp_run*.json`, Python-Assistent aus):
hoch Stelle 0 und 1 je 2/2, Stelle 2 0/2, in beiden Läufen identisch (−0,01 m). Das Ergebnis ist jetzt
reproduzierbar, der Ausfall also ein Logikfehler und keine Streuung.

**Korrektur 1, geneigte Sollhöhen in der Mittelphase:** Die Mittel- und Hinterachse wurden auf dieselbe Sollhöhe
(Oberkante + Radius + Durchhang + 2 cm) gehoben. Der starre Aufbau hielt damit auch das schon überquerte Vorderrad
2 cm über der Oberseite. Kein Rad hatte Kontakt, der Bot schwebte ohne Vortrieb (Nickwinkel 0°). Jetzt liegen
die Sollhöhen auf einer Geraden: Das zuletzt überquerte Rad setzt mit 1 cm auf, das angehobene hat 2 cm Luft.
Für die Achsen dahinter wird die Gerade fortgesetzt (Aufbau ca. 6° nach vorn geneigt).
Mit dem Python-Assistenten geprüft (Stelle 2 bestanden), dann in C++ übernommen.

**Messung C++ mit Korrektur 1** (`messungen/curb_results_c2_cpp_gerade_run1..4.json`):

| | Stelle 0 | Stelle 1 | Stelle 2 | runter |
| --- | --- | --- | --- | --- |
| hoch, 4 Durchläufe | 2/4 | 4/4 | 4/4 (3,5 bis 6,0 s) | 8/8 |

Hoch insgesamt 10/12. Geradeausfahrt unverändert (6,31 km/h, Bremsweg 0,22 m).

**Korrektur 2, Anhalten vor der Kante und Nase-runter-Schutz:** Bei den Ausfällen an Stelle 0 stand der Bot mit
25° **Nase unten** (der Test zeigt den Nickwinkel als Betrag). Die Mittelphase beginnt erst, wenn das Vorderrad
über der Kante ist. Dann ist das Mittelrad nur noch ca. 15 cm vor der Front. War es bis dahin nicht hoch genug,
traf es die Front zu tief und wurde dort gehalten, während das Hinterrad weiter stieg. Ob es reicht, hing von der
Geschwindigkeit ab, daher die Streuung. Neu:

- **Anhalten:** Ist das gerade anzuhebende Rad weniger als 8 cm vor der Kante und noch mehr als 2 cm unter Soll,
  bremst der Assistent bis zum Stillstand (600 N·s/m), bis das Rad oben ist. Das Original hält an Bordsteinen
  ebenfalls kurz an.
- **Nase-runter-Schutz:** Bei mehr als 10° Nase unten wird die Hubkraft der Räder hinter dem angehobenen Rad
  zurückgenommen.

Eingebaut in C++ (`HoldDistance`, `HoldError`, `HoldDamping`, `NoseDownLimit`) und Python
(`HOLD_DIST`, `HOLD_ERR`, `HOLD_C`, `NOSE_DOWN_LIMIT`).

**Messung C++ mit Korrektur 2** (`messungen/curb_results_c3_cpp_halten_run1..4.json`): hoch **0/12**. Der Bot hält wie
vorgesehen vor der Kante an. Das Rad erreicht seine Sollhöhe dann aber nicht mehr (Rollwinkel bis 17°), und der Bot
bleibt dauerhaft stehen. **Korrektur 2 ist verworfen.**

## 11. Abschluss Bordstein (20.10.2026)

**Endstand = bester gemessener Stand:** C++-Komponente `UStarshipCurbAssistComponent` mit seitengetrenntem Anheben
und geneigten Sollhöhen (Korrektur 1), ohne Anhalte-Logik.

- Quellcode C++ (`~/CarlaUE5/.../Vehicle/StarshipCurbAssistComponent.{h,cpp}`, Kopie in `unreal/ue5_carla_plugin/`)
  und Python (`scripts/carla/starship_curb_assist.py`) sind auf diesen Stand zurückgesetzt.
- Die aktuell gebaute Plugin-Bibliothek enthält die Anhalte-Logik noch. Im Blueprint ist sie über
  `HoldDistance = −1000` und `NoseDownLimit = 90` abgeschaltet und verhält sich damit wie der Endstand. Beim nächsten
  Plugin-Build entfallen diese beiden Eigenschaften, ihre Werte im Blueprint werden dann ignoriert.

| Ergebnis Endstand (4 Durchläufe, Town10HD) | |
| --- | --- |
| Bordstein hoch | **10/12** (Stelle 0: 2/4, Stelle 1: 4/4, Stelle 2: 4/4), 2,7 bis 6,0 s inkl. 1,2 m Anlauf |
| Bordstein runter | 8/8 (Stoßspitze unverändert 30 bis 46 m/s²) |
| Nickwinkel beim Hochfahren | 27–33° (geometrisch nötig ≥ 28°) |
| Rollwinkel | 1–6° |
| Geradeaus / Bremsen / Rückwärts | unverändert: 6,3 km/h, Bremsweg 0,22 m |
| Fahrzeug-Assets und -Werte | unverändert seit v2 |

**Benutzung:** Die Komponente `CurbAssist` hängt an `BP_Starship` und ist immer aktiv. Mit `manual_control.py`,
`starship_drive.py` oder ROS ist nichts weiter zu tun. Abschalten im Blueprint über *Curb Assist → Enabled*,
Logausgaben über *Debug Log*. Das Python-Skript `starship_curb_assist.py` ist nur noch Referenz bzw. für
CARLA-Installationen ohne neu gebautes Plugin gedacht. **Nicht zusammen mit der C++-Komponente betreiben.**

**Bekannte Grenzen:**

- Ca. 1 von 6 Hochfahrten bleibt hängen. Muster: Mittelrad erreicht die Bordsteinfront zu tief, der Bot steht mit
  der Nase nach unten. Abhilfe beim Fahren: kurz zurück und erneut anfahren.
- Hindernisse bis ca. 25 cm Höhe gelten als „Bordstein“ und werden erklettert.
- Runterfahren ohne Unterstützung, dabei harte Landung (≈ 4 g).
- Nachbildung, kein Nachbau der Starship-Mechanik: Ein starrer Aufbau mit Hubkräften ersetzt die Gelenkhebel.

## 12. Radabfrage auf Spherecast (21.10.2026)

Umstellung von Hand im Editor: `sweep_shape` aller sechs Rad-Blueprints von `RAYCAST` auf **`SPHERECAST`**.
[`s04_wheels.py`](../scripts/unreal_editor/ue5/s04_wheels.py) ist nachgezogen, damit ein Neuaufbau den Stand behält.

- Beobachtung beim Fahren: Mit Raycast taucht das Rad in die Bordsteinfront ein und bleibt stecken (der Strahl
  trifft nur einen Punkt unter der Radmitte, die Radkontur selbst kennt keine Kollision). Mit Spherecast erfasst die
  Abfrage die Kante schon mit der Radkontur, das Rad steigt auf statt einzutauchen. Der Bot quert Bordsteine damit
  leichter.
- Das widerspricht der Rechnung aus Abschnitt 10 (Punkt 5: Kraft nach hinten an der Kante). Diese Rechnung galt
  ohne Bordstein-Assistent. Mit Assistent überwiegt offenbar der Vorteil, dass das Rad nicht mehr einhakt.
- Noch nicht gemessen: Die Werte in Abschnitt 11 (hoch 10/12, runter 8/8) stammen von Raycast. Neue Messreihe mit
  `scripts/carla/starship_curb_test.py --assist --label spherecast` steht aus.
