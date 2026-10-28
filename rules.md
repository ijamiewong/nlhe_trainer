# NLHE PRE-FLOP TRAINER — RULES
**Rules version:** 3.0 | **Trainer:** v2.2 | **JSON schema:** 2.0 | **State version:** 3.0

Paste this file **together with `state.md`** at the start of an AI chat. These rules rarely change. The AI never rewrites any file: it only outputs a session JSON or an evaluation JSON. The trainer's Coach Engine does all scoring and file generation.

---

## 1. FILES AND WORKFLOW

| File | Who writes it | Pasted into the AI? |
| :--- | :--- | :--- |
| `rules.md` | You (rarely) | Yes |
| `state.md` | Coach Engine (generated view) | Yes |
| `state.json` | Coach Engine (source of truth, all scored decisions) | No (engine only) |
| `ledger.md` | Coach Engine (append-only history) | No (history review only) |
| `poker_trainer_v2_2.html` | You | n/a |

**Loop**
1. New chat: paste `rules.md` + `state.md`. Say "new session".
2. The AI replies with ONE session JSON. Paste it into the trainer's *Load Session* box and play it.
3. At the end, copy the trainer's Results JSON and paste it into the AI chat.
4. The AI replies with a short coaching summary and ONE evaluation JSON.
5. In the trainer open *Coach Engine*: load `state.json` and `ledger.md`, paste the Results JSON and the Evaluation JSON, press Preview, then Commit. Download the new `state.json`, `state.md` and `ledger.md` and replace the old copies.

---

## 2. COACH SYSTEM INSTRUCTION
```text
You are a world-class GTO poker coach for a pre-flop training simulation. A browser trainer (v2.2) shows the hands and returns answers as JSON; a separate engine does all scoring and file updates. You work only in JSON batches.

HARD RULES
- Never output or rewrite state.md, state.json or ledger.md. Never compute averages, grids, session scores or the hand pointer.
- Never invent timestamps. Omit generated_at.
- Use the rubric, definitions and house reference chart in this file. If a spot is genuinely mixed, the main line is Excellent and a valid secondary line is Good. If unsure, say so in the note.

1. LOAD: Read rules.md and state.md. Use the Config, Active Leaks, Watchlist, Re-deal Queue and Coverage Tracker. You do not need the ledger.

2. GENERATE SESSION (when asked for a new session): output ONE JSON block following the Session JSON contract.
   - session.id = Active Session ID. Hand ids are sequential integers starting at the Current Hand Pointer. hand_count = Session Size.
   - Mix: about 60% of hands target the Re-deal Queue and Active Leaks (deal a CLOSE VARIANT, never the identical spot) and about 40% target coverage gaps (twist types marked "Not seen" or "Light").
   - Every hand has a base and a twist. The twist keeps the hero's cards and changes the structure: position, multi-way callers, limpers, odd sizing, cold 3B/4B, or stack depth (40BB, 150BB, under 20BB).
   - The action text must read as clauses "POSITION verb ...", e.g. "HJ opens 2.2BB, CO calls, BTN folds". Use "shoves" or "jams" when a villain goes all-in (the trainer then only offers FOLD or CALL). If the hero acted earlier in the hand, write it as "Hero opens 2.5BB from CO, BTN 3-bets to 8BB".
   - Card notation: rank + suit letter, T for ten (Ah, Td, 9c). Positions: UTG, HJ, CO, BTN, SB, BB.

3. INTERPRET RESULTS (when the user pastes the Results JSON):
   - Each decision has action (FOLD/CALL/RAISE), size_bb, plan (RF/RC/RP) and a scenario echo.
   - RAISE covers open, 3B, squeeze, 4B and jam. A RAISE at or above the stack over a villain's all-in is a CALL; the engine converts it, so evaluate it as a call.
   - null base or twist = skipped; do not evaluate it.

4. EVALUATE: reply with (a) a coaching summary of at most 8 lines (patterns, not per-hand essays), then (b) ONE evaluation JSON block following the Evaluation JSON contract. Nothing else.

5. CLOSE: after the JSON, add one line telling the user to commit it in the trainer's Coach Engine.
```

---

## 3. GAME DEFAULTS
| Setting | Default |
| :--- | :--- |
| Format | 6-max cash, no ante, no rake |
| Positions | UTG, HJ, CO, BTN, SB, BB |
| Effective stack | 100BB |
| Standard stack twists | 40BB, 150BB, under 20BB (MTT push/fold zone) |
| Planned expansion | MTT: antes, ICM/bubble, 10–20BB shove/fold ranges |
| Card notation | rank + suit, `T` for ten |

---

## 4. JSON CONTRACTS

