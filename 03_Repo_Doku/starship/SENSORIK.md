# Sensorik

26 Sensoren, nachgebildet nach dem echten Starship: 9 Kameras, 4 Radare, 8 Ultraschallsensoren,
3 ToF-Tiefenkameras, GNSS und IMU. Die Anzahl von Kameras, Radaren und Ultraschall entspricht dem,
was über den Starship berichtet wurde (9 Kameras, 4 Radare, 8 Ultraschall, [MIT Technology Review 2018](https://www.technologyreview.com/2018/01/30/145935/why-sidewalk-delivery-robots-still-need-safety-drivers-too/amp/)).
Die Kamera-Anordnung folgt dem Starship-Patent [US20180253107A1](https://patents.google.com/patent/US20180253107A1/en).
Wo es keine Angaben gibt (genaue Montagepunkte, Radar- und Ultraschall-Werte), sind die Werte begründete Annahmen
und an vergleichbaren Seriensensoren orientiert.

Konfiguration: [`config/starship_sensors.json`](../config/starship_sensors.json).
**CARLA 0.10:** Anzeigen je Sensortyp mit Reitern und je eigener JSON-Datei in [`Sensorik/`](../Sensorik/README.md)
(gleiche Werte; das Skript unten ist der UE4-Stand mit allen Sensoren gleichzeitig).
Skript: [`scripts/carla/starship_sensors.py`](../scripts/carla/starship_sensors.py) (spawnt den Bot mit allen Sensoren,
zeigt sie live an; `--test 8` fährt 8 s geradeaus und speichert ein Übersichtsbild).

![Sensoren vorne](images/sensors_front.png)
![Sensoren hinten](images/sensors_rear.png)

Positionen in Metern relativ zum Fahrzeug-Ursprung (Boden, Mitte). Achsen wie CARLA: x vorne, y rechts, z oben.
Yaw 0 = nach vorne, 90 = nach rechts. Pitch negativ = nach unten.

## Kameras (9 × `sensor.camera.rgb`, 4 Bilder/s)

| ID | Gruppe | x | y | z | Yaw | Pitch | Auflösung | FOV |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| cam_front (K1) | Einzelkamera vorne | 0,28 | 0 | 0,42 | 0 | 0 | 640 × 480 | 80° |
| cam_front_left (K2) | Stereo vorne | 0,28 | −0,04 | 0,40 | 0 | −30 | 640 × 480 | 80° |
| cam_front_right (K3) | Stereo vorne | 0,28 | 0,04 | 0,40 | 0 | −30 | 640 × 480 | 80° |
| cam_left_front (K4) | Stereo links | 0,09 | −0,23 | 0,40 | −90 | −10 | 480 × 640 | 60° |
| cam_left_rear (K5) | Stereo links | −0,09 | −0,23 | 0,40 | −90 | −10 | 480 × 640 | 60° |
| cam_right_rear (K6) | Stereo rechts | −0,09 | 0,23 | 0,40 | 90 | −10 | 480 × 640 | 60° |
| cam_right_front (K7) | Stereo rechts | 0,09 | 0,23 | 0,40 | 90 | −10 | 480 × 640 | 60° |
| cam_rear_left (K8) | Stereo hinten | −0,28 | 0,04 | 0,40 | 180 | −10 | 640 × 480 | 80° |
| cam_rear_right (K9) | Stereo hinten | −0,28 | −0,04 | 0,40 | 180 | −10 | 640 × 480 | 80° |

Begründung (Patent US20180253107A1): eine Einzelkamera vorne plus Stereo-Paare vorne, links, rechts und hinten;
Stereo-Basis vorne/hinten 5–10 cm (hier 8 cm), seitlich 15–20 cm (hier 18 cm); vorderes Paar 20–40° nach unten
geneigt (hier 30°), um Bordstein und Boden direkt vor dem Bot zu sehen; Sichtfeld ca. 60 × 80°, seitliche Kameras
hochkant (deshalb 480 × 640); etwa 4 Bilder pro Sekunde (`sensor_tick` 0,25 s).

## ToF-Tiefenkameras (3 × `sensor.camera.depth`)

| ID | x | y | z | Yaw | Pitch | Auflösung | FOV |
| --- | --- | --- | --- | --- | --- | --- | --- |
| tof_front (T1) | 0,28 | 0 | 0,30 | 0 | −15 | 320 × 240 | 90° |
| tof_left (T2) | 0 | −0,23 | 0,30 | −90 | −15 | 320 × 240 | 90° |
| tof_right (T3) | 0 | 0,23 | 0,30 | 90 | −15 | 320 × 240 | 90° |

Begründung: Time-of-Flight wird für den Starship in Profilen genannt ([Wevolver](https://wevolver.com/specs/starship-technologies-starship-robot),
[3 Seas Europe](https://3seaseurope.com/the-estonian-robots-revolutionizing-last-mile-delivery/)). CARLA hat keinen ToF-Sensor;
eine Tiefenkamera mit niedriger Auflösung, im Skript auf 5 m abgeschnitten, kommt dem am nächsten. Vergleichbar sind
ToF-Sensoren der Klasse Infineon/pmd REAL3 (z. B. IRS2875C mit 240 × 180 Pixeln, [TechInsights](https://www.techinsights.com/ko/node/61619),
[Infineon ToF](https://infineon.com/cms/en/product/sensor/tof-3d-image-sensors/)). 320 × 240 liegt eine Stufe darüber;
für eine realistischere Simulation kann man auf 240 × 180 heruntergehen.

## Radar (4 × `sensor.other.radar`)

| ID | x | y | z | Yaw | H-FOV | V-FOV | Reichweite | Punkte/s |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| radar_front (R1) | 0,28 | 0 | 0,35 | 0 | 60° | 20° | 20 m | 1500 |
| radar_rear (R2) | −0,28 | 0 | 0,35 | 180 | 60° | 20° | 20 m | 1500 |
| radar_left (R3) | 0 | −0,23 | 0,35 | −90 | 60° | 20° | 20 m | 1500 |
| radar_right (R4) | 0 | 0,23 | 0,35 | 90 | 60° | 20° | 20 m | 1500 |

Begründung: 4 Radare wie berichtet, je eines pro Seite. Vergleichbar ist ein 60-GHz-Kurzstreckenradar für Roboter
wie TI IWR6843AOP (60–64 GHz, laut TI 120° Sichtfeld, für Lieferroboter beworben, [TI](https://www.ti.com/product/iwr6843aop),
[TI EVM](https://www.ti.com/product/IWR6843AOPEVM/part-details/IWR6843AOPEVM)). Das Sichtfeld ist hier bewusst
enger (60 × 20°), damit CARLAs Radar weniger Bodenechos liefert und die Punkte nicht verdünnt. Wer volle
Rundum-Abdeckung will, setzt `horizontal_fov` auf 90–120.

## Ultraschall (8 × `sensor.other.obstacle`)

| ID | x | y | z | Yaw | Pitch |
| --- | --- | --- | --- | --- | --- |
| us_front_left (U1) | 0,28 | −0,15 | 0,22 | −30 | 4 |
| us_front (U2) | 0,28 | 0 | 0,22 | 0 | 4 |
| us_front_right (U3) | 0,28 | 0,15 | 0,22 | 30 | 4 |
| us_right (U4) | 0 | 0,23 | 0,22 | 90 | 4 |
| us_rear_right (U5) | −0,28 | 0,15 | 0,22 | 150 | 4 |
| us_rear (U6) | −0,28 | 0 | 0,22 | 180 | 4 |
| us_rear_left (U7) | −0,28 | −0,15 | 0,22 | −150 | 4 |
| us_left (U8) | 0 | −0,23 | 0,22 | −90 | 4 |

Alle: Reichweite 3 m, `hit_radius` 0,15 m, auch statische Objekte.

Begründung: 8 Ultraschallsensoren wie berichtet, rundum verteilt (3 vorne, 3 hinten, je 1 seitlich). CARLA hat keinen
Ultraschallsensor; der Obstacle-Detektor (Kugel-Sweep) bildet die Keule nach. Vergleichbar ist z. B. MaxBotix
MB1242 (20–765 cm, 40 Hz, [MaxBotix](https://maxbotix.com/products/mb1242), [ArduPilot](https://ardupilot.org/rover/docs/common-rangefinder-maxbotixi2c.html)).
Die Reichweite ist auf 3 m begrenzt, weil der Bot nur 6 km/h fährt; Pitch +4° nach oben, weil der Sensor sonst
bei 22 cm Höhe die Fahrbahn als Hindernis meldet.

## GNSS und IMU

| ID | Typ | x | y | z | Rauschen |
| --- | --- | --- | --- | --- | --- |
| gnss (G) | `sensor.other.gnss` | 0 | 0 | 0,52 | Breite/Länge σ 0,00002° (ca. 2 m), Höhe σ 1 m |
| imu (I) | `sensor.other.imu` | 0 | 0 | 0,25 | Beschleunigung σ 0,02 m/s², Drehrate σ 0,002 rad/s |

Begründung: GNSS oben auf dem Deckel unter der Antenne; 2 m Streuung entspricht einem normalen GNSS-Empfänger ohne
RTK. IMU in Fahrzeugmitte; das Rauschen entspricht einfachen MEMS-Sensoren (Annahme, kein Datenblatt).

## Grenzen

- Anzahl und Art der Sensoren schwanken je nach Starship-Generation; neuere Quellen nennen bis zu 12 Kameras
  ([Sacra](https://sacra.com/research/starship)). Hier ist die 9-Kamera-Variante aus Patent und MIT-Artikel umgesetzt.
- Montagepunkte von ToF, Radar und Ultraschall sind nicht veröffentlicht und hier abgeleitet.
- ROS 2 ist noch nicht angebunden; die Klasse `SensorHub` im Skript ist als spätere Quelle für ROS-2-Publisher gedacht.
