---
name: mde-calculator
description: >
  Calculates the Minimum Detectable Effect (MDE) and minimum test duration for an
  A/B test using real Amplitude data. Use this skill WHENEVER the user asks to
  calculate MDE, minimum sample size, how many days a test needs, or the estimated
  end date of an experiment. Also triggers on: "MMV", "sample size", "test duration",
  "how long does the test need to run?", "calculate the test for [app] in [country]".
  The only manual inputs required are: app selection, country, and estimated launch date.
  Everything else is fetched automatically from Amplitude.
---

# MDE Calculator Skill

Calculates the minimum viable sample (MMV) and estimated test duration for an A/B test
using real funnel data from Amplitude for the selected app and country.

---

## Step 0 — Collect inputs

If not already provided, ask for:
1. **App / Project**: use `Amplitude:get_context` to list available projects (exclude "stage").
2. **Country**: specific country or "all countries" (no filter).
3. **Estimated launch date**: DD/MM/YYYY.

Do NOT ask for MDE — suggest it in Step 4 after seeing the data.

---

## Step 1b — Validate the step 2 (paywall view) event ⚠️

Step 2 of the funnel is the paywall view event. The correct event must represent **≥ 90%** of the entry point (SS). If it doesn't, the wrong event is being used.

**Validation rule:**
1. Run the funnel with `view_screen_pricing` as step 2
2. Check: `view_screen_pricing / SS ≥ 90%`?
   - ✅ Yes → use `view_screen_pricing`
   - ❌ No → try `view_screen_paywall` instead
3. Check: `view_screen_paywall / SS ≥ 90%`?
   - ✅ Yes → use `view_screen_paywall`
   - ❌ No → flag to the user — neither event captures the full paywall funnel

> Use only **one** event, whichever meets the ≥ 90% threshold. Never combine both.

**Known mapping:**
- iMote, AI Cleaner, Translator GO, QR Now → `view_screen_pricing` (≥90% ✓)
- Ereasy → `view_screen_paywall` (≥90% ✓ · `view_screen_pricing` only 15.8% ✗)

---

## Step 1c — Detect offer type ⚠️ CRITICAL FIRST STEP

The offer type determines which events to use and the conversion window.
**All funnels start from `[AppsFlyer] Install`.**

| Offer type | How to detect | Step 3 (offer event) | Step 4 (conversion) | Conv. window |
|-----------|--------------|----------------------|---------------------|-------------|
| **Free Trial** | `rc_trial_started_event` has high volume | `rc_trial_started_event` OR `rc_initial_purchase_event` | `rc_trial_converted_event` | trial_duration + 1 day |
| **Intro Offer** | `rc_initial_purchase_event` with product_id containing "intro" | `rc_initial_purchase_event` (filter: product_id contains "intro") | `rc_renewal_event` | intro_duration + 1 day |
| **No Offer** | `complete_purchase` dominates, no trial/intro | `complete_purchase` | — (3-step funnel only) | 30 days |

> **Always verify conversion window** by checking "Completed within X days" in the
> Amplitude funnel chart for that specific app before running queries.

---

## Step 1c — Choose the correct entry point

Before running any funnel, check the volume of `[AppsFlyer] Install` for the last 30 days.

**Rule:**
- If `[AppsFlyer] Install` ≥ 1,000 → use `[AppsFlyer] Install` as step 1
- If `[AppsFlyer] Install` < 1,000 → search for the app's "first open" event and use it instead

The "first open" event is app-specific. Search for it with `Amplitude:search` using terms like "first", "new user", "open first". Common names:
- `open_app_first_time` — QR Now, and likely other Leadtech apps with low install tracking
- `[Amplitude] New User` — generic Amplitude built-in (may not always work in funnels)

> ⚠️ `[AppsFlyer] Install` can be very low if AppsFlyer attribution is not properly
> configured for all traffic sources. The first-open event captures ALL new users
> regardless of attribution, making it a more reliable entry point in these cases.

---

## Step 2 — Fetch funnel data

> ⚠️ **Always use `type: "funnels"`, NOT `eventsSegmentation`.**
> All funnels start from `[AppsFlyer] Install` (or the first-open event if install < 1,000).

**Standard 4-step funnel (Free Trial):**
```
[AppsFlyer] Install → view_screen_pricing → [offer event(s)] → [conversion event]
```

**Standard 3-step funnel (No Offer):**
```
[AppsFlyer] Install → view_screen_pricing → complete_purchase
```

### FREE TRIAL — run 2 funnel queries, then combine

**Query A** — trial path:
```
Install → view_screen_pricing → rc_trial_started_event → rc_trial_converted_event
```
**Query B** — direct purchase path:
```
Install → view_screen_pricing → rc_initial_purchase_event → rc_trial_converted_event
```
**Combine results from Funnel A and B — with overlap detection:**

- If TC_A and TC_B are **very different** (one >> other): `TC_total = TC_A + TC_B` (minimal overlap, e.g. iMote)
- If TC_A ≈ TC_B (similar values): `TC_total = max(TC_A, TC_B)` (heavy overlap — same users going through both events in the same flow)

> When `complete_purchase` counts are much higher than `rc_trial_started_event` and both TC values are similar, it indicates users fire both events in sequence during the same subscription flow. In this case, summing would double-count.

`CP_total = step3_A + step3_B` (always sum step 3 — different events)
**Baseline CR = TC_total / Install**

