# Marketing Campaign Performance Analysis — ROI Diagnostic Dashboard

**A Power BI analysis built to expose a common and costly measurement error: how "Average ROI" is calculated determines which marketing channels look like winners — and the naive version of that calculation is wrong.**

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=flat&logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-Validated-blue)
![Status](https://img.shields.io/badge/Status-Validated-success)
![D](https://github.com/fawzy-911/Marketing-Campaign-Performance-Analysis/blob/main/assets/frame_chrome_win10_light.png)
---

## Business Context

Marketing performance data typically arrives with a pre-aggregated "Average ROI" figure per channel or region, produced by whoever built the source extract — usually a simple `AVERAGE()` across campaigns. That number gets used directly for budget allocation decisions: which channel gets more spend next quarter, which one gets cut.

This project starts from a simple question: **does a simple average of per-campaign ROI actually represent the return the business is getting on its total marketing spend?** The answer, verified against this dataset, is no — and the gap between the two numbers is large enough to reverse a budget decision.

---

## The Core Problem: Simple Average vs. Spend-Weighted ROI

A campaign-level `ROI` column has no way to account for how much was actually spent generating that number. A campaign with $500 in spend and a lucky 300% return sits in the same "average" bucket as a campaign with $50,000 in spend and a 20% return — even though the second campaign moved 100x more revenue.

**Verified finding, this dataset:**

| Channel | Simple Avg ROI (naive `AVERAGE()`) | Spend-Weighted ROI (`(ΣRevenue − ΣSpend) / ΣSpend`) | Gap |
|---|---|---|---|
| **Influencer Marketing** | **110.0%** | **59.3%** | **-50.7 pts** |
| **TV Commercials** | **45.0%** | **70.8%** | **+25.8 pts** |
| Radio Ads | 69.1% | 73.9% | +4.8 pts |
| Billboards | 69.2% | 70.5% | +1.3 pts |
| Search Engine Ads | 60.1% | 63.2% | +3.1 pts |
| Social Media Ads | 66.1% | 67.3% | +1.2 pts |
| Content Marketing | 61.4% | 62.0% | +0.6 pts |
| Email Marketing | 59.6% | 58.7% | -0.9 pts |

**Reading this table straight, a naive average tells a decision-maker to shift budget toward Influencer Marketing (it "looks" like the best channel at 110%) and away from TV Commercials (it "looks" like the weakest, at 45%). The spend-weighted truth says the opposite is closer to correct** — Influencer Marketing's real blended return on the dollars actually spent is roughly half what the naive number implies, while TV Commercials is meaningfully undervalued by the same naive calculation.

The same distortion exists at the regional level, smaller in magnitude but still directionally consequential — North America shows a **74.6% naive average vs. 68.1% weighted**, a 6.5-point overstatement of the region's real blended return.

### The correct DAX pattern

```dax
Total Spend =
sum('Marketing Campaign Performance'[Spend])

Total Revenue =
sum('Marketing Campaign Performance'[Revenue])

Aggregate ROI =                          -- source of truth: spend-weighted
DIVIDE ( [Total Revenue] - [Total Spend], [Total Spend], 0 )

Simple Avg ROI =                         -- kept deliberately, to expose the gap
AVERAGE ( 'Marketing Campaign Performance'[ROI] )

ROI Distortion =
[Simple Avg ROI] - [Aggregate ROI]

ROI Distortion Flag =
VAR Gap = [ROI Distortion]
RETURN
    SWITCH (
        TRUE (),
        Gap > 0.15, "⚠ Overstated",
        Gap < -0.15, "⚠ Understated",
        "✓ Aligned"
    )
```

`Simple Avg ROI` is retained in the model on purpose — not removed — specifically so the dashboard can put both numbers side by side and let the distortion be visible, rather than silently replacing one number with another and hiding the mistake that was actually happening upstream.

---

## Key Insights

| KPI | Result |
|---|---|
| **True Blended ROI (Aggregate ROI)** | 65.6% |
| **Naive Simple-Average ROI (misleading)** | 67.6% |
| **Total Spend** | $25.7M |
| **Total Revenue** | $42.5M |
| **Most distorted channel** | Influencer Marketing (naive average overstates true return by ~51 points) |
| **Most undervalued channel** | TV Commercials (naive average understates true return by ~26 points) |

**Secondary findings:**
- **Cost-per-acquisition varies more by industry than by channel.** Finance campaigns run the lowest CPA (~$22.94) despite not having the highest conversion rate — driven by lower cost-per-click rather than higher funnel efficiency. Automotive runs the highest CPA (~$26.67).
- **Click-through rate and conversion rate move independently**, not together — Retail has the highest CTR (2.29%) but the lowest conversion rate (9.02%), meaning Retail campaigns are good at generating interest but comparatively weak at converting it. This is a funnel-stage diagnosis a blended "performance score" would hide.

---

## Data Modeling

**Tables:** `Marketing Campaign Performance` (fact, 1,000 rows, campaign-grain), `Marketing Campaign Details` (channel-level reference), `Region Performance` (region-level reference), plus a dedicated `_Measures` table holding all DAX measures separate from any data table — standard practice for a clean, navigable measure list in the Fields pane.

**Design decision — two reference tables were audited, not trusted at face value.** Both `Marketing Campaign Details[Average ROI]` and `Region Performance[Average ROI]` were verified against the underlying `Marketing Campaign Performance` fact table and confirmed to be simple, unweighted averages — i.e., they carry the same distortion described above. They are retained in the model as **decoy/reference columns** for the dashboard's before/after comparison, but no measure or visual treats them as the source of truth for a business decision; `Aggregate ROI` is calculated independently from the fact table in every case.

---

## Known Limitations

Documented rather than silently patched, consistent with how this project's data was actually audited:

- **Raw `ROI` column data integrity issue.** 234 of 1,000 rows (23.4%) in `Marketing Campaign Performance[ROI]` do not reconcile with `(Revenue − Spend) / Spend` computed from that same row's own Spend and Revenue values — discrepancies are large (not rounding noise; e.g., a stated ROI of 1.93 against a recomputed 1.27 on the same row). Source unconfirmed — possibly late edits to Spend/Revenue after ROI was originally calculated upstream, or a generation artifact in the source extract. Flagged here rather than silently corrected, since silently rewriting a provided column is its own risk without knowing which value is authoritative.
- **`Best Campaign` measure inherits that data quality risk.** This measure identifies the top campaign via `CALCULATE(MAX(Campaign Name), ROI = MAX(ROI))` — meaning it selects on the raw, occasionally-inconsistent `ROI` column rather than a recomputed weighted figure. In a small number of cases, the "best campaign" surfaced by this card may not be the campaign with the strongest actual (Revenue − Spend)/Spend performance. Worth revisiting if this measure is used for anything decision-critical.
- **Report file hygiene.** An earlier draft `.pbix` included a second report page with two visuals bound directly to a default `AVERAGE(ROI)` aggregation on the raw column — bypassing the `Aggregate ROI` / `Simple Avg ROI` measure pair entirely and reintroducing the exact distortion this project exists to catch. That page was audited out before publishing; any additional report pages should route every ROI figure through the documented measures, never a default field-well aggregation on the raw column.

---

## Technical Stack

- **Power BI Desktop** — data modeling, DAX, report design
- **DAX** — explicit measure layer: weighted ROI, distortion detection, CPA, conversion/click-through rate, weighted channel ranking
- **Power Query (M)** — CSV ingestion, `_Measures` table structuring

---

## Dashboard Preview

*See `/assets` for full-resolution screenshots.*

---

## Contact

*[Mohamed Fawzy / www.linkedin.com/in/mohamed-fawzy-99449440a / hegabwork.26@gmail.com]*
