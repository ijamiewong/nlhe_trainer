# NLHE PRE-FLOP TRAINER - MASTER STATE FILE
**State file version:** 2.0 | **Compatible HTML trainer:** v2.0 (JSON schema 2.0) | **Last updated:** 2026-10-28

---

## 📑 SESSION CONFIGURATION
- **Active Session ID:** 03
- **Session Timestamp:** 2026-10-28 13:45 UTC
- **Current Hand Pointer:** 61
- **Session Size:** 10 hands (default; the user may request a different size)
- **Session Close Condition:** Session closes when the user pastes the completed Results JSON back into the chat. See Coach Instruction, step 5.
- **Analytics Baseline:** Session 02 onward. Session 01 is a pre-trainer archive and is excluded from all scoring, grids and leak counts.

### Game Defaults (assumed unless a scenario states otherwise)
| Setting | Default |
| :--- | :--- |
| Format | 6-max cash |
| Positions | UTG, HJ, CO, BTN, SB, BB |
| Effective stack | 100BB |
| Ante / rake | None |
| Standard stack twists | 40BB (short), 150BB (deep), **<20BB (MTT push/fold zone)** |
| Planned expansion | MTT: ante, ICM/bubble, 10–20BB shove/fold ranges. Not yet active; for now the <20BB spots are stack twists only. |
| Card notation | `T` for ten (e.g. `Th9h`). Session 01 history uses `10`; it is left as archived. |

---

## 🤖 GTO COACH SYSTEM INSTRUCTION
```text
You are a world-class GTO poker coach running a pre-flop training simulation. A separate HTML trainer (v2.0) presents the hands to the user and returns the answers as JSON. You never deal hands one at a time in chat. You work in batches through JSON.

1. LOAD STATE: Read this whole file first. Use the Player Performance Profile, Active Leaks, Re-deal Queue and Coverage Tracker to choose what to train. Treat Session 01 as history only.

2. GENERATE SESSION: Output ONE JSON block (schema in "JSON Contracts") of [Session Size] hands, starting at the Current Hand Pointer, with sequential integer hand ids. Each hand has:
   - base: hero position, hero cards, effective_stack_bb, and an unambiguous `action` string describing everything that happened before hero acts (e.g. "HJ opens 2.2BB, CO calls").
   - twist ("What If?"): the same hero cards, and a structurally different spot. Allowed twist types: position shift, multi-way flat callers, open-limpers / ISO, non-standard sizing, cold 4-bet or cold-call behind hero's open, 3-bet / 4-bet pots, stack change (40BB, 150BB, <20BB).
   - Mix of content: roughly 60% weighted to Active Leaks and the Re-deal Queue, 40% to uncovered spots in the Coverage Tracker. Do not repeat an exact hand + spot from the Re-deal Queue; use a close variant.
   - Output the JSON in a single code block with no commentary inside it.

3. INPUT: The user plays the session in the HTML trainer and pastes back the Results JSON. Per decision, the trainer returns `action` (FOLD / CALL / RAISE), `size_bb` (a number or null) and `plan` (RF, RC, RP, or empty). Valid plans: RF (Raise to Fold vs 4B), RC (Raise to Call 4B), RP (Raise to Push / 5B jam). Interpret the answers like this:
   - RAISE covers open, 3B, squeeze, 4B or jam. Work out which from the scenario text.
   - A jam is recorded as RAISE with size_bb equal to the effective stack.
   - Plan is scored only when action = RAISE. Ignore a plan on FOLD or CALL.
   - RAISE with no size_bb: score the sizing component 0 and flag "size missing".
   - RAISE with no plan: score the plan component 0 and flag "plan missing". (The trainer normally blocks this.)
   - A null base or twist means the hand was skipped. Record it as "Skipped" and exclude it from scoring.

4. EVALUATE: Evaluate every decision against GTO. For each hand give: the correct action / sizing / plan, one line of reasoning, and the three component scores. Do not write long essays. Output a compact feedback table, then the session summary.

5. SESSION CLOSE: Immediately after the evaluation, output a fully updated copy of this entire markdown document:
   - Append the session to the Master History Ledger using the Ledger Format. Add the real timestamp and hand range.
   - Recompute the Global Range Grid, Position Grid, Spot-Type Grid and Session Summary table from the scored decisions (never edit scores by hand).
   - Update Active Leaks: counts, last-seen, status. Promote a Watchlist item to Active when it appears 3 times across scored decisions. Mark a leak Resolved after 10 consecutive opportunities scored >= 80.
   - Rebuild the Re-deal Queue (max 10 spots) and Coverage Tracker.
   - Increment the Current Hand Pointer by the session size and the Active Session ID by 1. Update the Session Timestamp.
   - Add a Changelog line if the file structure or rules changed.
```

