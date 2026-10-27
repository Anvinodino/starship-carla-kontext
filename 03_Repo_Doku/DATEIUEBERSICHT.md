# Dateiübersicht: Sinn und Funktion aller Ordner und Dateien

Stand 22.10.2026, Branch `merge/clevon-starship-ein-repo`. Pfade relativ zum Repository.
„Ausführen?“ sagt, ob eine Datei beim normalen Betrieb gestartet wird (**ja**), nur bei Bedarf (Werkzeug, Test,
Neuaufbau) oder gar nicht (Daten, Doku, Asset).

## Wurzel

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `README.md` | Einstieg: beide Fahrzeuge, Plattformen, gemeinsame Ordnerstruktur, Schnellstart. | nein |
| `DATEIUEBERSICHT.md` | Diese Datei. | nein |
| `01_Clevon1/` | Alles zum Clevon 1 (Straßenroboter). | – |
| `02_starship_emodell/` | Alles zum Starship-Gehwegroboter im HKA-Design. | – |

---

# 01_Clevon1: Clevon 1

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `README.md` | Installation und Start (Stand CARLA 0.9.16 / Windows / WSL), Inhaltstabelle, Modellgrenzen. Für CARLA 0.10 siehe `Sensorik/ROS2/native/`. | nein |
| `HERKUNFT.json` | Herkunftsnachweis: welche Dateien aus welchem Arbeitsstand stammen, mit SHA-256-Prüfsummen. | nein |
| `requirements-windows.txt` | Python-Pakete für die Windows-Clients (`carla==0.9.16`, `numpy<2`, `pygame`). `pip install -r`. | einmalig |
| `.gitattributes` | Legt fest, dass `.uasset`, `.blend`, `.fbx`, Bilder und PDFs über Git LFS laufen. | nein |
| `.gitignore` | Schließt Python-Caches, venvs, Blender-Backups, Messdaten-Ordner und Unreal-Zwischenstände aus. | nein |

## 01_Clevon1/Modell: 3D-Modell

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `README.md` | Tabelle der Modelldateien und ihrer Verwendung. | nein |
| `Blender/Clevon1_Exportbasis.blend` | Blender-Stand, aus dem die FBX-Dateien exportiert wurden (Karosserie, Rig mit `Vehicle_Base` + 4 Radknochen). | nein |
| `Blender/Clevon1_Funktionen_Rueckfahrlicht.blend` | Korrigierter Arbeitsstand mit weißer Rückfahrlichtfläche. | nein |
| `Blender/ieem_ka_logo.jfif` | IEEM-Logo als Textur für die Beschriftung. | nein |
| `FBX/SK_Clevon1.fbx` | Skeletal Mesh: Karosserie mit Skelett, Grundlage des Fahrzeugs in Unreal. | Import |
| `FBX/SMC_Clevon1.fbx`, `FBX/SM_sc_Clevon1.fbx` | Weitere Static Meshes des Exports; laut Bericht dient ein zusätzlicher Kollisionsproxy den Sensor- und Raycast-Abfragen. Genaue Zuordnung ist im Repo nicht dokumentiert (bei Ricardo erfragen). | Import |
| `FBX/SM_Clevon1_BrakeCaliper_{FL,FR,RL,RR}.fbx` | Vier Bremssättel als eigene Meshes; folgen Federung und Lenkung, aber nicht der Raddrehung. | Import |
| `FBX/SM_Clevon1_CargoDoor.fbx` | Ladeklappe der CargoBox (beweglich über Constraint). | Import |
| `FBX/SM_Clevon1_Glass.fbx` | Glasflächen als eigenes Mesh (eigene Materialien). | Import |
| `FBX/SM_Clevon1_Lights.fbx` | Leuchtflächen mit getrennten Materialslots je Lichtfunktion. | Import |

