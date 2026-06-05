# Toronto Airbnb Market Intelligence Project

**Data Analysis Bootcamp | ORU**

A hands-on data analysis project using real Airbnb listing data from Toronto. Two tooling paths provided: Python (Google Colab) and SQL (BigQuery). Final deliverable: a Tableau Public dashboard.

---

## The Brief

You're a data analyst at a property management startup expanding into Toronto's short-term rental market. Leadership needs a market intelligence report before committing resources.

**Your job:** Analyze the Toronto Airbnb market to answer four questions, then deliver findings as a Tableau dashboard with a written recommendation.

**Questions to answer:**

1. **Supply concentration** — Which neighbourhoods and room types dominate the market?
2. **Demand signals** — Where is demand strongest? (Using review activity as a proxy for bookings)
3. **Competitive landscape** — Professional operators vs casual hosts: who controls the market?
4. **Regulatory compliance** — How does licensing vary across the city?

---

## Dataset

**Source:** [Inside Airbnb — Toronto](http://insideairbnb.com/get-the-data/)

**File:** `listings.csv` (compact version, ~15,776 rows)

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

**Note:** The compact `listings.csv` does not include price data. This project focuses on supply, demand (via review proxies), competition, and regulatory analysis. The absence of price is intentional for teaching — it forces you to think about proxy metrics instead of relying on the obvious variable.

---

## Setup

### Option A: Python (Google Colab)

1. Go to [colab.research.google.com](https://colab.research.google.com)
2. Create a new notebook
3. Upload `listings.csv` to the Colab session (Files panel → Upload)

```python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

df = pd.read_csv('listings.csv')
```

### Option B: SQL (BigQuery)

1. Go to [console.cloud.google.com/bigquery](https://console.cloud.google.com/bigquery)
2. Create a dataset: `airbnb_toronto`
3. Upload `listings.csv` as a table named `listings` (auto-detect schema)
4. Use the BigQuery SQL editor for all queries below

No credit card needed for the BigQuery sandbox (1 TB/month free queries).

---

## Step 1: Load & Profile

**Goal:** Understand the shape, quality, and quirks of the data before doing anything else.

### Python (Colab)

```python
# Shape
print(f"Rows: {df.shape[0]}, Columns: {df.shape[1]}")

# Data types
print(df.dtypes)

# Null check
print(df.isnull().sum())

# Key distributions
print(df['room_type'].value_counts())
print(df['minimum_nights'].describe())
print(df['number_of_reviews'].describe())
print(df['availability_365'].describe())

# How many hosts have multiple listings?
multi = df[df['calculated_host_listings_count'] > 1].shape[0]
print(f"Multi-listing host entries: {multi} / {df.shape[0]} ({multi/df.shape[0]*100:.1f}%)")

# License coverage
licensed = df[df['license'].notna() & (df['license'] != '')].shape[0]
print(f"Licensed: {licensed} / {df.shape[0]} ({licensed/df.shape[0]*100:.1f}%)")
```

### SQL (BigQuery)

```sql
-- Row count and null overview
SELECT
  COUNT(*) AS total_rows,
  COUNT(DISTINCT neighbourhood) AS unique_neighbourhoods,
  COUNT(DISTINCT host_id) AS unique_hosts,
  COUNTIF(reviews_per_month IS NULL) AS null_reviews_per_month,
  COUNTIF(last_review IS NULL) AS null_last_review,
  COUNTIF(license IS NOT NULL AND license != '') AS licensed_count
FROM `airbnb_toronto.listings`;

-- Room type distribution
SELECT
  room_type,
  COUNT(*) AS listing_count,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 1) AS pct
FROM `airbnb_toronto.listings`
GROUP BY room_type
ORDER BY listing_count DESC;

-- Minimum nights distribution
SELECT
  MIN(minimum_nights) AS min_val,
  MAX(minimum_nights) AS max_val,
  APPROX_QUANTILES(minimum_nights, 2)[OFFSET(1)] AS median_val,
  ROUND(AVG(minimum_nights), 1) AS avg_val
FROM `airbnb_toronto.listings`;

-- Multi-listing hosts
SELECT
  COUNTIF(calculated_host_listings_count > 1) AS multi_listing_entries,
  COUNT(*) AS total,
  ROUND(COUNTIF(calculated_host_listings_count > 1) * 100.0 / COUNT(*), 1) AS pct
FROM `airbnb_toronto.listings`;
```

**What you should find:**

- 15,776 listings across 140 neighbourhoods
- 67% entire homes, 33% private rooms, <1% shared/hotel
- Median minimum_nights = 28 (significant long-term rental presence)
- 3,231 listings (20%) have zero reviews
- 47% of listings come from hosts with multiple properties
- 57% of listings have a license on file

---

## Step 2: Clean & Transform

**Goal:** Create the derived columns that will power the analysis and dashboard.

### Python (Colab)

```python
# 1. Fill null reviews_per_month with 0
df['reviews_per_month'] = df['reviews_per_month'].fillna(0)

# 2. Categorize rental type based on minimum_nights
def rental_category(nights):
    if pd.isna(nights):
        return 'Unknown'
    elif nights <= 7:
        return 'Short-term (1-7)'
    elif nights <= 27:
        return 'Medium (8-27)'
    else:
        return 'Long-term (28+)'

df['rental_type'] = df['minimum_nights'].apply(rental_category)

# 3. Flag professional hosts
df['host_type'] = df['calculated_host_listings_count'].apply(
    lambda x: 'Professional' if x > 1 else 'Casual'
)

# 4. Flag licensed listings
df['is_licensed'] = df['license'].notna() & (df['license'] != '')

# 5. Flag active listings (has reviews OR has availability)
df['is_active'] = (df['number_of_reviews'] > 0) | (df['availability_365'] > 0)

# Verify transforms
print(df['rental_type'].value_counts())
print(df['host_type'].value_counts())
print(f"Licensed: {df['is_licensed'].sum()}")
print(f"Active: {df['is_active'].sum()}")
```

### SQL (BigQuery)

```sql
-- Create a cleaned view with all derived columns
CREATE OR REPLACE VIEW `airbnb_toronto.listings_clean` AS
SELECT
  *,
  COALESCE(reviews_per_month, 0) AS rpm_clean,

  CASE
    WHEN minimum_nights <= 7 THEN 'Short-term (1-7)'
    WHEN minimum_nights <= 27 THEN 'Medium (8-27)'
    ELSE 'Long-term (28+)'
  END AS rental_type,

  CASE
    WHEN calculated_host_listings_count > 1 THEN 'Professional'
    ELSE 'Casual'
  END AS host_type,

  CASE
    WHEN license IS NOT NULL AND license != '' THEN TRUE
    ELSE FALSE
  END AS is_licensed,

  CASE
    WHEN number_of_reviews > 0 OR availability_365 > 0 THEN TRUE
    ELSE FALSE
  END AS is_active

FROM `airbnb_toronto.listings`;

-- Verify
SELECT
  rental_type, COUNT(*) AS cnt
FROM `airbnb_toronto.listings_clean`
GROUP BY rental_type
ORDER BY cnt DESC;

SELECT
  host_type, COUNT(*) AS cnt
FROM `airbnb_toronto.listings_clean`
GROUP BY host_type;
```

---

## Step 3: Analyze

### Q1: Supply Concentration

**What we're asking:** Where is the supply? Which neighbourhoods have the most listings, and what types dominate?

#### Python

```python
# Top 15 neighbourhoods by listing count
top_hoods = (
    df.groupby('neighbourhood')
    .size()
    .reset_index(name='listing_count')
    .sort_values('listing_count', ascending=False)
    .head(15)
)
print(top_hoods)

# Neighbourhood × room type breakdown (top 15)
supply = (
    df[df['neighbourhood'].isin(top_hoods['neighbourhood'])]
    .groupby(['neighbourhood', 'room_type'])
    .size()
    .reset_index(name='count')
    .pivot(index='neighbourhood', columns='room_type', values='count')
    .fillna(0)
)
print(supply)

# Room type overall split
print(df['room_type'].value_counts(normalize=True).round(3))

# Rental type split
print(df['rental_type'].value_counts(normalize=True).round(3))
```

#### SQL

```sql
-- Top 15 neighbourhoods
SELECT
  neighbourhood,
  COUNT(*) AS listing_count,
  ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM `airbnb_toronto.listings_clean`), 1) AS pct_of_total
FROM `airbnb_toronto.listings_clean`
GROUP BY neighbourhood
ORDER BY listing_count DESC
LIMIT 15;

-- Neighbourhood × room type (top 15)
SELECT
  neighbourhood,
  room_type,
  COUNT(*) AS cnt
FROM `airbnb_toronto.listings_clean`
WHERE neighbourhood IN (
  SELECT neighbourhood
  FROM `airbnb_toronto.listings_clean`
  GROUP BY neighbourhood
  ORDER BY COUNT(*) DESC
  LIMIT 15
)
GROUP BY neighbourhood, room_type
ORDER BY neighbourhood, cnt DESC;

-- Rental type distribution
SELECT
  rental_type,
  COUNT(*) AS cnt,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 1) AS pct
FROM `airbnb_toronto.listings_clean`
GROUP BY rental_type
ORDER BY cnt DESC;
```

### Q2: Demand Signals

**What we're asking:** Where is demand strongest? We use `reviews_per_month` as a proxy for booking frequency, since Airbnb doesn't publish booking data publicly.

#### Python

```python
# Filter to active listings with reviews
active = df[df['number_of_reviews'] > 0].copy()

# Demand by neighbourhood
demand = (
    active.groupby('neighbourhood')
    .agg(
        avg_rpm=('reviews_per_month', 'mean'),
        median_rpm=('reviews_per_month', 'median'),
        total_reviews=('number_of_reviews', 'sum'),
        listing_count=('id', 'count')
    )
    .round(2)
)

# Top 15 by average review velocity
print(demand.sort_values('avg_rpm', ascending=False).head(15))

# Compare: high supply vs high demand
# Merge supply count with demand metrics
comparison = demand.copy()
comparison = comparison.sort_values('listing_count', ascending=False).head(20)
print(comparison[['listing_count', 'avg_rpm', 'total_reviews']])
```

#### SQL

```sql
-- Demand proxy by neighbourhood (active listings only)
SELECT
  neighbourhood,
  COUNT(*) AS listing_count,
  ROUND(AVG(rpm_clean), 2) AS avg_rpm,
  ROUND(APPROX_QUANTILES(rpm_clean, 2)[OFFSET(1)], 2) AS median_rpm,
  SUM(number_of_reviews) AS total_reviews
FROM `airbnb_toronto.listings_clean`
WHERE number_of_reviews > 0
GROUP BY neighbourhood
HAVING COUNT(*) >= 10  -- filter tiny neighbourhoods
ORDER BY avg_rpm DESC
LIMIT 15;

-- Supply vs demand scatter data
SELECT
  neighbourhood,
  COUNT(*) AS listing_count,
  ROUND(AVG(rpm_clean), 2) AS avg_rpm,
  SUM(number_of_reviews) AS total_reviews
FROM `airbnb_toronto.listings_clean`
WHERE number_of_reviews > 0
GROUP BY neighbourhood
HAVING COUNT(*) >= 10
ORDER BY listing_count DESC;
```

### Q3: Competitive Landscape

**What we're asking:** What fraction of the market is controlled by professional operators (hosts with more than one listing)?

#### Python

```python
# Overall split
print(df['host_type'].value_counts())
print(df['host_type'].value_counts(normalize=True).round(3))

# Host type × room type
host_room = (
    df.groupby(['host_type', 'room_type'])
    .size()
    .reset_index(name='count')
)
print(host_room)

# Performance comparison
perf = (
    df.groupby('host_type')
    .agg(
        listings=('id', 'count'),
        avg_rpm=('reviews_per_month', 'mean'),
        avg_availability=('availability_365', 'mean'),
        avg_min_nights=('minimum_nights', 'mean'),
        licensed_pct=('is_licensed', 'mean')
    )
    .round(2)
)
print(perf)

# Top operators
top_hosts = (
    df.groupby(['host_id', 'host_name'])
    .agg(listing_count=('id', 'count'))
    .sort_values('listing_count', ascending=False)
    .head(10)
)
print(top_hosts)
```

#### SQL

```sql
-- Host type overview
SELECT
  host_type,
  COUNT(*) AS listings,
  ROUND(AVG(rpm_clean), 2) AS avg_rpm,
  ROUND(AVG(availability_365), 0) AS avg_availability,
  ROUND(AVG(minimum_nights), 0) AS avg_min_nights,
  ROUND(COUNTIF(is_licensed) * 100.0 / COUNT(*), 1) AS licensed_pct
FROM `airbnb_toronto.listings_clean`
GROUP BY host_type;

-- Host type × room type
SELECT
  host_type,
  room_type,
  COUNT(*) AS cnt
FROM `airbnb_toronto.listings_clean`
GROUP BY host_type, room_type
ORDER BY host_type, cnt DESC;

-- Top 10 operators by listing count
SELECT
  host_id,
  host_name,
  COUNT(*) AS listing_count
FROM `airbnb_toronto.listings_clean`
GROUP BY host_id, host_name
ORDER BY listing_count DESC
LIMIT 10;
```

### Q4: Regulatory Compliance

**What we're asking:** What percentage of listings are licensed, and how does this vary by neighbourhood and host type?

#### Python

```python
# Overall compliance
print(f"Licensed: {df['is_licensed'].sum()} / {len(df)} ({df['is_licensed'].mean()*100:.1f}%)")

# Compliance by neighbourhood (top 20 by listing count)
compliance = (
    df.groupby('neighbourhood')
    .agg(
        total=('id', 'count'),
        licensed=('is_licensed', 'sum')
    )
)
compliance['pct_licensed'] = (compliance['licensed'] / compliance['total'] * 100).round(1)
compliance = compliance.sort_values('total', ascending=False).head(20)
print(compliance)

# Compliance by host type
print(
    df.groupby('host_type')['is_licensed']
    .agg(['sum', 'count', 'mean'])
    .round(3)
)

# Compliance by rental type
print(
    df.groupby('rental_type')['is_licensed']
    .agg(['sum', 'count', 'mean'])
    .round(3)
)
```

#### SQL

```sql
-- Overall compliance rate
SELECT
  COUNTIF(is_licensed) AS licensed,
  COUNT(*) AS total,
  ROUND(COUNTIF(is_licensed) * 100.0 / COUNT(*), 1) AS pct_licensed
FROM `airbnb_toronto.listings_clean`;

-- Compliance by neighbourhood (top 20 by volume)
SELECT
  neighbourhood,
  COUNT(*) AS total,
  COUNTIF(is_licensed) AS licensed,
  ROUND(COUNTIF(is_licensed) * 100.0 / COUNT(*), 1) AS pct_licensed
FROM `airbnb_toronto.listings_clean`
GROUP BY neighbourhood
ORDER BY total DESC
LIMIT 20;

-- Compliance by host type
SELECT
  host_type,
  COUNT(*) AS total,
  COUNTIF(is_licensed) AS licensed,
  ROUND(COUNTIF(is_licensed) * 100.0 / COUNT(*), 1) AS pct_licensed
FROM `airbnb_toronto.listings_clean`
GROUP BY host_type;

-- Compliance by rental type
SELECT
  rental_type,
  COUNT(*) AS total,
  COUNTIF(is_licensed) AS licensed,
  ROUND(COUNTIF(is_licensed) * 100.0 / COUNT(*), 1) AS pct_licensed
FROM `airbnb_toronto.listings_clean`
GROUP BY rental_type;
```

---

## Step 4: Export for Tableau

Before building the dashboard, export the cleaned and transformed data.

### Python (Colab)

```python
# Select columns for Tableau
export_cols = [
    'id', 'name', 'host_id', 'host_name', 'neighbourhood',
    'latitude', 'longitude', 'room_type', 'minimum_nights',
    'number_of_reviews', 'reviews_per_month', 'availability_365',
    'calculated_host_listings_count', 'number_of_reviews_ltm',
    'rental_type', 'host_type', 'is_licensed', 'is_active'
]

df[export_cols].to_csv('listings_clean.csv', index=False)
# Download from Colab: Files panel → right-click → Download
```

### SQL (BigQuery)

```sql
-- Export the cleaned view
-- In BigQuery console: run SELECT * FROM `airbnb_toronto.listings_clean`
-- Then click "Save Results" → CSV (local download)
-- Or use EXPORT DATA for GCS
SELECT * FROM `airbnb_toronto.listings_clean`;
```

---

## Step 5: Build the Tableau Dashboard

### Setup

1. Download and install [Tableau Public](https://public.tableau.com/) (free)
2. Open Tableau Public → Connect → Text File → select `listings_clean.csv`

### Sheet 1: Supply Map

**Chart type:** Horizontal stacked bar

1. Drag `neighbourhood` to Rows
2. Drag `Number of Records` (or `id` as COUNT) to Columns
3. Drag `room_type` to Color
4. Sort descending by listing count
5. Filter to Top 15 neighbourhoods by listing count
6. Title: "Listing Supply by Neighbourhood and Room Type"

**What to look for:** Waterfront Communities dominates. Most high-supply areas are entire-home heavy.

### Sheet 2: Demand vs Supply Scatter

**Chart type:** Scatter plot

1. Create a calculated field: right-click in Data pane → Create Calculated Field
   - Name: `Avg Reviews per Month`
   - Formula: `AVG([Reviews Per Month])`
2. Drag `neighbourhood` to Detail
3. Drag `CNT(id)` or `Number of Records` to Columns (this is listing count per neighbourhood)
4. Drag `Avg Reviews per Month` to Rows
5. Drag `SUM(Number of Reviews)` to Size
6. Add a reference line on the Y-axis (Analytics pane → drag Reference Line) for the overall average
7. Title: "Supply Volume vs Demand Intensity by Neighbourhood"

**What to look for:** Quadrants matter. Top-right = high supply + high demand (competitive). Top-left = low supply + high demand (opportunity). Bottom-right = high supply + low demand (saturated).

### Sheet 3: Host Type & Compliance

**Chart type:** Side-by-side — treemap + highlight table

**Left: Host Type Treemap**
1. Drag `host_type` to Color
2. Drag `room_type` to Detail
3. Drag `Number of Records` to Size
4. Label with `host_type` and `room_type`

**Right: Compliance Highlight Table**
1. Create a calculated field:
   - Name: `Licensed Pct`
   - Formula: `SUM(IF [Is Licensed] THEN 1 ELSE 0 END) / COUNT([Id])`
2. Drag `neighbourhood` to Rows (filter to Top 15)
3. Drag `Licensed Pct` to Text and Color
4. Format as percentage
5. Use a diverging color palette (red-green or orange-blue)

### Combine into a Dashboard

1. New Dashboard (Dashboard menu → New Dashboard)
2. Size: Automatic or 1200×800
3. Drag all three sheets onto the canvas
4. Add a title text box: "Toronto Airbnb Market Intelligence"
5. Add a filter: allow filtering by `room_type` across all sheets

### Publish

1. File → Save to Tableau Public
2. Create an account if you don't have one
3. The published URL is your portfolio link

---

## Deliverable Structure

Structure your findings like a working analyst would present to leadership:

**Executive Summary (2 sentences):**
State the overall market picture. Example: "Toronto's Airbnb market has 15,776 listings heavily concentrated in downtown/waterfront areas, with 67% being entire-home rentals. Nearly half the supply is controlled by professional multi-listing operators, and licensing compliance sits at 57% citywide with significant variation by neighbourhood."

**3 Key Findings (with evidence):**
Each finding = one sentence + one chart reference. No methodology.

**1 Recommendation:**
Based on the data, what should the company do? Which neighbourhoods, which property types, which host strategy? Be specific.

**Gaps & Next Steps:**
What this data cannot answer: pricing/revenue, actual booking rates, guest demographics, seasonal trends (this is a single snapshot). Acknowledging gaps builds credibility.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Python (pandas, matplotlib) OR BigQuery SQL | Data loading, profiling, cleaning, transformation, analysis |
| Google Colab OR BigQuery Console | Execution environment (browser-based, nothing to install) |
| Tableau Public | Dashboard creation and publishing |
| GitHub | Code and documentation hosting |

---

## License

This project uses data from [Inside Airbnb](http://insideairbnb.com/), available under a Creative Commons Attribution 4.0 International License.
