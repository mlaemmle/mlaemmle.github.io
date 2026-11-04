# wind.sim – Ertragssimulation von Windenergieanlagen

Eine HTML-Seite, die die Windertragsrechnung **direkt im Browser** ausführt (Python/Pyodide, nur numpy und pandas, kein Server-Backend). Vorlage ist das Excel-Tool *Windertragsberechnung* des Umwelt-Campus Birkenfeld (`Windertragsberechnung.xlsx`), die Theorie stammt aus den Vorlesungsfolien (`ee_03_windenergie_stud.pdf`) und steht in der Seite im Reiter **Grundlagen**.

```
wind.sim/
├── wind_sim.html              # Seite; lädt Winddaten und Anlagenkurven bei Bedarf aus data/
├── wind_sim_standalone.html   # dieselbe Seite mit eingebetteten Daten (eine Datei, ~1,4 MB)
├── data/
│   ├── standorte.js/.json     # Metadaten je Standort (Messhöhe, Quelle, Lücken …)
│   ├── <Standort>.csv/.js     # Stundenwerte 2025: wind_speed, wind_dir [, wind_speed_100m, wind_dir_100m]
│   └── anlagen.js/.json       # Leistungskurven (Excel-Vorlage + 8 reale Anlagentypen)
├── build_data.py              # erzeugt data/ (DWD, Open-Meteo/ERA5, Excel-Beispiel, windpowerlib-Kurven)
└── src/                       # Quellen: body.html, theory.html, app.js, wind_model.py, style.css, build.py
```

## Öffnen

* **Per Doppelklick:** `wind_sim.html` (Ordner `data/` muss daneben liegen) oder `wind_sim_standalone.html`. Die Daten werden per `<script>`-Tag geladen, ein Webserver ist nicht nötig.
* **Internet:** Pyodide, Plotly und KaTeX werden vom CDN geladen (erster Start ca. 10–20 s), ebenso Open-Meteo für „Eigener Standort“.
* **GitHub Pages:** Ordner ins Repository legen, nichts zu bauen.
* **Lokaler Server (optional):** `python3 -m http.server 8000` im Ordner `wind.sim`, dann `http://localhost:8000/wind_sim.html`.

## Neu bauen

```bash
python3 build_data.py      # Daten neu erzeugen (benötigt pandas, numpy, openpyxl, windpowerlib, curl)
python3 src/build.py       # beide HTML-Dateien aus src/ zusammensetzen
```

## Modell (wie im Excel-Tool)

1. Stundenwerte der Windgeschwindigkeit in Messhöhe `h_ref`.
2. **Höhenprofil** auf Nabenhöhe `h`:
   * exponentiell (Hellmann): `v = v_ref · (h/h_ref)^α`, `α = 1 / ln(√(h·h_ref) / z0)`
   * logarithmisch: `v = v_ref · ln(h/z0) / ln(h_ref/z0)`
3. **Leistungskennlinie** P(v), lineare Interpolation; Stundenmittel [kW] × 1 h = Ertrag [kWh].
4. Kennzahlen: Jahresertrag, Volllaststunden (= E / P_nenn), Auslastungsfaktor (= Volllaststunden / 8760 h), Weibull-Parameter (Maximum-Likelihood), Windleistungsdichte, Gesamtwirkungsgrad, Ertrag aus Weibull-/Rayleigh-Näherung.

Zusätzlich zum Excel: reale Anlagentypen mit Rotordurchmesser (→ Leistungsbeiwert c_P), optionale Luftdichte-Korrektur (Standardatmosphäre), pauschale Verluste, direkte ERA5-Winddaten in 100 m, Abruf beliebiger Orte, Diagramme (Leistungskurve mit Histogramm, Weibull/Rayleigh, Windrosen, Windprofil, Heatmap). Zeitstempel in UTC, Tages-/Monatswerte in UTC wie im Excel.

### Validierung gegen das Excel

Excel-Beispieldaten (8760 h), 2 MW, Nabenhöhe 100 m, z0 = 0,1 m, generische Kennlinie:

| | Excel | wind.sim |
|---|---|---|
| exponentiell | 3212,77 MWh (1606 h) | 3213,04 MWh (1607 h) |
| logarithmisch | 3256,38 MWh (1628 h) | 3256,57 MWh (1628 h) |

Die Abweichung unter 0,01 % entsteht, weil die Kennlinie hier interpoliert statt als Stufenfunktion (VLOOKUP) gelesen wird. Der Datensatz ist als „Generischer Standort“ im Dropdown enthalten. Die generische Kennlinie hält im Unterschied zur Excel-Kurve zwischen ca. 17 und 25 m/s die Nennleistung (im Excel fällt sie ab 20 m/s stufenweise ab); für diesen Datensatz ändert das den Jahresertrag nicht, weil in Nabenhöhe nie mehr als 20 m/s auftreten.