---

## 🔌 JSON CONTRACTS (must match the HTML trainer v2.0)

### Session JSON (AI → HTML)
The trainer requires `schema_version`, `session.id`, a non-empty `hands` array, and each hand needs `id`, `base` and `twist`, each with `hero`. Everything else is optional but shown if present.
```json
{
  "schema_version": "2.0",
  "session": { "id": "03", "generated_at": "2026-10-28T13:45:00Z", "hand_count": 10, "mode": "preflop" },
  "hands": [
    {
      "id": 61,
      "base":  { "hero": { "position": "BTN", "cards": ["9c","8c"] }, "effective_stack_bb": 100, "action": "HJ opens 2.2BB" },
      "twist": { "hero": { "position": "BTN", "cards": ["9c","8c"] }, "effective_stack_bb": 100, "action": "HJ opens 2.2BB, CO calls" }
    }
  ]
}
```
Rules: card format is rank + suit letter (`Ah`, `Td`, `9c`); `T` for ten. Positions are UTG, HJ, CO, BTN, SB, BB. `hand_count` equals the length of `hands`. Hand ids continue from the Current Hand Pointer and never repeat. Test sessions use ids 9001 and up so they never collide with real pointer ids.

### Results JSON (HTML → AI)
```json
{
  "schema_version": "2.0",
  "result_type": "completed_session",
  "session": { "id": "03", "completed_at": "ISO timestamp", "hand_count": 10 },
  "results": [
    { "hand_id": 61,
      "base":  { "action": "RAISE", "size_bb": 6.6, "plan": "RC" },
      "twist": { "action": "CALL",  "size_bb": null, "plan": "" } }
  ]
}
```
The results do not repeat the scenario text. The coach matches `hand_id` to the Session JSON it generated earlier in the same chat. If that JSON is not in the chat, ask the user to paste it.

---

## 🧮 SCORING RUBRIC

**Label to points** (used for every component):

| Label | Points |
| :--- | :--- |
| Excellent | 100 |
| Good | 80 |
| Sub-optimal | 50 |
| Blunder / Major Leak | 0 |

**Decision score** (one per base or twist decision) = average of its applicable components.
- FOLD / CALL: the **Action** component only.
- RAISE: **Action + Sizing + Plan**, averaged equally.

**Hand score** = average of the base and twist decision scores.

**Category and grid scores** = average of the decision scores in that bucket. A decision counts once for its own hero hand class, position or spot type.

**Sample size rule:** a score is shown whenever there is data, but a colour status is only assigned at **n >= 10 decisions**. Below that, status is ⚪ Provisional.

**Status thresholds (n >= 10):** 🟢 >= 80 | 🟡 60–79 | 🔴 < 60.

**Legacy note:** Session 02 was backfilled from its single combined label per decision (no separate component scores existed), so each decision there scores the label value directly. Sessions from 03 onward use the component rubric.

---

## 📊 PLAYER PERFORMANCE PROFILE & ANALYTICS
*Source: scored decisions only (Session 02 onward). Session 01 is excluded.*

### Session Summary
| Session | Hands | Decisions | Avg Score | Excellent | Good | Sub-optimal | Blunder | Focus / Notes |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 02 | 51–60 | 20 | 55 | 6 | 0 | 10 | 4 | First tracked session; heavy planning errors and OOP discipline |

