# Sensorik

Sensoranzeigen für `vehicle.starship.bot` (CARLA 0.10), aufgebaut wie die Clevon-Sensorik (`01_Clevon1/Sensorik`).
Die Anzeigen benötigen einen bereits gespawnten Starship-Bot und laufende Simulation (Play).
Kein Skript taktet die Welt oder verändert das Fahrzeug. Jede Anzeige entfernt beim Beenden nur ihre eigenen Sensoren.

| Skript | Sensoren | Anzeige |
|---|---|---|
| `manual_control_starship.py` | GNSS, IMU | Startet CARLAs `manual_control.py` mit Starship-GNSS/-IMU, spawnt und fährt den Bot. Navigationsdaten werden beim Beenden gespeichert. |
| `starship_cameras.py` | 9 RGB-Kameras | 5 Reiter: Front, Stereo vorne, links, rechts, hinten. Nur der aktive Reiter erzeugt Sensoren. |
| `starship_tof.py` | 3 ToF (Tiefenkamera) | 3 Reiter: vorne, links, rechts. Farbiges Tiefenbild bis 5 m, nächster Punkt, gültige Pixel. |
| `starship_radars.py` | 4 Radare | 3 Reiter: Vorne/Hinten, Links/Rechts (Fächer wie Clevon), Rundum-Draufsicht. Alle 4 Radare laufen dauerhaft. |
| `starship_ultrasonic.py` | 8 Ultraschall (Hindernissensor) | 2 Reiter: Draufsicht als Einparkhilfe (rot < 0,5 m, gelb < 1,5 m), Abstandsverlauf der letzten 10 s. |
| `starship_follow_cam.py` | – | Setzt die CARLA-Beobachteransicht laufend hinter den Bot (`--view back/top/side`). Für CARLA mit Fenster; steuert nichts, auch mit ROS-Steuerung nutzbar. |

Konfiguration: je Sensortyp eine JSON-Datei (`starship_cameras.json`, `starship_tof.json`, `starship_radars.json`,
`starship_ultrasonic.json`, `starship_navigation.json`). Die Werte stammen unverändert aus `config/starship_sensors.json`
(26 Sensoren), Begründungen und Quellen stehen in [docs/SENSORIK.md](../docs/SENSORIK.md). Neu festgelegt, weil die
Vorlage keine Messrate hatte: Radar 20 Hz, Ultraschall 20 Hz, ToF 10 Hz, GNSS 5 Hz, IMU 100 Hz (Annahmen).
Gemeinsamer Code: `starship_sensor_common.py` (Laden, Prüfen, Fahrzeugsuche, Spawnen, Reiterleiste).

## Start

Alle Startbefehle (CARLA, Bot spawnen, Anzeigen, Tests): [Haupt-README, Startbefehle](../README.md#startbefehle-carla-010).

```bash
cd ~/Documents/Codex/repos/last_mile-delivery-carla/02_starship_emodell/Sensorik
/home/ieem/CarlaUE5/.venv/bin/python manual_control_starship.py          # Bot spawnen und fahren
/home/ieem/CarlaUE5/.venv/bin/python starship_cameras.py                 # in einem zweiten Terminal
```

Ebenso `starship_tof.py`, `starship_radars.py`, `starship_ultrasonic.py`. Der Pfad zu `manual_control.py` ist
`~/CarlaUE5/PythonAPI/examples/manual_control.py` (änderbar über die Umgebungsvariable `CARLA_MANUAL_CONTROL`).

## Bedienung (alle Anzeigen)

- **1–9**: Reiter direkt; **Tab** / **Pfeil rechts**: weiter; **Shift+Tab** / **Pfeil links**: zurück.
- **S**: Messdaten nach `Messdaten/` speichern (Kameras PNG + `metadata.json`, ToF/Radar `.npz`, Ultraschall/Navigation `.json`).
- **Esc**: Anzeige schließen und eigene Sensoren entfernen.
- `--check` prüft Konfiguration und Abhängigkeiten ohne Server; `--vehicle-id` wählt bei mehreren Bots.
- `starship_cameras.py --preview` halbiert die Auflösung (weniger Grafikspeicher).

## Hinweise

- **Grafikspeicher:** Alle 9 Kameras und 3 ToF gleichzeitig haben den Editor (16 GB VRAM, Town10HD Epic) zum Absturz
  gebracht. Deshalb erzeugen Kamera- und ToF-Anzeige nur die Sensoren des aktiven Reiters. Nicht mehrere Kamera- oder
  ToF-Fenster parallel öffnen.
- Keine parallelen Sensorinstanzen über diese Anzeigen und die spätere ROS-2-Bridge starten.
- Ultraschall: Der CARLA-Hindernissensor meldet nur Treffer. Ohne Meldung seit `hold_s` (0,3 s) gilt die Keule als frei.
- ToF: ideale Tiefe ohne Rauschen und Mehrwegeffekte. Gespeichert wird die ungeclippte Tiefe in Metern.
- Ist-Raten hängen vom Welttakt ab (asynchron ca. 35 Hz in Town10HD), z. B. erreicht die IMU nicht 100 Hz.

## Getestet (20.10.2026, CARLA 0.10, Town10HD, Play im Editor)

Alle Anzeigen automatisch durchlaufen (alle Reiter, Speichern, Esc). Kameras 4,0 Hz Soll/Ist, ToF 10 Hz (Boden ab 0,35 m),
Radar alle 4 mit Treffern, Ultraschall U1–U3 erkennen ein Testfahrzeug in 1,85–2,27 m, GNSS 5,0 Hz, IMU ca. 36 Hz.

## ROS 2

Umgesetzt: CARLAs eingebautes ROS 2 plus Ergänzungsknoten, alle 26 Sensoren und die Fahrzeuglage unter
`/carla/starship/...`. Siehe [ROS2/README.md](ROS2/README.md). Die Abschnitte unten beschreiben die Vorbereitung.

### Vorbereitung ROS 2 (Stand vor der Umsetzung)

Die gemeinsame ROS-2-Schnittstelle (nächster Schritt) soll dieselben JSON-Dateien und `starship_sensor_common.py`
verwenden, wie die Clevon-Bridge:

- `role_name` ist eindeutig und mit `starship_` versehen und kann als Topic-/Frame-Name dienen
  (z. B. `/starship/starship_cam_front/image_raw`, Frame `starship/starship_cam_front`).
- Montage in `mount_m_deg` (CARLA: x vorne, y rechts, z oben), umrechenbar in ROS-TF (y links) wie bei Clevon.
- Datenformate je Typ: Kamera → `sensor_msgs/Image` + `CameraInfo` (`intrinsic_matrix` in `starship_cameras.py`),
  ToF → `sensor_msgs/Image` (32FC1, Meter), Radar → `PointCloud2` (`radar_xyz` in `starship_radars.py`),
  Ultraschall → `sensor_msgs/Range`, GNSS → `NavSatFix`, IMU → `Imu`.
- CARLA 0.10 bietet zusätzlich ein eingebautes ROS 2 (Sensorattribut `ros_name`, Logmeldung „ROS2 enabled“). Ob das die
  eigene Bridge ersetzen kann, ist beim nächsten Schritt zu prüfen.