## 01_Clevon1/Unreal: fertige Unreal-Assets

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `README.md` | Schritte zur Übernahme in ein CARLA-Projekt (Kopieren, Registrieren, `NearClipPlane=1.0`). | nein |
| `Content/Carla/Blueprints/Vehicles/Clevon1/BP_Clevon1.uasset` | Fahrzeug-Blueprint: verbindet Mesh, Physik, Räder; Licht-, Blinker-, Bremssattel- und Klappenlogik. | Asset |
| `…/Clevon1/BP_Clevon1_{FLW,FRW,RLW,RRW}.uasset` | Vier Rad-Blueprints: Radius, Lenkwinkel, Bremsmomente, Federung. | Asset |
| `Content/Carla/Static/Car/4Wheeled/Clevon1_CM/SK_Clevon1*.uasset` | Skeletal Mesh, Skelett und Physics Asset (Kollisionskörper). | Asset |
| `…/Clevon1_CM/AnimBP_Clevon1_CM.uasset` | Animations-Blueprint: überträgt Lenkung, Raddrehung, Federweg auf die Radknochen. | Asset |
| `…/Clevon1_CM/SM_*`, `SMC_*` | Importierte Bauteile: Bremssättel, Ladeklappe, Glas, Leuchten, Kollisionsproxy. | Asset |
| `…/Clevon1_CM/Materials_Blender/` | 41 Materialien und Texturen (Lack, Glas, Leuchten, Reifen, Logo). | Asset |

## 01_Clevon1/Fahrphysik

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `README.md` | Verweis auf die gültigen Werte, Kurzbeschreibung der Werkzeuge. | nein |
| `Bestandsaufnahme_20261008.md` | Endgültige Physikwerte (Masse 450 kg, Schwerpunkt, Räder, Federung, 6-kW-Kennlinie) mit Begründung. | nein |
| `physik_istwerte.py` | Liest die Physikwerte eines laufenden Clevon nur lesend aus und speichert sie als JSON-Nachweis. | Werkzeug |
| `test_elektroantrieb.py` | Berechnet das 6-kW-Antriebsprofil; `--apply` setzt es vorübergehend, `--restore` stellt zurück. | Werkzeug |

## 01_Clevon1/Sensorik: Sensoranzeigen und Profile

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `README.md` | Tabelle der Anzeigen und Bedienung. | nein |
| `clevon_cameras.json` | Profil der 6 RGB-Kameras: Montage, Auflösung, Sichtfeld, Rate. | nein |
| `clevon_radars.json` | Profil der 2 Front-Radare (kurz 30 m, mittel 80 m). | nein |
| `clevon_navigation.json` | Profil von GNSS und IMU. | nein |
| `clevon_sensors.json` | Profil des optionalen LiDAR (64 Kanäle) und Fahrzeugfilter `vehicle.clevon.1`. | nein |
| `clevon_sensor_common.py` | Gemeinsames Modul: Profile laden und prüfen, Fahrzeug finden, Sensoren anbringen/entfernen, ROS-Fahrzeugname. Von Anzeigen, Bridge und ROS-Skripten genutzt. | Modul |
| `manual_control_clevon.py` | Startet CARLAs `manual_control.py` mit Clevon-GNSS/-IMU; spawnt und fährt den Clevon, setzt `ros_name=clevon`. | **ja** |
| `clevon_cameras.py` | Kameraanzeige mit Reitern; nur das aktive Kamerapaar wird erzeugt (Grafikspeicher). | ja |
| `clevon_radars.py` | Anzeige der Messpunkte beider Radare. | ja |
| `clevon_lidar.py` | Bringt den optionalen LiDAR an und zeigt die Punktwolke. | ja |
| `clevon_follow_cam.py` | Setzt die CARLA-Beobachteransicht hinter den Clevon; erzeugt keine Sensoren. | optional |