### Global Range Grid
| Hand Category | Hands Included | Score (0-100) | n | Status | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Premium Pairs** | AA, KK, QQ, JJ, TT | 38 | 4 | ⚪ Provisional | JJ plan RP error; TT planned RF and folded to a 4B |
| **Medium Pairs** | 99, 88, 77 | — | 0 | ⚪ No data | |
| **Low Pairs** | 66, 55, 44, 33, 22 | — | 0 | ⚪ No data | |
| **Suited Aces** | A2s–A9s | 25 | 2 | ⚪ Provisional | SB flat trap with A3s (both decisions) |
| **Suited Broadways** | AKs–ATs excl. above, KQs, KJs, KTs, QJs, QTs, JTs | 75 | 6 | ⚪ Provisional | Good multi-way closing flat (KdQd); 3B sizing / planning drift |
| **Offsuit Broadways** | AKo, AQo, AJo, ATo, KQo, KJo, KTo, QJo, QTo, JTo | 38 | 4 | ⚪ Provisional | Cold-calling a 3B OOP; RC planned with unsuited hands; 150BB over-open |
| **Suited Connectors / Gappers** | T9s, 98s, 87s, 76s, 65s, 54s and one-gappers | 75 | 4 | ⚪ Provisional | Good OOP fold; 3B vs pure flat mix error |
| **Other Suited** | K9s-, Q9s-, J9s- … not covered above | — | 0 | ⚪ No data | |
| **Other Offsuit** | Weak Ax-o, Kx-o, Qx-o and so on | — | 0 | ⚪ No data | |

### Position Grid *(hero position per decision)*
| Position | Score | n | Status |
| :--- | :--- | :--- | :--- |
| UTG | — | — | Populated from Session 03 onward |
| HJ | — | — | Populated from Session 03 onward |
| CO | — | — | Populated from Session 03 onward |
| BTN | — | — | Populated from Session 03 onward |
| SB | — | — | Populated from Session 03 onward |
| BB | — | — | Populated from Session 03 onward |

### Spot-Type Grid
| Spot Type | Score | n | Status |
| :--- | :--- | :--- | :--- |
| Open (folded to) | — | — | Populated from Session 03 onward |
| Facing open: 3B | — | — | Populated from Session 03 onward |
| Facing open: flat / fold | — | — | Populated from Session 03 onward |
| Squeeze / multi-way | — | — | Populated from Session 03 onward |
| Facing 3B / 4B | — | — | Populated from Session 03 onward |
| Blind vs Blind | — | — | Populated from Session 03 onward |
| Short stack (<=40BB) | — | — | Populated from Session 03 onward |
| MTT zone (<20BB) | — | — | Populated from Session 03 onward |
| Deep (150BB+) | — | — | Populated from Session 03 onward |

*(Session 02 decisions were not tagged by position or spot; the tracker starts at Session 03.)*

---

## 🔴 ACTIVE EXPLOITABLE LEAKS
| # | Leak | Severity | First seen | Last seen | Evidence (S02+) | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| L1 | **SB Flatting Trap:** flat-calling single opens OOP from the SB with implied-odds hands (Ac3c) | High | S02 H52 | S02 H52 | H52 base + twist (2 of 2 failed) | Active |
| L2 | **Planning Inversion:** tags JJ/TT as `RF` or jams (`RP`) into better hands; tags unsuited broadways as `RC` instead of `RF` | High | S02 H54 | S02 H60 | H54 base + twist, H55 base, H60 base | Active |
| L3 | **Stack-depth inelasticity:** does not adjust opens at 150BB, over-squeezes into 40BB stacks | Medium | S02 H58 | S02 H59 | H58 twist, H59 twist | Active |
| L4 | **BvB / positional sizing:** 3B sizes too large when BB has position over an SB open | Medium | S02 H56 | S02 H56 | H56 base | Active |
| L5 | **Cold-calling a 3B OOP / folding blind defence vs a 4B** | High | S02 H55 | S02 H60 | H55 twist, H60 twist | Active |

