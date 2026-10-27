# Tutorial: Einen Lieferroboter in CARLA bauen

Oct 9, 2026 · @Anvin

## Worum es geht

Am Ende dieses Tutorials fährt ein sechsrädriger Lieferroboter nach dem Vorbild des Starship-Roboters durch eine virtuelle Stadt und nimmt seine Umgebung mit Kameras und anderen Sensoren wahr. Das Tutorial erklärt jeden Schritt so, dass man ihn ohne Vorwissen in 3D-Grafik oder Physik nachvollziehen kann: was man tut, warum man es tut und was es bringt.

Drei Programme arbeiten zusammen:

Blender ist ein kostenloses 3D-Programm. Hier entsteht die Form des Roboters, ähnlich wie beim Modellieren mit Knete, nur am Computer.

Unreal Engine ist eine Spiele-Engine, also die Software, mit der Computerspiele gebaut werden. Sie zeichnet die 3D-Welt und berechnet, wie Dinge fallen, rollen und zusammenstoßen.

CARLA ist ein Fahrsimulator für die Forschung an selbstfahrenden Fahrzeugen. Er baut auf Unreal Engine auf, liefert ganze Städte mit Straßen und Verkehr und lässt sich per Programmiersprache Python steuern.

Arbeitsablauf · 4 Stationen

Warum so viel Aufwand? Ein Simulator erlaubt, Software für einen Roboter zu testen, bevor es den Roboter gibt, ohne Risiko für Menschen und beliebig oft. Das klappt aber nur, wenn sich der virtuelle Roboter möglichst wie der echte verhält: gleiche Größe, gleiches Gewicht, gleiches Tempo, Sensoren an den gleichen Stellen. Deshalb steckt in vielen Schritten das Ziel, Werte des echten Roboters möglichst genau zu übernehmen.

## Der echte Starship-Roboter

Der Starship-Roboter ist ein kühlboxgroßer Lieferroboter, der auf Gehwegen in Schrittgeschwindigkeit Essen und Pakete ausliefert. Er wiegt leer etwa 25 kg, trägt bis zu 9 kg und fährt höchstens 6 km/h (Wikipedia). Um im Simulator echt zu wirken, muss unser Modell genau diese Bauteile und Eigenschaften nachbilden.

Bauteile · Seitenansicht

Die Sensoren des echten Roboters: 12 Kameras einschließlich Tiefenkameras, Radar und Ultraschall (Starship), dazu GPS und ein Lagesensor (Wikipedia). Was jeder davon tut, erklärt Schritt 7.

## Grundbegriffe ohne Vorwissen

Wer diese elf Begriffe kennt, versteht jeden folgenden Schritt. Physik-Kenntnisse braucht man nicht; die Vergleiche in der dritten Spalte reichen.

Rund ums 3D-Modell

Rund um die Physik

Rund um Unreal und CARLA

Blueprint: ein Bauplan für ein Objekt in Unreal mit allen Einstellungen, etwa „Rad, 9 cm Radius, 1 kg“. Aus einem Bauplan können beliebig viele Exemplare entstehen.

Asset: jede Datei im Unreal-Projekt, zum Beispiel ein Mesh, ein Material oder ein Blueprint.

Play: startet die Simulation im Editor. Erst dann läuft die Physik, und CARLA nimmt Befehle an.

Spawnen: ein Fahrzeug oder einen Sensor in der laufenden Simulation erscheinen lassen.

## Schritt 1 – Das Modell in Blender vorbereiten

Ziel: Ein 3D-Modell in der richtigen Größe, mit einem Skelett aus sieben Knochen, das Unreal ohne Missverständnisse übernimmt. Fast alle späteren Probleme lassen sich auf diesen Schritt zurückführen, deshalb lohnt hier Sorgfalt.

### 1.1 Die richtige Größe: Zentimeter statt Meter

Was man tut: In Blender die Szenen-Einheit auf 0,01 stellen (Scene Properties → Units → Unit Scale) und Modell und Knochen so skalieren, dass die Zahlen echte Zentimeter sind: Der Roboter ist 56 cm lang, ein Rad hat 9 cm Radius.

Warum: Blender rechnet standardmäßig in Metern, Unreal in Zentimetern. Steht in der Datei „0,56“, liest Unreal 0,56 cm. Der Roboter wäre winzig, seine Kollisionsform rutscht durch den Boden.

Was es bringt: Beide Programme lesen dieselbe Zahl als dieselbe Länge.

### 1.2 Das Skelett anlegen