## 01_Clevon1/Sensorik/ROS2: Bridge (CARLA 0.9.16, Humble, WSL)

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `README.md` | Befehle, Freigabe der Steuerung, Testskripte. | nein |
| `SCHNITTSTELLE.md` | Topics, Nachrichtentypen, Koordinaten, Zeitstempel der Bridge. | nein |
| `LICHT_UND_KLAPPE.md` | Beispielaufrufe für Licht, Blinker, Klappe über ROS. | nein |
| `clevon_ros_bridge.py` | Die Bridge: liest Sensordaten über die CARLA-Python-API, veröffentlicht sie in ROS 2, nimmt Fahrbefehle mit Watchdog an. | ja (Bridge-Modus) |
| `bridge_math.py` | Reine Rechenfunktionen: CARLA→ROS-Koordinaten, Quaternionen, Kameramatrix, Radarpunkte, Befehlsprüfung. | Modul |
| `vehicle_functions.py` | Licht- und Klappenbefehle für die Bridge (testbar ohne CARLA). | Modul |
| `manual_control_ros.py` | Windows-Fahrclient zum Fahren per Tastatur, während die Bridge die Sensoren stellt. | ja |
| `Start-ClevonROS2.ps1` | Startet die Bridge aus PowerShell in der WSL-Distribution (`-Spawn`, `-AllowControl`, `-Lidar`). | ja |
| `run_bridge.sh` | Lädt ROS-Umgebung und startet die Bridge mit ihrer Python-Umgebung (wird vom PS-Skript aufgerufen). | indirekt |
| `install_ros_humble.sh` | Installiert ROS 2 Humble und Python-Abhängigkeiten in Ubuntu 22.04. | einmalig |
| `build_interface.sh` | Baut das Nachrichtenpaket `clevon_msgs` nach `/opt/clevon_ros_ws`. | einmalig |
| `clevon_msgs/` | Eigenes ROS-Paket: `msg/VehicleControl.msg` (normierte Fahrbefehle), `srv/SetLights.srv` (Lichtzustand), `CMakeLists.txt`, `package.xml`, `README.md`. | Build |
| `probe_carla_connection.py` | Prüft lesend, ob CARLA aus WSL erreichbar ist. | Test |
| `verify_ros_environment.py` | Prüft die ROS-Kommunikation mit Test-Publisher und -Empfänger, ohne CARLA. | Test |
| `test_bridge_math.py` | 5 Unit-Tests für `bridge_math.py`. | Test |
| `test_vehicle_functions.py` | 6 Unit-Tests für die Licht-/Klappenlogik. | Test |
| `validate_interface.py` | Live-Test: Testfahrzeug, Sensoren, CameraInfo, TF, Steuerfreigabe ohne Anfahren. Ergebnis → `Validierung/interface_validation.json`. | Test |
| `validate_moving_interface.py` | Live-Test: Anfahren, Lenken, Bremsen über ROS. Ergebnis → `Validierung/ros_fahrtest.json`. | Test |
| `validate_vehicle_functions.py` | Live-Test für Licht und Klappe. Ergebnis → `Validierung/vehicle_functions_validation.json`. | Test |

## 01_Clevon1/Sensorik/ROS2/native: eingebautes ROS 2 (CARLA 0.10, Jazzy, Linux)

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `README.md` | Startreihenfolge und Dateiliste. | nein |
| `clevon_ros_sensors.py` | Bringt alle Clevon-Sensoren mit `ros_name` an; CARLA veröffentlicht sie dann selbst. | **ja** |
| `clevon_ros_extras.py` | rclpy-Knoten: Odometrie und TF `map → clevon` (fehlt im eingebauten ROS 2). | **ja** |
| `spawn_clevon.py` | Spawnt den Clevon ohne Fahrclient (`hero`, `ros_name=clevon`) zum Steuern über ROS. | bei ROS-Steuerung |
| `ros_ws/` | Minimales `carla_msgs` (nur `CarlaEgoVehicleControl`); `build.sh` baut ohne colcon, `setup.bash` lädt ROS + Paket. | einmalig bauen |

## 01_Clevon1/Dokumentation

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `README.md` | Inhaltsverzeichnis des Ordners. | nein |
| `Modellbeschreibung.md` | Aufbau, Rig, Export und Integration in Textform. | nein |
| `Sensorreferenzen.md` | Sensorparameter mit Referenzprodukten und Quellen. | nein |
| `Sensorpositionen_CARLA.csv` | Montagekoordinaten aller Sensoren (x, y, z, Roll, Pitch, Yaw). | nein |
| `Massvergleich_und_Sensoranordnung.pdf` | Maßzeichnungen und Herstellervergleich (Anhang A des Berichts). | nein |
| `Abbildungen/` | 14 Bilder: Bremssättel, Materialien, Licht-Blueprints, LiDAR-Test, Blender-Renders, Rig, offene Klappe. | nein |
| `Quellen_und_Entwicklung/` | Herstellerunterlagen: Produktblätter Clevon 1 und CargoBox, DHL-Fallstudie. | nein |
| `Zusaetzliche_Uploads/` | Tabelle zu Last-Mile-Fahrzeugen und öffentlicher Infrastruktur. | nein |

## 01_Clevon1/Validierung

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `README.md` | Was jeder Nachweis aussagt und was nicht. | nein |
| `interface_validation.json` | Ergebnis der Schnittstellenprüfung (Sensoren, TF, Steuerfreigabe, Watchdog). | nein |
| `ros_fahrtest.json` | Ergebnis des ROS-Fahrtests vom 09.10.2026. | nein |
| `vehicle_functions_validation.json` | Ergebnis der Licht-/Klappenprüfung. | nein |
| `physik_20261009T093026_973983Z.json` | Physik-Snapshot mit 6-kW-Profil nach Respawn. | nein |

