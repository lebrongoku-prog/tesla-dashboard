# Tesla Ladehistorie Dashboard

Kompakter Dauerkontext für Claude Code. Bitte bei jeder Änderung befolgen.

## Was das ist

Interaktives Web-Dashboard für Leonards Tesla-Ladehistorie: KPIs, Charts, Tabelle, Filter, Insights, PWA-fähig auf iOS.

- **Tech-Stack:** Single-file HTML mit inline CSS + Vanilla JS. Chart.js 4 als einzige externe Abhängigkeit (per CDN, mit SRI-Integrity). Kein Build-Prozess.
- **Deploy-Ziel:** GitHub Pages
  - Repo: `https://github.com/lebrongoku-prog/tesla-dashboard`
  - Live-URL: `https://lebrongoku-prog.github.io/tesla-dashboard/`
- **Datenquelle:** Google Sheet, gelesen im Frontend via Google Sheets API (OAuth Implicit Flow). Import in das Sheet passiert via Google Apps Script Web App (wöchentlicher Trigger + manueller Refresh-Button).
- **Nutzer:** Leonard (Schweizerdeutsch), fortgeschrittener Anwender ohne Entwicklerhintergrund. Möchte fertige Lösungen, keine schrittweisen Debug-Anleitungen.

## Dateistruktur

Im Projektroot:

| Datei/Ordner | Zweck | Ins Repo? |
|---|---|---|
| `index.html` | Das komplette Dashboard (HTML + CSS + JS in einer Datei) | ✅ Ja |
| `manifest.json` | Web App Manifest für PWA | ✅ Ja |
| `icons/` | 9 App-Icons (SVG + 8 PNG-Größen: 32/120/152/167/180/192/512/1024) | ✅ Ja |
| `README.md` | Repo-README (existiert auf GitHub, lokal ggf. nicht) | ✅ Ja |
| `CLAUDE.md` | Diese Datei | ✅ Ja |
| `.gitignore` | Existiert | ✅ Ja |
| `DASHBOARD_VORLAGE.md` | Persönliche Notizen/Vorlage von Leonard | ❌ Nein (siehe .gitignore) |
| `CLAUDE-CODE-ONBOARDING.md` | Übergabe-Notizen, bewusst privat | ❌ Nein |
| `Tesla-Web-Dashboard-Anleitung.docx` | Persönliche Anleitung | ❌ Nein |
| `PWA-Icon-Briefing.md`, `PWA-Vollbild-Briefing.md` | Wiederverwendbare Prompt-Vorlagen für andere Projekte | ❌ Nein |
| `.DS_Store` | macOS-Cruft | ❌ Nein |

**Aktueller `.gitignore`:**

```
.DS_Store
DASHBOARD_VORLAGE.md
Tesla-Web-Dashboard-Anleitung.docx
PWA-*.md
CLAUDE-CODE-ONBOARDING.md
```

## Pflicht-Workflow bei jeder Änderung

Diese Reihenfolge ist zwingend, sonst gibt es 404s auf GitHub Pages oder iOS zeigt alte Assets.

1. **Änderung machen** in `index.html`, `manifest.json` oder `icons/`.
2. **Bei Änderung an Icons oder wenn sich das Cache-Verhalten ändern soll:** Den Cache-Buster `?v=N` in `index.html` und `manifest.json` hochzählen. Aktueller Stand: `?v=2` in den `apple-touch-icon`- und `icon`-Links.
3. **JS-Syntax-Check vor jedem Commit.** Node ist auf diesem Mac **nicht** installiert (auch kein Homebrew/nvm/deno/bun, kein `gh`). Ersatz ist `osacompile` (JavaScriptCore, macOS-bordeigen) — erkennt Syntaxfehler mit Zeilennummer und Exit-Code 1:
   ```bash
   python3 -c "import re; html=open('index.html').read(); s=re.findall(r'<script(?![^>]*src=)[^>]*>(.*?)</script>', html, re.S); open('/tmp/_check.js','w').write(s[-1])" && osacompile -l JavaScript -o /tmp/_check.scpt /tmp/_check.js && echo "SYNTAX OK"
   ```
4. **JSON validieren:**
   ```bash
   python3 -c "import json; json.load(open('manifest.json'))"
   ```
5. **Commit + Push** mit sinnvoller Commit-Message (deutsch, Präsens).
6. **Nach Push die drei Deploy-Test-URLs prüfen** (in Inkognito, damit kein Cache):
   - `https://lebrongoku-prog.github.io/tesla-dashboard/` (Dashboard lädt)
   - `https://lebrongoku-prog.github.io/tesla-dashboard/manifest.json` (JSON kommt zurück)
   - `https://lebrongoku-prog.github.io/tesla-dashboard/icons/icon-180.png?v=2` (Blitz-Icon)
