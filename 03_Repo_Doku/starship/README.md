# Starship-Lieferroboter (HKA-Design) für CARLA

Ein 6-rädriger Lieferroboter nach Vorbild des Starship-Bots, als fahrbares CARLA-Fahrzeug
(`vehicle.starship.bot`) mit kompletter Sensor-Ausstattung.

Entwickelt und getestet mit **CARLA Source-Build, Branch `ue4-dev` (Commit `ac006dd1b`, Python-API 0.9.16)**
und **Unreal Engine 4.26.2 (CARLA-Fork)**.

**CARLA 0.10 (UE 5.5, Chaos):** umgesetzt mit 6 simulierten Rädern, Skripte in `scripts/unreal_editor/ue5/`,
jeder Schritt dokumentiert in [docs/IMPLEMENTIERUNG_CARLA_0.10.md](docs/IMPLEMENTIERUNG_CARLA_0.10.md).

![HKA-Bot in CARLA](docs/images/carla_hka_left.png)

## Inhalt

| Ordner | Inhalt |
| --- | --- |
| `models/` | FBX und Blender-Dateien. **`starship_hka_v5.fbx` ist die aktuelle Version** (HKA-Design). `starship_carla_v4.fbx` = gleiches Skelett, ursprüngliches Starship-Design. |
| `unreal/ue4.26_vehicle3/` | Fertige Unreal-4.26-Assets aus `Content/Carla/vehicle3` (Blueprint, Räder, Mesh, Skelett, Physics Asset, AnimBP, Materialien). |
| `config/starship_values.json` | Alle Fahrzeugwerte, direkt aus dem Editor ausgelesen. |
| `config/starship_sensors.json` | Sensor-Konfiguration: Typ, Position, Ausrichtung, Attribute. |
| `scripts/carla/` | Python-Clients: Sensoren spawnen und anzeigen, Fahren, Geschwindigkeitstest. |
| `scripts/unreal_editor/` | Editor-Python: FBX-Import, Werte auslesen. |
| `scripts/unreal_editor/ue5/` | CARLA 0.10: Aufbau im laufenden Editor per Remote Execution (`build_all.sh`, Schritte `s01`–`s07`). |
| `config/ue5/` | CARLA 0.10: Drehmomentkurve als CSV. |
| `Sensorik/` | CARLA 0.10: Sensoranzeigen je Sensortyp mit Reitern (Kameras, ToF, Radar, Ultraschall, GNSS/IMU), Vorlage Clevon. Siehe [Sensorik/README.md](Sensorik/README.md). |
| `docs/` | [Fahrzeugwerte](docs/FAHRZEUGWERTE.md), [Sensorik mit Begründung](docs/SENSORIK.md), [Umstieg auf CARLA 0.10](docs/CARLA_0.10.md), Bilder. |

## Startbefehle (CARLA 0.10)

Alle Befehle mit dem Python aus CARLAs venv (`carla`, `pygame`, `numpy` sind dort installiert). Pfade:

```bash
R=~/Documents/Codex/repos/last_mile-delivery-carla/02_starship_emodell
PY=/home/ieem/CarlaUE5/.venv/bin/python
```

### 1. CARLA starten

```bash
~/UnrealEngine5_carla/Engine/Binaries/Linux/UnrealEditor ~/CarlaUE5/Unreal/CarlaUnreal/CarlaUnreal.uproject -prefernvidia
```

Im Editor **Play** drücken. Alternativ, wenn Remote Execution aktiv ist (Project Settings → Plugins → Python):

```bash
python3 $R/scripts/unreal_editor/ue5/ue_remote.py $R/scripts/unreal_editor/ue5/s07_play.py   # Play starten
python3 $R/scripts/unreal_editor/ue5/ue_remote.py $R/scripts/unreal_editor/ue5/s07_stop.py   # Play beenden
```

### 2. Bot spawnen und fahren (eine Variante wählen)

| Befehl | Was |
| --- | --- |
| `cd $R/Sensorik && $PY manual_control_starship.py` | **Empfohlen.** CARLAs Fahrclient mit Starship-GNSS/-IMU; spawnt den Bot. Steuerung W/A/S/D, Q Rückwärts, Leertaste Handbremse, Esc Ende. |
| `$PY ~/CarlaUE5/PythonAPI/examples/manual_control.py --filter 'vehicle.starship.bot'` | Originaler CARLA-Fahrclient ohne Starship-Sensoren. |
| `$PY $R/scripts/carla/starship_drive.py --spawn 5` | Eigener schlanker Fahrclient mit Fahrer- und Beobachtersicht (V wechselt, Backspace neu spawnen). |

Der Bordstein-Assistent (C++-Komponente in `BP_Starship`) ist immer aktiv, nichts weiter zu starten.

