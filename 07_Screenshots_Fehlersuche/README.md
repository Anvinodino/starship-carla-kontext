# Screenshots aus der Fehlersuche: „Bot fällt durch die Straße“ (Nacht 07./08.10.2026, UE4)

Echte Screenshots aus der Sitzung, in der das Durchfallen behoben wurde. Die Editor-Bilder (Unreal Engine 4.26) hat
die KI beim Steuern des Editors aufgenommen. Die Bilder 08 und 09 stammen von einer CARLA-Kamera.
Im Dateinamen steht die Uhrzeit vom 08.10.2026. Gezeichnete Erklärgrafiken zu den Problemen stehen im Word-Bericht
`06_Eigene_Dokumente/Bericht_Starship_Bot_UE4.docx`.

**Ausgangslage:** Version 1 des Modells fiel durch die Straße. Version 2 kam in Unreal 100-mal zu klein an
(0,56 cm statt 56 cm). Grund: Blender rechnet in Metern, Unreal in Zentimetern, und im Export steckte ein Faktor 100.
Eine winzige Kollisionsbox rutscht zwischen zwei Rechenschritten durch den Boden.

## Ablauf

| Bild | Was man sieht | Bedeutung |
|---|---|---|
| `01_0022_PhysicsAsset_automatische_Kapsel` | Physics Asset nach dem Import: Unreal hat automatisch eine große **Kapsel** (lila) um den ganzen Bot gelegt, mit Rädern und Fahne. | **Falsch.** Ein Kollisionskörper, der bis unter die Räder reicht, berührt den Boden selbst. Dann trägt nicht die Federung den Bot. Dieser Stand war noch ein Zwischenschritt (Import mit Skalierung 100 als Notlösung). |
| `02_0022_Box_umfasst_Raeder` | Auf **Box** umgestellt und neu erzeugt (gelb = ausgewählt). Die Box umschließt weiterhin auch die Räder. | **Noch falsch**, gleiche Ursache. |
| `03_0027_neuer_Export_cm_wieder_Kapsel` | Eigentliche Lösung des Maßstabs: Modell in Blender auf echte Zentimeter umgerechnet und ohne Skalierungsfaktor neu exportiert (`starship_carla_v3.fbx`). Nach dem Import erzeugt Unreal wieder automatisch eine Kapsel. | Größe stimmt jetzt (56 × 58 × 88 cm). Die Kollisionsform muss erneut angepasst werden. |
| `04_0027_Box_umfasst_Raeder_und_Fahne` | Box erzeugt, aber noch zu groß: Räder und Fahne liegen darin. | **Falsch**, Box muss kleiner werden. |
| `05_0032_Box_nur_am_Rumpf_RICHTIG` | Box auf 55 × 45 × 33 cm gesetzt, nur um den Rumpf. Unterkante 16 cm über dem Boden, **Räder frei**. | **Richtig.** Die Räder (Federung) tragen den Bot, die Box schützt nur den Rumpf. Die Werte ließen sich mit Python nicht setzen und wurden direkt in die Asset-Datei geschrieben. |
| `06_0035_AnimBP_kompiliert` | Animation Blueprint (dreht und lenkt die Räder sichtbar) auf das neue Skelett umgestellt und kompiliert. | Notwendiger Folgeschritt nach dem Neuimport. |
| `07_0036_BP_Starship_Mesh_und_AnimBP` | Fahrzeug-Blueprint `BP_Starship`: neues Mesh und neues AnimBP eingetragen. | Fahrzeug nutzt jetzt das korrekt skalierte Modell. |
| `08_0045_CARLA_Bot_steht_auf_Strasse` | Kamerabild aus CARLA: Der Bot steht auf der Straße. | **Durchfallen behoben.** |
| `09_0045_CARLA_Vollgas_Bot_bewegt_sich_nicht` | Gleiche Kamera nach mehreren Sekunden Vollgas: Der Bot steht an derselben Stelle. | **Nächstes Problem:** Er fällt nicht mehr durch, fährt aber nicht. Ursache war der gekippte Hauptknochen: Die Federungs-Strahlen zeigten seitwärts, alle Räder galten als „in der Luft“. Die Räder sehen im Bild verdreht aus. |
| `10_0114_v4_Import_wieder_Kapsel` | Import von `starship_carla_v4.fbx`: alle Knochen ohne Drehung (zeigen in +Y, Roll 0). Unreal erzeugt wieder die Standard-Kapsel. | Lösung für das Nicht-Fahren; die Kollisionsform muss erneut gesetzt werden. |
| `11_0116_v4_Box_ueber_Raedern_vor_Korrektur` | Neue Box sitzt noch über den Rädern. | Vor der Korrektur. Danach wurde dieselbe Box wie in Bild 05 gesetzt (55 × 45 × 33 cm, Mitte 23,45 cm über `VehicleBase`). |
| `12_0119_AnimBP_v4` | AnimBP für den v4-Stand. | Danach fuhr der Bot (gegen 01:25 Uhr). |

## Danach (ohne Screenshot in diesem Ordner)

Weitere Korrekturen bis zum fahrenden Bot:
- `VehicleBounds` auf `NoCollision`: Der Bot stand sonst auf dieser Messbox.
- Schwerpunkt von (25, 0, −30) auf (0, 0, −15) cm.
- Später für das HKA-Modell die Tire Config: Ohne sie stürzte CARLA beim Spawnen ab.

Bilder vom fertigen Bot: `../04_Bilder/`.

**Nicht vorhanden:** ein Bild vom tatsächlich durch die Straße fallenden Bot. Die Ursache wurde über Messwerte
(Bounds 0,56 cm statt 56 cm) und Logs gefunden.
