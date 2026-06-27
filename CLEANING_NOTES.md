# Cleaning Notes — Mosquito Forecast × Weather (Seoul, 2026 season)

**Built:** 2026-06-27
**Date range:** 2026-05-01 → 2026-06-26 (57 days, current season to date)
**Sources:**
- Mosquito: 서울 열린데이터광장 — 서울시 모기예보제 정보 (`MosquitoStatus` Open API), daily, index 0–100 for three environments.
- Weather: data.go.kr — 기상청 지상(종관, ASOS) 일자료, station **108 (Seoul)**.

## How the data was pulled
- **Mosquito API returns one day per call** (no date-range support), so the 57 days were fetched one date at a time and combined. *(This matters for the weekly bot: a full May–Oct season = ~184 calls. Budget for it.)*
- **Weather API supports a date range** — all 57 days came in a single call.

## What was cleaned
- **Column renaming** to analysis-friendly names:
  - Mosquito: `MOSQUITO_VALUE_WATER/HOUSE/PARK` → `Idx_Water`, `Idx_House`, `Idx_Park`.
  - Weather: `tm`→`Date`, `avgTa`→`Temp_Avg`, `minTa`→`Temp_Min`, `maxTa`→`Temp_Max`, `sumRn`→`Rain_mm`, `avgRhm`→`Humidity_Avg`, `avgWs`→`Wind_Avg`.
- **Types:** all index/weather fields cast to numeric; `Date` cast to date.
- **Rainfall blanks:** the ASOS feed leaves `sumRn` empty on no-rain days; these were set to **0.0** (0 days needed it this pull — values already came as 0). Added a `Rained` flag (Rain/Dry) for easy filtering.
- **Duplicates:** none found (0 removed in each table).
- **Date continuity:** no missing days — all 57 calendar days present in both tables.

## Data-quality flags to know
- **`Idx_Water` is saturated:** it sits at the 100 cap on **39 of 57 days**. Treat it as "maxed out" rather than a true gradient. **Use `Idx_Park` and `Idx_House` as the main mosquito measures** — they vary across the full range (Park: 7.5–52.9).
- Weather ranges this pull: Temp_Avg 13.0–27.2 °C; 18 rainy days; one heavy-rain day (2026-06-20, 71.7 mm).

## What the data already shows (Pearson r, same-day)
| Relationship | Park index | House index |
|---|---|---|
| vs **Avg temperature** | **+0.81** | **+0.82** |
| vs Max temperature | +0.68 | +0.68 |
| vs Humidity | +0.02 | ~0.00 |
| vs Rainfall (same day) | +0.07 | +0.02 |

**Takeaway:** mosquito activity rises strongly with average temperature; same-day humidity and rainfall show essentially no linear relationship (a rain *lag* effect may exist but isn't tested here). This is the story the dashboard should foreground.

## Files
- `raw/` — untouched API pulls (`mosquito_raw_2026.csv`, `weather_raw_2026.csv`).
- `cleaned/`
  - `fact_mosquito.csv` — Date, Idx_Water, Idx_House, Idx_Park
  - `fact_weather.csv` — Date, Temp_Avg/Min/Max, Rain_mm, Humidity_Avg, Wind_Avg, Rained
  - `dim_date.csv` — calendar table (Year, Month, MonthName, Day, Weekday, WeekOfYear, IsWeekend)
  - `combined_mosquito_weather.csv` — flat joined table (quick-start option)
