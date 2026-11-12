
# Kreanara Digital Marketing — Marketing Funnel & Channel Performance Analysis

End-to-end analysis of a **Rp 7.8B digital marketing funnel** (308.7M impressions → 5.4M clicks → 145.5K conversions) across 6 channels — from raw CSVs to an interactive dashboard, including a **data quality audit** on a synthetic dataset with deliberately injected defects.

**🔗 Live dashboard:** [Looker Studio](https://datastudio.google.com/reporting/f09d0d15-172e-4457-9c08-875a0b66bee6)


![Dashboard preview]
<img width="897" height="694" alt="image" src="https://github.com/user-attachments/assets/f1176155-cc37-484e-9abe-ff870f1a5d63" />


---

## Project Overview

| | |
|---|---|
| **Business question** | Which marketing channels deserve more (or less) budget, and can the data be trusted to answer that? |
| **Dataset** | Synthetic ("Kreanara"), 5 CSVs · 55K+ rows · 4 data defects deliberately injected |
| **Stack** | Python (pandas) · PostgreSQL · Looker Studio |
| **Deliverables** | Interactive dashboard · SQL analysis suite · data quality audit with documented decisions |

### Pipeline

```
raw CSVs ──► pandas (cleaning & validation) ──► PostgreSQL (star-ish schema, 5 tables) ──► Looker Studio
```

### Dataset

| Table | Rows | Grain |
|---|---|---|
| `clients` | 50 | one row per client |
| `campaigns` | 500 | one row per campaign (channel, objective, budget, actual spend) |
| `ad_spend` | 3,000 | daily spend & funnel metrics per campaign |
| `leads` | 2,500 | one row per lead (score 1–100, status, acquisition channel) |
| `conversions` | 400 | one row per tracked conversion (revenue, CAC, profit) |

> ⚠️ The dataset is synthetic. Four defects were deliberately injected to simulate real-world data: channel name variants (`Instagram` / `IG` / `instagram`), ~10% NULL `conversions` in `ad_spend`, ~20% empty `acquisition_channel` in `leads`, and 15 orphaned conversion records.

---

## Key Findings

### 1. Budget misallocation
Instagram absorbs **~40% of total spend** (Rp 3.17B of Rp 7.83B) while performing at par on every efficiency metric. Meanwhile the highest-CPA channel (Google, Rp 58.5K) delivers the **highest lead quality score** (54.8 vs 52.0 average) — showing that cost-only optimization would cut the most valuable channel.

### 2. Cost metrics mislead without value metrics
The CPA ranking and the lead-quality ranking are nearly inverted. LinkedIn has the **cheapest CPL (Rp 1.9M) but the lowest lead score (49.3)** — the classic "cheap but weak" trap. Every budget recommendation in this project therefore pairs a cost metric with a value metric.

### 3. Volume-vs-Quality quadrant framework
Channels are segmented by lead volume and average lead score against portfolio averages into four actionable quadrants:

| Quadrant | Channels | Recommendation |
|---|---|---|
| **Scale Up** (low volume, high quality) | Google, TikTok | Increase reach, quality already proven |
| **Nurture** (high volume, high quality) | YouTube, Instagram* | Maintain and protect |
| **Optimize** (high volume, low quality) | — | Improve targeting before adding budget |
| **Cut** (low volume, low quality) | Facebook, LinkedIn | Deprioritize or rework |

<sub>*Instagram sits within 0.15 pts of the quality threshold — treated as directional, see Limitations.</sub>

---

## Data Quality Audit

Three material issues were found, verified, and resolved with documented decisions:

| # | Finding | Evidence | Decision |
|---|---|---|---|
| 1 | **Cross-objective lead misattribution** — dataset docs claim leads only come from `lead_gen` campaigns, but 48.8% of leads (1,221 / 2,500) attach to `conversions`-objective campaigns | Verification join `leads × campaigns.objective` | CPL defined strictly on `lead_gen` campaigns (1,279 leads); other leads treated as by-product. The initial unfiltered CPL distorted per-channel lead scores by **up to 4 points** (TikTok: 48.4 → 52.4) |
| 2 | **Revenue tracking covers only 0.3% of conversion volume** — `conversions` holds 400 tracked records vs 145,461 conversions in `ad_spend` | Row-count reconciliation | **ROAS excluded from reporting** (would be denominated against untracked volume). Revenue retained only as per-tracked-record metrics (AOV, avg CAC) with explicit n=400 caveat |
| 3 | **Cross-table spend discrepancy** — `campaigns.actual_spend_idr` totals ≈ Rp 13.2B vs `SUM(ad_spend.spend_idr)` ≈ Rp 7.8B | Per-campaign reconciliation query | `ad_spend.spend_idr` chosen as single source of truth (daily grain, date-filterable, consistent with CPA/CPC) |

---

## Methodology

### 1. Cleaning (Python / pandas)
- Normalized channel name variants to a single canonical label per channel
  
````  
  #  KATEGORIKAL — cek variasi penulisan
for col in ['channel', 'objective', 'status']:
    print(f"\n─── {col} ───")
    print(ca[col].value_counts(dropna=False))
    # Cek casing/spasi tersembunyi
    raw = ca[col].nunique()
    clean = ca[col].str.strip().str.lower().nunique()
    print(f"unique raw: {raw} | setelah normalisasi: {clean}")
    if raw != clean:
        print("  ⚠️ ADA VARIASI PENULISAN — perlu standardisasi")
````
- Quantified NULL rates (`ad_spend.conversions` ~10%, `leads.acquisition_channel` ~20%) before deciding handling
````
  # ═══ 4. HIPOTESIS: NaN conversions ↔ status kampanye
m = sp.merge(ca[['campaign_id','status','channel','objective']], on='campaign_id', how='left')
print("\n─── % conversions NaN per status ───")
print(m.groupby('status')['conversions'].apply(lambda s: s.isna().mean()*100).round(1))
print("\n─── % conversions NaN per channel ───")
print(m.groupby('channel')['conversions'].apply(lambda s: s.isna().mean()*100).round(1))
# Kalau ~10% merata di semua grup → missing at random
# Kalau terkonsentrasi di 'active' → missing not at random, JANGAN di-impute 0
````

- Removed/flagged orphaned conversion records failing FK integrity
````
  # ── PK & referential integrity
print("conv_id unik :", co['conv_id'].is_unique)
print("lead_id unik :", co['lead_id'].is_unique)   # True → 1:1

orphan = ~co['campaign_id'].isin(ca['campaign_id'])
print("Konversi dgn campaign_id tak dikenal:", orphan.sum())   # harus 0

conv_camps = co['campaign_id'].nunique()
print(f"Kampanye dgn konversi: {conv_camps}/500 "
      f"({conv_camps/500*100:.0f}%) | tanpa konversi: {500-conv_camps}")

# ── conversion_type: cek variasi penulisan (pelajaran 'channel')
print("\n", co['conversion_type'].value_counts(dropna=False))
print("raw:", co['conversion_type'].nunique(),
      "| normalisasi:", co['conversion_type'].str.strip().str.lower().nunique())

# ── conversion_date: parse + apakah jatuh dalam periode kampanye?
co['conversion_date'] = pd.to_datetime(co['conversion_date'], errors='coerce')
print("\nGagal parse:", co['conversion_date'].isna().sum())
m = co.merge(ca[['campaign_id','start_dt','end_dt']], on='campaign_id', how='left')
outside = (m['conversion_date'] < m['start_dt']) | (m['conversion_date'] > m['end_dt'])
print("⚠️ Konversi di luar periode kampanye:", outside.sum())

# ── Ekonomi per konversi (konsisten internal)
co['profit'] = co['revenue_idr'] - co['cac_idr']
print("\nProfit per konversi:")
print(co['profit'].describe())
print("Konversi rugi (revenue < cac):", (co['profit'] < 0).sum())

# ── revenue negatif / nol (refund? error?)
print("\nRevenue ≤ 0:", (co['revenue_idr'] <= 0).sum())

# ── ROAS per kampanye (HANYA 195 yg convert; pakai actual_spend)
rev_per_camp = co.groupby('campaign_id')['revenue_idr'].sum()
roas = ca.set_index('campaign_id').loc[rev_per_camp.index].assign(
    revenue=rev_per_camp,
    roas=rev_per_camp / ca.set_index('campaign_id')['actual_spend_idr']
)
print("\nROAS (195 kampanye yg convert):")
print(roas['roas'].describe())
print("Kampanye ROAS < 1 (rugi):", (roas['roas'] < 1).sum())
````


- Mapping eksplisit — casing DAN alias sekaligus.
 ````channel_map = {
    'instagram': 'Instagram',
    'ig':        'Instagram',   # ← alias, tak terdeteksi otomatis
    'youtube':   'YouTube',
    'google':    'Google',
    'facebook':  'Facebook',
    'tiktok':    'TikTok',
    'linkedin':  'LinkedIn',
}

cn['channel'] = cn['channel'].str.strip().str.lower().map(channel_map)

# ── Verifikasi hasil
print(cn['channel'].value_counts())
print("\nJumlah channel final:", cn['channel'].nunique())   # → 6

````

### 2. Modeling (PostgreSQL)
- 5-table schema with PK/FK constraints and CHECK constraints (see [`sql/01_schema.sql`](sql/01_schema.sql))
- All rates computed from raw counts at query time — **never** `AVG()` over stored per-row rates (unweighted averages mislead)
- NULL-safe math throughout (`NULLIF` denominators, `COALESCE` on nullable counts)
- LEFT JOINs wherever a dimension must keep zero-activity rows (inner joins were shown to overstate ROAS ~3x)

### 3. Analysis lenses (see [`sql/`](sql/))
| Lens | Metrics | File |
|---|---|---|
| Funnel overview | impressions → clicks → conversions, CTR, CVR | `02_funnel_overview.sql` 
````SELECT
	'impressions' AS funnel,
	SUM(impressions) AS total
FROM ad_spend 

UNION ALL

SELECT 
	'clicks' AS funnel,
	SUM(clicks) AS total
FROM ad_spend 

UNION ALL

SELECT 
	'conversions' AS funnel,
	SUM(conversions) AS total
FROM ad_spend 
````
| Channel cost efficiency | CPA, CPC per channel | `03_channel_performance.sql` |
| Lead generation | CPL (lead_gen only), weighted avg lead score, quadrant segmentation | `04_cpl_lead_quality.sql` |
| Data quality checks | coverage, reconciliation, misattribution | `05_data_quality_checks.sql` |
````
WITH spend_campaigns AS(
SELECT 
	campaign_id,
	SUM(actual_spend_idr) AS spend
FROM campaigns 
WHERE objective = 'lead_gen'
GROUP BY campaign_id
),

lead_data AS (
    SELECT
        campaign_id,
        COUNT(lead_id) AS total_leads,
        AVG(lead_score) AS avg_lead_score
    FROM leads
    GROUP BY campaign_id
),

channel_performance AS (
    SELECT
        c.channel,
        SUM(s.spend) AS total_spend_idr,
        COALESCE(
            SUM(l.total_leads),
            0
        ) AS total_leads,
        ROUND(
            SUM(s.spend) * 1.0
            / NULLIF(
                SUM(l.total_leads),
                0
            ),
            0
        ) AS cpl_idr,
        ROUND(
            SUM(
                l.avg_lead_score * l.total_leads
            ) / NULLIF(
                SUM(l.total_leads),
                0
            ),
            2
        ) AS avg_lead_score
    FROM campaigns c
	LEFT JOIN spend_campaigns s ON c.campaign_id = s.campaign_id
    LEFT JOIN lead_data l ON s.campaign_id = l.campaign_id
    GROUP BY  c.channel
)
SELECT
    channel,
    total_spend_idr,
    total_leads,
    cpl_idr,
    avg_lead_score,
    CASE
        WHEN total_leads >= (
            SELECT AVG(total_leads)
            FROM channel_performance
        )
        AND avg_lead_score >= (
            SELECT AVG(avg_lead_score)
            FROM channel_performance
        )
        THEN 'High Volume - High Quality'
        WHEN total_leads < (
            SELECT AVG(total_leads)
            FROM channel_performance
        )
        AND avg_lead_score >= (
            SELECT AVG(avg_lead_score)
            FROM channel_performance
        )
        THEN 'Low Volume - High Quality'

        WHEN total_leads >= (
            SELECT AVG(total_leads)
            FROM channel_performance
        )
        AND avg_lead_score < (
            SELECT AVG(avg_lead_score)
            FROM channel_performance
        )
        THEN 'High Volume - Low Quality'
        ELSE 'Low Volume - Low Quality'
    END AS performance_segment
FROM channel_performance
````
### 4. Dashboard (Looker Studio)
- KPI scorecards · proportional funnel · CPA & CPC bars (separate axes) · CPL bars · volume-vs-quality quadrant scatter with average reference lines
- Scope separation: overall funnel metrics vs an explicitly labeled **"Lead Gen Campaign only"** panel
- Channel filter + date range controls

---

## Limitations

- **Synthetic data**: rate metrics show far narrower variance across channels (CTR 1.74–1.80%, CVR 2.62–2.75%) than real-world benchmarks, so insights focus on budget allocation and absolute cost efficiency rather than rate comparisons.
- **Quality segmentation is directional**: lead scores span only ~5 points; channels near a threshold can flip quadrants.
- **ROAS not reported**: revenue tracking covers 0.3% of conversion volume (see audit finding #2).
- **On production data, my first steps would be**: validating conversion tracking (NULL semantics & attribution window) and reconciling derived metrics against raw counts.

---

## Repository Structure

```
kreanara-marketing-funnel/
├── README.md
├── data/
│   └── raw/                      # 5 source CSVs (or generation script)
├── notebooks/
│   └── 01_cleaning_validation.ipynb
├── sql/
│   ├── 01_schema.sql             # DDL: tables, constraints, enums
│   ├── 02_funnel_overview.sql
│   ├── 03_channel_performance.sql
│   ├── 04_cpl_lead_quality.sql
│   └── 05_data_quality_checks.sql
├── dashboard/
│   └── channel_performance.png   # dashboard screenshot(s)
└── docs/
    └── decisions.md              # decision log (optional, can live in README)
```

---

## Author

**Elnandi** — Data Analyst (BNSP certified) · 
