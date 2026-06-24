# Masterworks Art Market Analysis
 
A data pipeline and analysis project exploring the secondary art market through Masterworks' public sales data. This project covers data acquisition, cleaning, API scraping, SQL analysis, and business intelligence dashboarding.
 
---
 
## Project Overview
 
[Masterworks](https://www.masterworks.com) is a fintech platform that allows retail investors to buy fractional shares of blue-chip artworks. This project uses their publicly accessible GraphQL API to collect historical sale data across hundreds of artists, loads it into a PostgreSQL database, and analyzes it using SQL and Power BI.

 
## Project Structure
 

 
## Step 1: Artist Name Extraction and Cleaning
 
**Data source:** [Kaggle — WikiArt](https://www.kaggle.com/datasets/steubk/wikiart)
 
The dataset contains 80,020 unique images from 1,119 different artists across 27 styles. For this project, only the artist names were needed as a seed list to query the Masterworks API.
 
**Cleaning steps:**
1. Extracted unique artist names from the `artist` column
2. Stripped leading/trailing whitespace
3. Converted names to URL slugs (lowercase, spaces replaced with hyphens, periods removed) to match Masterworks' API format
```python
import pandas as pd
 
df = pd.read_csv("classes.csv")
 
# extract unique artists
artists = df["artist"].unique()
artist_df = pd.DataFrame(artists, columns=["artist"])
 
# normalize names
artist_df["artist_clean"] = artist_df["artist"].str.strip()
 
# convert to slugs for API requests
artist_df["slug"] = (
    artist_df["artist_clean"]
    .str.lower()
    .str.replace(" ", "-")
    .str.replace(".", "")
)
 
artist_df.to_csv("artists_clean.csv", index=False)
```
 
Output: `artists_clean.csv` with columns `artist`, `artist_clean`, `slug` — 1,119 artists ready to query.
 
---
 
## Step 2: Scraping the Masterworks GraphQL API
 
Masterworks exposes a GraphQL API at `https://api.masterworks.com/graphql`. Two queries are used:
 
**Query 1 — `ArtistDetails`:** Fetches the internal artist ID and metadata given a slug.
 
**Query 2 — `ArtistHistoricalReturn`:** Fetches all historical repeat sale pairs for an artist (works that Masterworks bought and later sold), paginating through all results.
 
The scraper loops through all 1,119 artist slugs, fetches the artist ID for each, then pulls all available sale records. Artists not found on Masterworks are skipped.

```python
import requests
import pandas as pd
import time
import os

# Config 
ENDPOINT = "https://api.masterworks.com/graphql"
ARTISTS_CSV = "artists_clean.csv"      
OUTPUT_CSV  = "masterworks_sales.csv"
PAGE_SIZE   = 50                   # max results per page
SLEEP_SEC   = 0.2                 # polite delay between requests

HEADERS = {
    "accept": "*/*",
    "accept-language": "en-US,en;q=0.9",
    "content-type": "application/json",
    "origin": "https://www.masterworks.com",
    "referer": "https://www.masterworks.com/",
    "user-agent": (
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
        "AppleWebKit/537.36 (KHTML, like Gecko) "
        "Chrome/149.0.0.0 Safari/537.36 Edg/149.0.0.0"
    ),
    "x-location": "https://www.masterworks.com/research",
}

# GraphQL queries 
ARTIST_DETAILS_QUERY = """
query ArtistDetails($slug: String) {
  artist(slug: $slug) {
    id
    slug
    name
    __typename
  }
}
"""

HISTORICAL_RETURN_QUERY = """
query ArtistHistoricalReturn(
  $artistId: ID!
  $first: Int
  $page: Int!
  $sortBy: WorkRepeatSalePairsSortBy
) {
  workRepeatSalePairs(
    first: $first
    page: $page
    where: { artistId: { equals: $artistId } }
    sortBy: [$sortBy]
  ) {
    edges {
      node {
        id
        slug
        saleDate
        workTitle
        workImageUrl
        salePriceAmountCents
        purchaseDate
        purchasePriceAmountCents
        moic
      }
    }
    pageInfo {
      total
      nextPage
    }
  }
}
"""

# Helpers
def gql(query, variables):
    resp = requests.post(
        ENDPOINT,
        json={"query": query, "variables": variables},
        headers=HEADERS,
        timeout=15,
    )
    resp.raise_for_status()
    data = resp.json()
    if "errors" in data:
        raise ValueError(f"GraphQL errors: {data['errors']}")
    return data["data"]


def get_artist_id(slug):
    data = gql(ARTIST_DETAILS_QUERY, {"slug": slug})
    artist = data.get("artist")
    if not artist:
        return None, None
    return artist["id"], artist["name"]


def get_all_sales(artist_id, artist_name, artist_slug):
    rows = []
    page = 0
    sort_by = {"attribute": "MOIC", "direction": "DESC"}

    while True:
        data = gql(
            HISTORICAL_RETURN_QUERY,
            {
                "artistId": artist_id,
                "first": PAGE_SIZE,
                "page": page,
                "sortBy": sort_by,
            },
        )
        pairs = data.get("workRepeatSalePairs", {})
        edges = pairs.get("edges", [])
        page_info = pairs.get("pageInfo", {})

        for edge in edges:
            node = edge["node"]
            rows.append(
                {
                    "artist":                   artist_name,
                    "artist_slug":              artist_slug,
                    "id":                       node.get("id"),
                    "slug":                     node.get("slug"),
                    "work_title":               node.get("workTitle"),
                    "image_url":                node.get("workImageUrl"),
                    "purchase_date":            node.get("purchaseDate"),
                    "purchase_price_usd_cents": node.get("purchasePriceAmountCents"),
                    "sale_date":                node.get("saleDate"),
                    "sale_price_usd_cents":     node.get("salePriceAmountCents"),
                    "moic":                     node.get("moic"),
                }
            )

        next_page = page_info.get("nextPage")
        if next_page is None or not edges:
            break
        page = next_page
        time.sleep(SLEEP_SEC)

    return rows


# Main
def main():
    artists_df = pd.read_csv(ARTISTS_CSV)

    # support either 'slug' or 'artist_clean' as the slug column
    if "slug" not in artists_df.columns:
        raise ValueError("CSV must have a 'slug' column")

    all_rows = []
    total = len(artists_df)

    for i, row in artists_df.iterrows():
        slug = str(row["slug"]).strip()
        print(f"[{i+1}/{total}] Fetching artist: {slug} ...", end=" ")

        try:
            artist_id, artist_name = get_artist_id(slug)
            if not artist_id:
                print("NOT FOUND — skipping")
                continue

            time.sleep(SLEEP_SEC)
            sales = get_all_sales(artist_id, artist_name, slug)
            print(f"{len(sales)} sales found")
            all_rows.extend(sales)

        except Exception as e:
            print(f"ERROR: {e} — skipping")

        time.sleep(SLEEP_SEC)

    if not all_rows:
        print("No data collected.")
        return

    df = pd.DataFrame(all_rows)

    # convert cents to dollars for readability
    df["purchase_price_usd"] = df["purchase_price_usd_cents"] / 100
    df["sale_price_usd"]     = df["sale_price_usd_cents"] / 100

    df.to_csv(OUTPUT_CSV, index=False)
    print(f"\nDone. {len(df)} total rows saved to {OUTPUT_CSV}")


if __name__ == "__main__":
    main()
```
 
**Output schema:**
 
| Column | Description |
|---|---|
| `artist` | Artist name |
| `artist_slug` | URL slug |
| `id` | Masterworks sale ID |
| `work_title` | Title of the artwork |
| `purchase_date` | Date Masterworks acquired the work |
| `purchase_price_usd` | Acquisition price in USD |
| `sale_date` | Date the work was sold |
| `sale_price_usd` | Sale price in USD |
| `moic` | Multiple on Invested Capital (sale price / purchase price) |


 
---
 
## Step 3: Loading Data into PostgreSQL
 
The cleaned CSV was loaded into a PostgreSQL database hosted on [Supabase](https://supabase.com). A synthetic `row_id` column was added as the primary key to handle cases where the same work appeared in multiple artist queries.
 
**Final table:** `masterworks_sales`
 
---

## Step 4: SQL Analysis

### 1. Which artists have generated the most total revenue?
 
```sql
SELECT
    artist,
    COUNT(*)                                       AS total_sales,
    ROUND(AVG(moic)::numeric, 2)                   AS avg_moic,
    ROUND(SUM(sale_price_usd)::numeric, 2)         AS total_revenue_usd,
    ROUND(AVG(sale_price_usd)::numeric, 2)         AS avg_sale_price_usd
FROM masterworks_sales
GROUP BY artist
ORDER BY total_revenue_usd DESC
LIMIT 5;
```
 
![Query 1 Results](https://github.com/user-attachments/assets/48318e92-a8e5-4a15-a5fa-392d9d076bdc)
 
---
 
### 2. What are the best performing individual sales by return on investment?
 
```sql
SELECT
    artist,
    work_title,
    ROUND(purchase_price_usd::numeric, 2)  AS purchase_price_usd,
    ROUND(sale_price_usd::numeric, 2)      AS sale_price_usd,
    ROUND(moic::numeric, 2)                AS moic,
    purchase_date,
    sale_date
FROM masterworks_sales
ORDER BY moic DESC
LIMIT 5;
```
 
![Query 2 Results](https://github.com/user-attachments/assets/87c259f5-a5ff-4606-92b6-966fe340a548)
 
---
 
### 3. What are the worst performing individual sales?
 
```sql
SELECT
    artist,
    work_title,
    ROUND(purchase_price_usd::numeric, 2)  AS purchase_price_usd,
    ROUND(sale_price_usd::numeric, 2)      AS sale_price_usd,
    ROUND(moic::numeric, 2)                AS moic,
    purchase_date,
    sale_date
FROM masterworks_sales
WHERE moic < 1
ORDER BY moic ASC
LIMIT 5;
```
 
![Query 3 Results](https://github.com/user-attachments/assets/b61b8918-43b9-4d6e-a241-da87399984bc)
 
---
 
### 4. Which artists have delivered the best average returns? (Min. 5 sales)
 
```sql
SELECT
    artist,
    COUNT(*)                          AS total_sales,
    ROUND(AVG(moic)::numeric, 2)      AS avg_moic,
    ROUND(MIN(moic)::numeric, 2)      AS min_moic,
    ROUND(MAX(moic)::numeric, 2)      AS max_moic
FROM masterworks_sales
GROUP BY artist
HAVING COUNT(*) >= 3
ORDER BY avg_moic DESC
LIMIT 5;
```
 
![Query 4 Results](https://github.com/user-attachments/assets/78d03bf9-70a9-4156-879d-3cd81a0a0c6f)
 
---
 
### 5. Which artist generated the most revenue each year?
 
```sql
WITH yearly_artist_revenue AS (
    SELECT
        EXTRACT(YEAR FROM sale_date::date)   AS sale_year,
        artist,
        SUM(sale_price_usd)                  AS total_revenue_usd,
        COUNT(*)                             AS total_sales
    FROM masterworks_sales
    WHERE sale_date IS NOT NULL
    GROUP BY sale_year, artist
),
ranked AS (
    SELECT *,
        RANK() OVER (PARTITION BY sale_year ORDER BY total_revenue_usd DESC) AS rank
    FROM yearly_artist_revenue
)
SELECT
    sale_year,
    artist,
    total_sales,
    ROUND(total_revenue_usd::numeric, 2) AS total_revenue_usd
FROM ranked
WHERE rank = 1
ORDER BY sale_year;
```
 
![Query 5 Results](https://github.com/user-attachments/assets/7c0535b8-a2cf-45df-b6d0-646e17348abb)
 
---
 
### 6. What was the most expensive painting sold each year?
 
```sql
WITH yearly_max AS (
    SELECT
        EXTRACT(YEAR FROM sale_date::date)   AS sale_year,
        artist,
        work_title,
        sale_price_usd,
        moic,
        RANK() OVER (
            PARTITION BY EXTRACT(YEAR FROM sale_date::date)
            ORDER BY sale_price_usd DESC
        )                                    AS rank
    FROM masterworks_sales
    WHERE sale_date IS NOT NULL
)
SELECT
    sale_year,
    artist,
    work_title,
    ROUND(sale_price_usd::numeric, 2) AS sale_price_usd,
    ROUND(moic::numeric, 2)           AS moic
FROM yearly_max
WHERE rank = 1
ORDER BY sale_year;
```
 
![Query 6 Results](https://github.com/user-attachments/assets/e9e9f5ff-6807-49e7-911d-88e8c72bc0fb)
 
---
