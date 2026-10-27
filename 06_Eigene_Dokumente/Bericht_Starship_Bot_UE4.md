# Starship-Bot in CARLA – Umsetzung & Sensorik

Oct 9, 2026 · @Anvin

## Überblick

Der Lieferroboter fährt als eigenes Fahrzeug `vehicle.starship.bot` in CARLA 0.9.16 (Unreal Engine 4.26), trägt ein eigenes HKA-Design und 26 simulierte Sensoren. Im Abschlusstest am 09.10.2026 (Karte Town02) erreichte er wie das Original rund 6 km/h, lenkte und bremste stabil, und alle Sensoren lieferten Daten.

Werkzeugkette:

1. **Blender 5.2** – Modell, Skelett (Armature) und FBX-Export in Zentimetern.
2. **Unreal Engine 4.26 (CARLA-Editor)** – Import als Skeletal Mesh, Physics Asset, Animation Blueprint, Rad- und Fahrzeug-Blueprint.
3. **CARLA** – Registrierung im Blueprint-Katalog, Steuerung und Sensoren über die Python-API.

| Datei / Asset | Ort | Zweck |
| --- | --- | --- |
| `starship_hka_v5.blend`, `starship_hka_v5.fbx` | `C:\Users\letie\Downloads` | aktuelles Modell im HKA-Design |
| `starship_carla_cm_v4.blend`, `starship_carla_v4.fbx` | `C:\Users\letie\Downloads` | Vorstufe: Maßstab und Bones korrigiert |
| `SK_Starship_Mesh_HKA`, Skeleton `SK_Starship_v4_SM_Starship_Skeleton` | `Content/Carla/vehicle3` | Mesh und Skelett |
| `PA_Starship_v4`, `AnimBP_Starship_v4` | `Content/Carla/vehicle3` | Physik-Box und Rad-Animation |
| `BP_Starship`, `BP_Starship_FLW/FRW/RLW/RRW` | `Content/Carla/vehicle3` | Fahrzeug und vier Rad-Blueprints |
| `starship_sensors.json`, `starship_sensors.py` | `C:\Users\letie\Downloads` | Sensor-Konfiguration und Laufzeit-Skript |

Weitere Meshes und Skelette im Ordner `vehicle3` (v2, v3, cm, ältere AnimBPs, `pa_starship`) sind Zwischenstände der Fehlersuche und werden nicht mehr verwendet.

## Blender: Modell, Knochen und Export

Das Modell wird in Zentimetern exportiert, und alle Bones zeigen nach +Y mit Roll 0. Nur so kommt das Skelett ohne versteckte Skalierung oder Drehung in Unreal an.

&#91;image: Starship-Modell in Blender mit eingeblendeten Bones: orange VehicleBase, blau Vorderräder, grau Mittelräder, grün Hinterräder\]

**Maße des Modells** (`SM_Starship`): 56,5 × 57,9 × 87,7 cm (L × B × H inkl. Fahne). Der Rumpf ist etwa 55 × 45 cm, seine Unterkante liegt bei 16 cm. Die Räder haben 9 cm Radius und 4 cm Breite.

**Skelett (Armature `Armature`)**

| Bone | Kopf-Position (cm) | Aufgabe | Gewichtung |
| --- | --- | --- | --- |
| `VehicleBase` | 0 / 0 / 9,05 | Root-Bone, Fahrzeugkörper | Rumpf, Deckel, Fahne |
| `Wheel_Front_Left` / `_Right` | ±19,1 / ±25,65 / 9,05 | Vorderräder, gelenkt und angetrieben | je Rad 100 % |
| `Wheel_Middle_Left` / `_Right` | 0 / ±25,65 / 9,05 | Mittelräder, nur Optik | je Rad 100 % |
| `Wheel_Rear_Left` / `_Right` | −19,1 / ±25,65 / 9,05 | Hinterräder | je Rad 100 % |

Alle Rad-Bones hängen direkt unter `VehicleBase` (Hierarchietiefe 2). Positive Y-Werte sind in Blender links.

**Bone-Ausrichtung – die wichtigste Einstellung**

- Jeder Bone: Tail = Head + (0, 15, 0) cm, also Richtung +Y, Roll = 0°. Damit ist sein lokales Achsenkreuz identisch mit dem des Modells.
- Früher zeigten die Bones nach oben (+Z). In Unreal war `VehicleBase` dadurch um 90° gekippt. PhysX nutzt diesen Bone als Fahrzeug-Koordinatensystem, die Federungs-Raycasts gingen seitwärts, und der Bot fuhr nicht.

**Maßstab**

- Szene: Unit System Metric, Unit Scale 0,01. Mesh-Daten und Bones sind auf Zentimeter-Werte skaliert, alle Objekt-Skalierungen stehen auf 1.
- Das Mesh darf beim Speichern nicht im Edit Mode sein, sonst überschreibt Blender die skalierten Daten beim Export wieder.

**FBX-Export** (`starship_carla_v4.fbx`)

| Einstellung | Wert |
| --- | --- |
| Selected Objects | an (Armature + `SM_Starship`) |
| Object Types | Armature, Mesh |
| Apply Unit | an |
| Apply Scalings | FBX Units Scale → *All Local* (`FBX_SCALE_NONE`), Scale 1,0 |
| Forward / Up | −Z Forward / Y Up |
| Primary / Secondary Bone Axis | Y / X |
| Add Leaf Bones | aus |
| Only Deform Bones | aus |
| Bake Animation | aus |
| Smoothing | Face |

Prüfung im FBX: `UnitScaleFactor` = 1,0, keine `Lcl Scaling`-Werte und keine `Lcl Rotation` an den Bones. Die Armature heißt bewusst `Armature`. Unreal überspringt diesen Knoten, sodass `VehicleBase` der Root-Bone wird.

### Eigenes Design: HKA-Lieferroboter

Das Modell trägt jetzt ein eigenes Erscheinungsbild mit rotem „HKA“ an beiden Seiten. So ist es optisch klar vom Starship-Original abgegrenzt. Rumpfmaße und Skelett blieben gleich, damit Physik, Rad-Animation und Sensorpositionen weiter passen.

