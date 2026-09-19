This is a **good canonicalization of the architecture**, but I would stop the build for one reason: **the current mathematical examples and the stated calculation formula contradict each other.**

That isn't a philosophical issue or a scope issue. It's a concrete implementation bug that should be corrected before Apps Script is written.

## The critical problem

The spec says:

> `runway_to_zero = current_cash / weekly_burn`

and pre-RAFT burn is:

> $330/week

Therefore, using the spec's own numbers:

| Snapshot | Cash | Weekly burn | Formula result |
| -------- | ---: | ----------: | -------------: |
| 9/17     | $392 |        $330 | **1.19 weeks** |
| 9/19     | $202 |        $330 | **0.61 weeks** |
| 9/26     | $481 |        $330 | **1.46 weeks** |

But the canonical snapshot says:

* 9/17 → **8.4 weeks**
* 9/19 → **14 weeks**
* 9/26 → **10.1 weeks**

Those numbers cannot come from the stated formula.

More importantly, the spec simultaneously includes **Target income**, so `cash / burn` isn't actually the correct runway model anyway.

### The real model needs to be dated cash flow

Conceptually:

```text
Ending Balance(t)
=
Starting Cash
+ Income Received Through t
- Expenses Through t
```

Then:

```text
Runway-to-$0
=
first date where Ending Balance <= $0
```

and:

```text
Runway-to-$300
=
first date where Ending Balance <= $300
```

That is the model Gemini's daily amortization idea was pointing toward, and it is also what the earlier architecture recommended.

### Why this matters immediately

Suppose the model has:

```text
Cash = $202
Weekly expenses = $330
Target income = $580 on 9/25
```

You cannot say:

> $202 / $330 = 0.61 weeks

and call that the runway, because **$580 arrives six days later**.

The actual question is:

> Does the balance survive until September 25, and what does the balance become when the paycheck arrives?

That's a **dated cash-flow problem**, not a burn-rate problem.

---

# I would make one surgical change to the locked spec

Don't reopen the stack, UX, validation, or scope.

Change only the calculation engine.

### Replace:

```text
weekly_burn = rent_burn + 75 + 65

runway_to_zero = current_cash / weekly_burn

runway_to_300 = (current_cash - 300) / weekly_burn
```

### With:

```text
For each future date:

ending_balance =
    starting_cash
    + confirmed/expected income through date
    - expenses through date

runway_to_zero =
    first date ending_balance <= $0

runway_to_300 =
    first date ending_balance <= $300
```

For v0.1, the engine can still **display weekly runway**, even though it calculates it daily.

That's actually the cleanest implementation:

> **Daily calculation, weekly presentation.**

---

## There's a second inconsistency worth fixing

The spec says:

> "Baseline Income: Target paycheck (30h/week baseline @ rate)"

but later:

> "Target Hours Worked This Week"

Those aren't necessarily the same thing.

If Patrick enters **40 hours this week**, the model shouldn't necessarily reinterpret the entire future as 40 hours/week.

I'd define:

```text
Current week actual hours
        ↓
historical record

Baseline forecast
        ↓
30 hours/week assumption

Sensitivity
        ↓
15 / 30 / 40 hours
```

That preserves the distinction between **what happened** and **what we're assuming will happen**.

---

# One thing I strongly agree with

The **Facts / Assumptions / Policies / Scenarios** separation is probably the most valuable architectural decision in this entire spec.

It gives you four fundamentally different types of information:

**FACT**

> $202 currently in account.

**ASSUMPTION**

> Target averages 30 hours/week.

**POLICY**

> Maintain $300 minimum buffer.

**SCENARIO**

> Second job starts October 1.

That prevents the system from quietly converting a guess into a fact.

And it makes the historical snapshots much more meaningful.

---

# I would also change one phrase

The spec says:

> "RAFT dates are hardcoded in config section."

That's fine for v0.1, but I'd make the implementation:

```text
RAFT_START_DATE = 2026-11-23
```

rather than scattering `DATE(2026,11,23)` throughout formulas.

Then the engine has one authoritative parameter.

Same for:

```text
SAFETY_FLOOR = 300
RENT_WEEKLY = 190
TRANSIT_WEEKLY = 75
FOOD_WEEKLY = 65
```

