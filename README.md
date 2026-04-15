# NHL Player Analytics Pipeline

A data pipeline and machine learning system for scraping, processing, and analyzing NHL skater statistics. Pulls live data from the NHL API, computes advanced per-60 metrics, clusters players by role, and identifies similar players using K-Nearest Neighbors. This is part of an ongoing project aimed at designing an end-to-end development pipeline with both backend and frontend interfaces. Currently, these scripts act as a backend workflow to a AWS based web application using AWS services such as s3, RDS, Lambda, API Gateway, and a public/private VPC.

---

## Project Structure

```
├── scraping.py        # NHL API data collection
├── Model.py           # ML modeling: KNN similarity + KMeans clustering
├── upload.py          # S3 upload for processed CSVs
└── files/             # Generated data files (gitignored)
    ├── skater_career_stats.csv
    ├── skater_current_stats.csv
    ├── personal_data.csv
    ├── forwards_data.csv
    └── team_standings.csv
```

---

## How It Works

### 1. `scraping.py` — Data Collection

Hits the [NHL Web API](https://api-web.nhle.com/) to pull current-season data for every active NHL skater.

- Fetches all 32 team rosters from live standings
- Collects per-player biographical data (position, height, weight, birthplace, etc.)
- Pulls full career `seasonTotals` for each skater
- Filters to current-season NHL stats and converts `avgToi` from `MM:SS` to seconds
- Merges personal + current stats and computes **per-60 minute rates**:
  - `goals_per_60`, `assists_per_60`, `points_per_60`, `shots_per_60`
- Filters to forwards only (C, L, R) and saves to `files/forwards_data.csv`
- Also scrapes team-level summary stats (shots, power play %, penalty kill %, etc.)

**Output files:**
| File | Contents |
|---|---|
| `skater_career_stats.csv` | Full career stats for all skaters |
| `skater_current_stats.csv` | Current-season stats only |
| `personal_data.csv` | Biographical info for all skaters |
| `forwards_data.csv` | Merged forwards data with per-60 metrics |
| `team_standings.csv` | Team-level season summary stats |

---

### 2. `Model.py` — ML Modeling

Runs two complementary ML models on the processed forwards data.

#### K-Nearest Neighbors (Player Similarity)
Finds the most statistically similar players to any given forward.

- Features used: `goals_per_60`, `assists_per_60`, `points_per_60`, `shots_per_60`, `avgToi`, `powerPlayPoints`
- Features are standardized with `StandardScaler` then weighted:

| Feature | Weight |
|---|---|
| `goals_per_60` | 2.0 |
| `assists_per_60` | 2.0 |
| `points_per_60` | 3.0 |
| `shots_per_60` | 1.5 |
| `avgToi` | 1.0 |
| `powerPlayPoints` | 1.5 |

- Uses **cosine similarity** via `sklearn.NearestNeighbors`
- Returns the 5 closest matches (excluding self) for any target player index

#### KMeans Clustering (Player Archetypes)
Groups forwards into role-based archetypes.

**3-cluster system:**
| Cluster | Label |
|---|---|
| 0 | Top-line scorer |
| 1 | Secondary / Two-way |
| 2 | Depth / Grinder |

**5-cluster system (optional):**
| Cluster | Label |
|---|---|
| 0 | Top-line scorer |
| 1 | Depth / Middle-six |
| 2 | Second-line scorer |
| 3 | Bottom-six / depth |
| 4 | Third-line / secondary scoring |

An **elbow method** plot is included to help choose the optimal `k`.

---

### 3. `upload.py` — S3 Upload

Uploads all generated CSVs from the `files/` directory to an AWS S3 bucket.

- Target bucket: `hockey.josephbensenportfolio.com`
- Files are uploaded to the `CSVs/` prefix
- Uses `boto3` with default AWS credential resolution (env vars, `~/.aws/credentials`, or IAM role)

---

## Getting Started

### Prerequisites

```bash
pip install pandas numpy scikit-learn matplotlib requests boto3
```

### Run the full pipeline

```bash
# 1. Scrape and process data from the NHL API
python scraping.py

# 2. Train models and generate clusters/similarity scores
python Model.py

# 3. Upload results to S3
python upload.py
```

> **Note:** AWS credentials must be configured before running `upload.py`. See the [boto3 credentials docs](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html).

---

## 🔧 Configuration

To find similar players for a different target, update the index in `Model.py`:

```python
target_idx = 326  # Change to any row index in forwards_data.csv
```

To switch between 3-cluster and 5-cluster archetypes, update:

```python
k = 3  # or k = 5
# and update the label map accordingly:
forwards_df = assign_labels(forwards_df, labels, CLUSTER_LABELS_3)
# or
forwards_df = assign_labels(forwards_df, labels, CLUSTER_LABELS_5)
```

---

## 📊 Data Source

All data is sourced from the public **NHL Web API**:
- `https://api-web.nhle.com/` — rosters, player landing pages
- `https://api.nhle.com/stats/rest/` — team summary stats

No API key required.