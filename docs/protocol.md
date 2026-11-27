# Evaluation Protocol and Benchmark Details

This document summarizes the datasets, sequence-disjoint splits, optical blackout simulation protocol, and evaluation metrics used in **BridgeTrack**.

> **Note:** Full evaluation scripts, standardized interruption-event index files, and trained checkpoints will be released upon paper acceptance.

---

## 1. Benchmarks and Splits

### 1.1 Underwater RGB--Sonar Tracking (`RGBS50`)

- **Dataset:** RGBS50 supplies synchronized perspective optical RGB frames and forward-looking imaging-sonar frames with independent bounding-box annotations in each sensor's native coordinate system.
- **Coordinate Convention:** Target localization is evaluated strictly against RGB ground truth, while sonar is consumed as an auxiliary modality in its native polar-sector coordinate frame without assuming spatial co-registration.
- **Zero-Leakage Partitioning:** All training, validation, and test splits are strictly disjoint by source video sequence. Recovery evaluation clips inherit the split assignment of their parent sequence.
- **Fixed Fine-Grained Evaluation Cohorts (9 RGBS50 test sequences):**
  - **51-Frame Cohort (53 events):** 27 events (Gap 40), 22 events (Gap 80), and 4 events (Gap 160), evaluated across post-return offsets $k \in \{0, \ldots, 50\}$.
  - **200-Frame Cohort (30 events):** Fixed common subset with sufficient post-return duration comprising 15 events (Gap 40), 13 events (Gap 80), and 2 events (Gap 160), evaluated across offsets $k \in \{0, \ldots, 199\}$ (4,093 valid event--frame instances).

### 1.2 RGB--Thermal Cross-Modal Generalization (`RGBT234` & `NOT-156`)

| Dataset | Split / Protocol | Sequences | Recovery Events |
| :--- | :--- | :---: | :---: |
| **RGBT234** | Training | 164 | — |
| | Validation | 35 | — |
| | Continuous test | 35 | — |
| | Interruption recovery (Gap 40) | 28 | 28 |
| | Interruption recovery (Gap 80) | 27 | 27 |
| | Interruption recovery (Gap 160) | 18 | 18 |
| | **All recovery gaps** | **28** | **73** |
| **NOT-156** | Official night test (zero-shot transfer) | 50 | — |

---

## 2. Strictly Causal Optical Blackout Protocol

Each interruption-recovery event consists of three consecutive phases:

1. **Visible Initialization Prefix ($a_t^r = 1$):** The tracker is initialized from $(b_0^r, b_0^s)$ at $t=0$ and runs causally on RGB and auxiliary frames prior to blackout.
2. **Optical Blackout Interval ($a_t^r = 0$, length $L \in \{40, 80, 160\}$ frames):** Hidden RGB frames are completely withheld (neither read nor encoded). Only the concurrent sonar (or thermal) stream and previously accumulated online states/memories are accessible.
3. **Post-Return Window ($a_t^r = 1$):** Optical observations resume, and tracking continues strictly causally without accessing any future frames or post-$t=0$ ground truth.

All controlled ablation variants share the identical online search policy: a local $5\sqrt{\mathrm{area}(b_{\mathrm{last}}^r)}$ crop resized to $384\times384$ during normal tracking, automatically switching to an aspect-preserving square-padded full-image crop ($384\times384$) during the first 10 returning RGB frames (or when $q_{\mathrm{prev}}^r < 0.5$).

---

## 3. Evaluation Metrics

### Continuous Tracking Metrics
- **AUC:** Area Under the Success Curve over IoU thresholds $[0, 1]$.
- **PR:** Precision at a 20-pixel center location error threshold.
- **NPR:** Scale-normalized Precision following TrackingNet conventions.

### Interruption-Recovery Metrics (at $\mathrm{IoU} \ge 0.5$)
- **First-Return Recovery (`R1`):** Percentage of recovery events achieving $\mathrm{IoU} \ge 0.5$ on the exact first returning RGB frame ($k = 0$).
- **Windowed Frame Success Rate (`SR_K`):** Pooled proportion of validly annotated post-return frames over offsets $k \in \{0, \ldots, K-1\}$ meeting $\mathrm{IoU} \ge 0.5$:

$$
\mathrm{SR}_{K} = 100 \cdot \frac{\sum_{e}\sum_{k=0}^{K-1} v_{e,k}\,\mathbb{I}\left[\mathrm{IoU}(\widehat{b}_{e,k}, b_{e,k}) \ge 0.5\right]}{\sum_{e}\sum_{k=0}^{K-1} v_{e,k}}
$$

where $v_{e,k} \in \{0, 1\}$ denotes annotation validity, and missing predictions on valid frames are counted as failures. Reported windows include short-horizon (`SR5`, `SR10`), medium-horizon (`SR51`), and extended persistence (`SR200`).

- **Instantaneous Success Rate (`P(k)`):** Offset-resolved success rate over valid events at post-return offset $k$:

$$
P(k) = 100 \cdot \frac{\sum_{e} v_{e,k}\,\mathbb{I}\left[\mathrm{IoU}(\widehat{b}_{e,k}, b_{e,k}) \ge 0.5\right]}{\sum_{e} v_{e,k}}
$$

- **Cumulative First Stable Recovery (`C(k)`) & `Stable@k`:** Let $\tau_e$ denote the offset of the third frame in the earliest run of **three consecutive validly annotated, RGB-available frames** achieving $\mathrm{IoU} \ge 0.5$ ($\tau_e = +\infty$ if never confirmed; any invalid annotation breaks a run):

$$
C(k) = \frac{100}{|\mathcal{E}|} \sum_{e \in \mathcal{E}} \mathbb{I}[\tau_e \le k]
$$

Under zero-based indexing, `Stable@10` equals $C(9)$, and `Stable@5` equals $C(4)$.