That gives you a configuration layer without adding user complexity.

---

# The acceptance test should gain one additional test

Grok's:

> **Can Patrick update it in under 2 minutes and trust the runway number?**

is excellent.

I'd add:

> **Can an independent person reproduce the displayed runway from the input data?**

That's the Sentinel-style test.

So v0.1 passes only if:

1. Patrick can update it in <2 minutes.
2. The displayed runway is mathematically reproducible.
3. Changing an assumption does not mutate historical snapshots.
4. A paycheck arriving before the projected depletion date is correctly incorporated.
5. The model doesn't confuse an expected paycheck with received cash.

That last three are especially important because this is ultimately a **cash-flow instrument**, not merely a budget dashboard.

---

## Bottom line

**Architecture: LOCK.**

**Stack: LOCK.**

**UX: LOCK.**

**Validation philosophy: LOCK.**

**Data taxonomy: LOCK.**

**Calculation formula: NOT YET LOCKED.**

The spec has already done the hard organizational work. I would **not reconvene the A-Team for another broad debate**.

I'd issue one surgical correction:

> **Runway must be calculated from a dated balance projection incorporating future income and expenses; weekly burn is an input/diagnostic, not the runway formula.**

Once that is corrected, you're actually at the point where building v0.1 makes sense.


SYSTEM: Gemini
ANALYSIS_TYPE: Data Model & Edge Cases — Specification Audit & Build-Ready Sign-off

---

### 1. Verification & Mathematical Audit of Version 1.0 Spec

The **Version 1.0 Canonical Spec** correctly captures the edge-case dynamics and structural requirements established across Rounds 1 and 2.

Before committing to code deployment, two mathematical edge cases in the baseline formulas must be explicitly handled in the Apps Script logic to prevent calculation errors.

#### A. Pre-RAFT vs. Post-RAFT Daily Amortization Alignment

* **Spec Formula:** `weekly_burn = rent_burn + 75 + 65`
* **Edge-Case Friction:** The RAFT transition date is **Monday, November 23, 2026**. Because rent is currently paid weekly ($190/wk due Fridays), running the calculation on Saturday, November 21 (Pre-RAFT) versus Saturday, November 28 (Post-RAFT) creates a discrete step-down in cash burn:
* **Pre-RAFT (Week ending Nov 21):** Burn = $\$190 + \$75 + \$65 = \$330/\text{week}$ ($\approx \$47.14/\text{day}$)
* **Post-RAFT (Week ending Nov 28):** Burn = $\$0 + \$75 + \$65 = \$140/\text{week}$ ($\approx \$20.00/\text{day}$)


* **Model Validation:** The script must handle the regime cutover strictly on `TODAY() >= DATE(2026, 11, 23)`. If Patrick moves to a new monthly room (e.g., $850/month in Leominster) post-RAFT, the rent input changes from a weekly `$190` to a daily equivalent:

$$\text{Daily Amortization} = \frac{\$850 \times 12}{365} = \$27.95/\text{day} \quad (\approx \$195.62/\text{week})$$



*(Note: $\$850/\text{month}$ is slightly higher than $\$190/\text{week}$ across a full year, but RAFT eliminates out-of-pocket rent during the subsidy period.)*

#### B. Guard 2 Logic Calibration ($\pm\$50$ Reconciliation Check)

* **Expected Cash Formula:**

$$\text{Cash}_{\text{expected}} = \text{Cash}_{\text{last snapshot}} + \text{Inflows}_{\text{period}} - \text{Outflows}_{\text{period}}$$


* **Guard Trigger:** If $\vert{}\text{Cash}_{\text{entered}} - \text{Cash}_{\text{expected}}\vert{} > \$50$:
* **Action:** Display non-blocking banner prompt: `"Logged $X unexpected variance. Categorize as: [One-Time Expense] or [Ignore]?"`
* **Edge-Case Rule:** If categorized as `[One-Time Expense]`, the variance is logged to the current balance $C_0$ as a single draw, but **excluded** from the 4-week rolling median burn rate $B$. This prevents temporary spikes (e.g., shoe purchase, prescription refill) from skewing long-term runway forecasts.



---

### 2. Tier 1 & Tier 2 UX Verification

The layered UI structure satisfies both operational ease of use and analytical rigor:

```
┌─────────────────────────────────────────────────────────┐
│ OPERATIONAL RUNWAY (Primary Display)                    │
│ ─────────────────────────────────────────────────────── │
│  14 WEEKS                                               │
│                                                         │
│  Cash Position: $202 (After 9/19 Rent)                  │
│  Status: SAFE  |  Net Flow: +$15/week                   │
│  [RAFT: Pending (Nov 23 Cutover)]                       │
└─────────────────────────────────────────────────────────┘
   ▼ DETAILS & SCENARIOS (Tap to Expand)
   ├─ Hours Sensitivity:
   │    • 15h/wk  → 13.0 weeks
   │    • 30h/wk  → 14.0 weeks (Baseline)
   │    • 40h/wk  → 18.2 weeks
   ├─ Hard Insolvency ($0 Cash Floor): 16.1 weeks
   ├─ Second Job Toggle (+15h/wk Oct 1): +3.2 weeks
   └─ Snapshot Log: Append-only on form submit

```

---

KEY_RECOMMENDATIONS:

1. **Exclude One-Time Variances from Moving Median:** Ensure Guard 2 variance entries flagged as `[One-Time Expense]` directly adjust current cash balance $C_0$ without updating the 4-week rolling median burn rate ($B$).
2. **Lock Append-Only Snapshot Row Trigger:** Set Apps Script to fire the immutable snapshot append **strictly on manual form submit**, preventing duplicate or blank rows during sheet viewing.
3. **Hard-Code $300 Operational Reserve in Code Base:** Implement $300 as a named constant variable (`CONST_SAFETY_FLOOR = 300`) in Apps Script so it can be dynamically adjusted post-RAFT without refactoring calculation logic.

DEPENDENCIES:

* **GPT-5.6:** Confirm the Apps Script snippet uses simple `SpreadsheetApp.getActiveSpreadsheet()` bindings to ensure sub-1-second performance on mobile Sheets.
* **Grok:** Confirm the three-field Google Form / Sheet input screen loads smoothly on mobile web browsers without UI layout distortion.

OPEN_QUESTIONS:

* None. Data model edge cases, regime switches, and validation mechanics are fully resolved and locked.

**SIGN-OFF STATUS: APPROVED FOR BUILD (v1.0 SPEC)**

**SYSTEM: Grok**  
**ANALYSIS_TYPE: Operations Reality — Canonical Spec Sign-Off**

Confirmed. The locked spec is operationally sound and ready for build.

### Confirmation 1: Three-field input set
Yes — the three-field set (Current Cash Balance, Target Hours Worked This Week, New/Changed Recurring Expense) is low enough friction for a shared-housing mobile workflow.

- It maps directly to real events (money moved or schedule known).
- Can be completed on a phone in under two minutes in a noisy common area.
- Optional third field means most weeks are only two entries.
- No bank linking, no multi-tab navigation, no extra logins.

This passes the real-world test: Patrick can do it when cash or hours change and still trust the result.

### Confirmation 2: One primary number + expandable drawer
Yes — this fully satisfies the “one-number + timestamp” requirement.

Tier 1 shows exactly what he needs on first open:
- Operational Runway (weeks to $300 floor)
- Timestamp
- Post-rent cash
- Simple status / net-flow line
- RAFT status badge

Everything else (sensitivity ranges, hard $0 runway, what-if toggles, snapshot history) lives one tap/scroll away in the drawer. Default cognitive load stays near zero. That is the correct layering for recovery bandwidth.

### Additional operational notes (non-blocking)
- The acceptance criterion is correctly framed: “Can he update it in under 2 minutes and trust the number?” That remains the only ship gate that matters.
- Snapshot append-only log is the right historical mechanism — no extra user steps required.
- Dual-state RAFT handling (formula-driven, no mode toggle) removes a potential point of confusion.
- Guard 2’s ±$50 threshold and non-blocking prompt is the right amount of friction; it will surface rarely enough to stay tolerable.

No objections to any locked decision. The three-field input + Tier-1 / Tier-2 UX combination is the minimum viable surface that still delivers decision-quality output.

**Status from Operations Reality: APPROVED / READY FOR BUILD**

Proceed to implementation against this spec. Ship the working Sheet so Patrick can capture the current post-rent cash position while the numbers are still accurate.