Was man tut: Eine Armature (so heißt ein Skelett in Blender) mit sieben Knochen anlegen und die Knochen genau so benennen:

Jeder Rad-Knochen sitzt genau in der Radmitte, alle hängen am Rumpf-Knochen. Die Armature selbst heißt Armature, dann überspringt Unreal sie beim Import und macht VehicleBase zum obersten Knochen.

Warum: Unreal bewegt später jedes Rad über seinen Knochen: drehen beim Fahren, schwenken beim Lenken. Ohne eigenen Knochen wäre ein Rad fest mit dem Rumpf verschweißt.

### 1.3 Die Gewichtung

Was man tut: Jeden Teil des Modells genau einem Knochen zuordnen: Radgeometrie zu 100 % zum Rad-Knochen, alles andere zu VehicleBase (Vertex Groups mit den Knochennamen).

Warum: Nur so dreht sich beim Drehen eines Rad-Knochens auch nur dieses Rad und nicht ein Stück Rumpf.

### 1.4 Die Knochen richtig ausrichten

Was man tut: Jeden Knochen so legen, dass er waagerecht nach +Y zeigt (Spitze 15 cm in Y-Richtung vom Kopf), und den Roll-Winkel auf 0 stellen.

Warum: Jeder Knochen hat ein eigenes Achsenkreuz. Die Physik in Unreal nimmt das Achsenkreuz des Rumpf-Knochens als „Richtung des Fahrzeugs“: X ist vorne, Z ist oben. Zeigt der Knochen nach oben, ist sein Z nach dem Export seitlich gekippt. Dann sucht die Federung den Boden zur Seite hin, findet ihn nie, und der Roboter fährt nicht.

Knochen-Ausrichtung · falsch und richtig

Das sieht in Blender ungewohnt aus, weil alle Knochen zur Seite zeigen. Für Unreal ist es genau richtig.

### 1.5 Exportieren als FBX

File → Export → FBX mit diesen Einstellungen:

Gestaltung: In unserem Projekt bekam das Modell zusätzlich ein eigenes HKA-Design (anthrazit, roter Schriftzug, Antenne mit Signallicht), um sich optisch vom Starship-Original abzugrenzen.

## Schritt 2 – Import in Unreal und die Physik-Box

Ziel: Das Modell liegt als bewegliches Objekt im Unreal-Projekt und hat eine einfache Kollisionsform, mit der die Physik rechnen kann.

### 2.1 Importieren

Was man tut: Im CARLA-Editor im Content Browser in einen eigenen Ordner wechseln (bei uns Content/Carla/vehicle3), Import wählen und die FBX-Datei öffnen.

Kontrolle: Das Mesh per Doppelklick öffnen. Die Maße sollten etwa 56 × 58 × 66 cm betragen. Steht dort ein Hundertstel oder das Hundertfache, ist beim Export der Maßstab falsch, dann zurück zu Schritt 1.1.

Unreal legt drei Dinge an: das Skeletal Mesh (Form und Farben), das Skeleton (die Knochen) und das Physics Asset (die Kollisionsform).

### 2.2 Die Physik-Box einstellen

Was man tut: Das Physics Asset öffnen. Im Werkzeugbereich Primitive Type auf Box stellen, Auto Orient to Bone ausschalten und Generate All Bodies klicken. Danach die Box so einstellen, dass sie nur den Rumpf umschließt:

Warum eine einfache Box: Die Physik prüft viele Male pro Sekunde, ob etwas zusammenstößt. Mit 1700 Eckpunkten wäre das langsam und wackelig, mit den 8 Ecken einer Box geht es schnell und stabil. Für die Frage „Streift der Roboter eine Wand?“ reicht eine Box völlig.

Warum die Box nicht bis zum Boden reicht: Den Roboter sollen die Räder tragen, nicht die Box. Reicht die Box bis auf die Straße, schleift der Rumpf wie ein Schlitten, und die Räder verlieren den Halt. 16 cm Abstand lassen den Federn Platz.

Warum keine Kollisionsformen an den Rädern: Um die Räder kümmert sich die Fahrzeug-Physik selbst (Schritt 3). Eigene Boxen an den Rad-Knochen würden mit der Federung in Konflikt geraten.

## Schritt 3 – Die Räder einstellen

Ziel: Die Physik weiß, wie groß, schwer, gefedert und griffig jedes Rad ist. Das legt man in vier kleinen Bauplänen fest: einen pro Rad, das wirklich fährt.