| Merkmal | Starship-Vorbild | HKA-Modell |
| --- | --- | --- |
| Farben | weißer Rumpf, schwarzer Unterbau | anthrazitgrauer Rumpf, hellgrauer Deckel, rote Akzente |
| Antenne | hohe Stange mit orangefarbener Fahne | kürzere Antenne (62 cm) mit rotem Signallicht |
| Front | glatt | rotes Lichtband auf 31 cm Höhe |
| Seiten | glatt | Schriftzug „HKA“ in Rot (11 cm hoch), schwarze Rammschutzleisten |
| Deckel | glatt | Griffleiste |

Dateien: `starship_hka_v5.blend` und `starship_hka_v5.fbx`; in Unreal `SK_Starship_Mesh_HKA` mit neuen Materialien `hka_body`, `hka_red`, `hka_lid`. Das Modell nutzt dasselbe Skelett wie v4, Physics Asset und AnimBP gelten unverändert.

&#91;image: HKA-Lieferroboter in Blender, Ansicht schräg von vorne links\]

&#91;image: HKA-Lieferroboter in Blender, Seitenansicht rechts mit rotem HKA-Schriftzug\]

Die Veränderungen betreffen die Gestaltung. Ob sie für eine Veröffentlichung urheber- und markenrechtlich ausreichen, ist eine Rechtsfrage und hier nicht geprüft.

## Unreal Engine: Assets und Einstellungen

In Unreal besteht der Bot aus fünf Teilen: Skeletal Mesh, Physics Asset mit einer Box, Animation Blueprint für die Räder, vier Rad-Blueprints und dem Fahrzeug-Blueprint `BP_Starship`. Alle liegen in `Content/Carla/vehicle3`.

### 1. Import des Skeletal Mesh

| Option | Wert |
| --- | --- |
| Import as Skeletal | an |
| Skeleton | leer (neues Skeleton anlegen) |
| Create Physics Asset | an (wird danach ersetzt) |
| Import Animations / Materials / Textures | aus (Materialien `orange`, `white`, `black`, `dark_gray` existieren bereits) |
| Import Uniform Scale | 1,0 |

Ergebnis: `SK_Starship_Mesh_v4` mit 7 Bones und den Bounds 56,5 × 57,9 × 87,7 cm. Ein Faktor 100 an dieser Stelle wäre ein Warnsignal für einen falschen Export.

### 2. Physics Asset `PA_Starship_v4`

- Genau ein Body auf `VehicleBase`, eine Box, Physics Type *Default*.
- Box im Bone-Raum: Center (0, 0, 23,45) cm, X = 55, Y = 45, Z = 33 cm. In Weltkoordinaten reicht sie von 16 bis 49 cm über dem Boden.
- Die Räder haben keine eigenen Bodies. Sie tragen den Bot über die Federung, die Box berührt die Straße nie.
- Unreal 4.26 Python kann Physics-Asset-Bodies nicht bearbeiten. Die Box-Werte wurden deshalb direkt in der `.uasset` gesetzt; im Editor geht das über *Body Setup → Primitives → Boxes*.

### 3. Animation Blueprint `AnimBP_Starship_v4`

Parent Class *Vehicle Anim Instance*, Skeleton `SK_Starship_v4_SM_Starship_Skeleton`. AnimGraph:

1. **Mesh Space Ref Pose**
2. **Wheel Handler** – dreht und lenkt die vier physikalischen Räder.
3. **Copy Bone** `Wheel_Rear_Right` → `Wheel_Middle_Right` (nur Rotation).
4. **Copy Bone** `Wheel_Rear_Left` → `Wheel_Middle_Left` (nur Rotation).
5. **Component To Local** → **Output Pose**

PhysX simuliert nur vier Räder (`WheeledVehicleMovementComponent4W`). Die Mittelräder drehen deshalb nur optisch mit den Hinterrädern mit.

&#91;image: AnimGraph von AnimBP\_Starship\_v4: Mesh Space Ref Pose, Wheel Handler, zwei Copy Bone, Component To Local, Output Pose\]

### 4. Rad-Blueprints `BP_Starship_FLW / FRW / RLW / RRW`

Parent Class *VehicleWheel*.

| Einstellung | Vorne (FLW, FRW) | Hinten (RLW, RRW) |
| --- | --- | --- |
| Shape Radius / Width | 9 / 4 cm | 9 / 4 cm |
| Mass | 1 kg | 1 kg |
| Steer Angle | 30° | 0° |
| Affected by Handbrake | aus | an |
| Max Brake / Handbrake Torque | 20 / 40 Nm | 20 / 40 Nm |
| Suspension Max Raise / Drop | 2 / 2 cm | 2 / 2 cm |
| Suspension Natural Frequency / Damping | 9 Hz / 0,8 | 9 Hz / 0,8 |
| Lat Stiff Max Load / Value, Long Stiff | 1 / 17, 3000 | 1 / 17, 3000 |
| Tire Config | **CommonTireConfig** (CARLA) | **CommonTireConfig** (CARLA) |
| Auto Adjust Collision Size | **aus** | **aus** |
| Dont Create Shape (im Editor: UsePhysAssetShape) | aus | aus |

*Auto Adjust Collision Size* maß einen falschen Radius von etwa 4 cm, der Bot saß dadurch 5 cm zu tief. Die Rad-Kollisionsform (`Wheel_Shape`) hob ihn an, sodass die Federung keinen Bodenkontakt bekam.

&#91;image: Class Defaults von BP\_Starship\_FLW: Shape, Wheel, Tire und Suspension\]

### 5. Fahrzeug-Blueprint `BP_Starship`

Parent Class *BaseVehiclePawn* (CARLA).

| Bereich | Einstellung | Wert |
| --- | --- | --- |
| Mesh | Skeletal Mesh / Anim Class | `SK_Starship_Mesh_v4` / `AnimBP_Starship_v4_C` |
| Mesh | Center Of Mass Offset (COM Nudge) | (0, 0, −15) cm, vorher (25, 0, −30) |
| VehicleBounds | Collision Preset | **NoCollision**, vorher *Vehicle* |
| Vehicle Movement | Mass / Drag | 35 kg / 0,3 (vorher 50 kg) |
| Wheel Setups | 0–3 | FLW `Wheel_Front_Left`, FRW `Wheel_Front_Right`, RLW `Wheel_Rear_Left`, RRW `Wheel_Rear_Right` |
| Engine | Torque Curve | 10 Nm @ 1000, 15 Nm @ 1400–4200, 8 Nm @ 6500 U/min |
| Engine | Max RPM / MOI | 720 / 0,05 kg·m² (vorher 6500 / 1,0) |
| Engine | Damping full / zero engaged / zero disengaged | 0,01 / 0,05 / 0,02 |
| Differential | Typ | Open Front Drive |
| Transmission | Automatik, Final Ratio | an, 1 Gang (Übersetzung 1,0), Final Ratio 3,4 (vorher 6 Gänge, 3,421) |

