# Last-Mile-Delivery-Roboter in CARLA

F&E-Projektarbeit an der Hochschule Karlsruhe, Institut für Energieeffiziente Mobilität (SS 2026):
zwei Lieferroboter als fahrbare CARLA-Fahrzeuge mit Sensorik und ROS-2-Anbindung.
Autoren: Anvin Le, Ricardo Rauch.

| Ordner | Fahrzeug | CARLA-ID |
| --- | --- | --- |
| [`01_Clevon1/`](01_Clevon1/README.md) | Clevon 1, Straßenroboter (4 Räder, 6-kW-Antrieb, Licht, Ladeklappe) | `vehicle.clevon.1` |
| [`02_starship_emodell/`](02_starship_emodell/README.md) | Starship-Gehwegroboter im HKA-Design (6 Räder, Bordstein-Assistent) | `vehicle.starship.bot` |
| [`00_Abgabe/`](00_Abgabe/) | Projektbericht (PDF, V3-7) und Video-Tutorial Starship mit ROS 2 | – |

## Plattform

CARLA 0.10.0 (Source-Build mit `ENABLE_ROS2=ON`), Unreal Engine 5.5.4, ROS 2 Jazzy, Ubuntu 24.04.
Die Clevon-Ordner enthalten zusätzlich den früheren Stand für CARLA 0.9.16 / UE 4.26 mit eigener ROS-2-Bridge.

## Klonen

```bash
mkdir -p ~/carla_projekt && cd ~/carla_projekt
git clone -b clevon-starship-carla-0.10 https://git.ieem-ka.de/rari1014/last_mile-delivery-carla.git
cd last_mile-delivery-carla
git lfs pull
```

## Anleitung

**[Anleitung_Bots_starten.pdf](Anleitung_Bots_starten.pdf)**: vollständige Schritt-für-Schritt-Anleitung für
beide Fahrzeuge, auch für Einsteiger. Sie deckt alle Funktionen ab: Terminal-Grundlagen, Klonen, einmalige
Vorbereitung (ROS 2, `carla_msgs`), Import des Starship in CARLA, CARLA-Startvarianten, Fahren, alle
Sensoranzeigen, ROS 2 je Fahrzeug (Sensoren, Odometrie, Steuern), Slalom-Demo, Fehlerbehebung und Spickzettel.
Alle Befehle zum Kopieren; die Pfade gehen vom Klon unter `~/carla_projekt/last_mile-delivery-carla` aus.

Der folgende Schnellstart ist eine Kurzfassung davon.

## Schnellstart

```bash
# CARLA mit Fenster und ROS 2
~/UnrealEngine5_carla/Engine/Binaries/Linux/UnrealEditor ~/CarlaUE5/Unreal/CarlaUnreal/CarlaUnreal.uproject \
  /Game/Carla/Maps/Town10HD_Opt -game -windowed -ResX=1280 -ResY=720 -quality-level=Low -prefernvidia --ros2

# selbst fahren
~/CarlaUE5/.venv/bin/python 02_starship_emodell/Sensorik/manual_control_starship.py
~/CarlaUE5/.venv/bin/python 01_Clevon1/Sensorik/manual_control_clevon.py

# ROS-2-Demo: Slalom über ROS 2, zwei Fahrzeugkameras aus ROS 2 angezeigt
# (auch für den Clevon im Starship-Ordner starten; build.sh nur beim ersten Mal)
cd 02_starship_emodell/Sensorik/ROS2 && bash ros_ws/build.sh && source ros_ws/setup.bash
~/CarlaUE5/.venv/bin/python ros2_slalom_demo.py                   # Starship
~/CarlaUE5/.venv/bin/python ros2_slalom_demo.py --vehicle clevon  # Clevon
```

Details:

- Starship: [`02_starship_emodell/README.md`](02_starship_emodell/README.md),
  Sensorik [`Sensorik/README.md`](02_starship_emodell/Sensorik/README.md),
  ROS 2 [`Sensorik/ROS2/README.md`](02_starship_emodell/Sensorik/ROS2/README.md),
  Umsetzungsprotokoll [`docs/IMPLEMENTIERUNG_CARLA_0.10.md`](02_starship_emodell/docs/IMPLEMENTIERUNG_CARLA_0.10.md)
- Clevon: [`01_Clevon1/README.md`](01_Clevon1/README.md),
  Sensorik [`Sensorik/README.md`](01_Clevon1/Sensorik/README.md),
  ROS 2 (CARLA 0.10) [`Sensorik/ROS2/native/README.md`](01_Clevon1/Sensorik/ROS2/native/README.md)