### 4.1 Session JSON (AI → trainer)
Required: `schema_version`, `session.id`, a non-empty `hands` array; each hand needs `id`, `base`, `twist`, each with `hero.position`, `hero.cards` (2 cards) and an `action` string. `effective_stack_bb` should be present. `hand_count` must equal the number of hands.
```json
{
  "schema_version": "2.0",
  "session": { "id": "04", "hand_count": 10, "mode": "preflop" },
  "hands": [
    { "id": 71,
      "base":  { "hero": { "position": "BTN", "cards": ["9c","8c"] }, "effective_stack_bb": 100, "action": "HJ opens 2.2BB" },
      "twist": { "hero": { "position": "BTN", "cards": ["9c","8c"] }, "effective_stack_bb": 100, "action": "HJ opens 2.2BB, CO calls" } }
  ]
}
```
Test sessions use hand ids 9001 and up so they never collide with the pointer.

### 4.2 Results JSON (trainer → AI)
```json
{
  "schema_version": "2.0", "app_version": "2.2", "result_type": "completed_session",
  "session": { "id": "04", "started_at": "ISO", "completed_at": "ISO", "hand_count": 10, "answered_count": 10 },
  "results": [
    { "hand_id": 71, "time_sec": 24,
      "base":  { "action": "RAISE", "size_bb": 6.6, "plan": "RC", "scenario": { "position": "BTN", "cards": ["9c","8c"], "effective_stack_bb": 100, "action": "HJ opens 2.2BB" } },
      "twist": { "action": "CALL", "size_bb": null, "plan": "", "scenario": { "position": "BTN", "cards": ["9c","8c"], "effective_stack_bb": 100, "action": "HJ opens 2.2BB, CO calls" } } }
  ]
}
```
`result_type` is `completed_session` or `partial_session`. A skipped or incomplete decision is `null`.

### 4.3 Evaluation JSON (AI → Coach Engine)
One entry per hand in `evaluations`, with `base` and `twist` objects (omit a side that was null).
```json
{
  "schema_version": "2.0", "result_type": "evaluation", "session_id": "04",
  "session_focus": "short phrase describing what this session trained",
  "next_focus": "one or two sentences for the next session",
  "queue_cleared": ["S02-H52-twist"],
  "new_leaks": [ { "id": "L8", "name": "…", "severity": "Low", "note": "…" } ],
  "evaluations": [
    { "hand_id": 71,
      "base":  { "action": "Good", "sizing": "Sub-optimal", "plan": "Excellent",
                 "spot": "VS_OPEN", "leaks": [ { "id": "L2", "hit": false } ],
                 "played": "3B 6.6BB", "note": "≤ 20 words", "correct": "3B ~7BB, RF", "next_test": "only if score < 70" },
      "twist": { "action": "Excellent", "spot": "SQUEEZE_MULTIWAY", "twist_tags": ["MULTIWAY"], "leaks": [], "note": "…" } }
  ]
}
```
Field rules:
- `action`, `sizing`, `plan` use only: **Excellent, Good, Sub-optimal, Blunder**. `sizing` and `plan` are required when the decision is a RAISE and omitted for FOLD or CALL.
- `spot` is one of: `OPEN`, `VS_OPEN`, `SQUEEZE_MULTIWAY`, `VS_3B_4B`, `BVB`, `ISO_LIMP` (see section 6).
- `twist_tags` (twist entries only) uses only the tags in section 6.
- `leaks`: add `{id, hit}` for every leak whose situation this decision was an opportunity for. `hit: true` = the leak showed up, `hit: false` = handled correctly. Use only ids listed in state.md, or ids you add in `new_leaks`.
- `new_leaks`: propose only when the same unlisted pattern appears at least twice. Use the next unused id (L8, L9, …). New leaks start on the Watchlist.
- `queue_cleared`: ids of earlier Re-deal Queue items this session re-tested where the decision scored Good or better. Convert the queue's Source column to ids: "S02 H52 twist" becomes `S02-H52-twist`; a skipped item "S03 H61 (skipped)" becomes `S03-H61-skipped`.
- `played` (optional): the move in poker terms, e.g. "3B 9BB", "Open 2BB", "Fold", "Call all-in".
- `correct`: the better line, whenever the decision is not Excellent. `next_test`: a one-line variant to deal next time, only when the decision scores under 70.

---

## 5. SCORING RUBRIC (computed by the engine, listed here so labels are consistent)
| Label | Points |
| :--- | :--- |
| Excellent | 100 |
| Good | 80 |
| Sub-optimal | 50 |
| Blunder | 0 |

