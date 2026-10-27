# Fahrzeugwerte

Ausgelesen am 18.10.2026 aus dem funktionierenden UE4-Fahrzeug (`Content/Carla/vehicle3`).
Maschinenlesbar: [`config/starship_values.json`](../config/starship_values.json).
Einheiten: cm, kg, Grad, Nm, U/min. Achsen wie in Unreal/CARLA: X vorne, Y rechts, Z oben.

## Geometrie

HKA-Mesh: 56,5 × 55,4 × 65,7 cm (L × B × H, mit Antenne). Radstand 38,2 cm, Spurweite 51,2 cm.

| Knochen | X | Y | Z | Eltern |
| --- | --- | --- | --- | --- |
| VehicleBase | 0 | 0 | 9 | – |
| Wheel_Front_Left | 19,1 | −25,6 | 9 | VehicleBase |
| Wheel_Front_Right | 19,1 | 25,6 | 9 | VehicleBase |
| Wheel_Middle_Left | 0,1 | −25,6 | 9 | VehicleBase |
| Wheel_Middle_Right | 0,1 | 25,6 | 9 | VehicleBase |
| Wheel_Rear_Left | −19,1 | −25,6 | 9 | VehicleBase |
| Wheel_Rear_Right | −19,1 | 25,6 | 9 | VehicleBase |

Materialslots: `dark_gray`, `black`, `hka_body` (im BP auf `dark_gray` überschrieben), `hka_red`, `hka_lid`.

## Fahrzeug (BP_Starship, Parent `BaseVehiclePawn`, `WheeledVehicleMovementComponent4W`)

Ergibt ca. 6 km/h Höchstgeschwindigkeit, wie der echte Starship.

| Bereich | Einstellung | Wert |
| --- | --- | --- |
| Chassis | Mass | 35 kg |
| Chassis | COM Nudge (Mesh → Body Instance) | 0 / 0 / −15 cm |
| Chassis | Chassis Width / Height | 50 / 50 cm |
| Chassis | Drag Coefficient | 0,3 |
| Chassis | Inertia Tensor Scale | 1 / 1 / 1 |
| Chassis | Linear Damping (Mesh) | 0,01 |
| Motor | Max RPM | 720 |
| Motor | MOI | 0,05 |
| Motor | Damping Full Throttle / Zero Throttle eingekuppelt / ausgekuppelt | 0,01 / 0,05 / 0,02 |
| Motor | Torque Curve (U/min → Nm) | 1000 → 10, 1400 → 15, 4200 → 15, 5000 → 12, 6500 → 8 |
| Getriebe | Automatik | an |
| Getriebe | Vorwärtsgänge | 1 Gang: Ratio 1,0, Down 0,5, Up 0,9 |
| Getriebe | Final Ratio / Rückwärts | 3,4 / −3,4 |
| Getriebe | Gear Switch Time / Auto Box Latency | 0,1 s / 0,7 s |
| Getriebe | Neutral Gear Up Ratio / Clutch Strength | 0,15 / 10 |
| Differential | Typ | Open Front Drive (Splits/Bias Standard) |
| Lenkung | Steering Curve | Standard |

Bei Max RPM 720 wirkt nur der erste Punkt der Torque Curve, effektiv ca. 10 Nm.

## Räder (BP_Starship_FLW / FRW / RLW / RRW, Klasse `VehicleWheel`)

| Einstellung | Vorne | Hinten |
| --- | --- | --- |
| Shape Radius / Width | 9 / 4 cm | 9 / 4 cm |
| Mass | 1 kg | 1 kg |
| Damping Rate | 0,25 | 0,25 |
| Steer Angle | 30° | 0° |
| Affected by Handbrake | nein | ja |
| Max Brake / Hand Brake Torque | 20 / 40 Nm | 20 / 40 Nm |
| Suspension Max Raise / Drop | 2 / 2 cm | 2 / 2 cm |
| Suspension Natural Frequency / Damping Ratio | 9 / 0,8 | 9 / 0,8 |
| Lat Stiff Max Load / Value, Long Stiff Value | 1 / 17, 3000 | 1 / 17, 3000 |
| Tire Config | CommonTireConfig | CommonTireConfig |
| Auto Adjust Collision Size / Dont Create Shape | aus / aus | aus / aus |

Wheel Setups im BP: 0 FLW → `Wheel_Front_Left`, 1 FRW → `Wheel_Front_Right`, 2 RLW → `Wheel_Rear_Left`, 3 RRW → `Wheel_Rear_Right`.

## Kollision

| Element | Wert |
| --- | --- |
| Physics Asset | `PA_Starship_v4`: 1 Box auf `VehicleBase`, keine Rad-Bodies |
| Box X × Y × Z | 55 × 45 × 33 cm |
| Box-Mitte (Knochenraum) | 0 / 0 / 23,45 cm → Unterkante 16 cm, Oberkante 49 cm über Boden |
| Collision | Query and Physics, Preset/Object Type Vehicle |
| VehicleBounds (Box-Komponente) | Extent 28 × 23 × 24 cm, Z 69 cm, **NoCollision** |

Hinweis: VehicleBounds stammt noch vom alten Mesh mit Fahne. Für das HKA-Mesh passt eher Extent 28 × 28 × 33 cm bei Z 33 cm.
