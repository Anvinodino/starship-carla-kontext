# ROS 2 (CARLA 0.10, eingebautes ROS 2 + Ergänzungsknoten)

Test am 20.10.2026: CARLAs eingebautes ROS 2 veröffentlicht 18 der 26 Starship-Sensoren selbst. Der Ergänzungsknoten
[`starship_ros_extras.py`](starship_ros_extras.py) liefert den Rest: 8 Ultraschall (`Range`), Fahrzeuglage
(`Odometry`, TF `map → starship`). Damit sind **alle 26 Sensoren und die Fahrzeuglage in ROS 2** verfügbar,
unter festen Namen `/carla/starship/...`.

| Teil | Wer veröffentlicht | Skript |
| --- | --- | --- |
| 9 Kameras, 3 ToF, 4 Radar, GNSS, IMU, `/clock`, TF Sensor → Fahrzeug | CARLA (`--ros2`) | `starship_ros_sensors.py` bringt die Sensoren mit `ros_name` an |
| 8 Ultraschall, Odometrie, TF `map → starship`, TF Ultraschall → Fahrzeug | eigener rclpy-Knoten | `starship_ros_extras.py` |
| fester Fahrzeugname `starship` statt `vehicle<ID>` | `ros_name` beim Spawnen | `manual_control_starship.py --ros-name starship` (Standard) |

## Voraussetzungen

- ROS 2 Jazzy (Ubuntu 24.04): einmalig [`install_ros_jazzy.sh`](install_ros_jazzy.sh) ausführen (fragt nach sudo).
- CARLA mit **`--ros2`** starten (zwei Bindestriche). Mit `-ros2` bleibt ROS 2 aus. Das Log schreibt in beiden Fällen
  „ROS2 enabled...“, wirklich aktiv ist es nur, wenn davor „ROS2: Enabling ROS2...“ steht
  (`CarlaSettings.cpp` prüft `FParse::Param(..., "-ros2")`, Unreal stellt selbst noch einen `-` davor).
- CARLA ist mit `ENABLE_ROS2=ON` gebaut (`libcarla-ros2-native.so`). Das ist im lokalen Build der Fall.

## Start

Je ein Terminal. `R=~/Documents/Codex/repos/last_mile-delivery-carla/02_starship_emodell`.

```bash
# 1. CARLA eigenständig, ohne Fenster, mit ROS 2 (Grafikspeicher, siehe unten)
~/UnrealEngine5_carla/Engine/Binaries/Linux/UnrealEditor ~/CarlaUE5/Unreal/CarlaUnreal/CarlaUnreal.uproject /Game/Carla/Maps/Town10HD_Opt -game -RenderOffScreen -quality-level=Low -prefernvidia --ros2

# 2. Bot spawnen und fahren (setzt ros_name=starship -> /carla/starship/...)
cd $R/Sensorik && /home/ieem/CarlaUE5/.venv/bin/python manual_control_starship.py

# 3. CARLA-Sensoren mit ros_name anbringen (alle Gruppen oder Auswahl: nav radar tof cameras)
/home/ieem/CarlaUE5/.venv/bin/python $R/Sensorik/ROS2/starship_ros_sensors.py

# 4. Ergänzungsknoten (Ultraschall, Odometrie, TF map -> starship)
source $R/Sensorik/ROS2/ros_ws/setup.bash && /home/ieem/CarlaUE5/.venv/bin/python $R/Sensorik/ROS2/starship_ros_extras.py

# 5. Empfangen
source /opt/ros/jazzy/setup.bash
ros2 topic list
ros2 topic echo /carla/starship/starship_us_front
ros2 run tf2_ros tf2_echo map starship_cam_front
```