Warum nur vier Räder? Die Fahrzeug-Physik von Unreal kennt nur Fahrzeuge mit vier Rädern. Wir geben ihr deshalb die Vorder- und Hinterräder. Die beiden Mittelräder drehen nur optisch mit (Schritt 4). Für das Fahrverhalten macht das kaum einen Unterschied, weil der Roboter langsam und auf ebenen Wegen fährt.

Was man tut: Im Content Browser per Rechtsklick Blueprint Class wählen, als Elternklasse VehicleWheel suchen und vier Baupläne anlegen: BP_Starship_FLW (vorne links), FRW (vorne rechts), RLW (hinten links), RRW (hinten rechts). In jedem die folgenden Werte eintragen.

Federung kurz erklärt: Jedes Rad hängt an einer virtuellen Feder. Die Frequenz sagt, wie hart die Feder ist: Ein weiches Sofa federt langsam, ein Trampolin schnell. Die Dämpfung sagt, wie schnell das Wippen danach aufhört: Ohne Dämpfung hüpft der Roboter nach jeder Bodenwelle weiter, mit zu viel Dämpfung ist er bockig.

## Schritt 4 – Die Räder sichtbar drehen lassen

Ziel: Die Räder drehen sich im Bild mit, wenn der Roboter fährt, und die Vorderräder schwenken beim Lenken.

Warum ein eigener Schritt? Die Physik rechnet zwar aus, wie schnell sich jedes Rad dreht, bewegt aber nicht das Bild. Das übernimmt ein Animation Blueprint: ein kleiner Ablaufplan, der in jedem Bild die Knochen des Skeletts in die passende Stellung bringt. Ohne ihn würde der Roboter mit stillstehenden Rädern über die Straße gleiten.

Was man tut: Rechtsklick auf das Skeleton → Create → Anim Blueprint, als Elternklasse VehicleAnimInstance wählen. Im AnimGraph diese Bausteine von links nach rechts verbinden:

Mesh Space Ref Pose – die Grundstellung des Skeletts als Ausgangspunkt.

Wheel Handler – holt sich aus der Physik Drehung und Lenkwinkel der vier physikalischen Räder und dreht deren Knochen.

Copy Bone von Wheel_Rear_Right nach Wheel_Middle_Right, nur Copy Rotation an – das rechte Mittelrad dreht sich genau wie das Hinterrad.

Copy Bone von Wheel_Rear_Left nach Wheel_Middle_Left – dasselbe links.

Component To Local und Output Pose – wandelt das Ergebnis um und gibt es aus.

Danach Compile und Save klicken.

Warum kopieren die Mittelräder von hinten? Die Physik kennt nur vier Räder (Schritt 3). Die Mittelräder haben deshalb keine eigene Drehzahl. Da alle Räder einer Seite ungefähr gleich schnell rollen, übernehmen sie die Drehung des Hinterrads. Das Hinterrad statt des Vorderrads, weil es nicht lenkt: Sonst würden die Mittelräder beim Abbiegen mit einschwenken.

## Schritt 5 – Das Fahrzeug einstellen

Ziel: Ein Bauplan für das ganze Fahrzeug, der Modell, Räder, Animation, Gewicht und Motor zusammenführt.

Was man tut: Einen Blueprint mit der CARLA-Elternklasse BaseVehiclePawn anlegen (bei uns BP_Starship). Diese Klasse bringt alles mit, was CARLA von einem Fahrzeug erwartet. Dann in den Class Defaults eintragen:

### 5.1 Modell, Animation und Räder zuordnen

Die Reihenfolge vorne links, vorne rechts, hinten links, hinten rechts ist vorgeschrieben, sonst verwechselt die Physik Vorder- und Hinterachse.

### 5.2 Gewicht und Schwerpunkt

Achtung: Der Schwerpunkt muss zwischen Vorder- und Hinterachse liegen. In unserer Vorlage lag er 25 cm vorne, also vor den Vorderrädern. Dann verteilt die Physik das Gewicht unsinnig auf die Räder.

### 5.3 Motor und Getriebe

Auch ein Elektroroboter wird in Unreal wie ein Auto mit Motor und Getriebe beschrieben. Die Werte sind so gewählt, dass der Bot wie das Original höchstens etwa 6 km/h fährt.

Vom Motor zum Tempo · Rechenweg

### 5.4 Die Messbox ausschalten

