# Parametric Income Protection Insurance for Gig Workers

> A system that detects real-world disruptions and automatically compensates delivery partners for lost income — no paperwork, no claims, no waiting.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Meet Arjun — Why This Matters](#2-meet-arjun--why-this-matters)
3. [Why Food Delivery](#3-why-food-delivery)
4. [Platform Choice — Mobile App](#4-platform-choice--mobile-app)
5. [What Counts as a Disruption](#5-what-counts-as-a-disruption)
6. [Zone-Based Architecture](#6-zone-based-architecture)
7. [Input Signals and Data Sources](#7-input-signals-and-data-sources)
8. [Data Ingestion Strategy](#8-data-ingestion-strategy)
9. [Signal Normalization](#9-signal-normalization)
10. [Zone Severity Computation](#10-zone-severity-computation)
11. [Trigger Logic](#11-trigger-logic)
12. [User Filtering and Scalability](#12-user-filtering-and-scalability)
13. [Presence Estimation](#13-presence-estimation)
14. [Payout Computation](#14-payout-computation)
15. [Insurance Policy Design](#15-insurance-policy-design)
16. [ML-Based Premium Pricing](#16-ml-based-premium-pricing)
17. [AI Chatbot and Explainability](#17-ai-chatbot-and-explainability)
18. [Fraud Defense and Anti-Spoofing](#18-fraud-defense-and-anti-spoofing)
19. [System Summary](#19-system-summary)

---

## 1. Problem Statement

Delivery partners earn only when they complete orders. There are no shifts, no fixed salaries, and no safety nets. When heavy rain hits, when a platform goes down, or when a curfew is imposed, their income doesn't pause — it stops entirely.

Traditional insurance doesn't work here. Filing a claim takes time. Proving income loss is complicated. And by the time a payout arrives, the worker has already absorbed the hit.

This system takes a different approach. Instead of waiting for workers to report a loss, it watches for disruptions in real time and triggers payouts automatically — based on measurable signals like rainfall intensity, traffic congestion, and platform uptime. No claims. No paperwork. Just protection.

This model is called **parametric insurance**: payouts are linked to observable parameters, not subjective assessments.

---

## 2. Meet Arjun — Why This Matters

Arjun is a Swiggy delivery partner based in Bengaluru. On a normal day, he earns around **₹650** completing 13–15 orders across a 10-hour shift.

On **June 15**, heavy rain from the southwest monsoon lashes the city. Roads flood. Customers stop ordering. Arjun parks his bike and waits. By end of day, he has completed zero orders.

**Without this system:** Arjun loses ₹650. No compensation. No recourse. He messages Swiggy support, gets no reply, and absorbs the loss out of his weekly budget.

**With Kavach Silver (₹49/week):**

- At 11:30 AM, the system detects heavy rainfall in Arjun's zone. Rain score hits 0.82, crossing the trigger threshold.
- Arjun was active in Zone 017 (Koramangala) for 3.5 hours before the event.
- His effective hours are estimated at 3.5. His average order rate is 1.4 orders/hour. Average earning per order is ₹45.
- Payout: `1.4 × 3.5 × 0.85 × 45 × 0.82 ≈ ₹154`
- His weekly cap is ₹3,000. ₹154 is well within the cap. Payout is approved.
- **By 1:45 PM, ₹154 is in his account. He didn't file anything.**

That's the entire point of this system. Not a claims form. Not a three-day wait. A real number, in real time, for a real person who just lost real income.

---

## 3. Why Food Delivery

Before building anything, we had to pick the right domain. Three gig categories were evaluated.

| Segment | How Disruptions Hit Income | How Easy to Measure | Decision |
|---|---|---|---|
| Food Delivery | Immediately — orders drop the moment conditions worsen | High — demand signals are real-time | Selected |
| E-Commerce Delivery | Delayed — parcels can be rescheduled without immediate income loss | Medium | Rejected |
| Grocery Delivery | Indirect — depends on store supply and shelf availability | Low — hard to track reliably | Rejected |

Food delivery was the clear choice. The income impact is direct, measurable, and immediate — which is exactly what parametric insurance needs to function fairly.

---

## 4. Platform Choice — Mobile App

We chose a **mobile-first application** as the primary interface. This is not incidental — it is the only sensible choice for this user base, and the system's core functionality depends on it.

**Why not web?** Delivery partners work entirely on their bikes. They don't open laptops between deliveries. A web interface would mean they never actually engage with the product — at signup or when a payout arrives.

**Why mobile works:**

- **Real-time GPS** — the presence detection and fraud prevention layers run on device-level location data. A web app cannot provide persistent background location tracking reliably across devices.
- **Push notifications** — when a payout fires, the worker needs to know within minutes. Push notifications to a native app are reliable and immediate. Web-based alerts are not.
- **Existing behaviour** — delivery partners already work from the Swiggy or Zomato app on their phone. A mobile insurance product fits into a workflow they already live in.
- **Offline resilience** — mobile apps can cache state locally. If a worker's connection drops mid-disruption, presence data is not lost.

The mobile app handles subscription management, live zone status, payout history, and the AI chatbot. The backend and admin layer are web-accessible, but the worker-facing product is mobile-only.

---

## 5. What Counts as a Disruption

A disruption is any external factor that reduces or prevents a delivery partner from completing orders.

We split disruptions into two categories because they need to be handled differently in the payout logic.

**Soft Disruptions** — Work is still possible, but harder and slower. Income dips but does not hit zero.

- Heavy rain
- Traffic congestion from events or accidents
- High wind speeds making riding unsafe
- Extreme heat reducing how long a person can safely work outdoors

**Hard Disruptions** — Work is structurally impossible. Income is zero regardless of effort.

- Flooding that makes roads impassable
- Curfews or lockdowns blocking delivery activity
- Platform outages where no orders can be placed or received

A soft disruption warrants partial compensation scaled to severity. A hard disruption warrants full estimated income replacement.
---

## 6. Zone-Based Architecture

The system does not track every user individually. It works at the **zone level**.

A zone is a predefined geographic unit — a locality or delivery cluster within a city. Each zone has a unique **Zone ID** pre-assigned before the system runs. All external API calls use the zone's coordinates. All severity scores and payouts flow from the Zone ID.

This is a scalability decision. A city with 10,000 delivery partners across 50 zones needs 50 API calls to get environmental data — not 10,000.
---

## 7. Input Signals and Data Sources

Every 10 minutes, the system fetches a set of signals for each zone. These are the raw inputs that determine whether a disruption is occurring and how severe it is.

**Environmental**

| Signal | What It Captures | Source |
|---|---|---|
| `rain_intensity` | Millimetres of rainfall per hour | OpenWeatherMap API |
| `flood_risk` | Waterlogging risk derived from accumulated rainfall | Derived from rain data |
| `wind_speed` | Wind speed in km/h | OpenWeatherMap API |
| `temperature_extreme` | Heat index beyond safe outdoor working limits | OpenWeatherMap API |

**Regulatory**

| Signal | What It Captures | Source |
|---|---|---|
| `delivery_allowed_flag` | Whether delivery is legally permitted in the zone | Mock / authority feed |

**Platform**

| Signal | What It Captures | Source |
|---|---|---|
| `platform_uptime` | Whether the delivery platform is online and accepting orders | Internal / Mock |

**Infrastructure**

| Signal | What It Captures | Source |
|---|---|---|
| `traffic_congestion_index` | Ratio of current travel time to free-flow travel time | Google Maps API / Mock |

---

## 8. Data Ingestion Strategy

All signals are fetched every **10 minutes** per zone using a polling-based approach.

The 10-minute interval balances responsiveness with cost. Most disruptions evolve over 10–30 minutes — frequent enough to catch developing conditions, not so frequent that we are hammering external APIs unnecessarily. The interval is configurable and can be tightened during monsoon season or known high-risk periods.

---

## 9. Signal Normalization

Raw signals arrive in different units. Before any aggregation can happen, they all need to be on the same scale. Every signal is normalized to a value between 0 and 1, where 0 is no disruption and 1 is maximum disruption.

$$rain\_score = \min\left(\frac{rain\_mm}{100},\ 1\right)$$

*Example: 40mm of rain → 40/100 = **0.40**. 120mm of rain → capped at **1.0**.*

$$flood\_score = \begin{cases} 1 & \text{if } rain\_mm \geq 100 \\ 0 & \text{otherwise} \end{cases}$$

*Example: 80mm of rain → flood_score = **0**. 105mm → flood_score = **1** (waterlogging risk).*

$$wind\_score = \min\left(\frac{wind\_speed}{20},\ 1\right)$$

*Example: 10 km/h wind → **0.50**. 25 km/h → capped at **1.0**.*

$$traffic\_score = \min\left(delay\_ratio,\ 1\right)$$

*Example: delay_ratio of 0.6 means travel is 60% slower than normal → traffic_score = **0.60**.*

Binary signals map directly:

$$delivery\_allowed\_flag = false \Rightarrow score = 1$$

$$platform\_uptime = false \Rightarrow score = 1$$

*Example: Curfew imposed in Zone 042 → delivery_allowed_flag = false → score = **1** regardless of weather.*

> **These thresholds are illustrative values for the prototype.** The denominators (100mm for rain, 20 km/h for wind) will be calibrated from historical income data per city — the rainfall level that hurts delivery volumes in Bengaluru will differ from Mumbai or Hyderabad.

---

## 10. Zone Severity Computation

Normalized signals are combined into a single **zone severity score** — one number in [0, 1] representing how disrupted a zone is right now.

### Hard Override

If either of the following is true, severity is immediately set to 1:

```
IF delivery_allowed_flag == false OR platform_uptime == false
  zone_severity = 1
```

These are not gradual conditions — they make work categorically impossible. Passing them through the continuous formula would dilute their severity. The hard override handles them cleanly.

### Continuous Severity

For soft disruptions:

$$zone\_severity = \max(rain\_score,\ flood\_score,\ wind\_score,\ traffic\_score)$$

**Why MAX, not average?**

- **Delivery is limited by bottlenecks, not averages.** If rain is 0.9 and traffic is 0.2, the rider's experience is defined by the rain — not a blend of the two. Averaging gives 0.55 and implies "moderate conditions." MAX gives 0.9, which is what the rider actually faces.
- **Averaging dilutes genuine severity.** If wind is 0.1 and temperature is 0.15, including them in an average pulls the score down even though nothing improved. MAX is immune to this — adding mild signals never lowers the score.
- **Mirrors how real risk works.** A delivery shift is ruined by its worst condition, not an average of all of them. MAX captures that accurately.

*Example:*

```
rain_score    = 0.82  ← heaviest signal, drives the result
flood_score   = 0.00
wind_score    = 0.20
traffic_score = 0.15

zone_severity = max(0.82, 0.00, 0.20, 0.15) = 0.82
```

### What zone_severity actually represents

`zone_severity` is not a direct income loss percentage. It is a measure of **how severe conditions in the zone were at the time of the event** — on a scale from 0 (no disruption) to 1 (maximum disruption).

It is used as a **scaling factor in the payout formula** for soft disruptions. A severity of 0.82 means conditions were severe enough that we scale the payout to 82% of the estimated income the worker would have earned. A severity of 0.40 means conditions were moderate — the payout covers around 40% of estimated earnings.

For hard disruptions (curfew, platform outage), severity is always 1 and the full estimated income is compensated — no scaling applied.
---

## 11. Trigger Logic

The trigger check is a single threshold comparison:

$$\text{if } zone\_severity \geq \theta \implies disruption\_event = TRUE$$

When a disruption event fires for a zone, the system moves to find which users in that zone are eligible for a payout. The threshold $\theta$ is configurable per zone and calibrated against historical income data.

---

## 12. User Filtering and Scalability

A large city can have tens of thousands of active delivery partners. When a disruption triggers, the system needs to instantly find eligible users in the affected zone. A full database scan does not scale.

**Solution: Redis Geo.** User locations are stored in Redis as a geospatial index. A single `GEORADIUS` query returns everyone within a zone boundary in sub-millisecond time, regardless of total user count. Redis was chosen specifically because it is in-memory — geospatial lookups that take seconds in a relational database take microseconds in Redis.

After the GEORADIUS query, two filters apply:

1. **Zone match** — confirm the user falls within the exact zone boundary
2. **Active status** — confirm they were online and working when the disruption triggered

### Continuous Scanning During an Active Event

When a disruption event is triggered for a zone, the system does not do a one-time lookup and stop. **It keeps running GEORADIUS scans every 10 minutes for the full duration of the event.** This matters because new riders may enter the zone after the event started — and they deserve coverage too.

*Example:* A heavy rain event triggers in Zone 017 at 11:30 AM. Arjun is already there. Ravi, another rider, enters Zone 017 at 11:48 AM. At the 12:00 PM scan, Redis picks up Ravi's location, confirms he is active, and adds him to the eligible pool. His presence time accumulates with each subsequent scan until the event ends.

The GPS refresh cycle and the GEORADIUS scan are tightly coupled — both run every 10 minutes — ensuring no eligible rider who enters during an event is missed.

User GPS is refreshed every 10 minutes, matching the polling cycle. There is a small positional lag trade-off — a rider who crossed a zone boundary recently might be mapped to the previous zone for up to 10 minutes — but zone boundaries are large enough that this is negligible in practice.
---

## 13. Presence Estimation

Not everyone physically in a disrupted zone was actively working when the event hit. Presence estimation determines how long a user was effectively on the job during the disruption window — which directly feeds the payout calculation.

$$effective\_hours = \min(presence\_time,\ \bar{h}_{user},\ H_{max})$$

Each of the three variables guards against a different type of over-claiming:

**`presence_time` — Did you actually show up and stay?**

This is the time the system detected the user as active in the zone during the disruption, captured across the 10-minute GPS polling cycles.

*Why it matters:* Without this, anyone who briefly passed through the zone could claim income loss. Someone who entered Zone 017 at 1:55 PM — just 5 minutes before a 2-hour event ended — should not receive the same payout as someone who was present for the full 2 hours.

*Example:* Arjun was in Zone 017 from 9 AM onwards. The event triggered at 11:30 AM. His presence_time during the event window is **1.5 hours** (11:30 AM to 1:00 PM when he left).

**$\bar{h}_{user}$ — How long do you normally work in a day?**

This is the user's personal historical average daily working session, derived from past platform activity.

*Why it matters:* A part-time rider who typically works 3-hour sessions should not be credited for 8 hours of presence just because they happened to be in the zone for a long stretch. Without this, a worker idling most of the day could claim far more than they would have actually earned.

*Example:* Ravi typically works 4-hour sessions. On the day of the event, he was present for 6 hours. His $\bar{h}_{user}$ = 4 hours caps his credit to a realistic session, not his total time in the zone.

**$H_{max}$ — A hard ceiling for the insurer**

A fixed upper limit applied regardless of presence or personal history. It protects the insurer from edge cases — for example, a user whose historical average is artificially inflated due to a data anomaly.

*Example:* $H_{max}$ is set to 10 hours. No single event can credit a worker with more than 10 effective hours, even if their presence_time was 12 hours and their $\bar{h}_{user}$ is 11.

**Putting it together:**

```
Arjun:       presence_time = 3.5h,  avg_hours = 10h,  H_max = 10h
             effective_hours = min(3.5, 10, 10) = 3.5h  ✓

Ravi:        presence_time = 6h,    avg_hours = 4h,   H_max = 10h
             effective_hours = min(6, 4, 10) = 4h      ← avg caps him

Late entrant: presence_time = 0.1h, avg_hours = 8h,  H_max = 10h
             effective_hours = min(0.1, 8, 10) = 0.1h  ← only gets credit for time actually present
```
---

## 14. Payout Computation

The payout formula differs based on whether the disruption was soft or hard.

### Soft Disruption

Work was possible but impaired. The payout reflects partial income loss, scaled to how severe the disruption was.

**Step 1 — Estimate how much work was lost**

$$activity\_score = avg\_rate \times effective\_hours \times demand\_multiplier$$

`activity_score` represents the estimated volume of work the rider lost — the number of orders they would have completed if the disruption had not happened. `avg_rate` is their historical orders per hour. `demand_multiplier` accounts for the fact that even if a rider is willing to work, fewer customers place orders during bad conditions — available work itself drops.

*Example: Arjun's avg_rate = 1.4 orders/hour. effective_hours = 3.5h. demand_multiplier = 0.85 (15% fewer orders during rain). activity_score = 1.4 × 3.5 × 0.85 = **4.16 estimated lost orders**.*

**Step 2 — Translate to income, scaled by zone severity**

$$payout = activity\_score \times earning\_per\_order \times zone\_severity$$

`zone_severity` is the scaling factor — it reflects how disruptive conditions actually were in the zone at that time. A severity of 0.82 means we pay out 82% of the estimated lost earnings. A severity of 0.40 means conditions were moderate — the payout covers 40%.

*Example: activity_score = 4.16, earning_per_order = ₹45, zone_severity = 0.82. Payout = 4.16 × 45 × 0.82 ≈ **₹154**.*

### Hard Disruption

Work was impossible. Full estimated income is compensated, with no severity scaling.

$$activity\_score = avg\_rate \times effective\_hours$$

$$payout = activity\_score \times earning\_per\_order$$

There is no `demand_multiplier` because there was no demand at all. There is no `zone_severity` scaling because the disruption was total — the worker had zero opportunity to earn anything.

### Cap Enforcement

$$final\_payout = \min(remaining\_cap,\ payout)$$

Cumulative weekly payouts cannot exceed the plan cap. This keeps the system financially predictable for the insurer.
---

## 15. Insurance Policy Design

### Plans and Pricing

Users subscribe weekly. The premium for each plan is designed to sit at roughly **1–3% of the weekly payout cap** — making the cost of coverage proportionally small against the maximum possible benefit.

| Plan | Weekly Premium | Weekly Payout Cap | Premium as % of Cap | Best For |
|---|---|---|---|---|
| Silver | ₹49 | ₹3,000 | ~1.6% | Part-time or occasional workers |
| Gold | ₹99 | ₹6,000 | ~1.7% | Regular full-time delivery partners |
| Diamond | ₹179 | ₹12,000 | ~1.5% | High-frequency, high-earning workers |

For a worker like Arjun earning ₹650/day, Silver at ₹49/week means paying roughly 1.1% of his weekly income to protect against disruption events that could wipe out an entire day or more.

> **These values are illustrative.** The final base premiums and caps will be derived through actuarial modelling — using historical disruption frequency, average payout amounts per zone, and target loss ratios — before the product is priced for production.

Arjun, on Silver, received ₹154 during the June 15 event — well within his ₹3,000 weekly cap. If a second disruption hits later in the week, he is still covered.

### Eligibility

To qualify for a payout, a user must be physically present in the affected zone when the disruption occurred, have been actively working for at least the minimum presence threshold, and hold an active subscription with remaining cap.

### Fraud Prevention

Three layers run continuously:

- **GPS consistency checks** flag physically impossible movements (appearing in two distant zones within minutes)
- **Activity mismatch detection** cross-references GPS presence with actual platform order activity — presence without any order attempts raises a flag
- **Behavioural anomaly detection** identifies users who appear in zones only during disruption windows and are otherwise inactive

A full adversarial defense architecture is described in Section 18.

---

## 16. ML-Based Premium Pricing

Premiums are not flat. A zone that floods every monsoon costs more to insure than one that rarely sees significant rain. The system uses an XGBoost model to price premiums dynamically per zone each week.

### Why Raw Data Is Not Enough

A single polling snapshot — "it rained 40mm today" — tells you very little about how risky a zone truly is. Raw data cannot capture:

- **Consistency** — does it rain here every week, or was that a one-off?
- **Spikes** — has this zone seen extreme events recently?
- **Trends** — is it getting worse as monsoon deepens, or improving?

So instead of feeding raw readings to the model, we convert historical time-series into three structured dimensions per signal:

- **Pattern** (`avg_7d`) — the zone's typical behaviour over the past week
- **Extreme** (`max_7d`) — the worst single reading in the past week
- **Trend** (`avg_3d`) — what the last 3 days look like relative to the full week

Together, these capture not just what happened, but how consistently and how severely.

### Features

| Category | Features | What They Capture |
|---|---|---|
| Rain | `rain_avg_7d`, `rain_max_7d`, `rain_avg_3d` | Pattern, spike, recent trend |
| Traffic | `traffic_avg_7d`, `traffic_max_7d`, `traffic_avg_3d` | Pattern, spike, recent trend |
| Wind | `wind_avg_7d`, `wind_max_7d`, `wind_avg_3d` | Pattern, spike, recent trend |
| Temperature | `temp_avg_7d`, `temp_max_7d`, `temp_avg_3d` | Pattern, spike, recent trend |
| Flood History | `flood_days_7d` | How many days in the past week crossed the flood threshold |
| Events | `event_score` | Severity of scheduled events in the zone that week |
| Forecast | `forecast_rain_7d`, `forecast_wind_7d`, `forecast_temp_7d` | Expected conditions over the next 7 days |

### event_score

This is not a binary flag. `event_score` is a continuous value in [0, 1] that reflects how significantly a known upcoming event is expected to disrupt the zone.

A large-scale event like an IPL match at M. Chinnaswamy Stadium — drawing 40,000+ people, closing roads, and spiking traffic for hours — might score around **0.8–0.9**. A smaller community event like a local Durga Puja celebration might score around **0.3–0.4** depending on expected footfall and road impact. A regular weekday with no special event scores close to **0.0**.

The score is assigned based on estimated attendee volume, road closure data, and historical congestion patterns from similar past events.

### Forecast Features

The model is not just backward-looking. Three forward-looking signals are included, sourced from the same OpenWeatherMap API call used for real-time polling:

- `forecast_rain_7d` — average expected rainfall over the next 7 days
- `forecast_wind_7d` — average expected wind speed over the next 7 days
- `forecast_temp_7d` — average expected temperature over the next 7 days

This means the model prices in both what has happened (historical features) and what is about to happen (forecast features). A zone entering a heavy monsoon week will carry a higher risk score than one exiting it — even if their past 7 days looked identical.

### Model

An **XGBoost model** outputs a `zone_risk` score between 0 and 1 for each zone each week.

XGBoost was chosen because it handles the mix of continuous values, flags, and counts naturally; is robust to occasional API failures; runs inference fast enough to reprice all zones nightly; and produces feature importance scores that the chatbot can use to explain why a worker's premium changed.

### Premium Formula

$$risk\_multiplier = 1 + zone\_risk$$

$$premium = base\_price \times risk\_multiplier$$

A zone with `zone_risk = 0.8` costs 1.8× the base rate. `zone_risk = 0.1` costs 1.1×. Workers in safer zones pay less — which is fair, and creates a natural incentive for the product to expand into lower-risk areas.
---

## 17. AI Chatbot and Explainability

The system makes automated financial decisions that directly affect how much a worker earns in a given week. It has a responsibility to explain those decisions clearly.

The chatbot surfaces the exact signals, scores, and thresholds behind every decision — in plain language, in response to natural questions.

| Question | What the Bot Explains |
|---|---|
| "Why did I get ₹154 today?" | Zone, severity score, effective hours credited, payout formula applied |
| "It was raining. Why didn't I get anything?" | Severity fell below threshold, or presence check failed |
| "Why is my premium higher this week?" | Which features drove the zone_risk score up |
| "Am I covered right now?" | Subscription status, remaining weekly cap |

The goal is for this to feel like a financial tool the worker can trust, not a black box that occasionally deposits money.

---

## 18. Fraud Defense and Anti-Spoofing

> **Context:** A coordinated syndicate of 500 delivery workers organised via Telegram is using GPS-spoofing apps to fake presence in disruption zones and drain the liquidity pool. Simple GPS verification is obsolete. The response is a multi-layered, behaviour-aware fraud detection architecture.

### Core Philosophy

Kavach does not trust any single signal. We validate **presence over time**, **realistic behaviour under disruption**, and **system-level coordination patterns** — making it extremely difficult to spoof all simultaneously at scale.

---

### Layer 1 — Multi-Signal Presence Validation

GPS coordinates alone are trivially spoofed. Kavach cross-validates location against signals that require physical presence to fake simultaneously. Signals are configurable by deployment tier — the MVP uses the first four (always available on-device); enriched signals are added as integrations mature.

| Signal | Availability | What We Check | Why It's Hard to Fake |
|---|---|---|---|
| GPS coordinates | Always | Zone match + movement pattern | Used as 1 of N signals, never alone |
| Device accelerometer | Always (on-device) | Motion consistent with a stranded rider — minor shifts, idle vibration | A phone at home shows flat/zero acceleration |
| App session activity | Always (Kavach app) | Active session duration + interactions during disruption window | Real workers stay on the app waiting; fraudsters don't |
| Battery + connectivity | Always (on-device) | Bad weather degrades signal quality; clean GPS in a storm is suspicious | Real disruptions cause measurable network degradation |

> **MVP Rule:** Payout validation uses the 4 always-available signals. GPS alone is never sufficient at any tier.

---

### Layer 2 — Behavioural Consistency Engine

Real disruptions change how workers behave. An XGBoost model trained on historical disruption-day patterns flags mismatches between disruption severity and user behaviour:

```
behavioral_score = f(
  order_acceptance_rate,      // Real workers try and fail — fraudsters don't try
  app_session_duration,       // Genuinely stranded workers stay active longer
  movement_entropy,           // Real riders shift position; spoofers stay perfectly static
  historical_behavior_delta   // Compare today vs user's own baseline on similar days
)

if zone_severity >= HIGH AND behavioral_score > normal_threshold:
    flag_as_suspicious
```

A genuine delivery partner trapped in a flood zone will have high app session time, multiple failed order attempts, and low but non-zero movement (seeking shelter). A fraudster at home will have zero order attempts, perfectly static GPS, and normal phone behaviour.

---

### Layer 3 — Coordinated Ring Detection

The 500-person syndicate is detectable at the **zone level**, not just the user level.

**Statistical Spike Detection**

```
zone_trigger_rate = payout_triggers_in_zone / active_users_in_zone (per 10-min window)

if zone_trigger_rate > (historical_avg × 3.0):
    action: hold all new payout events in zone, escalate to manual review
```

**Synchronisation Fingerprinting**

Coordinated attacks have unnatural timing patterns that organic disruptions do not:

- Payout trigger timestamps cluster within seconds of each other (bots fire simultaneously)
- Device model homogeneity — fraud rings often use the same device and OS version (bulk-provisioned phones)
- GPS precision uniformity — spoofed coordinates often have identical decimal precision; real GPS has natural noise
- Social graph proximity — users sharing network towers, IP subnet ranges, or similar install dates are flagged as a cluster

```
cluster_risk = weight(timing_sync) + weight(device_homogeneity) + weight(gps_precision) + weight(network_proximity)

if cluster_risk > threshold:
    freeze zone payouts
    auto-notify fraud review team
```

---

### Layer 4 — Three-State Payout Decision

A binary approve/reject system punishes honest workers with genuine network drops. Kavach uses three outcomes:

| Outcome | Condition | Action |
|---|---|---|
| **Auto-Approve** | 3 or more of 4 core signals match, behavioural score normal | Payout in 15 minutes. No friction for honest workers. |
| **Soft-Flag** | 1–2 of 4 signals match, or behavioural anomaly detected | Payout held for 2–4 hours. Auto-releases if no new contradictions emerge. Worker notified via app. |
| **Hard-Block** | Fewer than 1 signal matches, or zone fraud alert active | Payout blocked. Worker notified with a specific reason code. Can appeal within 48 hours with platform activity proof. |

The soft-flag with auto-release is the most important UX decision here. A genuine worker in a flood zone may have patchy GPS (expected in heavy rain), low app activity (platform itself is down), and unusual behaviour vs their own baseline. If they are genuine, no new contradiction signals appear and the payout releases automatically — no action required from the worker.

---

### Layer 5 — Continuous Model Retraining

Fraud patterns evolve. The fraud detection models are retrained weekly on confirmed fraud cases (ground truth negative labels), confirmed legitimate payouts (positive labels for behavioural baseline), and new spoofing patterns detected in the wild.

The XGBoost behavioural model updates its thresholds dynamically per zone based on a rolling 30-day fraud rate.

---

### What Makes This Hard to Beat

| Attack Vector | Defence |
|---|---|
| Single-point GPS spoof | Accelerometer + session activity cross-validation (always-available on-device) |
| Sustained GPS spoof | Movement entropy + behavioural mismatch flags it over time |
| Staying active on app while at home | Behavioural engine flags normal activity during high-severity disruption |
| 500-person coordinated ring | Statistical spike detection + synchronisation fingerprinting at zone level |
| Honest worker penalised | Three-state soft-flag with auto-release protects legitimate payouts |

> Kavach makes fraud economically and computationally expensive by requiring consistency across independent signal categories, detecting coordinated attacks at the zone level, and protecting honest workers through a delay-based soft-verification system.

---

## 19. System Summary

| Principle | How We Achieve It |
|---|---|
| **Scalability** | Redis Geo for instant zone-level user lookup — no full database scans |
| **Fairness** | Presence estimation and personalised average hours prevent over-compensation |
| **Accuracy** | MAX aggregation ensures the worst condition drives severity, not a diluted average |
| **Risk Control** | Weekly caps, fraud detection layers, and coordinated ring detection keep insurer exposure predictable |
| **Intelligence** | XGBoost for both risk pricing and fraud detection; AI chatbot for full explainability |

The entire system flows from one insight: gig workers face real, measurable income risk from real, measurable external events. When you can measure the disruption objectively, you can compensate for it automatically.

Arjun did not file a claim. He did not call anyone. He parked his bike, waited out the rain, and found ₹154 in his account before lunch. That is what this system does.
