# Content Refresh Prioritization: Ranking Pages for Review Under an Honest Validation Standard

**Misbah Abdullah** — FlyRank ML Internship Capstone, 2026

---

## Abstract

Which pages should a content team review first, out of far more candidates than they have time for? This paper builds and honestly validates a model that ranks pages by likelihood of decline, using FlyRank's pseudonymized content-refresh dataset (30,000 pages, 44 features). A client-grouped validation split — as opposed to a naive random split, which inflated results to a false-looking 1.00 Precision@50 due to client-identity leakage — produces an honest Precision@50 of 0.62, a 2.8x improvement over a hand-written baseline rule (0.220). The paper documents two distinct leakage failure modes found and fixed during development, translates the validated model into a practical, archetype-based action playbook, and is explicit about what this work cannot claim: causal impact, generalization to unseen clients, or validity beyond this single dataset snapshot.

---

## Introduction / Problem Statement

FlyRank's content teams face a real constraint: far more pages exist than any reviewer has time to check individually. A hand-written rule can encode a few thresholds a person thought of in advance (e.g. "stale AND visible"), but real decline is driven by an interaction of staleness, position, visibility, and momentum that a fixed rule can't weigh adaptively.

This paper's research question: **can a learned model rank pages for review more effectively than a hand-written baseline rule, under a validation standard honest enough to trust?** The "honest enough to trust" qualifier is not decorative — this project's own development process surfaced two separate ways an evaluation can silently lie (feature leakage and split leakage, both documented in Results), which is exactly the risk this paper is built to guard against and disclose plainly.

This sits in FlyRank's Lane 2 (Refresh / Content Opportunity Scoring): the decision is which pages a reviewer looks at first; the action is refresh, monitor, investigate, or deprioritize; the cost of a wrong call is either wasted reviewer time (false positive) or a real decline going unnoticed (false negative) — both real costs given finite review capacity.

---

## 1. Question

**Research question:** does a learned ranking model outperform a hand-written baseline rule at prioritizing pages for content review, under a validation split that prevents both feature-level and client-identity leakage?

**Decision this supports:** which pages a content reviewer with limited weekly capacity looks at first, and what action (refresh, investigate, deprioritize, monitor) they take once they do.

---

## 2. Data

**Release used:** the FlyRank ML internship starter dataset (`data/raw/content_refresh_anonymized.csv`) — 30,000 pseudonymized pages, 44 columns, one point-in-time snapshot. The full 79M-row warehouse release was explored earlier in the track for schema and feature-engineering practice, but the capstone model itself is built and validated on the starter release for reproducibility within the scope of this track.

**What was excluded, and why:**
- `trend_direction` and `trend_pct` — these directly define the label (`is_declining`), so using them as features would be circular
- `impressions_last_30d` / `prev_30d`, `clicks_last_30d` / `prev_30d`, `sessions_last_30d` / `prev_30d` — these are the raw values `trend_direction` is computed from; a leakage experiment confirmed including just one of these (`impressions_last_30d`) let a model achieve a suspicious 1.00 Precision@50, exposing the leak (see Results)
- `content_id`, `client_id` — used for grouping and validation splitting, never fed to the model as predictive features

**Public safety:** all identifiers in this dataset are pseudonymized (hashed content/client IDs); no real client names, URLs, or private query text appear anywhere in this paper or its underlying notebooks.

---

## 3. Methodology

**Label definition:** `is_declining_label = 1` if `trend_direction == "down"`, else `0`. This is a proxy label (a same-window bucket), not a causal or future-verified outcome — named explicitly as a limitation below.

**Features (23 final, after leakage fixes):** numeric features (e.g. `avg_position`, `word_count`, `engagement_rate`, `ai_traffic_pct`) plus one-hot encoded categoricals (`content_type`, `main_intent`, `competition_level`, `age_tier`, `freshness_tier`, `word_count_tier`, `position_tier`, and others) — all independent of the label-construction window, confirmed via a column-by-column leakage audit.

**Baseline (hand-written rule):** a weighted score combining staleness (0.4), log-scaled visibility (0.3), position penalty (0.2), and a declining flag (0.1), producing action labels (`refresh_now` / `monitor` / `deprioritize`). Precision@50 = 0.220 on the full 30,000-row set.

**Model:** Random Forest and Logistic Regression, compared side by side.

**Validation design — the central methodological finding of this paper:** a naive random 80/20 split scored a perfect 1.000 Precision@50 — not real signal, but an artifact of only ~32 distinct clients in the dataset, so a random split let 31 of them appear in both train and test, allowing the model to partially memorize client identity rather than learn genuine decline patterns. A client-grouped split (`GroupShuffleSplit` on `client_hash_id`, zero train/test client overlap) produces the honest number: Precision@50 = 0.62.