Bei den CARLA-Serienfahrzeugen ist `VehicleBounds` ebenfalls *NoCollision*. Mit Kollision stand der Bot auf seiner eigenen Bounding-Box statt auf den Rädern.

&#91;image: BP\_Starship Class Defaults: Anim Class AnimBP\_Starship\_v4\_C und Skeletal Mesh SK\_Starship\_Mesh\_v4\]

&#91;image: BP\_Starship Wheel Setups: vier Rad-Blueprints mit Bone-Namen\]

Die Motor- und Getriebewerte stammen aus der Tempo-Abstimmung auf 6 km/h, Details im Abschnitt „Belege für die Parametrisierung“. Außerdem überschreibt `BP_Starship` die Rumpffarbe (Material-Slot `hka_body`) mit dem Material `dark_gray`, damit der Rumpf anthrazit erscheint.

## Belege für die Parametrisierung

Jeder Wert hat eine von fünf Herkünften: **belegt** (Quelle zum echten Starship oder Sensor), **gemessen** (am 3D-Modell), **UE-Standard** (Voreinstellung von Unreal 4.26, nachgelesen im Engine-Quellcode `VehicleWheel.cpp` und `WheeledVehicleMovementComponent.cpp`), **Fehlersuche** (im Test als nötig erwiesen) oder **Annahme** (plausibel gewählt, nicht belegt). Annahmen sind die Stellen, an denen echte Messdaten den Bot noch verbessern können.

**Fahrzeug und Räder**

