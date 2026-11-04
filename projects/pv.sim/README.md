# PV-Ertragsprognose mit pvlib

Eine einzelne HTML-Seite, die **pvlib direkt im Browser** (Pyodide/WebAssembly) ausführt – kein Server-Backend.
Aus Standort, Anlagengröße, Neigung und Ausrichtung berechnet sie stündlich die Einstrahlung auf die Modulebene, die Zelltemperatur und den Ertrag und fasst die Ergebnisse zu Tages-, Monats- und Jahreswerten zusammen. Vorlage ist der Excel-Rechner (`excel_tool/`).

**Credits:** Prof. Dr. Manuel Lämmle auf Vorlage des Excel-Rechners von Prof. Dr. Henrik te Heesen, Umwelt-Campus Birkenfeld, Institut für Betriebs- und Technologiemanagement (IBT). Erstellt für das Modul „Erneuerbare Energie“ im Studiengang Sustainable Business Development.

```
pvlib_simulator/
├── pvlib_simulator.html             # Oberfläche + eingebettetes Python-Modell; lädt data/*.js bei Bedarf
├── pvlib_simulator_standalone.html  # dieselbe Seite mit eingebetteten Wetterdaten (eine einzelne Datei, ~1,8 MB)
├── data/
│   ├── standorte.js / .json    # Name, Koordinaten, Höhe, Datei, UTC-Offset je Standort
│   ├── <Standort>.js           # Wetterdaten als Script (wird per <script>-Tag geladen, geht auch bei file://)
│   └── <Standort>.csv          # Rohdaten: bhi, dhi, temp_air, wind_speed, pressure_hpa, rel_humidity
└── convert_try_to_csv.py       # erzeugt data/ aus excel_tool/TRY_*.xlsx
```

## Theorie

Die theoretischen Grundlagen (Strahlung, Sonnenstand, Transposition, Zelltemperatur, Leistung, Kennzahlen, Unterschiede zum Excel) stehen direkt in der Seite im Reiter **Grundlagen**.

## Öffnen

* **Einfach per Doppelklick:** `pvlib_simulator.html` (Ordner `data/` muss daneben liegen) oder `pvlib_simulator_standalone.html` (eine Datei, auch für Vorschau-Fenster ohne Basis-URL oder zum Verschicken). Die Wetterdaten werden per `<script>`-Tag geladen, es ist kein Webserver nötig.
* **Internet:** Pyodide, numpy/pandas/scipy und pvlib werden beim Start vom CDN geladen (ca. 20–30 s beim ersten Mal, danach im Browser-Cache). Ohne Internet startet die Seite nicht; Open-Meteo-Abruf braucht ebenfalls Internet.

Alternativ mit lokalem Server:

```bash
cd pvlib_simulator
python3 -m http.server 8000
# dann http://localhost:8000/pvlib_simulator.html öffnen
```

## GitHub Pages

Den Ordner `pvlib_simulator/` ins Repository legen und Pages aktivieren. Es ist nichts zu bauen.
`data/*.js` müssen mit ausgeliefert werden (ca. 1,7 MB gesamt, es wird nur der gewählte Standort geladen).

## Neue TRY-Standorte hinzufügen

1. `TRY_<Name>.xlsx` nach `excel_tool/` legen, Name und Koordinaten in `STANDORTE` in `convert_try_to_csv.py` ergänzen.
2. `python3 pvlib_simulator/convert_try_to_csv.py` ausführen (benötigt `openpyxl`); erzeugt CSV, JSON und JS.
3. Die Standalone-Variante enthält die Daten eingebettet und muss danach neu gebaut werden.

Hinweis: Basel und Lörrach liegen im selben DWD-Gitterfeld und haben deshalb identische Wetterdaten.

## Eigene Standorte (Online-Abruf)

„Eigener Standort“ ruft stündliche Daten der **Open-Meteo Archive API** (ERA5-Reanalyse, kostenlos, ohne API-Key, CORS-fähig) für ein wählbares Jahr ab; die Ortssuche nutzt die Open-Meteo Geocoding-API.
PVGIS (EU-JRC) liefert die besseren TMY-Daten für Europa, sendet aber keine CORS-Header und ist daher nicht direkt aus dem Browser nutzbar.

## Modell

| | pvlib-Standard | Excel-kompatibel |
|---|---|---|
| Sonnenstand | NREL-SPA (Stundenmitte) | gleich |
| Transposition | Perez (allsitescomposite1990) | isotrop, Albedo einstellbar |
| Zelltemperatur | SAPM (mit Wind) je Montageart | T<sub>amb</sub> + ΔT·G/1000 (ΔT je Montageart wie im Excel) |
| Reflexionsverluste | IAM (physikalisch, n = 1,526) | keine |
| Leistung | PVWatts-DC × Derating | gleich |

Zeitkonvention: Zeitreihen intern in UTC, Zeitstempel = Beginn der Stunde. Tages-/Monatswerte werden in Standardzeit des Standorts (TRY: MEZ = UTC+1) aggregiert.

### Abweichungen zum Excel-Tool

Beim Nachrechnen mit identischen Eingaben (Excel-Wetterdaten, `Excel-kompatibel`) zeigt sich:

* Sonnenhöhe und Einfallswinkelformel stimmen überein (Abweichung der Sonnenhöhe < 0,5°).
* Das Excel bestimmt den **Sonnenazimut** über die Stunde der wahren Ortszeit (`<= 12`) – in der Stunde um den Sonnenhöchststand liegt die Sonne dadurch auf der falschen Seite des Meridians (bis ca. 58° Fehler, ~7 % der Tagesstunden).
* Die **reflektierte Strahlung** wird im Excel mit `DNI + DHI` statt mit der Globalstrahlung auf die Horizontale (`BHI + DHI`) berechnet und ist dadurch etwa 40 % zu hoch.

Im Testfall (49,6° N / 7,16° O, 45°, Ost) liegt das pvlib-Ergebnis im Excel-kompatiblen Modus deshalb ca. 4 % unter dem Excel-Wert.
