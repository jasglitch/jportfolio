# FITPASS Partner Health & Churn Early Warning System
## Functional Requirements: Sensing & Reasoning Layers

**Owner:** Jasmine Sahoo, Lead PM
**Scope of this doc:** What the system watches, how it decides something is a real risk, and what it hands off to engineering to build. Action layer (email, dashboard, CRM sync) is covered in separate specs.

---

## 1. Sensing Layer: What we're monitoring

The system pulls four signal categories per partner, refreshed monthly (weekly if real-time infra is available).

| Signal | Definition | Source system | Why it matters |
|---|---|---|---|
| Slots listed | Number of bookable slots a partner has made available this cycle | Partner App | Drop in listings is often the first sign a partner is disengaging, before payout even moves |
| Slots booked | Number of listed slots actually booked by users | Booking engine | Utilization rate = booked / listed. Low utilization can be a partner problem (bad listing, pricing) or a demand problem (city-level) |
| Attendance confirmed | Bookings that converted to an actual check-in | Partner App check-in log | Distinguishes "booked but ghosted" from real usage. A partner with high bookings but low attendance has a different problem than one with low bookings |
| Payout per cycle | Total amount paid to the partner this cycle, and its trend over the last 3 cycles | Payments/Finance system | The commercial signal. Ties directly to whether the partner has a reason to stay |

**Derived inputs (computed, not raw):**
- Utilization rate = slots booked / slots listed
- Attendance rate = attendance confirmed / slots booked
- Payout trend = % change in payout, month over month, trailing 3 months
- Listing trend = % change in slots listed, trailing 3 months
- City demand index = average utilization across all partners in that city this cycle (used to separate partner-specific decline from market-wide softness)

**Update cadence:** Monthly cycle, computed in the last week of the month for that month's data. (Weekly cadence is a v2 consideration once alert volume and false-positive rate are validated at monthly cadence.)

---

## 2. Reasoning Layer: How risk is decided

This is not a single threshold rule. It's a two-step check: **detect, then verify.**

### Step 1: Detection (per partner, per cycle)
A partner is provisionally flagged if any of the following hold:
- Utilization rate has dropped more than 25% versus its own trailing 3-month baseline
- Payout has declined for 2 or more consecutive cycles
- Slots listed has declined more than 20% versus trailing 3-month baseline

These are partner-specific thresholds, not fleet-wide averages, because a small yoga studio and a large multi-court sports facility have very different normal ranges.

### Step 2: Verification (before it becomes a flag a human sees)
Before a provisional flag is surfaced, the system checks it against context to reduce false positives:
- **City demand check:** If city-wide utilization also dropped this cycle, the decline is more likely market-wide, not partner-specific. The flag is still raised but annotated as "city softness," which changes the suggested action from "call the partner" to "no action needed, monitor."
- **Single vs. sustained signal:** A one-cycle dip is logged but not escalated to a city head unless it persists into a second cycle, or unless it's a Tier A/B partner where even a one-cycle dip has outsized revenue impact.
- **Confidence score:** Each flag carries a confidence level (high/medium/low) based on how many of the three detection signals fired together. Multiple signals firing together = high confidence.

Only flags that clear verification get routed to the action layer.

### Tiering logic (A–E)
Every partner is tiered using two inputs:
1. **Payout per slot** (commercial value to FITPASS)
2. **City demand density** (whether the partner operates in a high or low demand market)

A high payout-per-slot partner in a low-demand city carries different risk than the same payout-per-slot partner in a saturated metro; tiering accounts for both rather than payout alone. Tier boundaries (what payout-per-slot range = Tier A vs B, etc.) need to be set using actual FITPASS payout data before this ships. This doc flags that as an open input, not a resolved threshold.

---

## 3. What's explicitly NOT in scope for the reasoning layer (v1)
- No predictive/ML churn scoring in v1. Detection is rule-based on trailing baselines. This keeps the system explainable, defensible in a stakeholder review, and buildable without a data science dependency.
- No adaptive/self-learning thresholds in v1. The "before/after" review each cycle is a **human-reviewed comparison** (did contacted partners recover, yes/no), not an automated model retrain. If thresholds need adjusting, a PM/analyst adjusts them manually based on that review. This is a deliberate scope cut, not a gap, state it this way if asked in an interview.

---

## 4. Engineering handoff summary
- **Inputs required:** slots listed, slots booked, attendance confirmed, payout, all at partner-month grain, plus city-level rollups for demand index.
- **Compute:** trailing 3-month baselines and trend calculations per partner, refreshed monthly.
- **Output:** a per-partner risk record (tier, flag status, confidence, contributing signals, suggested action text) that the action layer (email/dashboard) consumes.
- **Open decisions before build:** exact tier payout thresholds, whether verification runs at monthly or weekly cadence, and where the confidence score display lives (dashboard only, or also in the email).
