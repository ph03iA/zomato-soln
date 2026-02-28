# Improving Kitchen Prep Time (KPT) Prediction to Optimize Rider Assignment and Customer ETA at Zomato

## Understanding the Problem

Zomato needs accurate **Kitchen Prep Time (KPT)** predictions to:

- Dispatch riders at the right time (not too early, not too late)
- Show customers reliable ETAs
- Minimize rider idle/wait time at restaurants

### The Core Issue

The current "Food Order Ready" (FOR) signal is **unreliable** because:

1. **Rider-influenced marking:** Merchants mark food ready when the rider arrives, not when food is actually done
2. **No visibility into kitchen-wide load:** The system is blind to non-Zomato kitchen load (dine-in, Swiggy, walk-ins)
3. **Inconsistent behavior:** Manual marking behavior varies significantly across merchants and time periods

These limitations introduce noise and bias into KPT labels, leading to inaccurate predictions, early rider arrivals, increased waiting time, and fluctuating ETAs for customers.

---

## Proposed Solutions

Solutions are structured across **4 pillars**: Signal De-noising, New Signal Sources, Live Rush Detection, and Scalability.

---

### Pillar 1: De-noising the Existing FOR Signal

> **Problem:** FOR timestamps are contaminated by rider arrival times.

#### Solution A — Rider-Arrival Cross-Referencing Filter

- Compare `FOR_timestamp` with `rider_arrival_timestamp`. If `|FOR - rider_arrival| < threshold` (e.g., 30 seconds), flag this as a **rider-influenced FOR** and down-weight or discard it from training data.
- Build a per-merchant **reliability score** based on the historical percentage of rider-influenced FORs.
- Use this score to weight that merchant's labels in model training.

#### Solution B — Probabilistic Label Correction

- For flagged (noisy) FORs, estimate the **true prep time** using:
  - Historical median KPT for that restaurant + item category + time-of-day
  - Bayesian correction: blend the noisy FOR with the prior distribution
- This gives a **corrected label** that is more useful than raw FOR.

#### Solution C — Gamified Merchant Incentive for Accurate Marking

- Introduce a **"Prep Accuracy Score"** visible to merchants on the Mx app.
- Reward accurate FOR marking with better visibility/ranking on the platform.
- Penalize consistently inaccurate marking (e.g., reduced order routing priority).
- This aligns merchant behavior with data quality goals.

---

### Pillar 2: New Input Signals Beyond FOR

#### Solution D — Order Complexity Score

Compute a real-time **complexity score** for each order:

- Number of items
- Item-level prep difficulty (tagged via historical data: biryani > dal)
- Customization count (extra cheese, no onion, etc.)
- Shared ingredient overlap across concurrent orders (batching potential)

Feed this as a feature into KPT prediction.

#### Solution E — Merchant App Micro-Signals (Passive Instrumentation)

Track **implicit signals** from the Mx app without requiring manual input:

- **Order acceptance delay** — longer delay = kitchen is busy
- **Screen interaction patterns** — frequent app checks = less busy; no interaction = swamped
- **Concurrent active orders** on the Zomato platform

These are zero-effort signals from the merchant's perspective.

#### Solution F — Rider Feedback Loop (Post-Pickup)

- After pickup, prompt the rider (1-tap): *"Was food ready when you arrived?"*
  - Options: `Ready` / `Waited < 2 min` / `Waited 2-5 min` / `Waited > 5 min`
- This creates a **ground truth signal** independent of the merchant's FOR marking.
- Aggregate over time to calibrate per-merchant KPT bias.

---

### Pillar 3: Capturing Live Restaurant Rush (Beyond Zomato Orders)

> This is the hardest sub-problem — Zomato has no visibility into dine-in or competitor orders.

#### Solution G — IoT-Based Kitchen Load Sensing (Premium Tier)

Deploy a **lightweight IoT device** near the kitchen/counter:

- **Sound-level sensor (decibel meter):** Kitchen noise correlates strongly with activity. A CNN model on audio amplitude patterns (not raw audio, for privacy) can classify rush levels as `Low / Medium / High`.
- **Thermal/IR people counter** at the restaurant entrance: counts foot traffic in real-time.

Data is streamed to Zomato's backend and used as a real-time rush feature.

**Scalability:** Start with top 5K high-volume restaurants; cost per unit < $10 for basic sensors.

#### Solution H — POS Integration (Medium-Term, High Impact)

- Partner with major POS providers (Petpooja, dotPe, Posist) used by restaurants.
- Ingest **total active orders** across all channels (dine-in, Swiggy, counter).
- This gives direct visibility into kitchen load without any hardware.
- **Privacy-safe:** only aggregate order count + item count, no competitor-specific data.

#### Solution I — Proxy Rush Estimation (No Hardware, Fully Scalable)

Use freely available proxies for restaurant busyness:

- **Google Maps Popular Times API** — real-time busyness data for the location
- **Time-of-day + day-of-week rush patterns** — learned from historical data per restaurant
- **Weather and event data** — rain increases delivery orders; nearby events increase dine-in
- **Zomato dine-in page signals** — table booking rate, waitlist size (if available)
- **Area-level demand surge** — if nearby restaurants are busy on Zomato, this one likely is too

Build a **composite rush index** from these signals.

---

### Pillar 4: Scalability Design

| Solution | Scalability | Cost | Signal Quality |
|----------|-------------|------|----------------|
| Rider-influenced FOR filter (A) | All 300K merchants | Zero | High |
| Label correction (B) | All 300K merchants | Zero | Medium-High |
| Merchant incentive (C) | All 300K merchants | Low | Medium |
| Order complexity score (D) | All 300K merchants | Zero | High |
| Mx app micro-signals (E) | All 300K merchants | Zero | Medium |
| Rider feedback (F) | All 300K merchants | Low | High |
| IoT sensors (G) | Top 5–10K | Medium | Very High |
| POS integration (H) | ~50K (POS-using) | Low | Very High |
| Proxy rush estimation (I) | All 300K merchants | Zero | Medium |

#### Tiered Rollout Strategy

- **Tier 1 (All merchants, Day 0):** Solutions A, B, D, E, I — pure software, no merchant effort
- **Tier 2 (All merchants, Month 1):** Solutions C, F — require UX changes in Mx/rider app
- **Tier 3 (Top merchants, Quarter 1):** Solutions G, H — hardware/partnerships

---

## Bonus: Simulation Framework

A simplified simulation to demonstrate the impact of signal de-noising:

```python
import numpy as np

class KPTSimulator:
    def __init__(self, num_restaurants=1000, num_days=30):
        self.num_restaurants = num_restaurants
        self.num_days = num_days

    def generate_true_kpt(self, base_kpt, rush_level, order_complexity):
        """True KPT = base + rush_factor + complexity_factor + noise"""
        rush_factor = rush_level * np.random.uniform(3, 8)
        complexity_factor = order_complexity * np.random.uniform(2, 5)
        noise = np.random.normal(0, 2)
        return max(5, base_kpt + rush_factor + complexity_factor + noise)

    def generate_noisy_for(self, true_kpt, rider_arrival_time):
        """Simulate merchant FOR marking influenced by rider arrival"""
        rider_influenced = np.random.random() < 0.35  # 35% of FORs are rider-influenced
        if rider_influenced:
            return rider_arrival_time
        return true_kpt + np.random.normal(0, 1)

    def apply_denoising(self, for_signal, rider_arrival, historical_median):
        """Apply rider-cross-reference filter + Bayesian correction"""
        if abs(for_signal - rider_arrival) < 0.5:  # rider-influenced
            return 0.7 * historical_median + 0.3 * for_signal
        return for_signal

    def run(self):
        errors_before, errors_after = [], []
        for _ in range(self.num_restaurants * self.num_days * 10):
            base_kpt = np.random.uniform(10, 35)
            rush = np.random.choice([0, 1, 2], p=[0.5, 0.35, 0.15])
            complexity = np.random.uniform(0, 3)
            true_kpt = self.generate_true_kpt(base_kpt, rush, complexity)
            rider_arrival = true_kpt + np.random.uniform(-5, 10)
            noisy_for = self.generate_noisy_for(true_kpt, rider_arrival)
            corrected = self.apply_denoising(noisy_for, rider_arrival, base_kpt + 5)
            errors_before.append(abs(noisy_for - true_kpt))
            errors_after.append(abs(corrected - true_kpt))

        p50_before, p90_before = np.percentile(errors_before, [50, 90])
        p50_after, p90_after = np.percentile(errors_after, [50, 90])
        return {
            "P50 error (before)": round(p50_before, 2),
            "P50 error (after)": round(p50_after, 2),
            "P90 error (before)": round(p90_before, 2),
            "P90 error (after)": round(p90_after, 2),
            "P50 improvement %": round((p50_before - p50_after) / p50_before * 100, 1),
            "P90 improvement %": round((p90_before - p90_after) / p90_before * 100, 1),
        }

sim = KPTSimulator()
print(sim.run())
```

---

## Mapping to Success Metrics

| Metric | Impact From |
|--------|-------------|
| **Avg rider wait time** | De-noised FOR (A, B) + rush detection (G, H, I) → dispatch rider later when kitchen is actually busy |
| **ETA prediction error (P50/P90)** | All signals improve prediction → tighter distributions |
| **Order delay & cancellation** | Better KPT → less over-promising → fewer delays → fewer cancellations |
| **Rider idle time** | Accurate dispatch timing = rider arrives when food is ready, not 10 min early |

---

## Key Novelty Points

1. **Passive instrumentation** via Mx app interaction patterns (zero merchant effort)
2. **Sound-based IoT rush detection** — cheap, privacy-preserving, and directly measures kitchen activity
3. **Rider-as-sensor feedback loop** — creates an independent ground truth channel
4. **Bayesian label correction** — statistically fixes noisy training data without discarding it
5. **Google Popular Times as a rush proxy** — free, available for most merchants, no integration needed