### Watchlist (Session 01 history, unconfirmed; promoted to Active after 3 scored occurrences)
- Too tight in early positions: KsQs UTG fold (H17), 66 CO vs UTG fold (H04).
- Multi-way flats with AQo (H15, H40).
- Flatting OOP from the SB with broadways (H34, H39).
- Offsuit open sizing (QcJd HJ 2.2BB, H24).
- Plan inversion on non-premium hands: Sub-optimal plans on H45, H48, H49.

### Resolved Leaks
*None yet.*

---

## 🎯 RE-DEAL QUEUE (max 10; built from Blunder / Sub-optimal decisions, deal a close variant, not the identical spot)
| Source | Original Spot | What to test next |
| :--- | :--- | :--- |
| S02 H52 twist | SB Ac3c vs BTN 3.5BB open, called | SB vs large BTN open with a suited ace: 3B or fold |
| S02 H60 base + twist | SB TT 3B, planned RF; folded to a 23BB 4B | Pair vs a steal, RC plan; call or jam vs a 4B |
| S02 H55 twist | CO AdQs cold-calling a BTN 3B OOP | Offsuit Broadway facing a 3B: fold or 4B bluff |
| S02 H58 twist | CO AdJc 150BB, 2BB open, RC plan | Deep-stack open sizing and RF plan |
| S02 H59 twist | BTN KsQs vs 40BB SB squeeze spot | Short-stack blind in the pot: flat vs 3B |
| S02 H54 | UTG JJ open, RP plan; CO JJ 3B, RP plan | Medium-strength pairs: RC, not RP |
| S02 H56 base | BB KdQd 3B 12BB vs SB 3BB open | BB vs SB: IP sizing about 9BB |

## 🧭 COVERAGE TRACKER (twist types: counts across scored sessions)
| Twist Type | S02 count | Total | Gap? |
| :--- | :--- | :--- | :--- |
| Position shift | 3 | 3 | |
| Multi-way flat callers | 3 | 3 | |
| Non-standard sizing (3.5BB, 3BB, etc.) | 3 | 3 | |
| Cold 3B / 4B facing hero | 3 | 3 | |
| 40BB stack | 1 | 1 | Light |
| 150BB stack | 1 | 1 | Light |
| <20BB (MTT push/fold zone) | 0 | 0 | ❗ Not seen |
| Open-limpers / ISO | 0 | 0 | ❗ Not seen |
| Cold 4B / cold call behind hero's open | 0 | 0 | ❗ Not seen |
| 5B / jam decisions (RP) | 0 | 0 | ❗ Not seen |

---

## 🗂️ MASTER HISTORY LEDGER

### Ledger Format (for sessions appended from 03 onward)
```text
### ⏱️ Session NN | YYYY-MM-DD HH:MM UTC | Hands A–B | Avg score XX
*   **Hand NN:** POS | CARDS
    - *Base (spot):* Action/size [Plan: XX] [GTO: label — note] [Score: A/S/P → XX]
    - *Twist (spot):* Action/size [Plan: XX] [GTO: label — note] [Score: A/S/P → XX]
```
A / S / P = Action / Sizing / Plan component scores (S and P are `—` for FOLD and CALL).

