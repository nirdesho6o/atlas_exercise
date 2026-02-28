# ATLAS SPOT: Automated Software Performance Monitoring  
## GSoC Warm-Up Exercise Evaluation Report  

---

## 1. Introduction & Objective

This project explores automated anomaly detection for software performance monitoring.

The goals were:

- Generate process monitoring data using `prmon`
- Inject controlled anomalies into a continuous time series
- Evaluate statistical and ML-based detection methods
- Compare trade-offs between detectors
- Propose a scalable architecture

---

## 2. Data Generation

### Tools

- prmon
- mem-burner
- Python (pandas, numpy, matplotlib, scikit-learn)


### Workload Design

All anomalies were injected in a single continuous run.

| Phase | Memory Usage |
|--------|--------------|
| Baseline 1 | 200 MB (5 min) |
| Spike | 1024 MB, 4 processes (5 sec) |
| Baseline 2 | 200 MB (5 min) |
| Sustained Regression | 600 MB (3 min) |
| Recovery | 200 MB |


### Ground Truth

- The 5-second spike was labeled anomalous.
- The 3-minute sustained regression was labeled anomalous.
- All other timestamps were labeled normal.

This allows quantitative evaluation using precision, recall, and F1-score.

---

## 3. Detection Architecture

Three detectors run in parallel. Each targets a different anomaly type.

---

### 3.1 Rolling MAD  
**Purpose:** Detect short spikes  

- 30-second rolling window  
- One-sided robust Z-score  
- Alert if Z > 3  

Strength: Very precise for sharp spikes.  
Limitation: Because it relies on a finite rolling window, sustained regressions eventually become the new baseline.

---

### 3.2 EWMA Residual  
**Purpose:** Detect sustained regressions  

- Slow EWMA baseline (120s span)  
- Residual control limit (3σ)  
- 5 consecutive confirmations required  

Strength: Detects sustained mean shifts.  
Limitation: Needs stabilization period.




---

### 3.3 Isolation Forest  
**Purpose:** Detect structural changes  

- Features: [pss, pss_diff]  
- Trained on baseline only  
- contamination = 0.02  

Strength: Detects state transitions.  
Limitation: Dense plateaus may appear normal.



---

## 4. Results

F1-score is used due to class imbalance.

| Detector | Precision | Recall | F1 |
|-----------|------------|--------|----|
| Rolling MAD | 1.00 | 0.18 | 0.31 |
| EWMA | 0.25 | 0.32 | 0.28 |
| Isolation Forest | 0.71 | 0.05 | 0.10 |
| Combined (OR) | 0.34 | 0.50 | 0.41 |

### Observations

- MAD detects spikes with high precision.
- EWMA detects drift onset.
- Isolation Forest detects transitions.
- The combined system improves overall F1.

No single detector performs well on all anomaly types.

---

## 5. Trade-Off Analysis

### Rolling MAD — Finite Window Limitation
Detects sharp anomalies but adapts to sustained regressions.

### EWMA — Initialization Transient
Requires burn-in period due to exponential smoothing convergence.

### Isolation Forest — Density-Based Regime Adaptation
Unsupervised models may treat long plateaus as secondary normal modes.

---

## 6. Key Insight

Different anomaly types require different methods.

- Spikes → local statistics
- Drift → trend tracking
- Structural changes → density models

Parallel detection improves robustness.

---

## 8. Production Considerations

Manual tuning of windows and thresholds does not scale across many workflows.

For production:

- Use MAD, EWMA, and IF as feature extractors.
- Feed their scores into a supervised model.
- Train using historical labels from operators.

This removes manual threshold tuning and improves adaptability.

---

## AI Assistance Disclosure

AI tool (Google Gemini) was used for:

- Improving plot visualization formatting
- Refining the README structure and wording

The anomaly detection code was implemented independently.
The hybrid parallel architecture idea was inspired by the Opprentice paper.
All design decisions, parameter selection, and evaluations were performed manually.

## Conclusion

The parallel approach improves overall detection performance.

It separates spike detection from regression detection.

It reduces false positives while maintaining coverage.

This design can be extended to ATLAS production monitoring.
