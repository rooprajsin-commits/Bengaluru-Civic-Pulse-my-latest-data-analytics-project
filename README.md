# Bengaluru Civic Pulse
### BBMP grievance and air-quality analytics, with an auto-refreshing pipeline

An end-to-end data analyst project on Bengaluru's civic complaints (BBMP wards and zones),
combined with **live weather and air-quality data from a public API**. It answers questions a
city operations team actually asks:

- Which zones and wards carry the heaviest complaint burden per resident?
- Where is BBMP missing its resolution SLA, and does that hurt citizen ratings?
- **Does rainfall drive pothole and waterlogging complaints, and by how much?**
- What is open and overdue right now, and how is the air in each zone today?

| Tool | What it does in this project |
|---|---|
| **Python** | API extraction, data generation/cleaning, ETL, statistics, charts, Excel build, scheduler |
| **API** | Open-Meteo weather (archive + forecast) and air-quality APIs, free, no key needed |
| **SQL** | SQLite star schema, 9 reporting views, 11 business queries (CTEs, window functions) |
| **Excel** | Power Query auto-refresh, KPI dashboard, INDEX/MATCH ward lookup, AVERAGEIFS, CORREL, charts |
| **Power BI** | Star-schema model, 25+ DAX measures, 4-page report, automatic page refresh |

---

## Architecture

```
 Open-Meteo APIs ──┐
 (rain, temp, AQI) │     src/api_client.py
                   ├──►  src/data_generator.py ──► src/etl.py ──► SQLite  (data/db/civic_pulse.db)
 BBMP ward master ─┘     (grievance stream)        clean,         │  views (sql/02_views.sql)
 (data/raw)                                        transform      │
                                                                  ├──► data/processed/*.csv
                                                                  │      ├── Excel (Power Query, refresh 1 min)
                                                                  │      └── Power BI (Import)
                                                                  └──► PostgreSQL (optional)
                                                                         └── Power BI DirectQuery + auto page refresh
 src/run_pipeline.py --loop 1  →  repeats everything every minute
```

## Project structure
```
bengaluru-civic-pulse/
├── src/
│   ├── config.py                 paths, API endpoints, zones, categories, SLA rules
│   ├── api_client.py             EXTRACT - Open-Meteo weather + air quality (with cache/fallback)
│   ├── data_generator.py         grievance source: real BBMP export, or rain-driven simulation
│   ├── etl.py                    TRANSFORM + LOAD + EXPORT (SQLite, optional PostgreSQL, CSV)
│   ├── run_pipeline.py           orchestrator; --loop N for live refresh
│   ├── analysis.py               Python EDA -> reports/insights.md + charts
│   └── build_excel_dashboard.py  builds the Excel dashboard
├── sql/
│   ├── 01_schema.sql             star schema (dims + facts + refresh log)
│   ├── 02_views.sql              reporting views
│   └── 03_analysis_queries.sql   11 business questions
├── excel/   Civic_Pulse_Dashboard.xlsx, EXCEL_AUTO_REFRESH.md
├── powerbi/ POWERBI_BUILD_GUIDE.md, measures.dax, powerquery_tables.m, civic_pulse_theme.json
├── reports/ insights.md + PNG charts (regenerated each run)
├── data/raw/bbmp_wards.csv       48 Bengaluru wards across the 8 BBMP zones
└── run_showcase.bat              one-click live demo (Windows)
```

## Quick start
```bash
pip install -r requirements.txt
python src/run_pipeline.py             # first run builds ~21 months of history (~1 min)
python src/analysis.py                 # insights + charts in reports/
python src/build_excel_dashboard.py    # only needed if you delete the workbook
```
Then follow `excel/EXCEL_AUTO_REFRESH.md` (5 min) and `powerbi/POWERBI_BUILD_GUIDE.md`.

## Live showcase (auto refresh)
1. Double-click `run_showcase.bat` (or run `python src/run_pipeline.py --loop 1`).
2. Every minute the pipeline: calls the API for the latest weather and air quality, adds the
   complaints that arrived since the last run, moves complaints from *Registered* to *In Progress*
   to *Resolved/Closed* as time passes, reloads the database and re-exports the CSVs.
3. Excel (Power Query, 1-minute refresh) and Power BI (DirectQuery + automatic page refresh) pick
   up the change on their own. Point to the **Last refreshed** time and the **Complaints today** tile.

## Key findings (prototype data, regenerated in `reports/insights.md`)
- Heavy-rain days bring roughly **+87% pothole** and **+119% waterlogging** complaints versus dry days
  (daily rain vs rain-sensitive complaints r ≈ 0.76; yesterday's rain still predicts today's potholes).
- **Mahadevapura** breaches SLA on ~50% of complaints vs ~19% in East; its rating is the lowest (3.1/5).
- Bellandur, Whitefield, Varthur and HSR Layout are the wards to prioritise.
- Complaints resolved within SLA are rated far higher than breached ones (Q9 in `03_analysis_queries.sql`).

## About the data (be upfront about this in interviews)
- **Weather and air quality are real** when the API is reachable (`source = open-meteo` in the
  tables); if the machine is offline the pipeline falls back to cached or clearly-labelled
  `simulated` values so the demo never breaks.
- **Ward master**: 48 real Bengaluru wards in the 8 BBMP zones; population and area are approximate.
- **Complaint records are simulated** for this prototype, using BBMP's real categories and channels
  (Sahaaya app, 1533 helpline). The simulation is driven by the real rainfall, so the rain-impact
  analysis is something the pipeline has to discover, not a hard-coded number.
- **Swap in real data**: BBMP grievance exports are published on open-data portals such as
  OpenCity (data.opencity.in, search "BBMP grievances") and data.gov.in. Save the file as
  `data/raw/bbmp_grievances_real.csv`, adjust `COLUMN_MAP` in `src/data_generator.py` to its
  headers, and rerun.
- Context: in 2025 Bengaluru's civic administration began moving from BBMP to the Greater Bengaluru
  Authority and new city corporations. This project uses the BBMP 8-zone structure that the
  historical public data is organised by.

## Resume bullets
- Built an automated **Python ETL pipeline** integrating the **Open-Meteo REST API** with 130K+
  BBMP civic-complaint records across 48 Bengaluru wards, refreshing a **SQL star schema** every minute.
- Wrote **SQL** reporting views using CTEs and window functions (RANK, LAG, rolling averages) to
  score wards on complaint burden, SLA breach and citizen rating.
- Found that heavy rain raises pothole complaints ~87% and waterlogging complaints ~119% (r = 0.76),
  supporting a recommendation to pre-deploy repair crews on rain-forecast days.
- Delivered an auto-refreshing **Excel** dashboard (Power Query, INDEX/MATCH, AVERAGEIFS, CORREL) and
  a 4-page **Power BI** report (star schema, 25+ DAX measures, automatic page refresh).

## Likely interview questions
- *Why a star schema?* One fact table per process (complaints, weather, air quality) joined through
  shared dimensions (date, zone, ward), so filters work across all of them and DAX stays simple.
- *How do you avoid Excel reading a half-written file?* Each CSV is written to a temp file and
  swapped in atomically (`os.replace`) with retries if the file is locked.
- *How is the refresh incremental?* Weather only re-pulls the last 14 days; complaints only append
  records newer than the last timestamp; statuses are recomputed as of "now".
- *What would you do with real data?* Replace the simulation via `bbmp_grievances_real.csv`, add
  data-quality checks (unmatched wards, duplicate IDs), and schedule with Task Scheduler or cron.