### ⏱️ Session 01 | 2026-10-28 13:00 UTC | Hands 1–50
*📁 Pre-trainer archive. Unscored: excluded from the grids, leaks, coverage and summary. History review only. Hands 1–30 are base-only. Plans were mostly not recorded. Notation uses `10` for ten.*
*   **Hand 01:** BTN | JdJs | Base (vs HJ+CO Open): 3B 9BB [GTO: Excellent]
*   **Hand 02:** BB | Ks4s | Base (vs SB Open): 3B 12BB [GTO: Excellent]
*   **Hand 03:** UTG | AcJd | Base (Folded to): Open 2BB [GTO: Sub-optimal Sizing]
*   **Hand 04:** CO | 6h6c | Base (vs UTG Open): Fold [GTO: Sub-optimal/Too Tight]
*   **Hand 05:** SB | QcJs | Base (vs BTN Open): 3B 7.5BB [GTO: Sub-optimal Sizing]
*   **Hand 06:** HJ | Ac5c | Base (Folded to): Open 2BB [GTO: Excellent]
*   **Hand 07:** BTN | KhQd | Base (vs HJ Open): Call [GTO: Excellent]
*   **Hand 08:** BB | 8s7s | Base (vs HJ Open): Call [GTO: Excellent]
*   **Hand 09:** CO | Qd9d | Base (Folded to): Open 2BB [GTO: Excellent]
*   **Hand 10:** UTG | 10h9h | Base (Folded to): Fold [GTO: Excellent]
*   **Hand 11:** BTN | AsQs | Base (vs UTG Open): 3B 7.5BB [GTO: Excellent]
*   **Hand 12:** SB | KdJc | Base (vs CO Open): 3B 8.8BB [GTO: Excellent]
*   **Hand 13:** HJ | 4h4c | Base (Folded to): Open 2BB [GTO: Good]
*   **Hand 14:** BB | Jc7c | Base (vs BTN Open): Call [GTO: Excellent]
*   **Hand 15:** CO | AcQd | Base (vs UTG Open + HJ Flat): Call [GTO: Sub-optimal Flat]
*   **Hand 16:** BTN | 9h8h | Base (vs CO Open): Call [GTO: Excellent]
*   **Hand 17:** UTG | KsQs | Base (Folded to): Fold [GTO: Sub-optimal/Too Tight]
*   **Hand 18:** BB | Ad2d | Base (vs HJ Open): 3B 8.8BB [GTO: Excellent]
*   **Hand 19:** SB | 7h7c | Base (vs BTN Open): 3B 10BB [GTO: Excellent]
*   **Hand 20:** CO | Kc10s | Base (Folded to): Open 2BB [GTO: Excellent]
*   **Hand 21:** UTG | 4s4d | Base (Folded to): Fold [GTO: Excellent]
*   **Hand 22:** BTN | Ks9s | Base (vs HJ Open): Call [GTO: Excellent]
*   **Hand 23:** SB | AcQs | Base (vs CO Open): 3B 8.8BB [GTO: Excellent]
*   **Hand 24:** HJ | QcJd | Base (Folded to): Open 2.2BB [GTO: Sub-optimal Offsuit Open]
*   **Hand 25:** BB | 5h4h | Base (vs HJ Open): Call [GTO: Excellent]
*   **Hand 26:** CO | KsJc | Base (Folded to): Open 2BB [GTO: Excellent]
*   **Hand 27:** BTN | 7d6d | Base (vs HJ Open): Call [GTO: Excellent]
*   **Hand 28:** BB | Ac10s | Base (vs BTN Open): 3B 8BB [GTO: Good]
*   **Hand 29:** SB | 9c9d | Base (vs BTN Open): 3B 10BB [GTO: Excellent]
*   **Hand 30:** UTG | KhQh | Base (Folded to): Open 2.5BB [GTO: Excellent]
*   **Hand 31:** BTN | 8d7d 
    - *Base (vs HJ Open):* Call [GTO: Excellent]
    - *Twist (vs UTG Open + HJ Flat):* Fold [GTO: Excellent]
*   **Hand 32:** BB | AhJc 
    - *Base (vs SB Open):* 3B 12BB [GTO: Excellent]
    - *Twist (vs CO 2BB Open):* 3B 8BB [GTO: Excellent]
*   **Hand 33:** CO | Qs10s 
    - *Base (vs UTG Open):* Call [GTO: Excellent]
    - *Twist (Folded to):* Open 2BB [GTO: Excellent]
*   **Hand 34:** SB | KhQc 
    - *Base (vs UTG Open):* Call [GTO: Sub-optimal Flat OOP]
    - *Twist (vs BTN Open):* 3B 8BB [GTO: Excellent]
*   **Hand 35:** HJ | Jh10h 
    - *Base (vs UTG Open):* Call [GTO: Sub-optimal Flat OOP]
    - *Twist (Folded to):* Open 2.2BB [GTO: Excellent]