CARLA legt um jedes Fahrzeug eine Box namens VehicleBounds. Sie gibt die Größe des Fahrzeugs an, etwa für Sensoren. Ihre Collision Presets müssen auf NoCollision stehen. Sonst steht der Roboter auf dieser Box wie auf einem Sockel, die Räder tragen kein Gewicht, und er fährt keinen Millimeter.

### 5.5 Lenkung

Der Simulationsbot lenkt wie ein Auto über die Vorderräder. Der echte Starship lenkt nach seinem Fahrwerkspatent sehr wahrscheinlich wie ein Panzer, also über unterschiedlich schnelle Räder links und rechts. Unreal und CARLA bieten von Haus aus nur die Auto-Lenkung an. Panzerlenkung hätte eigenen C++-Code und einen Neubau von CARLA erfordert. Für Sensortests ist der Unterschied gering; der Simulationsbot kann nur nicht auf der Stelle drehen.

## Schritt 6 – In CARLA anmelden, testen und fahren

Ziel: CARLA kennt das Fahrzeug unter einem Namen und kann es auf Befehl in die Stadt setzen.

### 6.1 Anmelden

Was man tut: Im Ordner Content/Carla/Blueprints/Vehicles die VehicleFactory öffnen und einen neuen Eintrag anlegen: Make starship, Model bot, Class BP_Starship. Speichern.

Warum: CARLA führt eine Liste aller Fahrzeuge, die per Programm gerufen werden können. Erst mit dem Eintrag gibt es den Roboter unter dem Namen vehicle.starship.bot.

### 6.2 Simulation starten

Im Editor die Karte laden und Play drücken (Alt+P). Erst jetzt läuft die Physik, und CARLA nimmt Befehle an.

Tipp: Die große Karte Town10 braucht auf einem Rechner mit 16 GB Arbeitsspeicher fast den ganzen Speicher und kann den Editor abstürzen lassen. Zum Testen eignet sich die kleinere Karte Town02: Sie startet in rund 25 Sekunden statt mehrerer Minuten.

### 6.3 Fahren

In einem Terminal (PowerShell) nacheinander eingeben:

cd C:\Users\letie\Downloads

py -3.11 starship_drive.py

Es öffnet sich ein Fenster mit zwei Bildern: links die Sicht vom Roboter aus, rechts eine Kamera, die dem Roboter folgt. Mit V wechselt man zwischen Ansicht von hinten oben und Draufsicht.

Alternativ geht auch das CARLA-Beispiel manual_control.py --filter vehicle.starship.bot im Ordner D:\CARLA\PythonAPI\examples.

### 6.4 Woran man erkennt, dass alles stimmt

Der Roboter steht nach dem Erscheinen ruhig auf seinen Rädern, fällt nicht durch den Boden und schwebt nicht.

Bei Gas rollt er los und erreicht nach etwa einer Sekunde rund 6 km/h.

Die Räder drehen sich sichtbar, die Vorderräder schwenken beim Lenken.

Bei Bremse hält er an.

Wenn etwas davon nicht klappt, hilft die Fehler-Checkliste am Ende.

## Schritt 7 – Sensoren anbringen

Ziel: Der Roboter bekommt dieselben Sinne wie das Original, damit man Software zum Erkennen von Hindernissen und zum Navigieren testen kann.

### 7.1 Was die Sensoren tun

Jeder Sensortyp hat Stärken und Schwächen. Erst zusammen ergeben sie ein verlässliches Bild, so wie Menschen Sehen, Hören und Tasten kombinieren.

### 7.2 Wo sie sitzen und warum

Sensorpositionen · Draufsicht

Kameras auf 40–42 cm, rundum: so hoch wie möglich am Rumpf, damit sie weit sehen und die Räder nicht im Bild sind. Die Anordnung (eine Einzelkamera und ein nach unten geneigtes Paar vorne, je ein Paar an den Seiten und hinten) stammt aus einem Starship-Patent (US 2018/0253107 A1).

Tiefenkameras auf 30 cm, leicht nach unten geneigt: Sie sollen Bordsteine, Stufen und Löcher kurz vor dem Roboter finden. Bei 15° Neigung sieht ihre Bildmitte etwa 1,1 m vor dem Bot auf den Boden. Vorne und seitlich, weil der Roboter dorthin fährt und abbiegt.

Radar auf 35 cm, an jeder Seite: für Radfahrer und Autos, die sich nähern, auch bei schlechtem Wetter.