Python für den Ergänzungsknoten: rclpy und carla im selben Interpreter. Das CARLA-venv `~/CarlaUE5/.venv` hat dafür
`pyyaml` bekommen (für rclpy), das Repo-venv `.venv-carla010` geht ebenso. Vorher immer `ros_ws/setup.bash` laden
(ROS 2 Jazzy + `carla_msgs`). `--no-ultrasonic` / `--no-odometry` schalten Teile ab, `--rate` setzt die Ausgaberate
(Standard 20 Hz).

## Steuern über ROS 2

CARLA legt für Fahrzeuge mit `role_name=hero` (setzt `manual_control.py` automatisch) einen Subscriber an:
**`/carla/starship/vehicle_control_cmd`**, Typ **`carla_msgs/msg/CarlaEgoVehicleControl`**, QoS best effort.

`carla_msgs` gibt es für Jazzy nicht als Paket. [`ros_ws/src/carla_msgs`](ros_ws/src/carla_msgs) enthält nur diese eine
Nachricht, mit Feldern und Typnamen exakt wie in CARLA (`LibCarla/source/carla/ros2/types/CarlaEgoVehicleControl.h`,
DDS-Typ `carla_msgs::msg::dds_::CarlaEgoVehicleControl_`) und im offiziellen `carla-simulator/ros-carla-msgs`.
Bauen ohne colcon und sudo: `ros_ws/build.sh` (einmalig), danach `source ros_ws/setup.bash`.

```bash
/home/ieem/CarlaUE5/.venv/bin/python $R/Sensorik/ROS2/spawn_starship.py   # Bot ohne Fahrclient (hero, ros_name=starship)
source $R/Sensorik/ROS2/ros_ws/setup.bash
ros2 topic pub -r 10 /carla/starship/vehicle_control_cmd carla_msgs/msg/CarlaEgoVehicleControl '{throttle: 0.7}'
ros2 topic pub -r 10 /carla/starship/vehicle_control_cmd carla_msgs/msg/CarlaEgoVehicleControl '{throttle: 0.6, steer: -1.0}'   # links
ros2 topic pub --once -w 1 /carla/starship/vehicle_control_cmd carla_msgs/msg/CarlaEgoVehicleControl '{brake: 1.0}'
```

**Einzelbefehle immer mit `--once -w 1`** (wartet, bis CARLAs Subscriber verbunden ist). Ein neuer Publisher braucht
ca. 1–3 s zur DDS-Discovery, vorher Gesendetes geht bei best effort verloren. Am Clevon gemessen: Ein 3-s-Bremsbefehl
kam gar nicht an, `--once -w 1` kam 6 von 6 Mal an.

Felder: `throttle` 0..1, `steer` −1..1 (**CARLA-Konvention: positiv = rechts**, anders als ROS-Winkel), `brake` 0..1,
`hand_brake`, `reverse`, `gear`, `manual_gear_shift`.

Getestet (Bot als `hero` mit `ros_name=starship`, ohne `manual_control`):

| Befehl | Ergebnis |
| --- | --- |
| `throttle: 0.7` | 6,1 km/h nach ca. 2 s (Höchstgeschwindigkeit) |
| `throttle: 0.6, steer: -1.0` | Linkskurve, Gierwinkel −175° → +40° in 4 s, 5,4 km/h |
| `brake: 1.0` | Stillstand innerhalb 1 s |

Wichtig:

- **Nicht gleichzeitig mit `manual_control` steuern:** Der Fahrclient setzt die Steuerung in jedem Takt aus der Tastatur
  und überschreibt die ROS-Befehle (auch der Autopilot steuert selbst). Zum Steuern über ROS den Bot ohne
  Fahrclient spawnen: Rolle `hero` und `ros_name=starship`, wie im Test.
- **Kein Not-Halt:** Getestet: Bleiben die Befehle aus, fährt der Bot mit dem letzten Befehl unbegrenzt weiter
  (5 s ohne Befehl, weiter 6,1 km/h). CARLA hat dafür keine Absicherung. Die Clevon-Bridge bremst nach 0,5 s ohne
  gültigen Befehl. Für den Starship fehlt das noch. Bis dahin zum Anhalten immer ausdrücklich `brake: 1.0` senden.