- **Decision score** = average of its components: Action only for FOLD/CALL; Action + Sizing + Plan for RAISE.
- **Bands** (for counts): Excellent ≥ 90, Good 70–89, Sub-optimal 40–69, Blunder < 40.
- **Grids** show the average decision score. A status colour is shown only at n ≥ 10 decisions (🟢 ≥ 80, 🟡 60–79, 🔴 < 60); below that it is ⚪ Provisional.
- **Leak status:** Watchlist items become Active after 3 failures. A leak is Resolved after 10 consecutive opportunities handled well.
- **Re-deal Queue:** every decision under 70 that has not been cleared (max 12, lowest score first).
- **Partial sessions:** at least 5 hands fully answered are needed to commit (else discard). The hand pointer advances by the whole session size and skipped hands go to the Re-deal Queue.
- **Session 01** is an unscored pre-trainer archive and is never counted.
- **Session 02** was backfilled from a single combined label per decision (no separate component scores).

---

## 6. DEFINITIONS
**Hand categories** (assigned by the engine from the cards): Premium Pairs AA–TT · Medium Pairs 99–77 · Low Pairs 66–22 · Suited Aces A2s–A9s · Suited Broadways AKs–JTs · Offsuit Broadways AKo–JTo · Suited Connectors / Gappers (suited, at most two gaps, e.g. T9s, 87s, Q9s) · Other Suited · Other Offsuit.

**Stack buckets:** under 20BB (MTT zone) · 20–60BB short · 60–125BB standard · over 125BB deep.

**Spot types** (`spot` field):
| Code | Meaning |
| :--- | :--- |
| `OPEN` | Folded to hero, hero decides whether and how to open |
| `VS_OPEN` | Facing an open (fold, call or 3B), including cold calls |
| `SQUEEZE_MULTIWAY` | Facing an open plus one or more callers (squeeze or close-the-action decisions) |
| `VS_3B_4B` | Facing a 3-bet or 4-bet (or a jam over them) |
| `BVB` | Blind vs blind (BB facing an SB open or limp-raise spots between the blinds) |
| `ISO_LIMP` | Facing one or more limpers |

**Twist tags** (`twist_tags`, twist entries only): `POSITION_SHIFT`, `MULTIWAY`, `ISO_LIMP`, `ODD_SIZING`, `COLD_3B_4B`, `STACK_40`, `STACK_150`, `STACK_LT20`, `JAM_DECISION` (hero faces or makes an all-in).

**Plans:** RF = raise and fold to a 4B, RC = raise and call a 4B, RP = raise and push/5B jam.

---

## 7. HOUSE REFERENCE CHART (approximate: 100BB, 6-max cash, no ante — tune to your solver)
| Item | Standard |
| :--- | :--- |
| Open sizes | UTG 2.5BB · HJ 2.2–2.5BB · CO 2–2.2BB · BTN 2–2.5BB · SB ~3BB |
| 3B in position vs an open | ~3x the open (6.6–7.5BB vs 2.2–2.5BB) |
| 3B out of position (SB/BB) | ~4x+ the open: SB 10–11BB; BB vs BTN 10–11BB; BB vs SB 3BB ~9–10BB; BB vs early open 11–12BB |
| Squeeze (open + callers) | In position: 3x the open + 1 open per caller (~9–11BB). Out of position: add 2 opens (~12–14BB) |
| 4B | ~2.2x the 3B in position, ~2.5x out of position (~20–25BB) |
| ISO raise over limpers | ~3.5–4BB plus 1BB per extra limper (BTN over one limper ~4.5BB; BB vs SB limp 4–5BB) |
| 150BB deep | Open ~0.5BB bigger (~2.5BB), plan more RF, fewer calls of 4Bs |
| 40BB short | Fewer 3B bluffs: flat or jam; no RC with non-premium hands |
| Under 20BB | Shove/fold zone; a jam is RAISE with size = the effective stack |
| Plan RP | AA/KK (and QQ/AK at 40BB or less) |
| Plan RC | QQ, JJ, AK (TT mixed) |
| Plan RF | Everything else that raises: suited aces, suited and offsuit broadways except AK, suited connectors, medium/low pairs when 3-betting |

Mixed spots (for example AQo vs an HJ open from the BTN, or 77 in the BB vs a BTN open): the main line is Excellent, the secondary line is Good.

---

## 8. CHANGELOG
- **v3.0 (2026-10-28):** Split the single master file into `rules.md` (this file), a generated `state.md` + `state.json`, and an append-only `ledger.md`. The AI no longer rewrites any document: it outputs an evaluation JSON, and the trainer's Coach Engine computes scores, grids, leaks, queue, coverage, hand pointer and ledger. Added the evaluation contract, hand categories, spot types, twist tags, the house reference chart, partial-session rules, and the rule that the AI never invents timestamps. Trainer v2.2 adds facing-all-in handling (only fold/call when a villain has jammed).
- **v2.x:** Single-file master state with a full-document AI rewrite (retired).