**Leakage checks performed:**
1. **Feature-level** — confirmed no label-source columns present in the final feature set
2. **Split-level** — confirmed zero client overlap under the grouped split, versus 31 clients overlapping under a naive random split

---

## 4. Results (vs. Baseline)

| Method | Precision@50 | Notes |
|---|---|---|
| Hand-written baseline rule | 0.220 | Fixed thresholds, no learning |
| Model, naive random split | 1.000 | **Not a real result** — client-identity leakage (31/32 clients overlap train/test) |
| Model, honest client-grouped split | **0.62** | Zero client overlap; the number this paper stands behind |

Under the honest split, both Logistic Regression and Random Forest independently reached 0.62 Precision@50, but agreed on only 13 of their 50 flagged pages — indicating two structurally different routes to a similar hit rate, not one dominant, simple signal. Permutation importance under the honest model spreads across four features (`days_with_impressions`, `days_with_sessions`, `content_age_days`, `avg_position`) rather than one or two dominating — a meaningfully different picture from the pre-fix leaky version, where two features alone accounted for the overwhelming majority of the model's decisions.

**Error analysis:** of the honest model's top 50, 31 were correctly identified as declining and 19 were false positives (38% error rate at the top of the ranking) — a real, disclosed number, not glossed over. The false positives cluster around genuinely stale, moderate-traffic pages that have actually plateaued rather than declined — the model over-weights staleness in the same way the baseline rule did, a real, nameable failure mode rather than random noise.

*[Chart: Baseline vs. leaked vs. honest Precision@50 — see `work/figures/`]*

*[Chart: Archetype distribution — see `work/figures/archetype_distribution.png`]*

---

## 5. Limitations

- **Proxy label, not verified outcome:** `is_declining` is a same-window bucket (`trend_direction`), not a causally verified future decline
- **Single snapshot, not tested over time:** this is one point-in-time dataset; the model's stability across different months or seasons is unverified
- **Unbalanced client history:** clients have differing amounts of history; some clients are `access_profile = gsc_only`, structurally missing engagement signals
- **Causal blindness:** this data can show that a page's traffic changed; it cannot explain why (algorithm update, competitor, seasonality). Nothing here supports a causal claim about what refreshing a page will do
- **Generalization is unproven beyond this dataset:** validated on ~32 clients in one pseudonymized release; not proven to generalize to FlyRank's live production client base or the full 79M-row warehouse
- **38% false-positive rate in the top 50:** this is decision-support for a human reviewer, not a system reliable enough to act on unsupervised

---

## 6. Ranked Recommendations

This paper's recommendations are the direct output of the Content Action Playbook, re-ranked to surface actionable archetypes before non-actionable ones (a fix applied after an initial run showed non-declining "stable_visible" pages outscoring genuinely actionable pages by raw probability alone):

| Archetype | Recommended Action | Rationale |
|---|---|---|
| `stale_visible_declining` | Refresh content | Update body content, verify facts/dates, re-optimize for current queries |
| `strong_position_declining` | Investigate SERP change | Ranking well but losing clicks — check for SERP features (snippets, PAA) before assuming content is the problem |
| `weak_position_declining` | Deprioritize or restructure | Deep-position decline rarely responds to a content refresh alone |
| `stable_visible` | Monitor only | No action needed this cycle |
| `low_signal` | Deprioritize | Near-zero real traffic, not worth reviewer time |

**Human review is required before acting on any recommendation** — confirm seasonal effects are ruled out, confirm visibility reflects real query diversity (not one branded query), and never auto-publish, auto-prune, or treat the ranked order as strict priority without archetype context.

---

## 7. Artifacts

- Archetype distribution chart (`work/figures/archetype_distribution.png`)
- Baseline vs. honest-model vs. leaked-model Precision@50 comparison (table above)
- Permutation importance chart (top features under the honest model)

---

## Reproducibility

Full notebooks, in order:
`work/notebooks/w03_data_contract.ipynb` → `w04_baseline_score.ipynb` → `w05_model.ipynb` → `w06_validation_audit.ipynb` → `w07_action_playbook.ipynb` → `capstone.ipynb` (this paper).

Repository: [github.com/MisbahSangi/flyrank-ml-internship-misbah](https://github.com/MisbahSangi/flyrank-ml-internship-misbah)

Anyone with the repo can rerun the full pipeline end to end using the same pseudonymized starter dataset.

---

## Acknowledgments & Data Credit

Built on the FlyRank ML Internship dataset. Data and program credit: [https://flyrank.ai](https://flyrank.ai)