---

# 02_starship_emodell: Starship-Gehwegroboter (HKA-Design)

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `README.md` | Einstieg: Startbefehle CARLA 0.10 (Fahren, Sensoren, ROS 2, Tests, Neuaufbau) und UE4-Schnellstart, Stolperfallen. | nein |
| `.gitattributes` | Markiert `.uasset`, `.fbx`, `.blend`, Bilder als binär; Zeilenenden LF für `.py`/`.sh`. | nein |
| `.gitignore` | Schließt Blender-Backups, Logs, Testbilder, venvs und Messdaten aus. | nein |

## 02_starship_emodell/models: 3D-Modell

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `starship_hka_v5.blend` | **Aktuelles Modell** in Blender: eigenes HKA-Design (anthrazit, rotes „HKA“ seitlich, Antenne mit rotem Licht), 7 Knochen (`VehicleBase` + 6 Räder), alle Knochen in +Y, Roll 0. | nein |
| `starship_hka_v5.fbx` | Export davon in cm ohne Skalierung. Quelle für den Import in UE4 und UE5. | Import |
| `starship_carla_cm_v4.blend`, `starship_carla_v4.fbx` | Vorgängerstand mit gleichem Skelett im ursprünglichen Starship-Aussehen (Fahne). Nur Referenz. | nein |

## 02_starship_emodell/unreal

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `ue4.26_vehicle3/BP_Starship.uasset` | UE4-Fahrzeug-Blueprint (PhysX, `WheeledVehicleMovementComponent4W`, Vorderradantrieb). | Asset (UE4) |
| `ue4.26_vehicle3/BP_Starship_{FLW,FRW,RLW,RRW}.uasset` | Vier UE4-Rad-Blueprints mit `CommonTireConfig`. | Asset (UE4) |
| `ue4.26_vehicle3/SK_Starship_Mesh_HKA.uasset`, `SK_Starship_v4_SM_Starship_Skeleton.uasset`, `PA_Starship_v4.uasset` | Mesh, Skelett, Physics Asset (eine Box 55 × 45 × 33 cm). | Asset (UE4) |
| `ue4.26_vehicle3/AnimBP_Starship_v4.uasset` | UE4-AnimBP: Wheel Handler + 2 × Copy Bone (Mittelräder drehen nur optisch mit). | Asset (UE4) |
| `ue4.26_vehicle3/{dark_gray,black,hka_body,hka_red,hka_lid}.uasset` | Materialien des HKA-Designs. | Asset (UE4) |
| `ue5_carla_plugin/Source/Carla/Vehicle/StarshipCurbAssistComponent.{h,cpp}` | C++-Komponente **Bordstein-Assistent** für CARLA 0.10: erkennt Bordsteine, hebt die Räder seitengetrennt mit geregelter Kraft an (Nachbildung des Starship-Kipphebels). Wird ins CARLA-Plugin kopiert und mitgebaut. | Build |

Hinweis: Für CARLA 0.10 liegen **keine fertigen `.uasset`** im Repo. Die UE5-Assets werden mit den Skripten in
`scripts/unreal_editor/ue5/` im Editor erzeugt.

## 02_starship_emodell/config

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `starship_values.json` | Alle Fahrzeugwerte des UE4-Stands, direkt aus dem Editor ausgelesen. Die UE5-Skripte übernehmen daraus Geometrie, Masse, Bremsen usw. | nein |
| `starship_sensors.json` | Ursprüngliches Gesamtprofil aller 26 Sensoren (UE4-Stand), Quelle der Profile in `Sensorik/`. | nein |
| `ue5/Curve_Starship_Torque.csv` | Drehmomentkurve für Chaos (flach 1,0 bis 1000 U/min, ohne Kopfzeile). | Import |