Ultraschall auf 22 cm, als Ring: unten am Rumpf, damit auch niedrige Hindernisse wie Poller oder Tiere auffallen. Die Aufteilung (3 vorne, 3 hinten, je 1 seitlich) entspricht der Einparkhilfe von Autos. Sie sind 4° nach oben geneigt, sonst hält der Sensor beim Anfahren die Straße für ein Hindernis.

GPS auf dem Deckel: Eine Satellitenantenne braucht freie Sicht zum Himmel.

Lagesensor in der Mitte: Dort misst er die Bewegung des ganzen Roboters, nicht zusätzliches Schwanken einer Ecke.

### 7.3 Wie man sie in CARLA anbringt

In CARLA werden Sensoren nicht in den Fahrzeug-Bauplan eingebaut, sondern während der Simulation per Python „angeschraubt“ (Fachwort: attach). So machen es alle CARLA-Fahrzeuge. Der Vorteil: Man kann Sensoren ändern, ohne Unreal zu öffnen.

Konfigurationsdatei starship_sensors.json: Pro Sensor steht dort Name, Typ, Position in Metern (x vorne, y rechts, z oben, Ursprung am Boden unter der Fahrzeugmitte), Blickrichtung und Einstellungen wie Bildgröße oder Reichweite.

Skript starship_sensors.py: Es lässt den Roboter erscheinen, liest die Datei und bringt jeden Sensor an. Im Kern sind das drei Zeilen pro Sensor:

bp = world.get_blueprint_library().find("sensor.camera.rgb")   # Sensortyp wählen
bp.set_attribute("fov", "80")                                   # Einstellung setzen
cam = world.spawn_actor(bp, carla.Transform(carla.Location(x=0.28, z=0.42)), attach_to=robot)  # anschrauben

Daten abholen: Jeder Sensor ruft bei jeder Messung eine Funktion auf (cam.listen(...)), die das Bild oder die Messwerte entgegennimmt. Das Skript zeigt alles in einem Fenster an.

Starten (Play muss laufen):

py -3.11 starship_sensors.py --spawn 0

Nächster Schritt: Die Messwerte sammelt das Skript bereits an einer Stelle. Von dort aus werden sie später über ROS2 an Roboter-Software weitergegeben.

## Typische Fehler und was dahintersteckt

Alle diese Fehler sind beim Bau dieses Roboters tatsächlich aufgetreten. Die Tabelle beginnt beim Symptom, also bei dem, was man sieht.

Zwei Faustregeln:

Nach jedem größeren Schritt einmal testen. Dann weiß man, welcher Schritt einen Fehler verursacht hat.

Physik-Einstellungen nur im Bauplan in Unreal ändern und speichern. Änderungen über CARLA-Befehle während der Simulation, etwa apply_physics_control, haben bei diesem Roboter Werte zerstört und später Abstürze ausgelöst.

## Quellen

Starship Technologies: Our Robots – 12 Kameras inkl. Tiefenkameras, Radar, Ultraschall, sechs Räder mit Wippen

Wikipedia: Starship Technologies – Gewicht, Zuladung, Höchstgeschwindigkeit, GPS und Lagesensor

Patent US 2018/0253107 A1, Starship Technologies – Anordnung der Kameras

Patent US 2018/0194411 A1, Starship Technologies – Fahrwerk und Lenkung

The Robot Report: ESPROS-Chips in Starship-Robotern – Tiefenkamera-Chip

MIT Technology Review, 30.01.2018 – Anzahl Radare und Ultraschallsensoren

Unreal Engine 4.26, Quellcode des Plugins PhysXVehicles – Standardwerte für Räder und Fahrzeuge

Ausführliche Belege und die technische Fehleranalyse stehen im Bericht „Starship-Bot in CARLA – Umsetzung & Sensorik“.


Bauteil | Aufgabe | Was das für die Simulation bedeutet
Rumpf mit Deckel | schützt Ladung, Akku und Elektronik | bestimmt Größe und Gewicht; wird zur Kollisionsbox
6 Räder | fahren, lenken, tragen das Gewicht | jedes Rad braucht einen eigenen Knochen im Skelett
Wippe (Bogie) | lässt Mittel- und Hinterrad gemeinsam kippen, damit der Roboter Bordsteine überwindet (Starship) | wird im Simulator vereinfacht, die Federung jedes Rads übernimmt die Aufgabe
Sensoren | Kameras, Radar, Ultraschall, GPS und Lagesensor erfassen Umgebung und Position | werden in CARLA als virtuelle Sensoren angebracht (Schritt 7)
Antenne | Funkverbindung zur Fernaufsicht, Sichtbarkeit | nur optisch