*   **Hand 36:** BTN | 4h4d 
    - *Base (vs CO Open):* 3B 6.6BB [GTO: Good]
    - *Twist (Folded to):* Open 2BB [GTO: Excellent]
*   **Hand 37:** BB | Kh10h 
    - *Base (vs UTG Open):* Call [GTO: Excellent]
    - *Twist (vs UTG + HJ + CO Multi-way):* Call [GTO: Excellent]
*   **Hand 38:** CO | AdJd 
    - *Base (vs HJ Open):* 3B 6.6BB [GTO: Excellent]
    - *Twist (Folded to):* Open 2BB [GTO: Excellent]
*   **Hand 39:** SB | 10s9s 
    - *Base (vs CO Open):* Call [GTO: Sub-optimal Flat OOP]
    - *Twist (vs CO Open + BTN Flat):* Call [GTO: Sub-optimal Flat OOP]
*   **Hand 40:** BTN | AsQc 
    - *Base (vs UTG Open):* Fold [GTO: Excellent]
    - *Twist (vs UTG Open + HJ Flat):* Call [GTO: Sub-optimal Multi-way Flat]
*   **Hand 41:** CO | 9s9h 
    - *Base (Folded to):* Open 2BB [Plan: RC] [GTO: Excellent]
    - *Twist (vs HJ 2.2BB Open):* 3B 6.6BB [Plan: RC] [GTO: Excellent]
*   **Hand 42:** BTN | AdJh 
    - *Base (vs UTG Open):* Fold [GTO: Excellent]
    - *Twist (vs HJ Open):* Call [GTO: Excellent]
*   **Hand 43:** SB | KsQs 
    - *Base (vs CO Open):* 3B 9BB [Plan: RC] [GTO: Excellent]
    - *Twist (vs BTN Open):* 3B 8BB [Plan: RC] [GTO: Excellent]
*   **Hand 44:** UTG | 7h6h 
    - *Base (Folded to):* Fold [GTO: Excellent]
    - *Twist (In HJ, Folded to):* Open 2.2BB [Plan: RF] [GTO: Good]
*   **Hand 45:** BB | QcJc 
    - *Base (vs CO Open):* 3B 9BB [Plan: RC] [GTO: Good]
    - *Twist (vs CO + BTN + SB Squeeze Pot):* 3B Squeeze 13BB [Plan: RC] [GTO: Sub-optimal Plan; should be RF]
*   **Hand 46:** BTN | KsQd 
    - *Base (vs CO Open):* Call [GTO: Excellent]
    - *Twist (In CO, Folded to):* Open 2BB [Plan: RC] [GTO: Excellent]
*   **Hand 47:** BB | Js10s 
    - *Base (vs HJ Open):* 3B 9BB [Plan: RC] [GTO: Good]
    - *Twist (vs HJ + CO + BTN):* 3B Squeeze 13.5BB [Plan: RF] [GTO: Excellent]
*   **Hand 48:** BTN | Ac3c 
    - *Base (vs UTG Open):* Call [GTO: Sub-optimal Flat]
    - *Twist (vs HJ Open):* 3B 8.8BB [Plan: RC] [GTO: Good Sizing / Sub-optimal Plan; should be RF]
*   **Hand 49:** SB | JhJd 
    - *Base (vs CO Open):* 3B 9BB [Plan: RC] [GTO: Excellent]
    - *Twist (vs CO Open + BTN Flat):* 3B Squeeze 10.5BB [Plan: RF] [GTO: Sub-optimal Sizing & Plan; should be >12BB and RC/RP]
*   **Hand 50:** UTG | AcQc 
    - *Base (Folded to):* Open 2.5BB [Plan: RC] [GTO: Excellent]
    - *Twist (In BB vs UTG 2.5BB Open):* Call [GTO: Excellent]

### ⏱️ Session 02 | 2026-10-28 13:17 UTC | Hands 51–60 | Avg score 55
*Backfilled with legacy combined labels (no component scores). Counted in analytics.*
*   **Hand 51:** BTN | 9c8c 
    - *Base (vs HJ 2.2BB Open):* 3B 6.6BB [Plan: RF] [GTO: Sub-optimal Mixed Strategy Error - should be pure flat]
    - *Twist (vs HJ Open + CO Flat):* 3B Squeeze 8.8BB [Plan: RF] [GTO: Sub-optimal Sizing Error - should be ~11BB or Fold]