7. Bei Icon-Änderung zusätzlich: iPhone-Testanleitung an Leonard senden (App vom Home-Bildschirm löschen, Safari-Daten für Domain löschen, neu installieren).

## Kern-Architektur

### Datenfluss

```
Google Sheet
    ↓ (Sheets API, OAuth-Bearer)
loadData() → fetchSheet() → rowsToObjects()
    ↓
parseRows() → allCharges (Master-State)
    ↓
Filter (filterMonths / filterYear / periodOffset / monthlyCountFilter)
    ↓
getVisibleCharges()
    ↓
renderAll() → renderKpiCards, renderSavingsCard, renderRecordCards,
              renderCostChart, renderChargeCountChart, renderPriceChart,
              renderLocationChart, renderWeekdayChart, renderHeatmap,
              renderPriceInsights, renderYearGrid,
              renderYearComparisonChart, renderTable
```

Bei leerem Zeitraum übernimmt `clearPeriodSections()` das Zurücksetzen.
Jahresraster und Jahresvergleich hängen bewusst **nicht** am Zeitfilter.

### Globaler State (Top of Script)

- `allCharges` — geparste Ladevorgänge, Master. Filter-unabhängig.
- `filterMonths` (1/3/6/12/null), `filterYear` (Zahl/null), `periodOffset` (Monate zurück), `monthlyCountFilter` (hebt Monate mit genau N Ladungen hervor)
- `yearGridYear` (Jahr im Jahresraster), `yearComparisonMetric` (`'count' | 'cost' | 'kwh'`)
- `tableSortBy` (`'date' | 'loc' | 'kwh' | 'cost' | 'price' | 'day' | 'time'`), `tableSortDir` (`'asc' | 'desc'`), `tableSearch` (String), `visibleTableRows` (Basis für die Detailansicht)
- `charts` — { chartId → Chart.js-Instanz }, wird von `destroyChart` verwaltet.
- `STORAGE_KEYS` — alle localStorage-Schlüssel an einer Stelle.

### Datenmodell (ein Ladevorgang)

```js
{ date: Date, year: Number, monthKey: 'YYYY-MM', monthLabel: 'Jan 26',
  loc: String, kwh: Number, price: Number, total: Number,
  dayIdx: 0..6 (0=Mo, 6=So), hour: 0..23 }
```

Spalten aus dem Sheet: `ChargeStartDateTime`, `SiteLocationName`, `QuantityBase` (kWh), `UnitCostBase` (Preis/kWh), `Total Inc. VAT`.

### Zentrale Funktionen

**DOM:** `el(id)`, `setText(id, text)`, `setHtml(id, html)` — ersetzen die früheren ~110 `document.getElementById`-Aufrufe.

**Formatierung:** `formatChf(n, decimals=2)`, `formatNumber(n, decimals=1)`, `formatDate`, `formatMonthLabel`, `formatTime`, `formatDayMonth`, `toMonthKey` (alle `de-CH`).

**Rechnen:** `sumBy`, `sumKwh`, `sumCost`, `average`, `roundTo`, `pricePerKwh(charge)`, `averagePricePerKwh(charges)`, `uniqueYears`.

**Daten:** `parseRows(rows)`, `groupByMonth(charges)` → {key, label, kwh, cost, count}, `shortLocationName` + `repairEncoding` (Umlaute, `decodeURIComponent(escape(str))` — deprecated aber funktioniert).

**Zeitraum:** `getVisibleCharges()`, `getPreviousPeriodCharges()` (Trend-Badges), `getVisibleDateRange()`, `getChargeDateBounds()`, `getMaxPeriodOffset()`, `getPeriodLabel()`, `shiftPeriod(dir)`.

**Chart-Bausteine:** `categoryAxis(fontSize, color)`, `valueAxis(formatTick, extra)`, `countAxis()`, `chartOptions(extra)`, `averageLineDataset(...)`, `highlightColors(...)`, `isDarkMode()`, `chartGridColor()`, `destroyChart(id)`. Diese ersetzen die früher in jedem Diagramm kopierten Achsen- und Farbdefinitionen.

**Rendern:** `renderTrendBadge(elId, current, previous, downIsGood)`, `clearPeriodSections()`. Die drei Diagramme mit Doppelrolle sind aufgeteilt: `renderChargeCountChart` → `renderMonthCalendar` / `renderChargeCountBars`; `renderCostChart` → `renderCostPerChargeChart` / `renderCostPerMonthChart` (+ `buildCostRowsWithForecast` für die Hochrechnung); `renderPriceChart` → `renderPricePerChargeChart` / `renderPricePerMonthChart`.

