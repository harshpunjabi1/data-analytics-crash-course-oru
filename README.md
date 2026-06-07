# Toronto Airbnb Market Intelligence

**Data Analysis Bootcamp | ORU**

A hands-on data analysis project using real Airbnb listing data from Toronto. Analyze 15,000+ listings to answer four business questions, then deliver findings as a Tableau Public dashboard.

---

## The Brief

You're a data analyst at a property management startup expanding into Toronto's short-term rental market. Leadership needs a market intelligence report before committing resources.

**Questions to answer:**

1. **Supply concentration** — Which neighbourhoods and room types dominate?
2. **Demand signals** — Where is demand strongest? (review activity as a booking proxy)
3. **Competitive landscape** — Professional operators vs casual hosts
4. **Regulatory compliance** — How does licensing vary across the city?

---

## Dataset

**Source:** [Inside Airbnb — Toronto](http://insideairbnb.com/get-the-data/)

**File:** `listings.csv`

| Column | Type | Description |
|--------|------|-------------|
| `id` | int | Unique listing identifier |
| `name` | string | Listing title |
| `host_id` | int | Unique host identifier |
| `host_name` | string | Host display name |
| `neighbourhood` | string | Toronto neighbourhood (140 unique) |
| `latitude` / `longitude` | float | Listing coordinates |
| `room_type` | string | Entire home/apt, Private room, Shared room, Hotel room |
| `minimum_nights` | int | Minimum booking length required |
| `number_of_reviews` | int | Total lifetime reviews |
| `last_review` | date | Date of most recent review |
| `reviews_per_month` | float | Average monthly review rate (NULL if no reviews) |
| `calculated_host_listings_count` | int | Number of listings this host operates |
| `availability_365` | int | Days available in next 365 days |
| `number_of_reviews_ltm` | int | Reviews in last 12 months |
| `license` | string | STR license number (NULL if unlicensed) |

> **Note:** The compact `listings.csv` does not include price data. This project uses review velocity and availability as proxy metrics — a more transferable skill than working with an obvious variable.

---

## Setup: BigQuery

1. Go to [console.cloud.google.com/bigquery](https://console.cloud.google.com/bigquery)
2. Create a dataset (e.g. `airbnb_toronto`)
3. Create table → Upload → select `listings.csv`
4. **Important:** Under Advanced options, set **"Number of errors allowed"** to `100`. The raw CSV has ~73 rows with broken quoting in the `name` column. This tells BigQuery to skip those rows instead of failing.
5. Auto-detect schema → Create table

No credit card required for the BigQuery sandbox (1 TB/month free).

---

## Step 0: Fix Data Quality

The CSV import loads broken rows from the `name` column's bad quoting. These rows have `NULL` across most fields or text fragments in the `id` column. Clean them out before doing anything else:

```sql
CREATE OR REPLACE TABLE `<project>.<dataset>.listings` AS
SELECT *
FROM `<project>.<dataset>.listings`
WHERE SAFE_CAST(id AS INT64) IS NOT NULL
  AND neighbourhood IS NOT NULL;
```

This overwrites the table in place, keeping only valid rows.

---

## Step 1: Profile the Data

Understand the shape and quality of the data before analyzing.

```sql
-- Row count and null overview
SELECT
  COUNT(*) AS total_rows,
  COUNT(DISTINCT neighbourhood) AS unique_neighbourhoods,
  COUNT(DISTINCT host_id) AS unique_hosts,
  COUNTIF(reviews_per_month IS NULL) AS null_review_per_month,
  COUNTIF(last_review IS NULL) AS null_last_review,
  COUNTIF(license IS NOT NULL AND license != '') AS licensed_count
FROM `<project>.<dataset>.listings`;
```

```sql
-- Room type distribution
SELECT
  room_type,
  COUNT(*) AS listing_count,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 1) AS pct
FROM `<project>.<dataset>.listings`
GROUP BY room_type
ORDER BY listing_count DESC;
```

```sql
-- Minimum nights distribution
SELECT
  MIN(minimum_nights) AS min_val,
  MAX(minimum_nights) AS max_val,
  APPROX_QUANTILES(minimum_nights, 2)[OFFSET(1)] AS median_val,
  ROUND(AVG(minimum_nights), 1) AS avg_val
FROM `<project>.<dataset>.listings`;
```

```sql
-- Multi-listing hosts
SELECT
  COUNTIF(calculated_host_listings_count > 1) AS multi_listing_entries,
  COUNT(*) AS total,
  ROUND(COUNTIF(calculated_host_listings_count > 1) * 100.0 / COUNT(*), 1) AS pct
FROM `<project>.<dataset>.listings`;
```

---

## Step 2: Analyze

### Q1: Where is the supply concentrated?

```sql
-- Top 15 neighbourhoods by listing count
SELECT
  neighbourhood,
  COUNT(*) AS listing_count,
  ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM `<project>.<dataset>.listings`), 1) AS pct_of_total
FROM `<project>.<dataset>.listings`
GROUP BY neighbourhood
ORDER BY listing_count DESC
LIMIT 15;
```

```sql
-- Top 15 neighbourhoods x room type
SELECT
  neighbourhood,
  room_type,
  COUNT(*) AS listing_count
FROM `<project>.<dataset>.listings`
WHERE neighbourhood IN (
  SELECT neighbourhood
  FROM `<project>.<dataset>.listings`
  GROUP BY neighbourhood
  ORDER BY COUNT(*) DESC
  LIMIT 15
)
GROUP BY neighbourhood, room_type
ORDER BY listing_count DESC;
```

```sql
-- Room type distribution
SELECT
  room_type,
  COUNT(*) AS listing_count
FROM `<project>.<dataset>.listings`
GROUP BY room_type
ORDER BY listing_count DESC;
```

### Q2: Where is demand strongest?

Using `reviews_per_month` as a proxy for booking frequency. Filtered to listings with at least one review, and neighbourhoods with at least 10 listings to avoid noise.

```sql
SELECT
  neighbourhood,
  COUNT(*) AS listing_count,
  ROUND(AVG(COALESCE(SAFE_CAST(reviews_per_month AS FLOAT64), 0)), 2) AS avg_rpm,
  APPROX_QUANTILES(COALESCE(SAFE_CAST(reviews_per_month AS FLOAT64), 0), 2)[OFFSET(1)] AS median_rpm,
  SUM(SAFE_CAST(number_of_reviews AS INT64)) AS total_reviews
FROM `<project>.<dataset>.listings`
WHERE SAFE_CAST(number_of_reviews AS INT64) > 0
GROUP BY neighbourhood
HAVING COUNT(*) >= 10
ORDER BY avg_rpm DESC
LIMIT 15;
```

### Q3: Professional operators vs casual hosts

A host with more than one listing is classified as a professional operator.

```sql
SELECT
  CASE
    WHEN calculated_host_listings_count > 1 THEN 'Professional'
    ELSE 'Casual'
  END AS host_type,
  COUNT(*) AS listing_count,
  ROUND(AVG(COALESCE(SAFE_CAST(reviews_per_month AS FLOAT64), 0)), 2) AS avg_rpm,
  APPROX_QUANTILES(COALESCE(SAFE_CAST(reviews_per_month AS FLOAT64), 0), 2)[OFFSET(1)] AS median_rpm,
  SUM(SAFE_CAST(number_of_reviews AS INT64)) AS total_reviews,
  ROUND(AVG(availability_365)) AS avg_availability
FROM `<project>.<dataset>.listings`
WHERE SAFE_CAST(number_of_reviews AS INT64) > 0
GROUP BY host_type
HAVING COUNT(*) >= 10
ORDER BY avg_rpm DESC;
```

### Q4: Licensing compliance

```sql
-- Overall licensing rate
SELECT
  CASE WHEN license IS NULL THEN 'No' ELSE 'Yes' END AS licensed_flag,
  COUNT(*) AS number_of_listings,
  ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM `<project>.<dataset>.listings`), 1) AS pct_of_total
FROM `<project>.<dataset>.listings`
GROUP BY licensed_flag
ORDER BY number_of_listings DESC;
```

```sql
-- Licensing by neighbourhood
SELECT
  neighbourhood,
  COUNT(CASE WHEN license IS NULL THEN id ELSE NULL END) AS unlicensed,
  COUNT(CASE WHEN license IS NOT NULL THEN id ELSE NULL END) AS licensed,
  ROUND(COUNT(CASE WHEN license IS NOT NULL THEN id ELSE NULL END) * 1.0 / COUNT(*), 2) AS licensed_pct
FROM `<project>.<dataset>.listings`
GROUP BY neighbourhood
ORDER BY licensed DESC;
```

---

## Step 3: Export for Tableau

Save your query results as CSVs for Tableau. In the BigQuery console, after running each query:

1. Click **Save Results** → **CSV (local file)**
2. Save the following result sets:
   - `q1_supply_by_neighbourhood.csv` — Top 15 neighbourhoods × room type query
   - `q2_demand_by_neighbourhood.csv` — Demand signals query (avg_rpm, listing_count, total_reviews)
   - `q3_host_type.csv` — Professional vs casual query
   - `q4_licensing.csv` — Licensing by neighbourhood query

Alternatively, export the full cleaned table for a single Tableau data source:

```sql
SELECT * FROM `<project>.<dataset>.listings`;
```

Save as `listings_full.csv` and connect Tableau to that single file.

---

## Step 4: Build the Tableau Dashboard

### Setup

1. Download [Tableau Public](https://public.tableau.com/) (free)
2. Open Tableau Public → Connect → Text File → select your exported CSV
3. If using the full table export, Tableau auto-detects types

### Sheet 1: Supply Map

**Chart type:** Horizontal stacked bar

1. Drag `neighbourhood` to **Rows**
2. Drag `Number of Records` to **Columns**
3. Drag `room_type` to **Color**
4. Sort descending by listing count
5. Right-click `neighbourhood` → Filter → Top 15 by Count of Records
6. Title: *"Top 15 Neighbourhoods by Listing Count"*

### Sheet 2: Demand vs Supply Scatter

**Chart type:** Scatter plot

1. Create a calculated field — Name: `Avg Reviews per Month`, Formula: `AVG([Reviews Per Month])`
2. Drag `neighbourhood` to **Detail**
3. Drag `CNT([Id])` to **Columns** (listing count per neighbourhood)
4. Drag `Avg Reviews per Month` to **Rows**
5. Drag `SUM([Number Of Reviews])` to **Size**
6. Analytics pane → drag **Reference Line** onto Y-axis for the overall average
7. Title: *"Supply Volume vs Demand Intensity"*

**Read the quadrants:** Top-right = high supply + high demand (competitive). Top-left = low supply + high demand (opportunity). Bottom-right = high supply + low demand (saturated).

### Sheet 3: Host Type & Compliance

Two views side by side:

**Left — Host Type Treemap:**
1. Create a calculated field — Name: `Host Type`, Formula: `IF [Calculated Host Listings Count] > 1 THEN 'Professional' ELSE 'Casual' END`
2. Drag `Host Type` to **Color**
3. Drag `room_type` to **Detail**
4. Drag `Number of Records` to **Size** and **Label**

**Right — Compliance Highlight Table:**
1. Create a calculated field — Name: `Is Licensed`, Formula: `IF ISNULL([License]) THEN 0 ELSE 1 END`
2. Create another — Name: `Licensed Pct`, Formula: `SUM([Is Licensed]) / COUNT([Id])`
3. Drag `neighbourhood` to **Rows** (filter Top 15 by count)
4. Drag `Licensed Pct` to **Text** and **Color**
5. Format as percentage, use a diverging color palette (red to green)

### Assemble the Dashboard

1. Dashboard menu → **New Dashboard**
2. Set size to **Automatic** or **1200 × 800**
3. Drag all three sheets onto the canvas
4. Add a title text box: *"Toronto Airbnb Market Intelligence"*
5. Optionally add a `room_type` filter that applies to all sheets

### Publish

1. File → **Save to Tableau Public**
2. Create a free account if needed
3. The published URL is your portfolio link

---

## Deliverable Structure

Present findings like a working analyst would:

**Executive Summary (2 sentences):** State the market picture. Example: "Toronto's Airbnb market has ~15,700 listings concentrated in downtown/waterfront areas, with 67% being entire-home rentals. Nearly half the supply is controlled by professional multi-listing operators, and licensing compliance varies significantly by neighbourhood."

**3 Key Findings (each = one sentence + one chart):** No methodology. Just finding and evidence.

**1 Recommendation:** Based on the data, what should the company do? Which neighbourhoods, which property types, which host strategy?

**Gaps & Next Steps:** What this data cannot tell you — pricing/revenue, actual booking rates, guest demographics, seasonal trends. Acknowledging limitations builds credibility.

---

## Tools

| Tool | Purpose |
|------|---------|
| BigQuery (SQL) | Data loading, profiling, cleaning, analysis |
| Tableau Public | Dashboard creation and publishing |
| GitHub | Documentation and code hosting |

---

## Data Source

[Inside Airbnb](http://insideairbnb.com/) — Creative Commons Attribution 4.0 International License.