Common query parameters:
```json
{
  "type": "funnels",
  "params": {
    "conversionSeconds": [see table below],
    "range": "Last 30 Days",
    "mode": "ordered",
    "metric": "CONVERSION",
    "countGroup": "User",
    "newOrActive": "active",
    "eventAbstraction": "Event",
    "nthTimeLookbackWindow": 365,
    "segments": [country and/or platform filter if needed]
  }
}
```

### INTRO OFFER — run 1 funnel query

```
Install → view_screen_pricing → rc_initial_purchase_event (filter: product_id contains "intro") → rc_renewal_event
```

Event filter for step 3:
```json
{
  "event_type": "rc_initial_purchase_event",
  "filters": [{"subprop_key": "product_id", "subprop_op": "contains",
               "subprop_type": "event", "subprop_value": ["intro"]}]
}
```
**Baseline CR = rc_renewal_event / Install**

### NO OFFER — run 1 funnel query

```
Install → view_screen_pricing → complete_purchase
```
**Baseline CR = complete_purchase / Install**

---

## Conversion window reference

| Offer duration | conversionSeconds |
|---------------|------------------|
| 3-day trial/intro | 345,600 (4 days) |
| 7-day trial/intro | 691,200 (8 days) |
| 14-day trial | 1,296,000 (15 days) |
| Unknown | 2,592,000 (30 days) |

---

## Step 3 — Calculate metrics

```
SS  = [AppsFlyer] Install count (step 1)
VSP = view_screen_pricing count (step 2)
CP  = offer event count (step 3)
TC  = conversion event count (step 4, or CP for No Offer)

CR baseline ★ = TC / SS
daily_SS        = SS / 30
daily_per_var   = daily_SS / 2   (100% exposure, 2 variants)
```

---

## Step 4 — Recommend optimal MDE

**Do NOT default to 10%.** Build a table for 5%, 7%, 10%, 15%, 20%.

```
p1 = CR_baseline
p2 = p1 × (1 + MDE/100)
δ  = p2 − p1
MMV = ⌈ 7.849 × (p1(1−p1) + p2(1−p2)) / δ² ⌉
days = ⌈ MMV / daily_per_variant ⌉
```

**Recommendation logic:**
1. Preferred range: **7–21 days**
2. Choose **smallest MDE within 7–21 days**
3. Explain trade-off naturally, then confirm with user

---

## Step 5 — Display results

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  MDE CALCULATOR — [App] · [Offer type] · [Country]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  📊 FUNNEL (last 30 days · [offer type])
  Install              [SS]     100.0%
  view_screen_pricing  [VSP]    [CR_vsp]%
  [Offer event(s)]     [CP]     [CR_cp]%
  [Conversion event]   [TC]     [CR_tc]%  ★

  📐 MDE OPTIONS
  MDE 5%  → [D_5] days  (+[pp_5]pp)
  MDE 7%  → [D_7] days  (+[pp_7]pp)
  MDE 10% → [D_10] days (+[pp_10]pp)
  MDE 15% → [D_15] days (+[pp_15]pp)
  MDE 20% → [D_20] days (+[pp_20]pp)

  🎯 TEST RESULTS
  ┌──────────────────────────────────────────────┐
  │ Offer type           │ [type]                 │
  │ Baseline CR          │ [CR]%                  │
  │ Min. sample (MMV)    │ [MMV]/variant          │
  │ Daily installs       │ [daily]/day            │
  │ Days to significance │ [days] days            │
  │ Launch date          │ [date]                 │
  │ Estimated end date   │ [date]                 │
  └──────────────────────────────────────────────┘
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Known apps reference

| App | Offer type | Entry point | Paywall event | Step 3 event(s) | Step 4 event | Conv. window |
|-----|-----------|------------|--------------|-----------------|-------------|-------------|
| iMote iOS & Android (100004822) | Free Trial | `[AppsFlyer] Install` | `view_screen_pricing` | `rc_trial_started_event` OR `complete_purchase` | `rc_trial_converted_event` | 30 days |
| Translator GO (100000686) | Intro Offer | `[AppsFlyer] Install` | `view_screen_pricing` | `rc_initial_purchase_event` (product_id contains "intro") | `rc_renewal_event` | 8 days |
| AI Cleaner iOS (100002377) | Free Trial | `[AppsFlyer] Install` | `view_screen_pricing` | `rc_trial_started_event` OR `rc_initial_purchase_event` | `rc_trial_converted_event` | 4 days |
| QR Now iOS & Android (100000690) | Free Trial | `open_app_first_time` ⚠️ | `view_screen_pricing` | `rc_trial_started_event` OR `complete_purchase` | `rc_trial_converted_event` | 30 days |
| Ereasy (100002476) | Free Trial | `open_app_first_time` ⚠️ | `view_screen_paywall` ⚠️ | `rc_trial_started_event` | `rc_trial_converted_event` | 30 days |
| Video Up (100000576) | Mixed (Trial 3d/7d + Intro + Annual) | `[AppsFlyer] Install` | `view_screen_pricing` ⚠️ 88.8% | `rc_trial_started_event` OR `complete_purchase` | `rc_trial_converted_event` | 30 days |

---

## Edge cases

- **All MDEs > 21 days**: not viable at current traffic. Combine countries or run globally.
- **All MDEs < 5 days**: use MDE 5% — traffic allows maximum sensitivity.
- **Multiple countries**: run per country, present comparison table.
- **Platform filter (iOS only)**:
  ```json
  {"group_type": "User", "op": "is", "prop": "platform",
   "prop_type": "user", "type": "property", "values": ["iOS"]}
  ```
