Use this as a **Google Doc-ready version** (copy-paste as-is):

---

# Improving Kitchen Prep Time (KPT) Prediction to Optimize Rider Assignment and Customer ETA at Zomato

## 1. Context

Zomato’s delivery ETA quality depends heavily on accurate Kitchen Prep Time (KPT), i.e., the time from order confirmation to food readiness. At Zomato’s scale (300K+ merchants), even small KPT improvements can significantly impact customer satisfaction, rider productivity, and operating costs.

Today, KPT labels rely mostly on merchant-marked Food Order Ready (FOR) events in the Merchant Experience app. These events are often noisy and biased, leading to inaccurate KPT predictions and downstream inefficiencies.

---

## 2. Problem Understanding

### Current signal limitations
- **Rider-influenced FOR marking:** Merchants may mark “ready” when rider arrives, not when food is truly ready.
- **No full-kitchen visibility:** System sees Zomato orders, but kitchen load also comes from dine-in, takeaways, and competitor platforms.
- **Human behavior bias:** Marking habits vary by staff, shift, merchant type, and busy periods.

### Downstream business impact
- Early rider arrivals and increased rider wait time at pickup
- ETA fluctuations and reduced customer trust
- Higher rider idle time and lower fleet efficiency
- Increased delay/cancellation risk during rush windows

---

## 3. Objective

Improve KPT prediction by strengthening input signals and system design beyond merchant-marked FOR, while ensuring scalability across small, medium, and large merchants.

---

## 4. Proposed Solution: Multi-Signal Readiness Intelligence Layer

Instead of treating FOR as ground truth, build a **confidence-weighted readiness system** using multiple event streams.

### 4.1 Derived “True Ready Time” (Label Denoising)
Use multiple timestamps per order:
- Order confirmed
- Item prep start (if POS/KDS available)
- Packing start/end
- Merchant FOR click
- Rider geofence arrival
- Handoff/pickup confirmation (OTP scan, pickup event)

Create a `Signal Confidence Score`:
- Lower FOR confidence when FOR repeatedly coincides with rider arrival
- Higher confidence when FOR-to-pickup lag is realistic and stable
- Adjust labels dynamically by merchant behavior profile

**Outcome:** Cleaner KPT labels for both training and online correction.

---

### 4.2 Kitchen Rush Index (Beyond Zomato Order Volume)
Introduce a real-time `KitchenRushIndex (0–1)` using proxies:
- POS ticket throughput in last 10–20 mins
- KDS active queue length
- Dine-in bill creation / occupancy proxy
- Staffing level or shift schedule
- Manual “rush mode” switch for non-integrated merchants
- Time/event/weather-based demand priors

**Outcome:** Better KPT under true kitchen congestion, not just Zomato demand.

---

### 4.3 Merchant Reliability & Behavior Calibration
Build a daily-updated merchant reliability profile:
- Late marker profile (FOR usually delayed)
- Early marker profile (FOR often premature)
- Rider-triggered profile (FOR tied to rider arrival)
- High variance profile (inconsistent operational behavior)

Apply merchant-specific correction factors to KPT inputs and dispatch timing.

**Outcome:** Lower systematic bias without requiring immediate merchant behavior change.

---

### 4.4 Two-Stage Rider Assignment Policy
Use prediction uncertainty and readiness confidence for dispatch:

1. **Soft assignment (pre-position):** Keep rider near pickup zone during final prep phase.  
2. **Hard dispatch (final commit):** Trigger when high-confidence readiness threshold is crossed.

**Outcome:** Reduced rider waiting while keeping ETA stable.

---

### 4.5 Merchant Workflow Improvements (Low Friction)
- One-tap statuses: `Cooking → Packing → Ready`
- “Suggested ready now” nudges based on live prep progression
- Accuracy-based merchant incentives (not speed-only)
- Lightweight flow for long-tail merchants, richer flow for integrated chains

**Outcome:** Better signal quality at source with minimal operational burden.

---

## 5. Scalable Rollout Architecture (300K+ Merchants)

### Tiered deployment
- **Tier 0 (long tail):** App-only signals + geofence + reliability correction
- **Tier 1 (mid-size):** Basic POS/KDS + queue proxies
- **Tier 2 (enterprise chains):** Deep POS/KDS integrations + optional IoT kitchen instrumentation

All tiers feed a common readiness API with varying signal richness.

---

## 6. Quantitative Simulation Plan (Bonus)

Run offline replay/simulation on historical orders:

### Scenarios
- **Baseline:** FOR-only KPT + current dispatch
- **Variant A:** Confidence-weighted FOR correction
- **Variant B:** A + KitchenRushIndex
- **Variant C:** B + two-stage dispatch

### Metrics
- Rider wait time at pickup (avg / P90)
- ETA error (P50 / P90)
- Delay and cancellation rates
- Rider idle time and utilization

**Expected directional impact:**  
- Rider wait: ↓ 15–30%  
- ETA P90 error: ↓ 8–20%  
- Delay/cancellation: ↓ 5–12%  
(Exact values depend on city and merchant mix.)

---

## 7. Success Metric Mapping

- **Average rider wait time:** Improved via better readiness timing + staged dispatch
- **ETA prediction error (P50/P90):** Improved via denoised labels + rush context
- **Order delay/cancellation rates:** Reduced by fewer mistimed pickups
- **Rider idle time:** Reduced by avoiding premature dispatch

---

## 8. Risks and Mitigations

- **Sparse integrations in long tail:** Use app-only proxy signals + fallback heuristics
- **Merchant adoption friction:** Keep UI changes minimal; incentive-led adoption
- **Data quality drift:** Continuous trust scoring, anomaly monitoring, auto-fallbacks
- **Operational complexity:** Progressive city-wise rollout with A/B governance

---

## 9. Why This Approach Is Strong

This proposal is not model-only. It improves:
1. **Signal truthfulness** (denoised readiness labels)  
2. **Operational context** (kitchen rush beyond Zomato orders)  
3. **Decision policy** (uncertainty-aware rider assignment)  
4. **Scalability** (tiered architecture across 300K+ merchants)

It directly connects to all required success metrics and evaluation criteria.

---

If you want, I can also give you:
- a **1-page executive summary** version, or
- a **presentation-ready (8-slide) version** of the same content.