## 02_starship_emodell/scripts/carla: Python-Clients

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `starship_drive.py` | Schlanker Fahrclient: links Fahrersicht, rechts Beobachter (V wechselt), WASD, Q rückwärts, Backspace neu spawnen. | ja |
| `starship_ue5_check.py` | Abnahmetest CARLA 0.10: Blueprint-Attribute, Physik im Spiel, Ruhelage, Höchstgeschwindigkeit, Bremsen, Lenken, Rückwärts, Handbremse. | Test |
| `starship_curb_test.py` | Bordsteintest an 3 festen Stellen in Town10HD (hoch/runter): Erfolg, Zeit, Nick-/Rollwinkel, Stoß. `--label` speichert nach `docs/messungen/`. | Test |
| `starship_curb_assist.py` | Bordstein-Assistent als Python-Client (Vorläufer der C++-Komponente). Nur für CARLA ohne neu gebautes Plugin; nicht zusammen mit der C++-Komponente nutzen. | optional |
| `starship_speed_test.py` | Misst Beschleunigung und Höchstgeschwindigkeit (8 s Vollgas). | Test |
| `starship_sensors.py` | UE4-Stand: spawnt den Bot mit allen 26 Sensoren und zeigt sie gleichzeitig an (`--test N`, `--sync`). Für CARLA 0.10 ersetzt durch `Sensorik/`. | UE4 |
| `../starship_sensor_plot.py` | Zeichnet die Sensoranordnung (Draufsicht und Seitenansicht) aus den JSON-Profilen; Abbildung im Bericht. | Werkzeug |

## 02_starship_emodell/scripts/unreal_editor: Editor-Python

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `starship_import_hka.py` | UE4: importiert `starship_hka_v5.fbx` auf das vorhandene v4-Skelett. | UE4 |
| `starship_dump_values.py` | UE4: liest alle Fahrzeugwerte nur lesend aus → `config/starship_values.json`. | UE4 |
| `ue5/ue_remote.py` | Führt ein Python-Skript im laufenden UE5-Editor aus (Remote Execution mit Linux-Loopback-Korrektur). Basis aller UE5-Schritte. | Werkzeug |
| `ue5/starship_common.py` | Gemeinsame Pfade und Hilfsfunktionen der UE5-Schritte (Asset-Pfade, Laden der UE4-Werte, Speichern). | Modul |
| `ue5/build_all.sh` | Führt die Schritte 1–6 nacheinander aus: Fahrzeug komplett neu aufbauen. | Neuaufbau |
| `ue5/s01_import_mesh.py` | Schritt 1: FBX als Skeletal Mesh mit neuem Skelett importieren, Maßstab prüfen. | Neuaufbau |
| `ue5/s02_physics_asset.py` | Schritt 2: Physics Asset mit einer Box auf `VehicleBase` (55 × 45 × 30 cm, Unterkante 19 cm). | Neuaufbau |
| `ue5/s03_anim_bp.py` | Schritt 3: AnimBP aus `AnimBP_EuropeanHGV` (6 Räder, Wheel Controller) auf das Starship-Skelett umgehängt. | Neuaufbau |
| `ue5/s04_wheels.py` | Schritt 4: sechs Chaos-Rad-Blueprints (r 9 cm, alle angetrieben, Lenkung vorne 30°, Federung für Bordsteine, Raycast) mit Instanzprüfung. | Neuaufbau |
| `ue5/s05_vehicle_bp.py` | Schritt 5: Drehmomentkurve importieren, `BP_Starship` (Parent `BaseVehiclePawnNW`) mit Masse, Motor, Getriebe, Allradantrieb, Ackermann anlegen. | Neuaufbau |
| `ue5/s06_register.py` | Schritt 6: `vehicle.starship.bot` in `VehicleParameters.json` eintragen (mit Sicherung; außerhalb des Editors). | Neuaufbau |
| `ue5/s07_play.py`, `ue5/s07_stop.py` | Play im Editor starten bzw. beenden (per Remote Execution). | Werkzeug |
| `ue5/s08_curb_assist_component.py` | Schritt 8: hängt die C++-Komponente `CurbAssist` an `BP_Starship`. | Neuaufbau |