Begriff | Was es ist | Vergleich aus dem Alltag
Mesh (Gitternetz) | Die Oberfläche des Modells, zusammengesetzt aus tausenden kleinen Dreiecken. Unser Roboter hat rund 1700 Eckpunkte. | Ein Drahtgeflecht, über das man Stoff spannt
Material | Legt fest, wie eine Fläche aussieht: Farbe, Glanz, rau oder glatt. | Lack und Folie auf dem Drahtgeflecht
Knochen (Bone) und Skelett | Unsichtbare Stäbe im Modell. Bewegt sich ein Knochen, bewegen sich die daran „angehängten“ Teile mit. Unser Skelett hat 7 Knochen: einen für den Rumpf, je einen pro Rad. | Die Knochen in einer Hand: Drehst du das Handgelenk, drehen sich die Finger mit
Gewichtung | Legt fest, welcher Teil des Modells an welchem Knochen hängt. | Die Sehnen, die Haut an den Knochen festhalten
Achsenkreuz | Jeder Knochen hat drei Richtungen: vorne, links, oben. | Ein Kompass mit Höhenangabe
FBX-Datei | Austauschformat, in dem Blender das Modell samt Skelett an Unreal übergibt. | Ein Umzugskarton mit Beschriftung


Begriff | Was es ist | Vergleich aus dem Alltag
Physik-Engine (PhysX) | Rechenprogramm in Unreal, das ausrechnet, wie Dinge fallen, rollen und zusammenstoßen. | Ein unsichtbarer Schiedsrichter, der jede Zehntelsekunde entscheidet, wo alles ist
Kollisionsform | Vereinfachte, unsichtbare Hülle, mit der die Physik rechnet. Bei uns eine Box um den Rumpf. Das detaillierte Mesh wäre zu aufwendig. | Ein Umzugskarton um eine Vase: Zum Stapeln zählt nur der Karton
Federung mit Messstrahl | Die Physik schickt von jedem Rad einen unsichtbaren Strahl nach unten. Trifft er die Straße, drückt eine virtuelle Feder das Fahrzeug hoch, und das Rad kann Kraft auf die Straße bringen. | Ein Blindenstock, der den Boden ertastet
Schwerpunkt | Der Punkt, an dem das ganze Gewicht „wirkt“. Liegt er hoch, kippt ein Fahrzeug leichter. | Ein Tablett kippt, wenn die schweren Gläser am Rand stehen
Motor, Drehzahl und Übersetzung | Der Motor dreht schnell, die Räder langsamer. Die Übersetzung legt fest, wie viel langsamer. Begrenzt man die Motor-Drehzahl, begrenzt man damit auch das Tempo. | Die Gänge am Fahrrad: kleiner Gang, viel Kraft, wenig Tempo


Knochen | Sitzt bei (cm) | Trägt
VehicleBase | Mitte, 9 cm hoch | Rumpf, Deckel, Antenne
Wheel_Front_Left / _Right | 19 cm vor der Mitte, 25,6 cm links bzw. rechts | Vorderrad
Wheel_Middle_Left / _Right | Mitte, außen | Mittelrad
Wheel_Rear_Left / _Right | 19 cm hinter der Mitte, außen | Hinterrad


Einstellung | Wert | Warum
Selected Objects | an (Armature und Modell markieren) | nur der Roboter, keine Kamera oder Lampe
Object Types | Armature, Mesh | Skelett und Form, sonst nichts
Apply Scalings | All Local, Scale 1,0 | keine versteckten Größenfaktoren in der Datei
Forward / Up | −Z / Y | Standard für Unreal
Primary / Secondary Bone Axis | Y / X | passt zur Knochen-Ausrichtung aus 1.4
Add Leaf Bones | aus | keine überflüssigen Zusatzknochen
Bake Animation | aus | der Roboter hat keine fertigen Bewegungsabläufe


Option | Wert | Warum
Import as Skeletal | an | Ein „Skeletal Mesh“ ist ein Modell mit Skelett, also mit beweglichen Teilen. Ein normales Modell wäre starr.
Skeleton | leer lassen | Unreal legt ein neues Skelett nach unseren 7 Knochen an.
Create Physics Asset | an | Erzeugt eine erste Kollisionsform, die wir in 2.2 anpassen.
Import Animations | aus | Bewegungen berechnet später die Physik, keine Vorlage.
Import Uniform Scale | 1,0 | Die Größe stimmt schon aus Blender.


