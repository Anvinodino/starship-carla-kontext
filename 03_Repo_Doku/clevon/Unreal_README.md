# Integration in CARLA

Die Clevon-Assets liegen in zwei Ständen vor. Für die aktuelle Plattform **CARLA 0.10.0 / UE 5.5.4** den
Chaos-Stand verwenden.

| Ordner | Plattform | Fahrphysik | Status |
| --- | --- | --- | --- |
| [`UE5_CARLA-0.10_Chaos/`](UE5_CARLA-0.10_Chaos/) | CARLA 0.10.0, Unreal Engine 5.5.4 | Chaos Vehicles | **aktuell**, Stand des Berichts |
| [`UE4_CARLA-0.9.16_PhysX/`](UE4_CARLA-0.9.16_PhysX/) | CARLA 0.9.16, Unreal Engine 4.26.2 | PhysX (`WheeledVehicleMovementComponent4W`) | früherer Stand, nur zur Nachvollziehbarkeit |

Beide Ordner enthalten die Unterstruktur `Content/Carla/...` und werden in den `Content`-Ordner des jeweiligen
CARLA-Projekts übernommen. Die Stände sind nicht austauschbar: UE4-Fahrzeug-Blueprints laufen nicht unter Chaos,
UE5-Assets lassen sich in UE 4.26 nicht öffnen.

## Unterschiede UE5 (Chaos) gegenüber UE4 (PhysX)

| Asset | Änderung |
| --- | --- |
| `BP_Clevon1` | Chaos-Fahrzeugbewegung, Licht- und Klappenlogik übernommen |
| `BP_Clevon1_FLW/FRW/RLW/RRW` | Radklassen `ChaosVehicleWheel` statt `VehicleWheel` |
| `Curve_Clevon1_Torque`, `Curve_Clevon1_Steering` | neu: Drehmoment- (6-kW-Modell) und Lenkkurve für Chaos |
| `AnimBP_Clevon1_Chaos` | neu: Animations-Blueprint mit Chaos-Wheel-Controller |
| `SK_Clevon1_SK_Clevon1_PhysicsAsset` | Kollisionskörper für Chaos angepasst |
| `SM_Clevon1_Lights`, `SM_Clevon1_Glass`, Glasmaterial | für UE5 überarbeitet |
| übrige 50 Dateien (Meshes, Skelett, Materialien, Texturen) | unverändert aus dem UE4-Stand übernommen |

Hinweis: Die unverändert übernommenen Dateien tragen noch das UE4-Dateiformat. Unreal Engine 5 lädt sie
direkt; sie werden erst beim nächsten Speichern im Editor in das UE5-Format umgeschrieben.

## Vorgehen (CARLA 0.10.0)

1. Mit `git lfs pull` sicherstellen, dass die echten Binärdateien vorliegen.
2. Unreal schließen und den Inhalt von `UE5_CARLA-0.10_Chaos/Content` in
   `~/CarlaUE5/Unreal/CarlaUnreal/Content` übernehmen. Die Unterstruktur `Carla/...` unverändert lassen.
3. Fahrzeug in `Content/Carla/Config/VehicleParameters.json` eintragen (Make `Clevon`, Model `1`,
   Class `/Game/Carla/Blueprints/Vehicles/Clevon1/BP_Clevon1.BP_Clevon1_C`, `NumberOfWheels` 4, `HasLights` true).
   Die Blueprint-ID ist dann `vehicle.clevon.1`.
4. In `Config/DefaultEngine.ini` ergänzen und Unreal neu starten:

```ini
[/Script/Engine.Engine]
NearClipPlane=1.0
```

5. Unreal öffnen, `/Game/Carla/Blueprints/Vehicles/Clevon1/BP_Clevon1` laden, kompilieren, **Play** starten und
   den Fahrclient gemäß Haupt-README ausführen. Fahrzeuggröße, Radkontakt, Licht, Klappe und Sensoransichten prüfen.

Für CARLA 0.9.16 gilt dasselbe mit `UE4_CARLA-0.9.16_PhysX/Content`; die Registrierung erfolgt dort über die
VehicleFactory statt über `VehicleParameters.json`.

## Vollständigkeit

Der Import auf eine unabhängige Installation ist noch nicht geprüft. Im Quellprojekt zeigt
**BP_Clevon1 → Asset Actions → Migrate** die benötigten Abhängigkeiten an. Referenzen auf CARLA-Basisassets
setzen eine passende CARLA-Installation voraus.

## Enthaltene Komponenten

`BP_Clevon1`: Fahrzeug-, Licht- und Klappenlogik. Vier Wheel-Blueprints: Rad-/Federungsparameter.
`Clevon1_CM`: Meshes, Skeleton, Physics Asset, Animations-Blueprint, Materialien und Texturen.
Dieser Ordner enthält keine auszuführenden Skripte. Ein FBX-Neuimport ist für die Übernahme der fertigen Assets nicht erforderlich.
