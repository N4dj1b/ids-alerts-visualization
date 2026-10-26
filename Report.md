# Lab 02 — Visualization of Intrusion Detection Alerts
**Course:** IA for Cybersecurity — 3rd Year, Option CyS  
**Student:** _your name_  **Date:** _date_  
**Tool:** Python (Pandas, NumPy, Matplotlib, Seaborn) in Google Colab

## 1. Use case and objective
IDS sensors (Snort, Suricata) produce thousands of alerts per day, mostly low-priority noise. The objective is to use visualization to find the alerts that matter, define one automatic detection rule, and evaluate it on anomalous and normal data.

## 2. Dataset
No real dataset is used. One day of IDS alerts (19 Oct 2026) is **simulated** with the fields a Snort/Suricata log provides: `timestamp, src_ip, dst_ip, dst_port, signature, category, priority`, plus a ground-truth label `is_attack` used **only for evaluation**.

| Component | Alerts | Description |
|---|---|---|
| Normal background | 1,800 | ~150 external IPs, 8 signatures, busier 08:00–18:00; priority-1 alerts are rare and isolated |
| Benign noisy source | 450 | Authorised internal Nessus scanner `10.0.0.99`, 02:00–03:00, low priority |
| **Injected anomaly** | 650 | `198.51.100.23` sends Log4j RCE exploit attempts (priority 1) to `10.0.0.50/51`, ports 443/8080, 14:20–14:31 |
| **Total** | **2,900** | |

The scanner is deliberately included as a trap: it generates many alerts but is harmless, so a rule based on volume alone fails.

## 3. Methodology
1. Generate the data (fixed random seed, reproducible).
2. Explore: types, priority and category distributions, top sources.
3. Visualize (three figures below).
4. Define a detection rule from what the plots show.
5. Evaluate on the attack day, on five normal-only days, across thresholds, and against a naive baseline.

## 4. Visualizations
| Figure | File | What it shows |
|---|---|---|
| 1. Stacked timeline by priority | `figures/01_alerts_timeline.png` | A large priority-1 spike at 14:20–14:31; the low-priority scanner plateau at 02:00–03:00 |
| 2. Heatmap signature × hour | `figures/02_signature_hour_heatmap.png` | Hour 14 is dominated by the Log4j signature, hour 2 by the Nessus signature |
| 3. Top sources: volume vs. priority-1 | `figures/03_top_sources_volume_vs_severity.png` | By volume the scanner looks as threatening as the attacker; by severity only the attacker stands out |

## 5. Detection rule
> **Raise an alert when one source IP generates at least T = 15 priority-1 alerts within a 5-minute window.**

On normal data the maximum is 2 priority-1 alerts per (IP, window); the attacker produces 295, 308 and 47 in three consecutive windows. T = 15 sits well between the two.

## 6. Evaluation
**Attack day (alert level)**

| TP | FP | FN | TN | Precision | Recall | F1 | FPR |
|---|---|---|---|---|---|---|---|
| 650 | 0 | 0 | 2,250 | 1.00 | 1.00 | 1.00 | 0.00 |

**Normal-only days (5 days, different seeds):** 0 alerts raised, so 0 false alarms.

**Threshold sensitivity**

| T | 1 | 2 | 3–30 | 50–100 | 300 |
|---|---|---|---|---|---|
| Precision | 0.70 | 0.997 | 1.00 | 1.00 | 1.00 |
| Recall | 1.00 | 1.00 | 1.00 | 0.93 | 0.47 |

**Baseline (volume-only rule, ≥ 15 alerts of any priority per 5 min):** recall 1.00 but 422 false positives (the scanner), precision 0.61, F1 0.76. Our rule gives F1 = 1.00.

## 7. Main findings
1. The anomaly is located at 14:20–14:31 from `198.51.100.23` against the web servers, signature *Log4j RCE*.
2. Raw volume is misleading; combining **severity and rate** separates an attack from a noisy but benign scanner.
3. The rule detects every attack alert and raises no false alarms on five normal days.
4. It is robust for T between 3 and 30. Too low a T causes false positives, too high a T misses the shorter last attack window.

## 8. Limitations and future work
- Data is simulated, so the clean separation is optimistic; real data has heavier tails.
- A fixed 5-minute window misses low-and-slow attacks, and per-IP counting misses distributed attacks.
- Next steps: per-host adaptive baselines (Z-score, Isolation Forest, Session B) and validation on a real dataset (CIC-IDS2017, UNSW-NB15).