**Auth:** `initGoogleAuth` → `consumeTokenFromUrl` / `restoreTokenFromSession` → `startDataLoad`; `signIn`. OAuth Implicit Flow, Token in `sessionStorage`.

**Refresh:** `triggerRefresh(event)` — ruft Apps Script Web App auf. Token per `getRefreshToken()` einmalig via `prompt()` abgefragt und in `localStorage` gespeichert (Key: `tesla-refresh-token`). **Shift-Klick auf Refresh-Button = Token zurücksetzen** (`forgetRefreshToken`).

### Chart.js-Konventionen

- Chart.js 4.5.0 via CDN (SRI-Integrity gesetzt, nicht anfassen).
- Eigenes `topLabelPlugin`: zeichnet Zahlen über Balken. `ds.topLabel = false` deaktiviert es pro Dataset. `ds.topLabelFmt(val, idx)` für custom Format.
- Linien-Diagramme: **keine Flächenfüllung** (`fill: false`), **keine Kurven** (`tension: 0`). Bewusste Design-Entscheidung.

## UI- und Namens-Konventionen

- **Sprache:** Alle User-facing Strings auf Deutsch (Schweizerdeutsch), z.B. „Ladevorgänge", „Ø Preis / kWh", „CHF".
- **Zahlen:** `.toLocaleString('de-CH')`, CHF als Präfix (nicht Suffix), Tausender-Trenner `'` (kommt aus de-CH automatisch).
- **Farben:**
  - Primär-Blau `#3b82f6`, Grün `#22c55e`, Lila `#a855f7` (Ø Preis/kWh — **nicht** Grün, war korrigiert), Orange `#f97316`, Cyan `#06b6d4`.
  - Gradient-Header: `#0C4A6E → #0EA5E9`.
  - **Flächen-, Text- und Linienfarben stehen als CSS-Variablen in `:root`** (`--surface`, `--surface-sunken`, `--btn-bg`, `--field-bg`, `--border`, `--divider`, `--text-strong`, `--text-body`, `--text-control`, `--text-muted`, `--text-label`, `--row-*`, `--shadow-card`, `--overlay-bg`). Der Dunkelmodus überschreibt in `body.dark` **nur diese Variablen** — neue Bausteine deshalb mit Variablen statt Hex-Werten gestalten, dann stimmt der Dunkelmodus automatisch.
  - `--text-label-fixed` ist Absicht: Diese Beschriftungen hatten historisch nie eine dunkle Variante und bleiben in beiden Modi `#94a3b8`.
  - Diagrammfarben liegen im JS unter `CHART_COLORS`, nicht im CSS.
- **Tabelle:** Monatliche Gruppen abwechselnd weiß / hellblau via `.month-even` / `.month-odd` (Farben über `--row-even` / `--row-odd`). **Keine Borders** (wurde probiert und wieder verworfen). Die `.month-*`-Regeln müssen im CSS **nach** `tbody tr:hover` stehen — gleiche Spezifität, es entscheidet die Reihenfolge.
- **KPI-Trend-Badges:** `.good` (grün, positive Entwicklung), `.bad` (rot), `.neutral` (grau). Semantik pro KPI: Ladevorgänge → up=good, Kosten & Ø Preis → down=good.
- **Tooltips:** Kein Gedankenstrich als Trennlinie. Beträge vertikal untereinander ausgerichtet.
- **KPI-Zusatzinfos:** In derselben Zeile wie der Wert (via `.kpi-value-row`), NIE als separate Zeile oder eigene Kachel.
- **Zeitfilter:** 1M/3M/6M/1J/Alle + dynamisch generierte Jahres-Buttons (aus `allData.year`).
- **Bei 1M-Filter:** Die drei Reihe-2-Diagramme zeigen Einzelladevorgänge statt Monatswerte. Ladevorgänge-Chart wird zum Kalender-Overlay.

## Bekannte Gotchas / Stolpersteine