## Datenquellen (alle Stundenwerte des Kalenderjahres 2025, UTC)

| Eintrag | Quelle | Anemometerhöhe | Ø v (Messhöhe) | Hinweis |
|---|---|---|---|---|
| Generischer Standort | Beispieldatensatz aus der Excel-Vorlage | 10 m | 3,56 m/s | Standort nicht dokumentiert, in der Seite ohne Excel-Bezug beschriftet |
| Berlin-Tempelhof | DWD, Station 433 | 10 m | 3,42 m/s | |
| Hannover | DWD, Station 2014 | 10 m | 3,65 m/s | **361 h Lücke** linear interpoliert |
| Soltau (Lüneburger Heide) | DWD, Station 4745 | 10 m | 2,87 m/s | |
| Cuxhaven | DWD, Station 891 | 10 m | 5,02 m/s | |
| Norderney | DWD, Station 3631 | 11,8 m | 5,87 m/s | Insel |
| Helgoland | DWD, Station 2115 | 10 m | 6,32 m/s | 112 h interpoliert |
| Leuchtturm Kiel | DWD, Station 2961 | 34 m | 8,48 m/s | 192 h interpoliert |
| Kap Arkona (Rügen) | DWD, Station 183 | 24 m | 6,39 m/s | |
| Freiburg | DWD, Station 1443 | 12 m | 2,22 m/s | |
| Feldberg/Schwarzwald | DWD, Station 1346 | 18,4 m | 7,63 m/s | Ersatz für den **Schauinsland** (dort keine aktuellen DWD-Winddaten) |
| Passau (Station Fürstenzell) | DWD, Station 5856 | 12 m | 2,51 m/s | 8 km von Passau |
| Nordsee – Deutsche Bucht | ERA5 (Open-Meteo) | 10 m und 100 m | 7,51 / 9,30 m/s | Offshore-Punkt bei alpha ventus |
| Ostsee – Arkonabecken | ERA5 (Open-Meteo) | 10 m und 100 m | 6,92 / 8,67 m/s | Offshore-Punkt |
| Eigener Standort | ERA5 (Open-Meteo), Live-Abruf | 10 m und 100 m | – | beliebiger Ort, Jahr wählbar |

* **Schauinsland:** Die DWD-Station „Schauinsland (UBA)“ (04424) hat nur Daten bis 1986. ERA5 am Schauinsland-Punkt liefert 2,2 m/s in 10 m und unterschätzt Gipfelwinde stark; deshalb ist der Feldberg (8 km entfernt) als Ersatz eingebaut.
* **Leuchtturm Alte Weser** (DWD, 42,6 m) wurde wegen 27 % fehlender Stunden nicht aufgenommen.
* Die Anemometerhöhe ist die Geberhöhe laut DWD-Stationsmetadaten für Mitte 2025; sie ist in der Seite einstellbar.

Lizenzen: DWD-Daten gemäß GeoNutzV (Quelle: Deutscher Wetterdienst, Climate Data Center). Open-Meteo/ERA5: CC BY 4.0 (Weather data by Open-Meteo.com; Generated using Copernicus Climate Change Service information).

## Anlagentypen

Generische Anlage (normierte Kennlinie, skaliert mit Nennleistung) sowie Enercon E-70/2300, Vestas V90/2000, Enercon E-115/3200, Nordex N131/3600, Enercon E-141/4200, Senvion 6.XM152 (6,3 MW), Vestas V164/8000 und V164/9500 (Offshore). Kennlinien: Herstellerangaben aus der [windpowerlib](https://github.com/wind-python/windpowerlib)-/OEDB-Turbinenbibliothek (Lizenz des Pakets: MIT; Datenquelle bitte bei Veröffentlichung nennen). Enden die Herstellerdaten vor 25 m/s (V90, N131, V164/9500), wird die Kurve bis 25 m/s auf Nennleistung fortgesetzt. Die Typen der Repowering-Übung (E-66/70, E-138 EP3) sind in der Bibliothek nicht enthalten; als Ersatz dienen E-70 und E-141.

## Grenzen

Grobe Abschätzung mit einem Wetterjahr: kein Messmast, keine Orografie/Wald, Höhenprofil nur für neutrale Schichtung, Stundenmittel statt Böen, keine Nachlauf-/Schall-/Schattenverluste (pauschal über „Sonstige Verluste“). Das Excel-Tool ist nicht für Schaltjahre ausgelegt; die Seite rechnet mit der tatsächlichen Stundenzahl.

## Nicht geprüft

Betrieb auf echtem GitHub Pages, per `file://` (nur über lokalen Server und als Standalone-Datei getestet) und in anderen Browsern als dem eingebauten.
