# Power BI Build Guide — Seoul Mosquito × Weather Dashboard

Follow top to bottom. ~20–30 minutes. You only need the four files in the `cleaned/` folder.

---

## 1. Import the data

1. Open **Power BI Desktop** → **Home → Get data → Text/CSV**.
2. Import all four from `cleaned/`, clicking **Load** each time:
   - `dim_date.csv`
   - `fact_mosquito.csv`
   - `fact_weather.csv`
   - (skip `combined_mosquito_weather.csv` — only use it if you'd rather do the quick single-table version in section 8)
3. After loading, open **Table view** (left rail) and confirm `Date` columns show a date icon. If any imported as text: select the column → **Column tools → Data type → Date**.

---

## 2. Build the model (relationships)

Go to **Model view** (left rail). You want a simple **star schema**: one date table connected to both fact tables.

Create these relationships (drag `Date` field from one table onto `Date` in the other, or **Home → Manage relationships → New**):

| From (many) | To (one) | Field | Cardinality | Cross-filter |
|---|---|---|---|---|
| `fact_mosquito[Date]` | `dim_date[Date]` | Date | Many-to-one (∗:1) | Single |
| `fact_weather[Date]` | `dim_date[Date]` | Date | Many-to-one (∗:1) | Single |

Then mark the date table: select **`dim_date`** → **Table tools → Mark as date table** → choose the **`Date`** column. (This makes time-intelligence and axis sorting behave.)

Your model should look like: `fact_mosquito` → `dim_date` ← `fact_weather`.

> Why a date dimension instead of joining the two tables directly? It lets every visual filter both mosquito and weather by the same calendar, and is the pattern you'll reuse for every future dashboard.

---

## 3. Create measures (DAX)

In **Report view**, right-click `dim_date` in the Data pane → **New measure**. Paste each one (one measure per "New measure"):

```DAX
Avg Park Index = AVERAGE(fact_mosquito[Idx_Park])
```
```DAX
Avg House Index = AVERAGE(fact_mosquito[Idx_House])
```
```DAX
Latest Park Index =
VAR d = MAX(fact_mosquito[Date])
RETURN CALCULATE(SUM(fact_mosquito[Idx_Park]), fact_mosquito[Date] = d)
```
```DAX
Avg Temp = AVERAGE(fact_weather[Temp_Avg])
```
```DAX
Max Temp = MAX(fact_weather[Temp_Max])
```
```DAX
Total Rainfall (mm) = SUM(fact_weather[Rain_mm])
```
```DAX
Rainy Days = CALCULATE(COUNTROWS(fact_weather), fact_weather[Rain_mm] > 0)
```
```DAX
Days Index Maxed (Water) = CALCULATE(COUNTROWS(fact_mosquito), fact_mosquito[Idx_Water] >= 100)
```

---

## 4. Recommended dashboard layout

One page, four zones. Build in this order.

### Top row — KPI cards
Insert **Card** visuals (Visualizations pane → Card) for:
- **Latest Park Index** (today's mosquito level)
- **Avg Park Index** (season average)
- **Avg Temp**
- **Total Rainfall (mm)**
- **Rainy Days**

### Middle-left — the headline chart (mosquito vs temperature over time)
- Insert a **Line and clustered column chart**.
- **X-axis:** `dim_date[Date]`
- **Column y-axis:** `Avg Park Index`, `Avg House Index`
- **Line y-axis:** `Avg Temp`
- This shows mosquito activity climbing with temperature — your key insight.

### Middle-right — correlation scatter
- Insert a **Scatter chart**.
- **X-axis:** `Avg Temp`   **Y-axis:** `Avg Park Index`
- **Values:** `dim_date[Date]` (so each dot = one day)
- Turn on **Analytics → Trend line**. You'll see a clear upward slope (r ≈ 0.81).

### Bottom-left — rainfall
- Insert a **Clustered column chart**.
- **X-axis:** `dim_date[Date]`   **Y-axis:** `Total Rainfall (mm)`

### Bottom-right / side — slicers
- **Slicer** on `dim_date[Date]` (set slicer type to **Between**) for date range.
- Optional **Slicer** on `dim_date[MonthName]`.

---

## 5. Polish
- **Title** the page: `Seoul Mosquito Forecast × Weather — 2026 Season`.
- Format the temperature line in a contrasting color (e.g. red) vs mosquito columns (blue/green).
- Sort the time charts ascending by Date (visual's "..." → Sort axis → Date → Ascending).

---

## 6. Notes that change how you read it
- **Use Park / House indices, not Water.** Water is capped at 100 most days (saturated), so it looks flat at the top and tells you little.
- Mosquito activity tracks **average temperature** (r ≈ 0.81), **not** same-day humidity or rainfall (r ≈ 0). Don't over-read rain on the same day; if you want, a future version can test rainfall with a 1–2 week lag (standing water → larvae).

---

## 7. Refreshing later
The CSVs are static snapshots of this pull. To update: re-run the data pull for new dates, overwrite the files in `cleaned/` (same names), and hit **Home → Refresh** in Power BI. All visuals update automatically.

---

## 8. Quick single-table alternative
If you'd rather not build relationships: import only `combined_mosquito_weather.csv`, and point every visual's fields at that one table. You lose the clean star-schema reuse but it works for a fast first dashboard. Measures become e.g. `AVERAGE(combined_mosquito_weather[Idx_Park])`.