| Parameter | Wert | UE-Standard (Pkw) | Herkunft | Begründung / Quelle |
| --- | --- | --- | --- | --- |
| Masse | 35 kg | 1500 kg | belegt | 25 kg leer plus bis 9,1 kg Zuladung ([Wikipedia](https://en.wikipedia.org/wiki/Starship_Technologies)) |
| Höchstgeschwindigkeit (Ziel) | 6 km/h | – | belegt | [Wikipedia](https://en.wikipedia.org/wiki/Starship_Technologies); Patent: bevorzugt 4–6 km/h |
| Physik-Box | 55 × 45 × 33 cm, Unterkante 16 cm | – | gemessen | Rumpfmaße des Modells |
| Schwerpunkt-Verschiebung | (0, 0, −15) cm | (0, 0, 0) | Fehlersuche | Problem 5 |
| Luftwiderstandsbeiwert | 0,3 | 0,3 | UE-Standard | bei 6 km/h praktisch ohne Einfluss |
| Radradius / -breite | 9 / 4 cm | 30 / 10 cm | gemessen | Rad-Geometrie des Modells |
| Radmasse | 1 kg | 20 kg | Annahme | kleines Kunststoffrad |
| Lenkwinkel vorne | 30° | 70° | Annahme | siehe Hinweis zur Lenkung unten |
| Federweg ein / aus | 2 / 2 cm | 10 / 10 cm | Annahme | kleiner Roboter mit kurzem Federweg |
| Federfrequenz / Dämpfung | 9 Hz / 0,8 | 7 Hz / 1,0 | Annahme | straffe Abstimmung für wenig Wanken |
| Bremsmoment / Handbremse | 20 / 40 Nm | 1500 / 3000 Nm | Annahme | grob auf die Masse heruntergerechnet |
| Reifen: Lat Stiff Max Load / Value | 1 / 17 | 2 / 17 | Annahme |  |
| Reifen: Long Stiff Value | 3000 | 1000 | Annahme | aus einer Fahrzeugvorlage übernommen |
| Auto Adjust Collision Size / Radform | aus / an | an / an | Fehlersuche | Problem 2 |
| VehicleBounds-Kollision | NoCollision | – | Fehlersuche | Problem 3, wie CARLA-Serienfahrzeuge |

**Lenkung: Auto-Lenkung statt Panzerlenkung wie im Original**

Der simulierte Bot lenkt wie ein Auto über zwei schwenkbare Vorderräder (Lenkwinkel 30°). Der echte Starship lenkt nach den Unterlagen sehr wahrscheinlich wie ein Panzer, also über unterschiedliche Raddrehzahlen links und rechts. Das ist eine bewusste Vereinfachung.

*Beleg für die Panzerlenkung im Original:* Das Starship-Fahrwerkspatent [US 2018/0194411 A1](https://patents.google.com/patent/US20180194411A1/en) (erteilt als US 10 239 378 B2, Priorität 02.11.2015) beschreibt im Hauptanspruch, dass die Drehachse jedes Rades bei Vorwärts-, Rückwärts- **und Kurvenfahrt** im Wesentlichen fest zum Roboter bleibt. Dazu können alle sechs Räder einzeln angetrieben sein, die hinteren Motoren getrennt je Seite. Feste Radachsen plus getrennt angetriebene Seiten sind genau das Prinzip der Panzerlenkung. Einschränkung: Ein Unteranspruch nennt lenkbare Vorderräder als mögliche Variante, und Starship selbst veröffentlicht keine Angabe zur Lenkung des Serienroboters.

*Warum die Simulation trotzdem wie ein Auto lenkt:*

1. **Unreal bietet nur Auto-Lenkung an.** Das Fahrzeugsystem von Unreal 4.26 (Plugin PhysXVehicles) enthält nur Fahrzeugkomponenten mit Lenkrad-Logik. Die darunterliegende Physik-Bibliothek PhysX kennt zwar einen Panzerantrieb (`PxVehicleDriveTank`), Unreal stellt ihn aber nicht als Komponente bereit; im Quellcode wird er nur beim Aufräumen erwähnt.
2. **CARLA ist auf Autos ausgelegt.** Die Fahrzeugklasse von CARLA, die Steuerbefehle (Gas, Bremse, Lenkwinkel), `manual_control.py`, der Autopilot und die ROS-Bridge erwarten einen Lenkwinkel. Panzerlenkung hätte eigenen C++-Code im CARLA-Plugin und einen kompletten Neubau von CARLA erfordert.
3. **Einfachheit und Aufwand.** Mit der Auto-Lenkung läuft der Bot mit unveränderter CARLA-Software und allen vorhandenen Werkzeugen. Für das Projektziel, Sensoren und ROS2-Schnittstellen zu testen, ist die Art der Lenkung zweitrangig.
4. **Geringe Auswirkung im normalen Fahrbetrieb.** Geradeaus und in sanften Kurven bewegen sich beide Varianten fast gleich. Der Hauptunterschied: Der Simulationsbot kann nicht auf der Stelle drehen und braucht einen Wendekreis.

Für ROS2 bleibt die Schnittstelle trotzdem gleich: Ein Fahrbefehl `cmd_vel` (Geschwindigkeit plus Drehrate) wird in der Simulation in Lenkwinkel umgerechnet, beim echten Roboter in Raddrehzahlen.

**Motor und Getriebe** (abgestimmt auf das belegte Tempo von 6 km/h)

| Parameter | Wert | vorher | Herkunft | Begründung |
| --- | --- | --- | --- | --- |
| Gänge | 1 Gang, Übersetzung 1,0 | 6 Gänge (3,615 … 0,683) | Abstimmung | Ein Elektro-Lieferroboter hat kein Schaltgetriebe; die 6 Gänge stammten aus einer Pkw-Vorlage. |
| Achsübersetzung (Final Ratio) | 3,4 | 3,421 | Abstimmung | gerundet, Gesamtübersetzung 3,4 |
| Maximaldrehzahl | 720 U/min | 6500 U/min | Abstimmung | begrenzt das Tempo: 720 U/min / 3,4 = 212 U/min am Rad ≈ 2,0 m/s bei 9 cm Radius; im Test 5,4–6,7 km/h |
| Drehmomentkurve | 10–15 Nm | gleich | Annahme | aus Pkw-Vorlage herunterskaliert; Bot erreicht sein Tempo in rund 1 s |
| Motor-Trägheit (MOI) | 0,05 kg·m² | 1,0 (UE-Standard) | Fehlersuche | der Pkw-Wert wirkte bei 9-cm-Rädern wie ein Schwungrad |
| Dämpfung voll / Leerlauf eingekuppelt / ausgekuppelt | 0,01 / 0,05 / 0,02 | 0,15 / 2,0 / 0,35 | Annahme | passend zur kleineren Trägheit verkleinert |

Messung im Abschlusstest (Vollgas, 8 s geradeaus): 90 % der Endgeschwindigkeit nach 1,0 s, höchstens 6,7 km/h, Nickwinkel beim Anfahren höchstens 3,6°, Vollbremsung bis zum Stillstand.

**Sensoren**

| Parameter | Wert | Herkunft | Begründung / Quelle |
| --- | --- | --- | --- |
| Anzahl Kameras | 9 sichtbar + 3 ToF = 12 | belegt | Starship, Patent, Vision Systems Design |
| Kamera-Anordnung, Stereo-Basis, Neigung | siehe Positionstabelle | belegt | Patent US 2018/0253107 A1 |
| Kamera-Sichtfeld | 80° quer bzw. 60° hochkant | belegt | Patent: etwa 60° × 80° |
| Kamera-Bildrate | 4 Bilder/s | belegt | Patent: bevorzugt 4 Bilder/s |
| Kamera-Auflösung | 640 × 480 | Annahme | im Patent nicht genannt |
| ToF-Auflösung | 320 × 240 | belegt | Datenblatt ESPROS epc660 |
| ToF-Reichweite / Sichtfeld | 5 m / 90° | Annahme | ToF misst typischerweise im Bereich weniger Meter |
| Anzahl Radar / Ultraschall | 4 / 8 | belegt | MIT Technology Review 2018 (ältere Generation) |
| Radar-Öffnungswinkel / Reichweite | 60° × 20° / 20 m | Annahme |  |
| Ultraschall-Reichweite | 3 m | Annahme | Größenordnung von Einparksensoren |
| GNSS-Rauschen | ca. 2 m | Annahme | frühe Generation laut Hersteller nur etwa 30 m genau (Vision Systems Design) |
| IMU-Rauschen | 0,02 m/s², 0,002 rad/s | Annahme |  |

## CARLA: Registrierung, Test und Fehlersuche

Der Bot ist in der `VehicleFactory` als Make `starship`, Model `bot` eingetragen und damit über die Python-API als `vehicle.starship.bot` spawnbar. Im Abschlusstest am 09.10.2026 in Town02 stand er stabil auf seinen Rädern, erreichte nach 1 s seine Reisegeschwindigkeit und fuhr höchstens 6,7 km/h.

&#91;image: HKA-Lieferroboter in CARLA (Town02), rechte Seite\]

&#91;image: HKA-Lieferroboter in CARLA (Town02), linke Seite\]

**Test-Ablauf**

1. Im CARLA-Editor *Play* starten (Town10HD\_Opt, Ladezeit etwa 4–5 min).
2. `python manual_control.py --filter vehicle.starship.bot` in `D:\CARLA\PythonAPI\examples` ausführen, oder das Testskript `starship_drive_test.py` laufen lassen (Leerlauf, Vollgas, Lenken, Bremsen mit Log von Höhe, Lage und Tempo).

### Gelöste Probleme – verständlich erklärt

Sechs Fehler verhinderten, dass der Bot startet oder fährt. Jeder hatte dieselbe Wurzel: Blender, Unreal und die Physik-Engine verstanden ein Detail des Modells unterschiedlich. Ein paar Begriffe vorab:

| Begriff | Bedeutung |
| --- | --- |
| Bone (Knochen) | Unsichtbares Gerüst im 3D-Modell. Jeder Bone hat eine Position und ein eigenes Achsenkreuz (vorne, links, oben). Der Rumpf hängt am Bone `VehicleBase`, jedes Rad an einem Rad-Bone. |
| Physik-Engine (PhysX) | Rechenprogramm in Unreal, das Schwerkraft, Federn, Reifen und Motor simuliert. |
| Federungs-Strahl | Unsichtbarer Messstrahl, den PhysX von jedem Rad nach unten schickt. Trifft er die Straße, drückt die Feder das Fahrzeug hoch und das Rad kann Antriebskraft übertragen. |
| Physics Asset | Vereinfachte Kollisionsform des Modells, bei uns eine Box um den Rumpf. Die Physik rechnet damit statt mit dem detaillierten Modell. |
| Blueprint | Bauplan eines Objekts in Unreal mit allen Einstellungen. |
| VehicleBounds | Box, die CARLA um jedes Fahrzeug legt, um seine Größe anzugeben, etwa für Sensoren und Verkehrsregeln. Sie soll nicht anfassbar sein. |
| Schwerpunkt | Der Punkt, an dem das Gewicht des Fahrzeugs wirkt. |

#### Problem 1: Falscher Maßstab

**Was passierte:** Version 1 fiel durch die Straße. Version 2 kam in Unreal 100-mal zu klein an, 0,56 statt 56 cm.

**Warum ist das ein Problem:** Blender speichert Maße in Metern, Unreal liest Zentimeter. Aus 0,56 m wurden 0,56 cm. Eine so winzige Physik-Box rutscht zwischen zwei Rechenschritten durch den Boden, und Rad-Radien, Federwege und Sensorpositionen passen nicht mehr.

**Lösung:** In Blender alle Maße von Modell und Bones auf echte Zentimeter-Werte umgerechnet und mit der Einheit 1 cm exportiert. Keine Skalierungsfaktoren mehr in der Datei.

**Warum das hilft:** Beide Programme lesen jetzt dieselbe Zahl als dieselbe Länge. Es gibt keinen versteckten Faktor, der an anderer Stelle wieder stören kann.

&#91;embedded content: Problem 1 · Maßstab vorher und nachher\]

#### Problem 2: Falsche Radgröße in der Physik

**Was passierte:** Der Bot saß 5 cm zu tief, die Räder steckten optisch im Boden.

**Warum ist das ein Problem:** Die Option *Auto Adjust Collision Size* misst den Radradius automatisch am Modell. Wegen der gedrehten Rad-Bones maß sie die Radbreite (4 cm) statt des Durchmessers. Mit dem falschen Radius rechnet PhysX Bodenkontakt und Federweg falsch.

**Lösung:** Automatik aus, Radius fest auf 9 cm. Die Rad-Kollisionsform war zwischenzeitlich abgeschaltet und ist seit der Bone-Korrektur (Problem 4) wieder an.

**Warum das hilft:** PhysX nutzt jetzt den gemessenen Radius des Modells. Mit korrekt ausgerichteten Bones sitzt die Radform genau am Rad und hebt den Bot nicht mehr an.

#### Problem 3: Der Bot stand auf seiner eigenen Messbox

**Was passierte:** Der Bot stand stabil auf genau 0 cm, bewegte sich aber bei Vollgas keinen Millimeter.

**Warum ist das ein Problem:** Die CARLA-Box `VehicleBounds` reicht vom Boden bis 88 cm und war auf „anfassbar“ gestellt. Der Bot stand auf dieser Box wie auf einem Sockel. Die Räder trugen kein Gewicht und konnten keine Kraft auf die Straße bringen.

**Lösung:** Kollision der Box ausgeschaltet (*NoCollision*). Das ist auch bei allen mitgelieferten CARLA-Autos so eingestellt; verglichen am BMW Isetta.

**Warum das hilft:** Die Box beschreibt nur noch die Größe. Getragen wird der Bot nur noch über Federn und Räder.

&#91;embedded content: Problem 3 · Messbox mit und ohne Kollision\]

#### Problem 4: Gekippter Haupt-Bone

**Was passierte:** Auch ohne Messbox meldeten alle vier Räder „in der Luft“. Der Motor drehte mit 6245 U/min hoch, der Bot lag auf seiner Physik-Box.

**Warum ist das ein Problem:** PhysX nimmt das Achsenkreuz des Haupt-Bones `VehicleBase` als Fahrzeug-Koordinatensystem. In Blender zeigte der Bone nach oben, in Unreal war er dadurch um 90° gekippt. Für PhysX war „unten“ deshalb seitwärts, und die Federungs-Strahlen gingen ins Leere.

**Lösung:** Alle Bones in Blender nach +Y mit Roll 0 ausgerichtet. Ihr Achsenkreuz entspricht jetzt genau dem des Modells.

**Warum das hilft:** „Unten“ ist wieder unten. Die Strahlen treffen die Straße, die Federn tragen den Bot, die Räder übertragen Kraft. Das war der entscheidende Fix: Danach fuhr der Bot.

&#91;embedded content: Problem 4 · Federungs-Strahlen vorher und nachher\]

#### Problem 5: Schwerpunkt vor der Vorderachse

**Was passierte:** Im Blueprint war der Schwerpunkt um 25 cm nach vorne und 30 cm nach unten verschoben, ein übernommener Wert eines Autos.

**Warum ist das ein Problem:** Die Vorderachse sitzt nur 19 cm vor der Mitte. Der Schwerpunkt lag also vor dem vordersten Rad. PhysX verteilt das Gewicht nach der Lage des Schwerpunkts auf die Räder; den Hinterrädern bliebe rechnerisch ein negatives Gewicht.

**Lösung:** Schwerpunkt mittig und 15 cm tiefer gesetzt.

**Warum das hilft:** Alle Räder bekommen eine plausible Last. Der tiefere Schwerpunkt macht den hohen, schmalen Bot in Kurven kippstabiler.

&#91;embedded content: Problem 5 · Lage des Schwerpunkts\]

#### Problem 6: Absturz beim Spawnen des HKA-Modells

**Was passierte:** Nach dem Neustart des Editors stürzte Unreal jedes Mal ab, sobald der Bot gespawnt wurde. Der Absturzbericht zeigte auf die CARLA-Funktion, die beim Start die Reibung der Reifen setzt.

**Warum ist das ein Problem:** Jedes Rad braucht eine Reifen-Konfiguration (*Tire Config*). Bei einem Test war sie durch ein nur im Arbeitsspeicher existierendes Objekt ersetzt und so gespeichert worden. Nach dem Neustart war das Feld leer. CARLA greift beim Spawnen darauf zu und löst damit einen Absturz aus. Zusätzlich ließ die abgeschaltete Radform (*UsePhysAssetShape*) das neue HKA-Mesh nicht sauber starten.

**Lösung:** Allen vier Rädern die Standard-Reifenkonfiguration `CommonTireConfig` von CARLA zugewiesen und die Radform wieder eingeschaltet. Gefunden über Symbol-Auflösung des Absturzberichts und einen Gegentest mit dem v4-Mesh.

**Warum das hilft:** Jedes Rad hat jetzt eine dauerhaft gespeicherte Reifen-Konfiguration, die auch nach einem Neustart vorhanden ist. Mit Radform erzeugt Unreal die Räder auf dem Standardweg.

#### Begleitproblem: Editor-Abstürze beim Laden

Der Rechner hat 16 GB Arbeitsspeicher. Town10 belegt im Editor zusammen mit *Play* fast alles davon, mehrfach brach Unreal mit „out of memory“ ab. Getestet wird deshalb in der kleineren Karte Town02: Sie lädt in rund 25 s statt 4–5 min.

**Werkzeuge der Fehlersuche**

- Editor-Python während *Play*: Motordrehzahl, Gang und `is_in_air` jedes Rads direkt am laufenden Fahrzeug auslesen.
- Linetraces in mehreren Höhen durch den Bot: zeigten, welche Komponente wo kollidiert.
- Vergleich mit einem Serienfahrzeug (BMW Isetta / Microlino) im selben Lauf.
- Änderungen am Blueprint-Default-Objekt während *Play* gelten für jeden neu gespawnten Bot. So ließ sich ohne Neustart testen, gespeichert wurde erst nach *Play*.

**Hinweise für eigene Skripte**

- `apply_physics_control()` aus der CARLA-API wirft den Bot aus der Welt. Physikwerte bitte im Blueprint ändern.
- Die Höchstgeschwindigkeit ist über die Motordrehzahl auf rund 6 km/h begrenzt, wie beim echten Starship.
- Einmal stürzte der Editor beim Start von *Play* mit `DXGI_ERROR_DEVICE_HUNG` ab, einem Grafiktreiber-Timeout unter DirectX 12.

## Recherche: Sensorik des echten Starship-Roboters

Starship nennt für die aktuelle Flotte 12 Kameras inklusive Time-of-Flight, dazu Radar, Ultraschall und neuronale Netze. Genaue Anzahlen für Radar und Ultraschall sowie die Einbaupositionen veröffentlicht die Firma nicht. Die Quellen widersprechen sich teilweise, weil sie verschiedene Robotergenerationen beschreiben.

| Sensor | Aktuell (Starship, Stand 2024–26) | Frühe Generation (2016–18) | Quelle |
| --- | --- | --- | --- |
| Kameras gesamt | 12, inkl. Time-of-Flight | mind. 9 bzw. 10 | [Starship: Our Robots](https://www.starship.xyz/?p=673), [Wikipedia](https://en.wikipedia.org/wiki/Starship_Technologies) |
| Stereo-Kameras (sichtbares Licht) | Anzahl nicht genannt | 3 Stereo-Paare = 6 | [Vision Systems Design](https://www.vision-systems.com/embedded/article/16742182/autonomous-robot-successfully-delivers-takeout-for-the-first-time) |
| Time-of-Flight-Kameras (3D) | enthalten, Chip ESPROS epc660 | 3 | [The Robot Report, 30.11.2022](https://www.therobotreport.com/espros-chips-in-use-in-starship-technologies-delivery-robots/), Vision Systems Design |
| Radar | enthalten, Anzahl offen | 4 | Starship, [MIT Technology Review, 30.01.2018](https://www.technologyreview.com/2018/01/30/145935/why-sidewalk-delivery-robots-still-need-safety-drivers-too/amp/) |
| Ultraschall | enthalten, 360°-Abdeckung | 8 | Starship, MIT Technology Review, Vision Systems Design |
| GNSS (GPS) | enthalten | enthalten, Genauigkeit ca. 30 m | Wikipedia, Vision Systems Design |
| IMU (Beschleunigung, Drehrate) | enthalten | enthalten | Wikipedia, Vision Systems Design |
| Rechner | nicht genannt | Nvidia Tegra K1 | Vision Systems Design |

**Patent zur Kameraanordnung:** Das Starship-Patent [US 2018/0253107 A1](https://patents.google.com/patent/US20180253107A1/) (Anmelder Starship Technologies OÜ, Priorität 02.11.2015) beschreibt 9 Kameras: eine Einzelkamera vorne mit waagerechter Sicht sowie Stereo-Paare vorne, links, rechts und hinten. Die Abstände der Paare betragen vorne und hinten 5–10 cm, seitlich 15–20 cm. Die beiden vorderen Paar-Kameras sind 20–40° nach unten geneigt, die Sichtfelder betragen etwa 60° × 80°, seitliche Kameras sind hochkant eingebaut. Die Kameras nehmen 4 Bilder pro Sekunde auf, der Roboter fährt bevorzugt 4–6 km/h. Zusammen mit den drei ToF-Kameras aus Vision Systems Design ergibt das die 12 Kameras, die Starship heute angibt. Ein Patent beschreibt eine Ausführungsform; dass der Serienroboter genau so aufgebaut ist, bestätigt es nicht.

**Time-of-Flight-Chip:** Starship nutzt den ESPROS epc660. Der Chip arbeitet bei voller Sonne und bei Dunkelheit und braucht wenig Strom ([The Robot Report](https://www.therobotreport.com/espros-chips-in-use-in-starship-technologies-delivery-robots/)). Laut Datenblatt hat er 320 × 240 Pixel und liefert 12-Bit-Distanzwerte je Pixel ([Datenblatt epc660, Kopie](https://atta.szlcsc.com/upload/public/pdf/source/231023/Datasheet_epc660-d67cd1b32b170f852a88b269f1d42613.pdf)).

**Verhalten bei Hindernissen:** Die Sensoren bilden eine „Bubble of Awareness“. Der Bot erkennt damit Fußgänger, Radfahrer oder Tiere und kann ausweichen oder in sicherem Abstand anhalten ([Starship](https://www.starship.xyz/?p=673)). Kamera und Ultraschall zusammen erzeugen diese Zone ([Vision Systems Design](https://www.vision-systems.com/embedded/article/16742182/autonomous-robot-successfully-delivers-takeout-for-the-first-time)).

**Eckdaten des echten Roboters:** höchstens 6 km/h, 25 kg leer, bis 9,1 kg Zuladung, im Schnitt 18 h Akkulaufzeit ([Wikipedia](https://en.wikipedia.org/wiki/Starship_Technologies)).

Offene Punkte: genaue Anzahl von Radar und Ultraschall in der aktuellen Generation sowie alle Einbaupositionen. Für die Simulation wurden die Werte der frühen Generation (4 Radar, 8 Ultraschall) übernommen.

## Sensor-Platzierung am Modell

Die Kameras sind so angeordnet, wie es Starship im Patent [US 2018/0253107 A1](https://patents.google.com/patent/US20180253107A1/) beschreibt. Für ToF, Radar, Ultraschall, GNSS und IMU gibt es keine veröffentlichten Einbaupositionen. Ihre Lage ist deshalb aus den Anforderungen des jeweiligen Sensors begründet. Insgesamt sitzen 26 Sensoren in vier Höhenbändern: Kameras auf 40–42 cm, Radar auf 35 cm, ToF auf 30 cm, Ultraschall auf 22 cm, GNSS auf dem Deckel.

&#91;embedded content: Draufsicht · 26 Sensoren am Rumpf\]

Vorne sitzen eine Einzelkamera und ein nach unten geneigtes Stereo-Paar, dazu je ein Stereo-Paar links, rechts und hinten. Darunter liegen an drei Seiten ToF, an allen vier Seiten Radar, und ein Ring aus acht Ultraschallsensoren.

&#91;image: Beschriftete Sensor-Marker am HKA-Modell, Ansicht schräg von vorne links: K Kamera (blau), T ToF (lila), R Radar (rot), U Ultraschall (grün), G GNSS (gelb)\]

&#91;image: Beschriftete Sensor-Marker am HKA-Modell, Ansicht schräg von hinten rechts\]

**Einbaupositionen** (Meter, CARLA-Achsen: x vorne, y rechts, z oben; Ursprung am Boden in Fahrzeugmitte)

| Nr. | ID (Frame-Name) | Typ | x | y | z | Gier / Nick | Beleg bzw. Begründung |
| --- | --- | --- | --- | --- | --- | --- | --- |
| K1 | `cam_front` | Kamera 640×480, 80°, 4 Bilder/s | 0,28 | 0 | 0,42 | 0° / 0° | Patent: Einzelkamera vorne, waagerechte Sicht |
| K2 / K3 | `cam_front_left` / `cam_front_right` | Stereo-Paar | 0,28 | −0,04 / 0,04 | 0,40 | 0° / −30° | Patent: Paar vorne, Basis 5–10 cm, 20–40° nach unten |
| K4 / K5 | `cam_left_front` / `cam_left_rear` | Stereo-Paar hochkant 480×640, 60° | 0,09 / −0,09 | −0,23 | 0,40 | −90° / −10° | Patent: Paar links, Basis 15–20 cm, hochkant |
| K6 / K7 | `cam_right_rear` / `cam_right_front` | Stereo-Paar hochkant | −0,09 / 0,09 | 0,23 | 0,40 | 90° / −10° | Patent: Paar rechts |
| K8 / K9 | `cam_rear_left` / `cam_rear_right` | Stereo-Paar | −0,28 | 0,04 / −0,04 | 0,40 | 180° / −10° | Patent: Paar hinten, Basis 5–10 cm |
| T1–T3 | `tof_front`, `tof_left`, `tof_right` | ToF 320×240, 90°, bis 5 m | 0,28 / 0 / 0 | 0 / −0,23 / 0,23 | 0,30 | 0/−90/90° / −15° | Anzahl 3: Vision Systems Design; Lage begründet (unten) |
| R1–R4 | `radar_front`, `_rear`, `_left`, `_right` | Radar 60° × 20°, 20 m | ±0,28 / 0 | 0 / ±0,23 | 0,35 | 0/180/−90/90° / 0° | Anzahl 4: MIT Technology Review 2018; Lage begründet |
| U1–U3 | `us_front_left`, `us_front`, `us_front_right` | Ultraschall 3 m | 0,28 | −0,15 / 0 / 0,15 | 0,22 | −30/0/30° / 4° | Anzahl 8 und 360°: MIT Technology Review, Vision Systems Design; Lage begründet |
| U4, U8 | `us_right`, `us_left` | Ultraschall | 0 | 0,23 / −0,23 | 0,22 | 90/−90° / 4° | wie U1–U3 |
| U5–U7 | `us_rear_right`, `us_rear`, `us_rear_left` | Ultraschall | −0,28 | 0,15 / 0 / −0,15 | 0,22 | 150/180/−150° / 4° | wie U1–U3 |
| G | `gnss` | GNSS, Rauschen ca. 2 m | 0 | 0 | 0,52 | – | begründet: freie Sicht zum Himmel |
| I | `imu` | IMU | 0 | 0 | 0,25 | – | begründet: nahe am Schwerpunkt |

**Begründung der Positionen ohne Herstellerangabe**

- **Kamerahöhe 40–42 cm:** Das Patent nennt keine Höhe, nur eine Robotergröße von etwa 60 cm. Wir nutzen die höchste Stelle des Rumpfs knapp unter der Deckelkante (43 cm). Dort ist die Sicht am weitesten und wird nicht von den Rädern verdeckt.
- **ToF vorne und seitlich, 30 cm, 15° nach unten:** ToF misst auf wenige Meter genaue Abstände. Es dient deshalb der Nahbereichserkennung von Bordsteinen, Stufen und Löchern. Bei 30 cm Höhe und 15° Neigung trifft der Mittelstrahl den Boden etwa 1,1 m vor dem Bot (0,30 m / tan 15°). Vorne und seitlich, weil der Bot dorthin fährt und abbiegt; hinten decken Stereo-Kameras, Radar und Ultraschall ab.
- **Radar auf jeder Seite, 35 cm:** Radar erkennt bewegte Objekte wie Radfahrer und Autos auch bei Dunkelheit und Regen. Je ein Radar pro Seite deckt die vier Hauptrichtungen ab. Mit 60° Öffnungswinkel bleiben schmale Lücken an den Ecken; dort messen die schrägen Ultraschallsensoren.
- **Ultraschall auf 22 cm, Ring aus 8:** Ultraschall erkennt nahe Hindernisse bis etwa 3 m und dient als letzte Sicherheitsstufe. Er sitzt am unteren Rumpf, um auch niedrige Hindernisse wie Poller, Tiere oder Kinderfüße zu erfassen. Die Aufteilung 3 vorne, 3 hinten, 1 je Seite entspricht den üblichen Einparksensoren bei Autos. Die 4° Neigung nach oben verhindert Bodenechos (im Test geprüft).
- **GNSS auf dem Deckel:** Eine Satellitenantenne braucht freie Sicht zum Himmel. Der Deckel ist die höchste freie Fläche.
- **IMU in der Rumpfmitte:** Je weiter eine IMU vom Drehpunkt entfernt ist, desto mehr Zusatzbeschleunigung misst sie bei Drehungen. In der Mitte nahe dem Schwerpunkt misst sie die reine Fahrzeugbewegung.

## Sensor-Implementierung in CARLA

Alle 26 Sensoren laufen in CARLA und liefern Daten (Abschlusstest am 09.10.2026, Town02). Die Kameras arbeiten wie im Patent mit 4 Bildern pro Sekunde, Radar, ToF, GNSS und IMU mit rund 34–37 Hz. Die Sensoren werden wie bei allen CARLA-Fahrzeugen zur Laufzeit per Python am Bot montiert, nicht im Blueprint.

**Aufbau**

- `starship_sensors.json` – die physische Ausstattung: je Sensor ID, CARLA-Typ, Einbauposition, Ausrichtung und Parameter. Die ID dient später als ROS2-Frame- und Topic-Name.
- `starship_sensors.py` – spawnt den Bot, montiert alle Sensoren aus der JSON-Datei und sammelt die Messwerte in einer Klasse `SensorHub`. Dazu kommt eine Live-Ansicht mit Steuerung per WASD. Option `--test N` fährt N Sekunden geradeaus und speichert einen Screenshot, `--sync` schaltet auf synchronen 20-Hz-Takt.

| Starship-Sensor | CARLA-Sensor | Umsetzung |
| --- | --- | --- |
| Kameras (1 einzeln + 4 Stereo-Paare) | `sensor.camera.rgb` | 640×480 quer bzw. 480×640 hochkant, 80° bzw. 60°, 4 Bilder/s (`sensor_tick` 0,25 s) |
| Time-of-Flight (3) | `sensor.camera.depth` | 320×240 wie epc660; Werte über 5 m werden verworfen |
| Radar (4) | `sensor.other.radar` | 60° × 20°, 20 m, 1500 Punkte/s |
| Ultraschall (8) | `sensor.other.obstacle` | Sweep 3 m, Radius 15 cm; ohne Ereignis für 0,25 s gilt „frei“ |
| GNSS | `sensor.other.gnss` | Rauschen ca. 2 m horizontal |
| IMU | `sensor.other.imu` | Rauschen 0,02 m/s², 0,002 rad/s |

&#91;image: Live-Ansicht aller Sensoren in Town02: oben Kamera vorne und Stereo-Kameras links, rechts, hinten; Mitte ToF-Tiefenbilder und das 30° nach unten geneigte vordere Stereo-Paar; unten Ultraschallring, Radar-Draufsicht und GNSS/IMU-Werte\]

**Testergebnisse**

| Sensorgruppe | Anzahl | Messrate | Prüfung |
| --- | --- | --- | --- |
| Kameras | 9 | 4,5 Hz | Bilder plausibel, Rate wie im Patent |
| ToF | 3 | ca. 36 Hz | Tiefenbilder plausibel |
| Radar | 4 | ca. 36 Hz | rund 120 Punkte im Bild |
| Ultraschall | 8 | nur bei Treffer | Leitkegel 1,5 m vor dem Bot erkannt: `us_front` meldet 0,81 m (Test vom 08.10.2026); keine Fehlalarme während der Fahrt |
| GNSS / IMU | je 1 | ca. 34 Hz | Werte plausibel |

Die hinteren Ultraschallsensoren erkannten zunächst die Straße als Hindernis, weil der Bot beim Anfahren leicht nickt. Seit alle acht um 4° nach oben geneigt sind, gibt es keine Fehlalarme mehr.

**Nächster Schritt: ROS2-Schnittstellen**

Der `SensorHub` ist die Quelle für die ROS2-Publisher. Geplant sind:

- Topics `/starship/<id>/…` mit Standard-Nachrichten: `Image` + `CameraInfo`, `PointCloud2` für Radar, `Range` für Ultraschall, `NavSatFix`, `Imu`, `Odometry`.
- TF-Baum `map → odom → base_link → <Sensor-ID>` aus den Einbaupositionen.
- Befehlseingang `/starship/cmd_vel` (`Twist`) und Simulationszeit `/clock`.

Zur Wahl stehen die carla-ros-bridge in WSL2 (kein Neubau von CARLA nötig) oder ein Neubau von CARLA mit `--ros2`. Im lokalen Build ist ROS2 derzeit ausgeschaltet (`Ros2 OFF`).

## Quellen

- [Starship Technologies: Our Robots](https://www.starship.xyz/?p=673) – 12 Kameras inkl. ToF, Radar, Ultraschall, „Bubble of Awareness“ (abgerufen 09.10.2026)
- [Wikipedia: Starship Technologies](https://en.wikipedia.org/wiki/Starship_Technologies) – Sensoren, 6 km/h, 25 kg, 18 h Akku
- [Vision Systems Design: Autonomous robot successfully delivers takeout for the first time](https://www.vision-systems.com/embedded/article/16742182/autonomous-robot-successfully-delivers-takeout-for-the-first-time) – 3 Stereo-Paare, 3 ToF, Ultraschall 360°, Tegra K1
- [MIT Technology Review, 30.01.2018](https://www.technologyreview.com/2018/01/30/145935/why-sidewalk-delivery-robots-still-need-safety-drivers-too/amp/) – 9 Kameras, 4 Radare, 8 Ultraschallsensoren
- [The Robot Report, 30.11.2022: ESPROS chips in use in Starship robots](https://www.therobotreport.com/espros-chips-in-use-in-starship-technologies-delivery-robots/) – ToF-Chip epc660
- [ESPROS epc660 Datenblatt (Kopie)](https://atta.szlcsc.com/upload/public/pdf/source/231023/Datasheet_epc660-d67cd1b32b170f852a88b269f1d42613.pdf) – 320 × 240 Pixel, 12-Bit-Distanz
- [CARLA: ROS2 native interface](https://carla-ue5.readthedocs.io/en/latest/ros2_native/) – `--ros2`, `ros_name`, `enable_for_ros()`

* [Patent US 2018/0253107 A1, Starship Technologies](https://patents.google.com/patent/US20180253107A1/) – Kameraanordnung, Stereo-Abstände, Neigung, Sichtfeld, 4 Bilder/s, Geschwindigkeit

- [Patent US 2018/0194411 A1 / US 10 239 378 B2, Starship Technologies](https://patents.google.com/patent/US20180194411A1/en) – Fahrwerk, feste Radachsen bei Kurvenfahrt, einzeln angetriebene Räder
