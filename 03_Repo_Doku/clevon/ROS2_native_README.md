# Clevon über CARLAs eingebautes ROS 2

Alternative zur bestehenden Bridge `../clevon_ros_bridge.py`: Die Sensordaten sendet CARLA 0.10 selbst
(Startparameter `--ros2`). Hier liegen nur das Anbringen der Sensoren, die fehlende Fahrzeuglage und `carla_msgs` für die
Steuerung. Gleiches Vorgehen wie beim Starship (`02_starship_emodell/Sensorik/ROS2`).
**Nicht gleichzeitig mit der Bridge betreiben** (doppelte Sensoren, doppelte Last).

Getestet am 20.10.2026 unter Ubuntu 24.04, ROS 2 Jazzy, CARLA 0.10 (lokaler Source-Build mit `ENABLE_ROS2=ON`).
Der Windows-/WSL-Betrieb der bestehenden Bridge ist davon nicht berührt und wurde hier nicht getestet.

| Datei | Zweck |
| --- | --- |
| `clevon_ros_sensors.py` | Bringt die Clevon-Sensoren aus den vorhandenen JSONs mit `ros_name = role_name` an (Gruppen `nav radar lidar cameras`; `--low-load`, `--preview` für die Kameras). Gleiche Konfigurationsprüfung wie Anzeigen und Bridge (`load_*_config` aus `clevon_sensor_common.py`). |
| `clevon_ros_extras.py` | rclpy-Knoten: `nav_msgs/Odometry` auf `/carla/clevon/odometry` und TF `map → clevon` (CARLA veröffentlicht keine Fahrzeuglage). |
| `spawn_clevon.py` | Clevon ohne Fahrclient spawnen (`role_name=hero`, `ros_name=clevon`), zum Steuern über ROS. |
| `ros_ws/` | Minimales `carla_msgs` (nur `CarlaEgoVehicleControl`, Felder wie in CARLA). `build.sh` baut ohne colcon/sudo, `setup.bash` lädt ROS + `carla_msgs`. |
| `../../manual_control_clevon.py --ros-name clevon` | Fahrclient setzt jetzt `ros_name` am Fahrzeug (Standard `clevon`) und an GNSS/IMU. |
| `../../clevon_follow_cam.py` | Beobachteransicht folgt dem Clevon (CARLA mit Fenster, `--view back/top/side`). |

## Start (je ein Terminal)

```bash
# 0. einmalig: ROS 2 Jazzy (siehe Starship: install_ros_jazzy.sh) und carla_msgs bauen
~/Documents/Codex/repos/last_mile-delivery-carla/01_Clevon1/Sensorik/ROS2/native/ros_ws/build.sh

# 1. CARLA mit ROS 2, mit Fenster (zum Zusehen) oder ohne (-RenderOffScreen statt -windowed ...)
~/UnrealEngine5_carla/Engine/Binaries/Linux/UnrealEditor ~/CarlaUE5/Unreal/CarlaUnreal/CarlaUnreal.uproject /Game/Carla/Maps/Town10HD_Opt -game -windowed -ResX=1280 -ResY=720 -quality-level=Low -prefernvidia --ros2

# 2a. selbst fahren: Fahrclient (spawnt den Clevon mit ros_name=clevon, GNSS/IMU inklusive)
cd ~/Documents/Codex/repos/last_mile-delivery-carla/01_Clevon1/Sensorik && /home/ieem/CarlaUE5/.venv/bin/python manual_control_clevon.py
# 2b. ODER über ROS steuern: Clevon ohne Fahrclient spawnen
/home/ieem/CarlaUE5/.venv/bin/python ~/Documents/Codex/repos/last_mile-delivery-carla/01_Clevon1/Sensorik/ROS2/native/spawn_clevon.py

# 3. Sensoren für ROS (Kameras schonend: --preview)
/home/ieem/CarlaUE5/.venv/bin/python ~/Documents/Codex/repos/last_mile-delivery-carla/01_Clevon1/Sensorik/ROS2/native/clevon_ros_sensors.py --preview

# 4. Fahrzeuglage
source ~/Documents/Codex/repos/last_mile-delivery-carla/01_Clevon1/Sensorik/ROS2/native/ros_ws/setup.bash && /home/ieem/CarlaUE5/.venv/bin/python ~/Documents/Codex/repos/last_mile-delivery-carla/01_Clevon1/Sensorik/ROS2/native/clevon_ros_extras.py

# 5. zusehen (nur mit Fenster)
/home/ieem/CarlaUE5/.venv/bin/python ~/Documents/Codex/repos/last_mile-delivery-carla/01_Clevon1/Sensorik/clevon_follow_cam.py
```