### 3. Sensoranzeigen (in weiteren Terminals, Bot muss existieren)

```bash
cd $R/Sensorik
$PY starship_cameras.py        # 9 Kameras, 5 Reiter (--preview: halbe Aufloesung)
$PY starship_tof.py            # 3 ToF-Tiefenkameras, 3 Reiter
$PY starship_radars.py         # 4 Radare, 3 Reiter inkl. Rundum-Draufsicht
$PY starship_ultrasonic.py     # 8 Ultraschall, Draufsicht + Verlauf
```

Alle Anzeigen: `--check` (ohne Server prüfen), `--vehicle-id N` (bei mehreren Bots), Tasten 1–9/Tab Reiter,
S speichern nach `Sensorik/Messdaten/`, Esc beenden. Kamera- und ToF-Anzeige nicht mehrfach parallel öffnen
(Grafikspeicher). Details: [Sensorik/README.md](Sensorik/README.md).

### ROS 2 (eingebautes ROS 2 von CARLA)

CARLA eigenständig mit `--ros2` starten (zwei Bindestriche; ohne Editor, sonst reicht der Grafikspeicher nicht),
Bot spawnen, dann:

```bash
~/UnrealEngine5_carla/Engine/Binaries/Linux/UnrealEditor ~/CarlaUE5/Unreal/CarlaUnreal/CarlaUnreal.uproject /Game/Carla/Maps/Town10HD_Opt -game -RenderOffScreen -quality-level=Low -prefernvidia --ros2
$PY $R/Sensorik/ROS2/starship_ros_sensors.py                      # 18 Sensoren mit ros_name (Strg+C entfernt sie)
source $R/Sensorik/ROS2/ros_ws/setup.bash && $PY $R/Sensorik/ROS2/starship_ros_extras.py   # Ultraschall, Odometrie, TF
source /opt/ros/jazzy/setup.bash && ros2 topic list               # /carla/starship/...
```

**Mit Fenster, um die Simulation zu sehen** (statt `-RenderOffScreen`; ca. 6,1 GB Grafikspeicher ohne Sensoren), dazu
die Nachführkamera, die die Beobachteransicht hinter den Bot setzt (`--view back|top|side`, steuert nichts):

```bash
~/UnrealEngine5_carla/Engine/Binaries/Linux/UnrealEditor ~/CarlaUE5/Unreal/CarlaUnreal/CarlaUnreal.uproject /Game/Carla/Maps/Town10HD_Opt -game -windowed -ResX=1280 -ResY=720 -quality-level=Low -prefernvidia --ros2
$PY $R/Sensorik/starship_follow_cam.py
```

Bot dafür mit `manual_control_starship.py` spawnen (setzt `ros_name=starship`), zum Steuern über ROS 2 stattdessen
ohne Fahrclient mit `spawn_starship.py` (der Fahrclient würde die ROS-Befehle überschreiben). Steuer-Topic
`/carla/starship/vehicle_control_cmd`, `carla_msgs` aus `Sensorik/ROS2/ros_ws`:

```bash
$PY $R/Sensorik/ROS2/spawn_starship.py                            # Bot als hero mit ros_name=starship, ohne Fahrclient
source $R/Sensorik/ROS2/ros_ws/setup.bash && ros2 topic pub -r 10 /carla/starship/vehicle_control_cmd carla_msgs/msg/CarlaEgoVehicleControl '{throttle: 0.7}'
```

Kein Not-Halt: Ohne neue Befehle fährt der Bot mit dem letzten weiter. Zum Anhalten
`ros2 topic pub --once -w 1 /carla/starship/vehicle_control_cmd carla_msgs/msg/CarlaEgoVehicleControl '{brake: 1.0}'`
(`-w 1` wartet auf CARLAs Subscriber, sonst kann ein Einzelbefehl verloren gehen). Topics, Messwerte, Einrichtung:
[Sensorik/ROS2/README.md](Sensorik/ROS2/README.md).

### 4. Tests

```bash
$PY $R/scripts/carla/starship_ue5_check.py        # Abnahme: Physik, Hoechstgeschwindigkeit, Bremsen, Lenken, Rueckwaerts
$PY $R/scripts/carla/starship_curb_test.py        # Bordstein hoch/runter an 3 festen Stellen (Town10HD), --label NAME speichert
```

### 5. Fahrzeug im Editor neu aufbauen (nur bei Änderungen am Modell)

Editor offen, Remote Execution aktiv, kein Play:

