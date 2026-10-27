# Clevon1 in CARLA

Fahrzeugmodell für CARLA 0.9.16 mit Fahrphysik, Lichtfunktionen, Ladeklappe und Sensorik.
Eine eigene Python-Bridge stellt Sensordaten und Fahrzeugbefehle in ROS 2 bereit.

> **Aktueller Stand: CARLA 0.10.0 / Unreal Engine 5.5.4.** Fahrzeug-Assets: [`Unreal/UE5_CARLA-0.10_Chaos`](Unreal/README.md).
> ROS 2 läuft dort über die **native** CARLA-Anbindung: [`Sensorik/ROS2/native`](Sensorik/ROS2/native/README.md).
> Die unten beschriebene Python-Bridge gehört zum früheren Stand (CARLA 0.9.16 / UE 4) und wird nicht mehr verwendet.

## Voraussetzungen und Installation

- CARLA 0.9.16 mit Unreal Engine 4.26.2; geprüft mit Server-Build 294096e.
- Windows-Python 3.10 mit CARLA-Python-API, NumPy und Pygame.
- Für ROS: WSL 2, Ubuntu 22.04 und ROS 2 Humble.
- Git LFS für die binären Modell- und Bilddateien: nach dem Klonen im Repository `git lfs pull` ausführen.

Zuerst das Fahrzeug nach [Unreal/README.md](Unreal/README.md) integrieren.
Die vollständige Übertragung auf eine zweite CARLA-Installation ist noch nicht bestätigt.
Alle folgenden PowerShell-Befehle gelten im Ordner **01_Clevon1** mit aktivierter Python-Umgebung:

```powershell
python -m pip install -r requirements-windows.txt
```

Die Fahrclients verwenden das originale CARLA-Beispiel `manual_control.py`.
Bei abweichendem Installationspfad vor dem Start setzen:

```powershell
$env:CARLA_MANUAL_CONTROL = 'D:/CARLA/PythonAPI/examples/manual_control.py'
```

Ohne diese Variable gilt `C:/CARLA/carla/PythonAPI/examples/manual_control.py`.

## Start ohne ROS

1. Unreal-Projekt öffnen und **Play** starten.
2. Fahrzeug starten:

```powershell
python .\Sensorik\manual_control_clevon.py
```

3. Gewünschte Sensoranzeigen jeweils in einem weiteren Terminal starten:

```powershell
python .\Sensorik\clevon_cameras.py
python .\Sensorik\clevon_radars.py
python .\Sensorik\clevon_lidar.py
```

GNSS und IMU laufen im Fahrclient. Im Fahrfenster steuern, zusätzliche Sensorfenster mit Esc schließen.
Nach einem Fahrzeug-Respawn die Sensoranzeigen neu starten. Optionen: `python <Skript> --help`.
Die vier JSON-Dateien unter Sensorik enthalten die Sensorparameter.

## Start mit ROS 2

| | Native ROS-2-Anbindung (**aktuell**) | Eigene Python-API-Bridge (**früher**, nicht mehr verwendet) |
|---|---|---|
| Ordner | [`Sensorik/ROS2/native/`](Sensorik/ROS2/native/README.md) | Dateien direkt in [`Sensorik/ROS2/`](Sensorik/ROS2/README.md) |
| Plattform | CARLA 0.10.0, UE 5.5.4, ROS 2 Jazzy, Ubuntu 24.04 | CARLA 0.9.16, UE 4.26.2, ROS 2 Humble, WSL 2 |
| Wer sendet die Sensordaten? | CARLA selbst (`--ros2`, DDS) | Bridge liest über die CARLA-Python-API und veröffentlicht |
| Skripte | `clevon_ros_sensors.py`, `clevon_ros_extras.py`, `spawn_clevon.py` | `clevon_ros_bridge.py` und Hilfsskripte |
| Steuerung | `/carla/clevon/vehicle_control_cmd` (`carla_msgs`) | `/clevon/control/command` (`clevon_msgs`), Licht/Klappe über Services |

Die native Anbindung ist in [`Sensorik/ROS2/native/README.md`](Sensorik/ROS2/native/README.md) beschrieben.
Der folgende Abschnitt betrifft nur die frühere Bridge.

### Frühere Bridge (CARLA 0.9.16 / UE 4)

Einmalig in Ubuntu im Ordner `Sensorik/ROS2` (Bridge-Dateien, nicht `native/`):

```bash
sudo bash install_ros_humble.sh
sudo bash build_interface.sh
```

Der Installer benötigt Internet und verändert die Ubuntu-Paketinstallation.
Danach Unreal **Play** starten. Es gibt zwei alternative Betriebsarten:

| Betriebsart | PowerShell-Befehle im Ordner 01_Clevon1 |
|---|---|
| Tastaturfahrt mit ROS-Sensordaten | `python .\Sensorik\ROS2\manual_control_ros.py`, danach in zweitem Terminal `.\Sensorik\ROS2\Start-ClevonROS2.ps1 -Lidar` |
| Fahrzeug über ROS steuern | `.\Sensorik\ROS2\Start-ClevonROS2.ps1 -Spawn -AllowControl -Lidar` |

Im zweiten Modus erzeugt die Bridge das Fahrzeug ohne separates Fahrfenster.
Die Ansicht erfolgt im Unreal-Viewport. `-Lidar` ist optional; `-CarlaHost IP` setzt die Serveradresse explizit.
Die separaten Sensoranzeigen und den normalen Fahrclient nicht zusätzlich starten.

In einem Ubuntu-Terminal:

```bash
source /opt/ros/humble/setup.bash
source /opt/clevon_ros_ws/install/setup.bash
ros2 topic list
```

Fahrfreigabe sowie Licht-/Klappenbefehle: [Bridge-Anleitung](Sensorik/ROS2/README.md).
Zuerst die Bridge mit Strg+C beenden, danach den manuellen Fahrclient schließen.

## Inhalt

| Ordner | Inhalt |
|---|---|
| [Unreal](Unreal/README.md) | Fahrzeug-Assets und Integrationsschritte |
| [Modell](Modell/README.md) | Blender-Quellen und korrigierte FBX-Dateien |
| [Sensorik](Sensorik/README.md) | Fahrclient, Sensoranzeigen und Konfigurationen |
| [Sensorik/ROS2](Sensorik/ROS2/README.md) | ROS 2: `native/` (aktuell, CARLA 0.10) und Python-API-Bridge direkt im Ordner (früher, CARLA 0.9.16) |
| [Fahrphysik](Fahrphysik/README.md) | Aktuelle Parameter und Prüfwerkzeuge |
| [Dokumentation](Dokumentation/README.md) | Modellbeschreibung, Zeichnungen und Referenzen |
| [Validierung](Validierung/README.md) | Testergebnisse und deren Aussagegrenzen |

## Modellgrenzen

450 kg Betriebsmasse und mehrere Fahrwerksparameter sind Modellannahmen. Der 6-kW-Antrieb ist eine
aggregierte Ersatzkennlinie. Die Klappe schließt unmittelbar; der LiDAR-Ursprung ist gegenüber dem
sichtbaren Gehäuse angehoben. Sechs Full-HD-Kameras erreichen über die Bridge nicht durchgehend
ihre Sollrate. Die Simulation ist asynchron; eine vollständige OEM- oder Echtzeitvalidierung liegt nicht vor.