## Steuern über ROS 2

Topic **`/carla/clevon/vehicle_control_cmd`**, Typ **`carla_msgs/msg/CarlaEgoVehicleControl`** (QoS best effort).
Nur ohne gleichzeitigen Fahrclient: `manual_control` überschreibt die Befehle in jedem Takt.

```bash
source ~/Documents/Codex/repos/last_mile-delivery-carla/01_Clevon1/Sensorik/ROS2/native/ros_ws/setup.bash
ros2 topic pub -r 10 /carla/clevon/vehicle_control_cmd carla_msgs/msg/CarlaEgoVehicleControl '{throttle: 0.5}'
ros2 topic pub --once -w 1 /carla/clevon/vehicle_control_cmd carla_msgs/msg/CarlaEgoVehicleControl '{brake: 1.0}'
```

- `steer` in CARLA-Konvention: positiv = rechts.
- **Einzelbefehle immer mit `--once -w 1`** (wartet, bis CARLAs Subscriber verbunden ist). Ein neuer Publisher braucht
  ca. 1–3 s zur DDS-Discovery. Was vorher gesendet wird, geht bei best effort verloren. Gemessen: Ein 3-s-Bremsbefehl
  kam gar nicht an, `--once -w 1` kam 6 von 6 Mal an.
- **Kein Not-Halt:** Ohne neue Befehle fährt das Fahrzeug mit dem letzten weiter. Anders als die Bridge
  (0,5-s-Watchdog) bremst hier nichts automatisch.

## Gemessen (CARLA mit Fenster, Town10HD, `-quality-level=Low`)

| Topic (`/carla/clevon/…`) | Typ | Rate | Bemerkung |
| --- | --- | --- | --- |
| `clevon_camera_*/image` (6), `…/camera_info` | `Image` (`bgra8`), `CameraInfo` | siehe unten | Soll laut JSON 1920×1080, 30 Hz |
| `clevon_front_radar_short/point_cloud`, `clevon_front_radar_mid/point_cloud` | `PointCloud2` | 18,0 / 18,6 Hz (Soll 18,2) | |
| `clevon_roof_lidar_preview/point_cloud` | `PointCloud2` | 39 Hz | ein Teilpaket je Simulationsschritt (`sensor_tick` 0), wie bei CARLA üblich |
| `clevon_gnss` | `NavSatFix` | 5,0 Hz | |
| `clevon_imu` | `Imu` | Welttakt (38 Hz; Soll 100) | |
| `odometry` | `Odometry` | 20 Hz | Ergänzungsknoten, Ground Truth |
| `vehicle_control_cmd` | `carla_msgs/CarlaEgoVehicleControl` | – | Steuerung getestet: Gas 0,5 → 14,5 km/h, Rechtskurve, Bremsen |

**Kameras und Simulationstakt.** Die Kameralast bremst CARLA selbst. Die Kameras können nie schneller liefern als
der Welttakt:

| Kameraprofil | Grafikspeicher | Welttakt = max. Kamerarate |
| --- | --- | --- |
| `--preview` (640×360, 10 Hz) | 9,9 GB (alle 11 Sensoren) | ≈ 38 Bilder/s, Kameras 10 Hz |
| `--low-load` (1280×720, 20 Hz) | 10,4 GB | 11,4 Bilder/s, Kameras 11,3 Hz |
| JSON (1920×1080, 30 Hz) | 12,0 GB | 6,8 Bilder/s, Kameras ≈ 7 Hz |

Für Echtzeit mit hoher Kamerarate reicht die RTX 4070 Ti SUPER (16 GB) bei 6 Full-HD-Kameras in Town10HD nicht.
Für vollständige, gleichmäßige Datensätze CARLA im synchronen Modus betreiben (dann läuft die Simulation langsamer
als Echtzeit, aber jeder Schritt hat alle Bilder). Nicht getestet.

## Hinweise

- CARLA-Startparameter `--ros2` mit zwei Bindestrichen, sonst bleibt ROS 2 aus (das Log meldet trotzdem „ROS2 enabled...“).
- `ros2 topic list --no-daemon` findet den rclpy-Knoten nicht immer sofort. `ros2 topic hz /carla/clevon/odometry` zeigt ihn.
- Signal-Handler erst nach dem CARLA-Verbindungsaufbau setzen (der Client überschreibt SIGINT). Strg+C und SIGTERM
  entfernen die Sensoren sauber.