## 02_starship_emodell/Sensorik: Sensoranzeigen und Profile (CARLA 0.10)

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `README.md` | Anzeigen, Bedienung, Grafikspeicher-Hinweise, Testergebnis. | nein |
| `starship_cameras.json` | Profil der 9 RGB-Kameras (Einzelkamera vorne + 4 Stereopaare, 4 Hz). | nein |
| `starship_tof.json` | Profil der 3 ToF-Tiefenkameras (320 × 240, 5 m, 10 Hz). | nein |
| `starship_radars.json` | Profil der 4 Radare (je Seite eins, 20 m, 20 Hz). | nein |
| `starship_ultrasonic.json` | Profil der 8 Ultraschallsensoren (CARLA-Hindernissensor, 3 m, 20 Hz). | nein |
| `starship_navigation.json` | Profil von GNSS (5 Hz) und IMU (100 Hz) mit Rauschwerten. | nein |
| `starship_sensor_common.py` | Gemeinsames Modul (gleiche Schnittstelle wie beim Clevon): Profile laden/prüfen, Bot finden, Sensoren anbringen/entfernen, Reiterleiste, ROS-Fahrzeugname. | Modul |
| `manual_control_starship.py` | Startet CARLAs `manual_control.py` mit Starship-GNSS/-IMU, spawnt den Bot mit `ros_name=starship`. | **ja** |
| `starship_cameras.py` | Kameraanzeige, 5 Reiter; nur die Kameras des aktiven Reiters existieren. | ja |
| `starship_tof.py` | ToF-Anzeige, 3 Reiter: farbiges Tiefenbild bis 5 m. | ja |
| `starship_radars.py` | Radaranzeige, 3 Reiter inkl. Rundum-Draufsicht. | ja |
| `starship_ultrasonic.py` | Ultraschallanzeige als Einparkhilfe und Abstandsverlauf. | ja |
| `starship_follow_cam.py` | Setzt die CARLA-Beobachteransicht hinter den Bot (`--view back/top/side`). | optional |
| `Messdaten/` | Ziel für gespeicherte Messdaten (Taste S); Inhalt wird nicht versioniert. | nein |

## 02_starship_emodell/Sensorik/ROS2

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `README.md` | Start, Steuerung über ROS 2, gemessene Topics und Raten, Lücken des eingebauten ROS 2, Grafikspeicher. | nein |
| `starship_ros_sensors.py` | Bringt 18 Sensoren (Kameras, ToF, Radar, GNSS, IMU) mit `ros_name` an; CARLA veröffentlicht sie unter `/carla/starship/…`. | **ja** |
| `starship_ros_extras.py` | rclpy-Ergänzungsknoten: 8 Ultraschall als `sensor_msgs/Range`, Odometrie, TF `map → starship` und Ultraschall-TF. | **ja** |
| `spawn_starship.py` | Spawnt den Bot ohne Fahrclient (`hero`, `ros_name=starship`) zum Steuern über `/carla/starship/vehicle_control_cmd`; `--remove` entfernt ihn. | bei ROS-Steuerung |
| `install_ros_jazzy.sh` | Installiert ROS 2 Jazzy auf Ubuntu 24.04 nach offizieller Anleitung. | einmalig |
| `ros_ws/build.sh` | Baut das minimale `carla_msgs` ohne colcon und ohne sudo. | einmalig |
| `ros_ws/setup.bash` | Lädt ROS 2 Jazzy und das lokale `carla_msgs`. | vor ROS-Befehlen |
| `ros_ws/src/carla_msgs/` | ROS-Paket mit nur `CarlaEgoVehicleControl.msg` (Felder exakt wie in CARLA), `CMakeLists.txt`, `package.xml`. | Build |
| `ros_ws/.gitignore` | Schließt die Build-Ausgaben aus. | nein |

## 02_starship_emodell/docs

| Datei/Ordner | Sinn und Funktion | Ausführen? |
|---|---|---|
| `IMPLEMENTIERUNG_CARLA_0.10.md` | **Hauptprotokoll** der Portierung auf CARLA 0.10: jeder Schritt, Stolperfallen, Messreihen, Bordstein-Analyse, Endstand. | nein |
| `FAHRZEUGWERTE.md` | Alle Fahrzeugwerte des UE4-Stands mit Einheiten (Geometrie, Motor, Getriebe, Räder, Kollision). | nein |
| `SENSORIK.md` | Alle 26 Sensoren mit Positionen, Parametern, Begründung und Quellen (Patent, Presse, Vergleichssensoren). | nein |
| `CARLA_0.10.md` | Ursprüngliche Planung des Umstiegs auf CARLA 0.10 (inzwischen umgesetzt). | nein |
| `images/` | Bilder: Blender-Knochen, HKA-Renderings, Sensorbeschriftung vorn/hinten, CARLA-Aufnahmen (UE4 und UE5), UE4-Editor-Screenshots; `sensor_labels.json` ordnet Sensor-IDs den Kürzeln K1–K9, T1–T3, R1–R4, U1–U8, G, I zu. | nein |
| `messungen/curb_results_*.json` | Rohdaten aller Bordstein-Messreihen: `basis`, `v1_federweg`, `v2_drop4`, `a1`–`a6` (Python-Assistent), `c1`–`c3` (C++-Komponente). Endstand = `c2_cpp_gerade`. | nein |