`starship_ros_sensors.py` liest dieselben JSON-Dateien wie die Anzeigen, setzt `ros_name = role_name` und hält die
Sensoren aktiv. Strg+C oder SIGTERM entfernt sie wieder. Den Signal-Handler erst nach dem Verbindungsaufbau setzen:
Der CARLA-Client überschreibt SIGINT auf C-Ebene, sonst wirkt Strg+C nicht.

## Gemessene Topics (alle gleichzeitig, eigenständiges CARLA, Town10HD)

| Topic (`/carla/starship/…`) | Typ | Rate (Soll) | Bemerkung |
| --- | --- | --- | --- |
| `starship_cam_*/image` (9) | `sensor_msgs/Image` | 3,98–4,12 Hz (4) | `bgra8`, seitlich hochkant 480×640 |
| `starship_cam_*/camera_info` (9) | `sensor_msgs/CameraInfo` | – | Lochkamera aus FoV, `plumb_bob`, keine Verzeichnung |
| `starship_tof_*/image` (3) | `sensor_msgs/Image` | 10,0 Hz (10) | **`bgra8` in CARLA-Tiefenkodierung**, nicht `32FC1` in Metern. Umrechnung: (R + G·256 + B·256²) / (256³−1) · 1000 m |
| `starship_tof_*/camera_info` (3) | `sensor_msgs/CameraInfo` | – | |
| `starship_radar_*/point_cloud` (4) | `sensor_msgs/PointCloud2` | 19,8–20,2 Hz (20) | Felder x, y, z, velocity, azimuth, altitude, depth |
| `starship_gnss` | `sensor_msgs/NavSatFix` | 4,99 Hz (5) | Kovarianz 0 |
| `starship_imu` | `sensor_msgs/Imu` | Welttakt (44–60 Hz; Soll 100) | |
| `/clock` | `rosgraph_msgs/Clock` | Welttakt | Simulationszeit; alle Header nutzen sie |
| `/tf` | `tf2_msgs/TFMessage` | – | CARLA: Sensor → `starship`; `frame_id` der Nachrichten = Sensorname |
| `starship_us_*` (8) | `sensor_msgs/Range` | 20 Hz | **Ergänzungsknoten.** `ULTRASOUND`, 0,02–3 m, Öffnungswinkel 0,2 rad (aus `hit_radius`), ohne Treffer `+inf` (REP 117). Gemessen: vorne 1,15 m gegen ein Testfahrzeug, hinten `inf` |
| `odometry` | `nav_msgs/Odometry` | 20 Hz | **Ergänzungsknoten.** Ground Truth, `map` → `starship`, Twist im Fahrzeugsystem. Geprüft: vorwärts x ≈ 1,5–1,8 m/s, Linkskurve +0,75 rad/s, Rechtskurve −0,90 rad/s |
| `/tf` `map → starship`, `/tf_static` Ultraschall → `starship` | | 20 Hz / einmalig | **Ergänzungsknoten.** TF-Kette `map → starship → Sensor` mit `tf2_echo` geprüft, auch für CARLAs Sensor-Frames |
| `vehicle_control_cmd` | `carla_msgs/CarlaEgoVehicleControl` | – | Subscriber von CARLA für `role_name=hero`, Steuerung getestet (siehe „Steuern über ROS 2“) |

QoS der Sensortopics: Reliable, Transient Local.

## Lücken des eingebauten ROS 2 und wie sie geschlossen sind

1. **Ultraschall:** `sensor.other.obstacle` hat keinen ROS-2-Publisher (`ROS2::ProcessDataFromObstacleDetection`
   schreibt nur eine Logzeile). Geschlossen durch den Ergänzungsknoten (`Range`).