```bash
$R/scripts/unreal_editor/ue5/build_all.sh                                                       # Schritte 1-6: Mesh, Physik, AnimBP, Raeder, BP, Registrierung
python3 $R/scripts/unreal_editor/ue5/ue_remote.py $R/scripts/unreal_editor/ue5/s08_curb_assist_component.py   # Bordstein-Assistent anhaengen
```

Bordstein-Assistent (C++) nach Änderungen bauen, Editor vorher schließen:

```bash
~/UnrealEngine5_carla/Engine/Build/BatchFiles/Linux/Build.sh CarlaUnrealEditor Linux Development -project=$HOME/CarlaUE5/Unreal/CarlaUnreal/CarlaUnreal.uproject -game -buildscw
```

## Schnellstart (CARLA UE4, gleicher Stand)

1. Den Ordner `unreal/ue4.26_vehicle3/` nach `CarlaUE4/Content/Carla/vehicle3/` kopieren.
   Der Pfad muss genau so heißen, weil die Assets sich gegenseitig über `/Game/Carla/vehicle3/` referenzieren.
2. Editor starten, `Content/Carla/Blueprints/Vehicles/VehicleFactory` öffnen, unter **Vehicles** einen Eintrag hinzufügen:
   Make `starship`, Model `bot`, Class `BP_Starship`, Number of Wheels `4`. Compile, Save All.
3. Play drücken, dann im Terminal:

   ```bash
   pip install pygame numpy
   python scripts/carla/starship_drive.py
   python scripts/carla/starship_sensors.py
   ```

Die Rad-Blueprints brauchen CARLAs `CommonTireConfig` (`/Game/Carla/Blueprints/Vehicles/CommonTireConfig`);
die ist in jedem CARLA-UE4-Projekt vorhanden.

## Schnellstart (CARLA 0.10, UE 5.5)

1. Editor mit `CarlaUnreal.uproject` öffnen, *Project Settings → Plugins → Python → Enable Remote Execution* einschalten.
2. `scripts/unreal_editor/ue5/build_all.sh` ausführen. Das legt Mesh, Physics Asset, AnimBP, 6 Räder und
   `BP_Starship` an und trägt `vehicle.starship.bot` in `VehicleParameters.json` ein.
3. Play drücken, dann `python scripts/carla/starship_ue5_check.py` (CARLA-0.10-Client) ausführen.
4. Bordsteine: C++-Komponente `UStarshipCurbAssistComponent` aus `unreal/ue5_carla_plugin/` ins CARLA-Plugin kopieren,
   Plugin bauen, `scripts/unreal_editor/ue5/s08_curb_assist_component.py` ausführen. Stand und Grenzen:
   [IMPLEMENTIERUNG_CARLA_0.10.md, Abschnitt 11](docs/IMPLEMENTIERUNG_CARLA_0.10.md).

## Selbst importieren (andere CARLA-Version)

1. `models/starship_hka_v5.fbx` als **Skeletal Mesh** importieren: Skeleton *None*, Create Physics Asset an,
   Import Animations aus. Die FBX ist in cm, ohne Skalierung, Wurzelknochen `VehicleBase`.
2. Physics Asset: alle Bodies bis auf `VehicleBase` löschen, dort eine Box 55 × 45 × 33 cm,
   Mitte 23,45 cm über dem Knochen (Unterkante 16 cm über Boden).
3. AnimBP (Parent `VehicleAnimInstance`): `Mesh Space Ref Pose` → `Wheel Handler` → 2 × `Copy Bone`
   (`Wheel_Rear_Left` → `Wheel_Middle_Left`, `Wheel_Rear_Right` → `Wheel_Middle_Right`, nur Rotation)
   → `Component To Local` → Output.
4. Vier Rad-Blueprints und das Fahrzeug-Blueprint mit den Werten aus [docs/FAHRZEUGWERTE.md](docs/FAHRZEUGWERTE.md) anlegen.
5. In der VehicleFactory registrieren (siehe oben).

## Bekannte Stolperfallen

- **Skalierung:** Blender-Exporte mit Knoten-Skalierung 100 lassen den Bot durch die Welt fallen. Die FBX hier sind korrigiert.
- **Knochen-Ausrichtung:** Alle Knochen müssen ohne Rotation sein (in UE identische Achsen), sonst rechnet PhysX das Fahrwerk seitwärts.
- **VehicleBounds** braucht `NoCollision`, sonst steht der Bot auf dieser Box statt auf den Rädern.
- **Tire Config** muss an allen Rädern gesetzt sein, sonst stürzt CARLA beim Spawnen ab.
- **Speicher:** Mit 16 GB RAM läuft der Editor mit Town10 oft voll. Town02 verwenden und nie zwei Editoren gleichzeitig öffnen.
- Die Mittelräder werden mit PhysX nicht simuliert (4W-Fahrwerk), nur optisch mitgedreht.