Wert | Einstellung | Bedeutung
Größe | 55 × 45 × 33 cm | Länge × Breite × Höhe des Rumpfs
Mitte | 23,45 cm über dem Rumpf-Knochen | Box reicht von 16 bis 49 cm über dem Boden
Hängt an | VehicleBase | bewegt sich mit dem Rumpf


Wert | Bei uns | Was er bedeutet | Warum diese Zahl
Shape Radius / Width | 9 cm / 4 cm | Halbmesser und Breite des Rades | am Modell gemessen
Auto Adjust Collision Size | aus | Unreal soll die Radgröße nicht selbst messen | die Automatik maß bei uns die falsche Seite des Rades und kam auf 4 cm statt 9 cm
Mass | 1 kg | Gewicht eines Rades | kleines Kunststoffrad; der Unreal-Standard von 20 kg gilt für Autoräder
Steer Angle | vorne 30°, hinten 0° | wie weit das Rad einlenkt | vorne lenken, hinten nicht, wie beim Auto
Affected by Handbrake | vorne aus, hinten an | ob die Handbremse dieses Rad bremst | wie beim Auto: Handbremse auf der Hinterachse
Max Brake Torque | 20 Nm | wie fest die Bremse zupacken kann | auf das geringe Gewicht heruntergerechnet; Autos haben 1500 Nm
Max Hand Brake Torque | 40 Nm | Stärke der Handbremse | doppelte Bremskraft, damit der Bot sicher stehen bleibt
Suspension Max Raise / Drop | 2 cm / 2 cm | wie weit die Feder ein- und ausfedern kann | kleiner Roboter auf glatten Wegen; Autos haben 10 cm
Suspension Natural Frequency | 9 | wie straff die Feder ist; höher heißt härter | eher straff, damit der hohe, schmale Bot wenig schaukelt
Suspension Damping Ratio | 0,8 | wie schnell das Nachschwingen aufhört | knapp unter 1 hört die Bewegung schnell und ohne Wippen auf
Tire Config | CommonTireConfig | Reifen-Grip-Einstellung | der CARLA-Standard; ohne Eintrag stürzt CARLA beim Start ab
Lat Stiff / Long Stiff | 1 / 17 / 3000 | wie stark der Reifen seitlich und längs greift | aus einer Fahrzeugvorlage übernommen, im Test unauffällig
Dont Create Shape | aus | die Physik bekommt eine Radform für Zusammenstöße | mit eingeschalteter Option stürzte unser Modell beim Start ab


Einstellung | Wert | Wozu
Skeletal Mesh | unser Modell aus Schritt 2 | die Form
Anim Class | der Animation Blueprint aus Schritt 4 | drehende Räder im Bild
Wheel Setups 0–3 | BP_Starship_FLW → Knochen Wheel_Front_Left, FRW → Wheel_Front_Right, RLW → Wheel_Rear_Left, RRW → Wheel_Rear_Right | sagt der Physik, welcher Rad-Bauplan an welchem Knochen sitzt


Einstellung | Wert | Bedeutung | Warum
Mass | 35 kg | Gesamtgewicht | 25 kg leer plus rund 10 kg Ladung, wie beim Original
Center Of Mass Offset | (0, 0, −15) cm | verschiebt den Schwerpunkt 15 cm nach unten | ein tiefer Schwerpunkt macht den hohen, schmalen Bot kippstabil, wie ein schwerer Akku unten im Rumpf


Einstellung | Wert | Bedeutung | Warum
Torque Curve | 10–15 Nm | Kraft des Motors | reicht, um in rund 1 s auf Tempo zu kommen
Max RPM | 720 U/min | höchste Motordrehzahl | begrenzt das Tempo, siehe Rechenweg unten
MOI | 0,05 | Trägheit des Motors, also wie schwer er in Schwung kommt | der Auto-Standard 1,0 wirkt bei 9-cm-Rädern wie ein schweres Schwungrad
Damping Rates | 0,01 / 0,05 / 0,02 | wie stark der Motor von selbst abbremst | passend zur kleinen Trägheit verkleinert
Forward Gears | ein Gang, Übersetzung 1,0 | Getriebe | ein Elektroroboter hat kein Schaltgetriebe
Final Ratio | 3,4 | feste Untersetzung zwischen Motor und Rad | ergibt mit 720 U/min das Zieltempo


Taste | Wirkung
W / S | Gas / Bremse
A / D | links / rechts lenken
Q | Rückwärtsgang an/aus
Leertaste | Handbremse
Backspace | am nächsten Startpunkt neu erscheinen
ESC | beenden