- **iOS PWA-Icons müssen ohne Alpha-Kanal sein** (RGB, nicht RGBA), sonst rendert iOS sie schwarz oder ignoriert sie. Bei Neu-Generierung: mit Pillow `img.convert('RGB')` vor Save.
- **GitHub Web-UI Upload verliert Ordnerstruktur**, wenn Dateien einzeln statt als Ordner gezogen werden. → Immer den Ordner selbst ziehen. (Mit lokalem Git und `git push` erledigt sich das.)
- **iOS cached Icons extrem aggressiv.** Bei Änderung: Cache-Buster `?v=N` hochzählen + Leonard bitten, App vom Home-Bildschirm zu löschen und Safari-Daten für die Domain zu leeren.
- **Token-Rotation ist vollzogen** (bestätigt 26.07.2026). Im Apps Script `Tesla API.gs` → `doGet()` wird gegen einen neuen Wert verglichen. Die zwei alten Token (`leo2024tesla`, `Fml2gnpg30…`) stehen zwar noch in der öffentlichen Git-Historie (Commits `2d42672` / `ee1cabb`), sind aber ungültig — Historie umschreiben unnötig.
- **Refresh-Token ist NICHT mehr im Code hardcoded** (Sicherheits-Fix). Neuer Token im Apps Script Properties Service / hardcodierter Vergleich; Frontend fragt via `prompt()` beim ersten Refresh und speichert in `localStorage['tesla-refresh-token']`. iPhone-Shortcut muss separat mit neuer Token-URL aktualisiert werden.
- **OAuth Redirect-URI ist fest verdrahtet** in `signIn()`: `https://lebrongoku-prog.github.io/tesla-dashboard/`. Bei URL-Änderung auch in der Google Cloud Console anpassen.
- **Cowork-Artifact-Version (Google Drive-basiert) und diese GitHub-Pages-Version sind architektonisch unterschiedlich** — nicht zeilenweise synchron halten. Änderungen hier nicht automatisch dorthin übertragen.
- **`ChargeStartDateTime`-Parsing** ist naiv (`new Date(...)`). Zeitzonen aus dem Tesla-Export werden als lokale Zeit interpretiert.
- **Chart.js SRI-Hash im Script-Tag darf beim Version-Upgrade nicht vergessen werden** — sonst bricht das ganze Dashboard.

## Design-Präferenzen (nicht diskutieren, direkt anwenden)

- Linien-Diagramme: gerade Linien, keine Flächenfüllung.
- Tabellen-Zeilen: Monatsweise Farbgruppen mit `.month-even`/`.month-odd`, keine Borders.
- KPI-Zusatzinfos: In der Value-Zeile, nicht als separate Kachel.
- Deutsche, knappe Antworten. Leonard möchte fertige Lösungen, kein iteratives Debugging.

## Nützliche Befehle

```bash
# Repo-Setup — ERLEDIGT (26.07.2026). Ordner ist mit origin/main verbunden,
# Push läuft über den macOS-Keychain (kein gh CLI installiert).
# Commit-Identität ist lokal gesetzt: lebrongoku-prog / @users.noreply.github.com
#
# Falls das Setup je wiederholt werden muss (ohne Merge-Konflikte):
#   git init -b main
#   git remote add origin https://github.com/lebrongoku-prog/tesla-dashboard.git
#   git fetch origin
#   git reset origin/main        # mixed: Arbeitsverzeichnis bleibt unangetastet
#   git checkout -- README.md    # liegt nur remote, lokal nachziehen

# JS-Syntax-Check (vor jedem Commit) — kein Node auf diesem Mac, daher osacompile
python3 -c "import re; html=open('index.html').read(); s=re.findall(r'<script(?![^>]*src=)[^>]*>(.*?)</script>', html, re.S); open('/tmp/_check.js','w').write(s[-1])" && osacompile -l JavaScript -o /tmp/_check.scpt /tmp/_check.js && echo "SYNTAX OK"

# manifest.json validieren
python3 -c "import json; json.load(open('manifest.json'))"

# Icons neu rendern (aus icons/icon.svg)
python3 <<'PY'
from PIL import Image
src = Image.open('icons/icon-1024.png').convert('RGBA')
for size in [512, 192, 180, 167, 152, 120, 32]:
    img = src.resize((size, size), Image.LANCZOS)
    bg = Image.new('RGB', img.size, (12, 74, 110))
    bg.paste(img, mask=img.split()[3])
    bg.save(f'icons/icon-{size}.png', 'PNG', optimize=True)
PY

# Deploy
git add -A && git commit -m "…" && git push
```

## Wichtige externe Endpunkte / IDs

- **Google Sheet ID:** `1YC4EyDAhOa-eJod_JU5afc48IRUb5A0kMcgoiWLROnc`
- **OAuth Client ID:** `49501547672-sc1ga0lcq200h4iuij33c7n4kf65gpra.apps.googleusercontent.com`
- **Apps Script Web App:** `https://script.google.com/macros/s/AKfycbzplDQKzviFvZkT-mb8xChF-9AN3Eu15mMC-et_JSFyhRHPCEw-2F7oYgW0ehwQGvJzJw/exec`
- **OAuth Redirect:** `https://lebrongoku-prog.github.io/tesla-dashboard/`
- Apps Script: wöchentlicher Trigger + manueller Refresh via `?refresh=true&token=<TOKEN>`. Token wird im Frontend per `prompt()` + `localStorage` verwaltet.