2. **Fahrzeuglage:** CARLA veröffentlicht kein `map → Fahrzeug` und keine Odometrie. Geschlossen durch den
   Ergänzungsknoten. Umrechnung wie CARLA (`TransformQuaternion.h`): Position (x, −y, z), Rotation (roll, −pitch, −yaw).
   Drehraten als axialer Vektor (−x, y, −z). Achsen aus Unreal-Matrix statt CARLA-Client-Vektoren (Vorzeichenfehler).
3. **Stabile Namen:** ohne `ros_name` heißt das Fahrzeug `vehicle<ID>`, bei jedem Spawn anders.
   `manual_control_starship.py` setzt `ros_name=starship`. Alle Skripte lösen den Namen wie CARLA auf
   (`ros_vehicle_name` in `starship_sensor_common.py`).
4. **ToF in Metern:** offen. CARLA sendet `bgra8` in Tiefenkodierung, wer `32FC1` braucht, muss umrechnen.
5. **Fahrbefehle über ROS:** funktioniert über `/carla/starship/vehicle_control_cmd` mit dem lokal gebauten
   `carla_msgs` (Abschnitt „Steuern über ROS 2“). Offen: Not-Halt bei ausbleibenden Befehlen.

## Gesamttest (20.10.2026)

Ablauf genau wie unter „Start“: CARLA `-game -RenderOffScreen -quality-level=Low --ros2`, `manual_control_starship.py`,
`starship_ros_sensors.py`, `starship_ros_extras.py`. Ergebnis: **29 Daten-Topics + 13 `camera_info` unter `/carla/starship/`**,
alle 26 Sensoren und die Odometrie mit Sollrate (Kameras 3,8–4,2 Hz, ToF 10 Hz, Radar 20 Hz, Ultraschall 20 Hz, GNSS 5 Hz,
Odometrie 20 Hz). Die IMU lief nur mit 28 Hz, weil der Welttakt unter Volllast auf ca. 28 Bilder/s fällt.
Grafikspeicher stabil bei 11,0 GB. Nach Strg+C sind alle Sensoren entfernt. `starship_ros_sensors.py` überspringt GNSS/IMU,
weil `manual_control_starship.py` sie schon (mit `ros_name`) angebracht hat.

Hinweise aus dem Test:

- `rgb<ID>/image` (ca. 29 Hz) ist die Anzeigekamera von `manual_control`. CARLA veröffentlicht sie mit, weil sie keinen
  `ros_name` hat. Harmlos, kostet aber Bandbreite.
- **Keine Restprozesse stehen lassen:** Ein noch verbundener `manual_control` aus einem früheren Lauf belegt Port 8000
  (Traffic Manager, neuer Start scheitert mit `get_trafficmanager ... std::exception`) und hat eigene Sensoren. Zusammen
  mit allen Starship-Sensoren reichte das für einen Grafikspeicher-Absturz.
- Einmal stürzte CARLA unabhängig von den Sensoren ab: `Assertion failed: Code == CURLM_OK` im Derived-Data-Cache
  von Unreal (`CurlHttpClient.cpp`). Ursache in der Engine, nicht untersucht.

## Grafikspeicher (wichtig)

| Laufumgebung | Grundlast | mit allen 18 Sensoren |
| --- | --- | --- |
| Editor + Play, Town10HD Epic | 10,3 GB | **Absturz** bei der 8. Kamera („Out of memory on Vulkan“), ca. 0,8 GB je Kamera |
| `-game -RenderOffScreen`, Town10HD | 6,0 GB | **11,1 GB, stabil** |

Für den ROS-Betrieb CARLA deshalb eigenständig ohne Editor starten (Befehl oben). `-quality-level=Low` hat die
Grundlast nur von 6,0 auf 5,9 GB gesenkt. Das Log zeigt „Quality Level = Null“ (Standardwert vor dem Einlesen). Ob
die Stufe wirkt, ist offen. Mit `manual_control` und allen Sensoren: 11,0 GB von 16 GB.