Sensor | Anzahl | Was er macht | Stärke | Schwäche | Vergleich
Kamera | 9 | macht Farbbilder, 4 pro Sekunde | erkennt, was etwas ist: Mensch, Auto, Ampel | sieht schlecht im Dunkeln; Entfernungen nur durch Rechnen | unsere Augen
Stereo-Kamerapaar | 4 Paare | zwei Kameras nebeneinander | aus dem Versatz der zwei Bilder lässt sich die Entfernung berechnen | rechenaufwendig | zwei Augen sehen räumlich
Tiefenkamera (Time-of-Flight) | 3 | schickt Licht aus und misst, wie lange es zurückbraucht; liefert ein Bild aus Entfernungen | misst Abstände direkt, auch bei Dunkelheit | nur wenige Meter Reichweite | eine Fledermaus, nur mit Licht
Radar | 4 | schickt Funkwellen aus und empfängt das Echo | misst Entfernung und Tempo bewegter Dinge, auch bei Regen und Nebel | grobes Bild, erkennt keine Formen | die Radarfalle der Polizei
Ultraschall | 8 | schickt Töne oberhalb des Hörbereichs aus und hört das Echo | billig und zuverlässig im Nahbereich bis etwa 3 m | sehr kurze Reichweite | die Einparkhilfe im Auto
GNSS (GPS) | 1 | empfängt Satellitensignale und berechnet die Position auf der Erde | weiß, wo auf der Welt der Roboter ist | auf einige Meter ungenau | das Navi im Handy
IMU (Lagesensor) | 1 | misst Beschleunigung und Drehung | sehr schnell und genau für kurze Zeit | Fehler summieren sich mit der Zeit | der Gleichgewichtssinn im Innenohr


Sensor am echten Roboter | CARLA-Sensortyp
Kamera | sensor.camera.rgb
Tiefenkamera | sensor.camera.depth (Werte über 5 m werden verworfen)
Radar | sensor.other.radar
Ultraschall | sensor.other.obstacle (CARLA hat keinen eigenen Ultraschall; dieser Sensor meldet Hindernisse auf einer Linie, das kommt dem am nächsten)
GPS | sensor.other.gnss
Lagesensor | sensor.other.imu


Was man sieht | Ursache | Lösung | Schritt
Roboter fällt durch die Straße | Modell und Kollisionsbox sind 100-mal zu klein, weil Meter als Zentimeter gelesen wurden | Maße in Blender auf echte Zentimeter umrechnen, Skalierung 1 | 1.1
Modell ist in Unreal winzig oder riesig | falscher Maßstab beim Export | wie oben; nach dem Import die Maße prüfen | 1.1, 2.1
Roboter sitzt zu tief, Räder stecken im Boden | Unreal hat die Radgröße selbst gemessen und sich geirrt | Auto Adjust Collision Size aus, Radius von Hand eintragen | 3
Roboter steht, fährt aber keinen Millimeter | die Messbox VehicleBounds trägt ihn wie ein Sockel | Kollision der Messbox auf NoCollision | 5.4
Motor heult hoch, Räder „in der Luft“, kein Vortrieb | Rumpf-Knochen ist gekippt, die Federung sucht den Boden zur Seite | Knochen in Blender nach +Y mit Roll 0 ausrichten | 1.4
Fahrverhalten unruhig, Hinterräder ohne Last | Schwerpunkt liegt vor der Vorderachse | Schwerpunkt mittig und tief setzen | 5.2
Unreal stürzt ab, sobald der Roboter erscheint | ein Rad hat keine Reifen-Einstellung (Tire Config) | allen Rädern CommonTireConfig zuweisen | 3
Roboter fährt dreimal so schnell wie das Original | Motor und Getriebe stammen aus einer Auto-Vorlage | ein Gang, Drehzahl auf 720 U/min begrenzen | 5.3
Ultraschall meldet beim Anfahren ein Hindernis | der Sensor sieht die Straße, weil der Bot beim Anfahren leicht nickt | Sensoren 4° nach oben neigen | 7.2
Editor stürzt beim Laden oder bei Play ab | Arbeitsspeicher voll (große Karte Town10) | kleinere Karte Town02 verwenden, nie zwei Editoren gleichzeitig | 6.2
Räder gleiten, ohne sich zu drehen | Animation Blueprint fehlt oder ist nicht zugewiesen | AnimBP anlegen und im Fahrzeug unter Anim Class eintragen | 4, 5.1