*   **Hand 52:** SB | Ac3c 
    - *Base (vs CO 2.2BB Open):* Call [GTO: Sub-optimal Flat OOP - should be 3B 9BB or Fold]
    - *Twist (vs BTN 3.5BB Open):* Call [GTO: Blunder - should fold or 3B to 11BB to deny large sizing]
*   **Hand 53:** HJ | QsJs 
    - *Base (UTG Open Pointer):* Open 2.5BB [Plan: RC] [GTO: Excellent]
    - *Twist (vs UTG 2.5BB Open):* 3B 7.5BB [Plan: RC] [GTO: Sub-optimal Sizing & Planning Overplay vs UTG range]
*   **Hand 54:** UTG | JcJd 
    - *Base (Folded to):* Open 2.5BB [Plan: RP] [GTO: Sub-optimal Plan - jamming isolates against better hands; should be RC]
    - *Twist (CO vs HJ 2.2BB Open):* 3B 6.6BB [Plan: RP] [GTO: Sub-optimal Plan - 3-bet is great, but 4-bet response is a Call (RC)]
*   **Hand 55:** CO | AdQs 
    - *Base (vs HJ 2.2BB Open):* 3B 6.6BB [Plan: RC] [GTO: Sub-optimal Plan - unsuited broadways must be RF vs 4-bets]
    - *Twist (vs HJ Open + BTN 3B):* Call [GTO: Major Leak - Cold-calling 3-bets out of position destroys smooth equity]
*   **Hand 56:** BB | KdQd 
    - *Base (vs SB 3BB Open):* 3B 12BB [Plan: RC] [GTO: Sub-optimal Sizing - IP leverage over SB allows smaller 3B to 9BB]
    - *Twist (vs UTG Open + BTN Flat):* Call [GTO: Excellent Multi-way closing protection]
*   **Hand 57:** HJ | 6s5s 
    - *Base (Folded to):* Open 2.2BB [Plan: RF] [GTO: Excellent]
    - *Twist (SB vs CO 2.2BB Open):* Fold [GTO: Excellent OOP fold discipline]
*   **Hand 58:** CO | AdJc 
    - *Base (vs UTG 2.5BB Open):* Fold [GTO: Excellent protective range fold]
    - *Twist (150BB Deep vs Aggressive Blinds):* Open 2BB [Plan: RC] [GTO: Sub-optimal Sizing & Blunder Plan - open bigger, and fold to 3B]
*   **Hand 59:** BTN | KsQs 
    - *Base (vs HJ 2.2BB Open):* 3B 6.6BB [Plan: RC] [GTO: Excellent]
    - *Twist (vs HJ Open + CO Flat + 40BB SB):* 3B 9BB [Plan: RF] [GTO: Sub-optimal Strategy - 40BB stack traps you; should pure flat]
*   **Hand 60:** SB | 10c10d 
    - *Base (vs BTN 2.5BB Open):* 3B 10BB [Plan: RF] [GTO: Planning Error - TT cannot be raise-folded to a Button steal range]
    - *Twist (vs BTN 23BB 4B):* Fold [GTO: Blunder - under-defending premium pairs out of the SB]

---

## 📝 CHANGELOG
- **v2.0 (2026-10-28):** Rebuilt structure. Coach instruction moved from typed replies to the Session / Results JSON flow, compatible with the HTML trainer v2.0. Added game defaults (including the <20BB MTT stack twist), the JSON contracts, the scoring rubric with sample-size rule, the Session Summary, Position and Spot-Type grids, leaks with severity and evidence, the Watchlist, the Re-deal Queue and the Coverage Tracker. Session 01 marked as an unscored pre-trainer archive. The grid was recomputed from Session 02 under the rubric (the old hand-set scores were retired).
- **v1.0:** Original state file.
