# Pantry Steward · 冰箱管家 — Business Specification

**Platform-independent.** This document specifies *what the product does*, not how to build it.
No framework, storage format, host API or language is assumed: the same rules can be implemented
in Java + Vue, ArkTS, a native desktop stack, or anything else.

**Scope.** Everything a fresh implementer needs to rebuild the product from zero: domain model,
business rules, derivations, validations, screen surfaces and acceptance criteria. UI wording is
part of the business (the product is Chinese-facing), so canonical Chinese labels are given as
`` `term` ``; identifiers elsewhere are English.

**Terminology** follows `GLOSSARY.md`; the hard-to-reverse decisions are in `docs/adr/`.
Those files are the domain authority — read them before renaming anything.

## How to read this document

- **MUST / MUST NOT** are binding rules; violating one is a defect.
- **Derived** values are never stored — recompute them (ADR-0003).
- **Snapshot** values are copied at confirmation time and never re-derived (ADR-0001, ADR-0002).
- Facts about the original implementation are marked *(observed)*; they document the shipped
  behaviour and should be reproduced unless an ADR says otherwise.
- Open questions are collected in §12.

---

## 1. Product summary

A local-first pantry and meal-planning butler. The user records a dietary profile and the real
contents of their fridge, chooses a health cycle, and confirms a long-term plan. The app then
generates a one-day, three-meal menu (早/中/晚) from **stock that already exists**, lets the user
review and accept it, derives a **缺菜清单** (shopping-gap list) from whatever stock is missing,
offers a **simulated** grocery flow to top the gaps up, and finally records what was actually
eaten — deducting stock and advancing progress only on that explicit confirmation.

Three principles drive every rule below:

1. **Recommendation ≠ execution.** Generating, editing or accepting a menu changes no stock, no
   intake and no progress. Only a confirmed actual meal does.
2. **One source of truth per fact.** Stages and execution records are immutable snapshots;
   the shopping-gap list is always recomputed, never stored.
3. **Honesty over polish.** Real intake is recorded even when it exceeds targets; unknown
   nutrition stays unknown and is never counted as zero; no feature claims to guarantee
   allergy safety or nutritional accuracy.

### Actors

| Actor | Description |
| --- | --- |
| **User** | The single local person. Owns profile, fridge, plans and history. |
| **Butler app** | The product. Suggests menus from stock, derives gaps, records facts. |
| **Online model** | Optional external provider that proposes menus and estimates missing nutrition. Never authoritative; every output is validated locally and never writes stock directly. |

### Navigation surfaces *(observed)*

Six always-present tabs: `首页` (0), `总览` (1), `冰箱` (2), `菜单` (3), `档案` (4), `设置` (5).
A separate in-app `模拟买菜` region (§7) replaces the tabs while open and has its own
`商品 / 购物车 / 订单` navigation plus `← 返回管家`.

---

## 2. Domain model

### 2.1 Entities

| Entity | Nature | Key fields (business-level) |
| --- | --- | --- |
| **Profile** | Mutable, versioned | scene, sex, age, height, weight, allergies, avoidances, preferences, special, activity, done, plan_approved |
| **Batch** (inventory item) | Immutable-fields snapshot, created once | id, name, quantity, unit, package_size, package_size_unit, average_item_grams, expires_on_day, expiry_precision, source, nutrition?, canonical_quantity, canonical_unit |
| **Cycle candidate** (`program`) | Recomputable offer, unconfirmed | id, name, days, variant, goal, trait, difference, daily_kcal, kcal_goal, protein_goal, fiber_goal, vitamin_c_goal |
| **Plan draft** | Ordered list of candidate ids, unconfirmed | — |
| **Stage** | Immutable snapshot of a confirmed candidate | id, started, days, kind (`health`/`experience`), targets (snapshotted), consumed totals, intake[] |
| **Day menu** | Mutable until consumed | id, day, title, meals[3], uses[], accepted, consumed, stage_id, revision, profile_revision, created, reason, goal, source, numeric, previous |
| **Meal** | Part of a day menu | slot (早/中/晚), title, minutes, uses[], steps |
| **Use** | One ingredient amount in a meal | food_id, name, quantity, unit, grams |
| **Shopping-gap row** (缺菜清单) | **Derived, never stored** | name, required, available, short |
| **Cart line** | User input, persisted | name, product, quantity, dropped |
| **Mock order** | Confirmed cart record | id, day, status (`待收货`/`已收货`), total_cents, lines[] |
| **Execution record** | Append-only, immutable snapshot | id, program_id, slot, program snapshot, at, day, title, recommendation snapshot, actual[], kcal, protein, fiber, vitamin_c, extra_kcal, nutrition_complete, nutrition_unknown |
| **Nutrition reference** | Per 100 g; provenance-tagged | kcal, protein, fiber, vitamin_c, source, basis?, estimated_at?, model? |
| **Inventory event** | Idempotency ledger | operation_id, type (`add`/`edit`/`discard`/`consume`), batch_ids[] |
| **App log** | Rolling, ≤24 entries | at, text |

### 2.2 Canonical terms

| Term | Meaning | Must not be confused with |
| --- | --- | --- |
| `健康方向` (health direction / scene) | A preference chosen at onboarding. **Not executable.** | A plan |
| `周期` (cycle) | A candidate duration in days. | A stage |
| `方案` (candidate) | An unconfirmed offer for a cycle, carrying that cycle's targets. | A stage |
| `阶段` (stage) | A confirmed candidate, holding a date range. Locks on its start date. | A cycle |
| `计划` (plan) | The ordered stage sequence; total days = sum of stages. | A candidate |
| `日菜单` (day menu) | One day's three meals. The unit the gap list is computed over. | A plan |
| `缺菜清单` | Derived `required − available` per ingredient. | A stored purchase list |
| `模拟商品` | Demo catalog item. **Not** the user's inventory. | A batch |
| `模拟订单` | Demo order record. No payment, no stock change. | A purchase |
| `确认收货` | Checking a mock order in as new stock batches. | A purchase or a meal |
| `营养参考` | Per-100 g values with provenance. | A measurement or a guarantee |
| `实际执行记录` | Confirmed actual eating. The **only** basis for progress and stock deduction. | A menu |
| `进度` (progress) | Confirmed meals ÷ planned meals. | Nutrition-goal attainment (separate) |

### 2.3 Structural limits

| Collection | Max |
| --- | --- |
| batches | 60 |
| day menus in memory | most recent **20 distinct days**, then ≤20 menus |
| stages in one plan | 12 |
| archived plans | 12 |
| cart lines | 80 |
| mock orders | 40 |
| execution records | 10000 |
| uses in one menu | 60 |
| uses in one meal | 8 (model may return ≤4) |
| intake snapshots per stage | 150 |
| app log | 24 (oldest dropped) |
| profile free-text fields | see §4.3 |

---

## 3. Onboarding and planning

### 3.1 Onboarding steps

| Step | Screen | Exit condition |
| --- | --- | --- |
| 0 | Choose `健康方向` | A scene is selected |
| 1 | Basics + dietary restrictions | Valid basics saved (`save_profile`) |
| 2 | Cycle candidates + plan composition | A plan is confirmed (`confirm_health_plan`) |
| 3 | First inventory entry | ≥1 batch exists (`finish_onboarding`) |
| 4 | Normal app (six tabs) | — |

Saving basics **regenerates all candidates** and resets the plan draft. Confirming a plan sets
`plan_approved`, starts the health window, and moves to step 3. Step 3 refuses to finish with zero
batches. Only after step 4 does the reminder checker run.

### 3.2 Profile fields and validation

| Field | Rule |
| --- | --- |
| `scene` | One of `减脂控卡`, `日常健康`, `健身增肌`, `轻负担饮食` |
| `sex` | `男性` or `女性` |
| `age` | Integer 12–100 (numeric-target computation additionally requires 19–78) |
| `height` | Integer 80–230 cm |
| `weight` | Number 25–300 kg |
| `allergies` / `avoidances` / `preferences` | Free text, trimmed |
| `special` | Free text (special conditions) |
| `activity` | `轻活动` (default), `中活动`, `高活动` |

**Numeric-target gate.** A candidate can carry numbers only when the profile is
*ready*: `special` empty, age 19–78, height 80–230, weight 25–300, sex known, **and** the computed
base intake is ≥ 1350 kcal (§3.3). Otherwise candidates carry zero targets, display the reason
`当前档案不能自动推算安全的数值目标；请由专业人员确认后再使用数值追踪。`, and **cannot be
confirmed**. Faking profile data must not be a way around this gate.

**Profile editing.** `重新设置用户画像` backs up the current state, archives the plan, clears
candidates and returns to step 0. **Nothing is saved until a new plan is confirmed**; cancelling
restores the backup exactly (profile, plan, stock). Editing is refused while a confirmation,
generation, removal or recognition is in flight.

### 3.3 Target mathematics *(observed)*

Let `w`=weight kg, `h`=height cm, `a`=age.

```
resting = 10w + 6.25h − 5a − 161        (sex = 女性)
resting = 10w + 6.25h − 5a + 5          (sex = 男性)
activity = 1.2 | 1.45 | 1.65            (轻活动 | 中活动 | 高活动)
sceneFactor = 0.93 | 1.00 | 1.05 | 0.96 (减脂控卡 | 日常健康 | 健身增肌 | 健身减脂)
base = floor( max(resting, resting × activity × sceneFactor) / 50 ) × 50
```

`base < 1350` ⇒ no numeric targets (see gate above).

Per-variant targets (`base` = the value above):

```
perKg     = 0.8   ; or 1.2 when scene ∈ {健身增肌, 健身减脂}
fiberRate = 0.014
kcal      = base

variant 平衡 (balance):  unchanged
variant 高蛋白 (protein): perKg   += 0.4
variant 高纤  (fiber):   fiberRate = 0.019
variant 加餐/控卡 (trim): perKg   += 0.1 ; fiberRate = 0.017
                         kcal = base + 150 if 健身增肌 else base − 150
                         clamp kcal to ≥ 1200          // prototype floor

protein = floor(w × perKg)
fiber   = floor(kcal × fiberRate)
vitaminC = 90 if 男性 else 75
```

A candidate's cycle totals are `daily × days` for kcal, protein, fiber, vitamin C.

Weekly health goals set at profile save (separate from candidate targets):

| Goal | Value |
| --- | --- |
| protein (per week) | `floor(weight × 0.8) × 7` g |
| fiber (per week) | 175 g (female) / 266 g (male) |
| calcium (per week) | 7000 mg |
| iron (per week) | 0; 126 mg if female ≤ 50; 56 mg if male |
| all four | set to **0** if age < 19, age > 50, or `special` non-empty |

### 3.4 Cycle candidates

Six cycles: **1, 3, 7, 15, 21, 30** days. Each cycle holds exactly three candidates.

- Candidate 1 is always `平衡` (baseline).
- Candidates 2 and 3 are drawn from `{高蛋白, 高纤, 加餐/控卡}` according to that cycle's seed:
  `second = 1 + (seed mod 3)`, `third = 1 + ((seed + 1) mod 3)` (1=protein, 2=fiber, 3=trim).
- Every candidate shows: goal text, execution trait, and **its difference from the baseline**
  (e.g. "蛋白质比「起步」多 N g/天，热量相同").
- **Regenerate this cycle's candidates** increments only that cycle's seed, replacing only that
  cycle's unconfirmed candidates. Confirmed stages and other cycles are untouched. Regenerating is
  refused while a confirmation or generation is running.
- The **1-day health cycle** and the optional **1-day experience** (§8) are separate entries.

### 3.5 Plan composition and confirmation

- A plan is an ordered list of candidate ids (repeats allowed, e.g. 7+7). Max 12 stages.
- The draft shows total days = Σ stage days, plus each stage's date range and targets.
- Draft operations: append, move up/down, remove, replace with another candidate.
- **Confirmation** (`confirm_health_plan`): every candidate must have `daily_kcal > 0`; each becomes
  a **stage snapshot** whose targets are copied (ADR-0001), starting where the previous stage
  ended, so ranges meet end-to-start and start today.
- The optional experience choice turns the single confirmed stage into `kind = experience` named
  `1 天轻食体验`.
- If the draft is empty, the currently selected candidate is used.
- A failed save restores the previous state but **keeps the composed draft** so work is not lost.

### 3.6 Stage locking and future-stage editing

- A stage is **locked** from the calendar day its start date arrives:
  `floor(started / 86400) ≤ today`. A locked stage cannot be moved, replaced, removed or
  retargeted, even if that day's meals are not yet confirmed; the UI must say why.
- Stages that have **not started** can be reordered, replaced with another candidate, removed, or
  retargeted. Every such edit recalculates later stages' date ranges and the plan totals.
- Confirmed progress and intake records are never affected by editing future stages
  (this is the reason stages are snapshots).
- **Current stage** = the first stage whose `[start, start+days)` covers today; if none covers
  today, the last stage. A plan is *expired* once the calendar passes the last stage's end.
  Targets always follow the stage whose dates cover today.

### 3.7 Progress and overview

- `meals_per_day = 3`.
- Stage progress `% = floor(confirmed meals in stage ÷ (days × 3) × 100)`, capped at 100.
- Plan progress = Σ confirmed ÷ Σ planned across stages.
- **Only a confirmed actual meal increments progress.** Generating, editing or accepting a menu
  never does.
- The `总览` tab shows: the long-term plan, the current stage, today's per-meal progress, and the
  current stage's kcal/protein target against actuals.
- Nutrition-goal attainment is reported **separately** from meal progress. Three confirmed meals
  do not imply the nutrition target was met.
- Archiving a plan (`选择下一份计划…`) appends the live stages to plan history (bounded to 12) and
  clears them. **Stock is not restored and history is not erased.** Long-term tracking reads the
  independent execution-record table.

---

## 4. Inventory

### 4.1 Batch creation and normalisation

A batch is created by manual entry, image recognition, or `确认收货`. It must survive this
normalisation unchanged, or be rejected with a user-readable reason:

| Field | Rule |
| --- | --- |
| `name` | 1–40 characters after trim |
| `quantity` | > 0 and ≤ 100000; integer when the unit counts pieces |
| `unit` | Convertible units: `g`,`kg`,`ml`,`L`,`个`. Package units: `盒`,`袋`,`瓶` |
| `package_size` | Required (> 0) with a package unit; must be **absent** (0/empty) with a convertible unit |
| `package_size_unit` | With a package unit: `g`, `ml`, or `piece` (integer when `piece`). With a convertible unit: must be empty |
| `average_item_grams` | > 0 allowed **only** when the canonical unit is `piece`; ≤ 100000 |
| `expires_on_day` | `0` = unknown (requires precision `unknown`); ≥ 0 otherwise (requires `exact` or `estimated`), ≤ 1000000 |
| `source` | One of `manual`, `receipt_image`, `food_image`, `legacy` |
| `nutrition` | Optional. If present it must be complete and valid (§4.3) |
| canonical amount | ≤ 100000000 |

**Canonical form.** Convertible units map to `g`, `ml`, or `piece` (`kg`→`g` ×1000, `L`→`ml` ×1000).
A package unit maps to its `package_size_unit` and `canonical_quantity = quantity × package_size`.
The canonical form is what all planning arithmetic uses. **Unknown/absent conversion must never be
presented as grams** — a volume (`ml`) batch is not gram-convertible, and a package batch without a
size has no gram form.

**Gram equivalent** (`batch_grams`): `g` ⇒ `canonical_quantity`; `piece` ⇒
`canonical_quantity × average_item_grams`; anything else ⇒ 0 (not gram-convertible).

### 4.2 Inventory operations

All writes go through one idempotent operation carrying an `operation_id`. Replaying the same id
is a no-op that succeeds. Operations:

| Type | Effect |
| --- | --- |
| `add` | Append 1..n new batches with fresh ids. Refused if it would exceed 60 batches. |
| `edit` | Re-normalise one existing batch in place, preserving its source and any image reference. |
| `discard` | Remove one batch. **Not** counted as eaten. |
| `consume` | Deduct a list of `(batch, quantity, unit)` from stock. |

`consume` rules (all must hold per line): the batch exists; quantity > 0 and ≤ that batch's
canonical quantity; the unit matches the batch's canonical unit; the batch is not expired;
piece quantities are integers; **no batch appears twice in one operation**. A batch reduced to zero
is removed and, if it was `临期`, increments `handled`. Otherwise the canonical quantity (and its
display quantity via the original unit factor) is reduced.

Every successful operation increments the state revision, clears the risk key, appends an
inventory event, writes a log line, and re-validates the whole state before committing.

**Atomicity.** Deduction, the execution record, and the stage's intake/consumed totals are written
**in one save**. On failure the pre-operation state is restored and the user is told the operation
did not take effect. A batch count that fails final validation rolls back entirely.

### 4.3 Nutrition reference

Per 100 g: `kcal` > 0 and ≤ 900; `protein` 0–100; `fiber` 0–100; `vitamin_c` 0–1000.

`source` is one of `用户填写`, `包装数据`, `AI 估算`. An estimate additionally carries `basis`
(assumptions such as raw/cooked or variety), `estimated_at`, and `model`. **Priority:** user-entered
and package data beat an AI estimate. See §9.

### 4.4 Freshness and reminders

| State | Condition |
| --- | --- |
| `保质期未知` | expiry unknown |
| `已过期` | expiry day < today |
| `临期` | 0 ≤ days remaining ≤ 3 |
| `正常` | otherwise |

- `临期清零` progress: `handled ÷ max(baseline, current 临期 count + handled)`, capped at 100.
  `baseline` is raised whenever stock is added; the window resets on a 7-day cycle.
- A periodic check (about once a minute, only in the normal app state, and only when no
  confirmation/generation/recognition is in flight) looks for batches expiring within **24 hours**
  that the currently active menu does not cover, or an active menu that has become invalid.
  It notifies once per distinct risk set (`risk_key`), unless reminders are off or snoozed.
  `稍后提醒` snoozes for 30 minutes.
- Reminders never create a menu by themselves when automatic generation is off; they prompt the
  user to generate manually.
- `fast_forward` (demo only) advances the app's own clock by 12 hours and re-checks.

### 4.5 Eligibility

| Helper | Eligible when |
| --- | --- |
| `plan_eligible` (gram planning) | not expired **and** `batch_grams > 0` **and** not excluded by allergy/avoidance |
| `recipe_eligible` (native units) | not expired **and** `canonical_quantity > 0` **and** canonical unit ∈ {`g`,`ml`,`piece`} **and** not excluded |

**Exclusion matching.** The free-text allergy/avoidance fields are split on `，` `、` `；` `;`
`,` into terms; a batch is excluded when any non-empty term is a substring of the batch name **or**
the batch name is a substring of the term (either direction).

**Ordering.** Expiring batches first, earliest expiry first; unknown expiry sorts last.

---

## 5. The day menu

### 5.1 Shape

One menu = one day = exactly three meals in fixed order `早`, `中`, `晚`. The menu keeps a merged
`uses` list (all meals flattened) so stock, budget and deduction have one place to look, plus a
per-meal `uses` list for display and editing.

Generation sources, shown to the user:
- `本地规则 · 三餐日菜单` — local rule path, numeric.
- `本地规则 · 营养未核算` — local rule path where nutrition references are missing (`numeric = false`).
- `在线 AI` — model-generated, numeric.
- `在线 AI · 营养未核算` — model-generated with insufficient references (`numeric = false`).
- `明确标记的演示菜单` — demo data.

### 5.2 Generation — local rule path

1. Refuse if paused, or if there is no current stage / the day's budget is exhausted.
2. Pick up to **6** batches, earliest expiry first, never the same batch twice, only batches with a
   known nutrition reference. Score = `expiry_order × 10`, reduced by 8 / 5 / 5 when the item's
   protein ≥ 8 g, fiber ≥ 2 g, vitamin C ≥ 10 mg respectively **and** the stage still has that
   gap. Lowest score wins.
3. If **no** batch has a known reference, fall back to `recipe_eligible` batches ordered by
   "has reference first, then earliest expiry", build the same three-meal shape with
   `numeric = false`, and tell the user it cannot be accepted yet.
4. If nothing is eligible at all, say why: expired, excluded, piece batches without quantities, or
   package batches without a per-package amount.
5. Distribute picks across the three meals round-robin. **Every picked batch is used.** A batch used
   more than once is split across the meals it lands in, so the merged usage never exceeds one
   batch.
6. Measure the merged nutrition. If it is complete and exceeds the day's remaining budget, scale
   every use down by `remaining ÷ estimate` (never below 1 unit) so the three meals share one
   budget.
7. Save the menu; if it still cannot be accepted, say why.

Meal titles in the rule path are the joined ingredient names plus the meal label; steps are the
generic three-step instructions. Meals must never be empty.

### 5.3 Generation — model path

The model receives: the user goal; the profile (scene, basics, allergies, avoidances, preferences);
the eligible inventory with ids, native quantities/units, gram estimates, remaining days and
per-100 g references; the current stage, its remaining daily kcal and remaining cycle nutrition;
the previous menu title; the trigger reason; and a demo flag. Volume or non-converted package
batches are **never** sent as grams. Body data is not sent with nutrition-estimation requests (§9).

The model must return: `feasible` (bool), `reason` (non-empty, ≤ 800 chars) and exactly three meals;
each meal has a title from the fixed cooking-method enumeration, 1–120 minutes, 1–4 ingredients,
and 2–8 structured steps. Every cited ingredient id must come from the offered inventory, must be
within that batch's available amount, and repeated use across meals counts against the total.
Steps must reference only ids used by that meal; free-text ingredients are rejected. When
`feasible` is false, the reason is shown and no menu is created.

**Local validation is mandatory** before a model menu becomes a candidate: structure, ingredient
ids and amounts against stock, allergy/avoidance compliance, nutrition completeness, and the day's
kcal budget. On failure the app feeds the **specific** error back to the model for **one**
automatic correction, then stops and leaves retry to the user. A model failure never falls back to
a local menu in place of a user-requested one.

**Stale results.** A model reply is discarded if, while waiting, the stock/goal/stage/profile
revision, state revision, time offset or paused/AI-switch state changed, or the menu was removed.
The user is told the old result was dropped and must regenerate.

### 5.4 Menu validity

`menu_issue` returns the first blocking reason, in this order:

1. menu missing → `这份菜单不存在。`
2. already eaten → `这份菜单已经记录过用餐。`
3. no ingredients → `这份菜单没有食材。`
4. menu's stage ≠ current stage → `当前阶段已变化，请重新生成菜单。`
5. any ingredient violates allergy/avoidance → `「<name>」违反过敏或忌口，不能接受。`
6. any ingredient lacks a complete nutrition reference → `有食材缺少每 100 g 营养参考，无法核算，不能接受。`
7. estimated kcal ≤ 0 → `这份菜单没有可核算的热量，不能接受。`
8. no current stage, or the day's budget is exhausted → `当前阶段未启用、已结束，或今天额度已用完。`
9. kcal > remaining daily budget → `整份菜单约 N kcal，超过今天剩余 M kcal。`

**Stock shortage does not block acceptance** (ADR-0002). Insufficient stock is a per-ingredient
status (`库存够` / `还差 N 克`) that the gap list carries.

Separately, **deduction readiness** requires the menu to be accepted **and** every use to be
coverable from eligible batches by the allocation below. Adding stock must never invalidate an
already accepted menu; a changed stage still does.

### 5.5 Allocation (which batch is spent)

Cover the day's uses from eligible batches held in memory:

- The menu's own batch is spent first, so an explicit "use batch N of the same ingredient" choice
  still decides where the deduction lands.
- Then the earliest-expiry batch of the same name and unit.
- A use may span several batches; **two uses never spend the same grams**.
- A piece use must fit inside one batch, because the average item weight is a per-batch number.
- Draws are coalesced by batch id before deduction, because one operation may not list a batch twice.

Nothing here modifies stock — allocation is a dry run until confirmation.

### 5.6 Menu editing

| Action | Rule |
| --- | --- |
| Change a meal's portion | Recompute proportionally; the result must still pass `menu_issue`, otherwise roll back with the reason. |
| Swap an ingredient | Cycle through other eligible batches; the result must still pass `menu_issue`, otherwise roll back. |
| Accept `接受今日菜单` | Requires `menu_issue` to be empty. Sets `accepted` and becomes the active menu. Message: an accepted menu whose stock is short becomes executable once the missing batches arrive — no regeneration. |
| Remove the menu | Requires explicit confirmation; deleting a menu does not restore stock and does not erase meal records. |

New menus are offered while older ones exist; the newest unconfirmed valid menu is the display
candidate. Only the most recent 20 distinct days are retained.

---

## 6. 缺菜清单 (derived shopping-gap list)

### 6.1 Derivation

Per merged ingredient name in the day menu:

```
required  = Σ use_grams(name)                       // merged across meals
available = Σ batch_grams(f) over plan_eligible batches with that name
short     = ceil( max(0, required − available) )
```

Rows with `short ≤ 0` are omitted. The list is **recomputed every time**; nothing is stored
(ADR-0003). Only two things can change a gap: the day menu's required amount, or stock availability.
Because a menu may only cite stock that existed at generation time, the list can never contain an
ingredient the menu never used — it is the *drift* between an already-generated menu and later stock
changes (ADR-0005).

### 6.2 Editing a row

The number the user types is **how much to top up**, so the write target is
`required := available + typed`, applied back onto the day menu.

- The typed value must be ≥ 0 (empty is rejected with `需要补齐的数量请填写 0 或更大的数字。`).
- Reallocation: if the ingredient appears in more meals than the target can cover, the surplus meals
  **lose** the ingredient rather than keeping a 0 g entry; otherwise the target is split
  proportionally across those meals, with the rounding remainder on the last one. Piece ingredients
  round to whole pieces using the batch's average item weight.
- The edit is refused (rolled back) if it would leave a meal with no ingredients, or if the
  resulting menu fails `menu_issue`.
- `不需要` (drop) is the same write with `required := available`, so the gap becomes zero without
  the list storing a number.

The list never records "already bought" or "received" — partial receipt re-derives automatically.

---

## 7. 模拟买菜 (simulated grocery region)

An in-app region inside the same window (no second app, no network capability). **Nothing is bought,
paid for, or added to stock by browsing, carting or ordering.** Every page is labelled
`模拟平台` / `模拟下单`.

### 7.1 Entry

`一键补齐` is enabled only **after the day menu is accepted**. Before acceptance the panel shows a
hint only: no cart is created and no navigation happens. Entering the region carries the current
derived gap list, marking each line `已对应模拟商品` or `无对应模拟商品`.

### 7.2 Catalog

Fixed demo data shipped with the product: **24 products**, no network, no real prices. Each product
has: name, category, spec (`<amount> <measure> / <pack unit>`), unit, price, on-sale quantity,
per-100 g nutrition reference, reference shelf life in days, and an emoji. One common ingredient
(e.g. 秋葵) is deliberately **absent** so the "no matching product" path is reachable.

| name | category | spec | unit | price | stock | kcal | protein | fiber | vitC | shelf days |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 菠菜 | 蔬菜 | 300 g / 把 | 把 | 4.80 | 20 | 23 | 3 | 2.2 | 28 | 3 |
| 青菜 | 蔬菜 | 400 g / 把 | 把 | 4.20 | 15 | 15 | 1.5 | 1.5 | 45 | 3 |
| 生菜 | 蔬菜 | 250 g / 把 | 把 | 3.90 | 10 | 15 | 1.4 | 1.3 | 8 | 4 |
| 番茄 | 蔬菜 | 500 g / 份 | 份 | 6.90 | 18 | 18 | 0.9 | 1.2 | 14 | 6 |
| 胡萝卜 | 蔬菜 | 500 g / 袋 | 袋 | 3.50 | 22 | 39 | 1 | 2.8 | 5.9 | 14 |
| 土豆 | 蔬菜 | 1 kg / 袋 | 袋 | 5.90 | 25 | 77 | 2 | 2.2 | 19 | 20 |
| 西兰花 | 蔬菜 | 400 g / 个 | 个 | 7.80 | 12 | 34 | 2.8 | 2.6 | 89 | 5 |
| 黄瓜 | 蔬菜 | 500 g / 份 | 份 | 4.90 | 16 | 16 | 0.8 | 0.5 | 2.8 | 5 |
| 蘑菇 | 蔬菜 | 200 g / 盒 | 盒 | 6.20 | 14 | 22 | 3.1 | 1 | 2 | 5 |
| 苹果 | 水果 | 1 kg / 袋 | 袋 | 12.90 | 30 | 53 | 0.4 | 2.4 | 4.6 | 20 |
| 香蕉 | 水果 | 1 kg / 把 | 把 | 9.90 | 20 | 93 | 1.4 | 1.2 | 8.7 | 5 |
| 橙子 | 水果 | 1 kg / 袋 | 袋 | 11.90 | 18 | 48 | 0.9 | 2.4 | 53 | 15 |
| 鸡蛋 | 肉蛋 | 10 枚 / 盒 | 盒 | 10.80 | 26 | 143 | 13 | 0 | 0 | 30 |
| 鸡胸肉 | 肉蛋 | 500 g / 份 | 份 | 15.90 | 12 | 133 | 24 | 0 | 0 | 3 |
| 猪里脊 | 肉蛋 | 500 g / 份 | 份 | 18.90 | 9 | 155 | 20 | 0 | 0 | 3 |
| 牛肉 | 肉蛋 | 500 g / 份 | 份 | 35.90 | 6 | 125 | 20 | 0 | 0 | 3 |
| 三文鱼 | 水产 | 300 g / 份 | 份 | 42.90 | 5 | 208 | 20 | 0 | 0 | 2 |
| 豆腐 | 豆制品 | 400 g / 盒 | 盒 | 4.80 | 20 | 76 | 8 | 1 | 0 | 5 |
| 豆浆 | 豆制品 | 1 L / 瓶 | 瓶 | 8.90 | 24 | 31 | 3 | 0.6 | 0 | 7 |
| 牛奶 | 乳品 | 1 L / 盒 | 盒 | 12.90 | 28 | 54 | 3.3 | 0 | 1 | 10 |
| 酸奶 | 乳品 | 200 g / 杯 | 杯 | 6.90 | 22 | 72 | 3.2 | 0 | 1 | 21 |
| 大米 | 主食 | 5 kg / 袋 | 袋 | 39.90 | 40 | 346 | 7.4 | 0.7 | 0 | 365 |
| 全麦面包 | 主食 | 400 g / 袋 | 袋 | 11.90 | 12 | 246 | 9 | 6 | 0 | 7 |
| 燕麦片 | 主食 | 1 kg / 袋 | 袋 | 19.90 | 16 | 367 | 15 | 10 | 0 | 180 |

Only `g`/`kg` specs yield a gram measure; every other spec yields no gram conversion.

### 7.3 Cart

- **Auto-fill on entering** adds one line per current gap row, **only for names never seen before**.
  User quantities, replacements and deletions are never overwritten.
- A line = gap ingredient name + chosen product + quantity + `dropped` flag.
- Quantity is counted in **packs**: `max(1, ceil(short ÷ spec_grams))`; products with no gram spec
  default to one pack. Manual quantity: 1–99.
- Flags: `无匹配` when no product matches (nothing to order), `可售不足` when quantity > on-sale
  quantity. An unmatched or over-stock line is **flagged, not dropped**, and offers `替换`.
- `替换` offers alternatives ordered by: same category first (matched by a keyword hint over the
  ingredient name), then spec closest to the gap; allergies and avoidances are excluded.
- Cart contents are **user input** and must persist across restarts; unit prices and a total are
  shown.

### 7.4 Order

- Only lines with a chosen product can be ordered; the ordered quantity is
  `min(cart quantity, on-sale quantity)`, discarding zero lines.
- `模拟订单` is written **only** after explicit user confirmation. Submitting writes the order and
  **nothing else**: stock, progress and execution records are unchanged. The order survives restarts.
- Order lines: name, product, quantity, unit, unit price. Total = Σ quantity × price.

### 7.5 确认收货 (receive)

- Available on a `待收货` order. Per line, the received quantity defaults to the ordered amount and
  **can only be reduced** (≥ 0); the shelf life defaults to the product's reference days counted
  from today and is editable (integer 0–3650).
- Only lines with `received > 0` become **new** batches — each with a new id and its own arrival
  date, never merged into an existing batch — so the `冰箱` tab shows them immediately.
- Batch form from the product spec: `g`/`kg`/`ml`/`L` ⇒ `quantity = amount × received`,
  `unit = measure`; `枚`/`个` with a package unit ⇒ `quantity = received`, `unit = product unit`,
  `package_size = amount`, `package_size_unit = piece`. Any other spec makes the line
  unconfirmable.
- On success the order becomes `已收货`, the checked cart lines are removed, and the unchecked
  remainder **re-derives into the gap list** — no separate "not received" bookkeeping.
- `确认收货` writes **no execution record** and does **not** move progress. Buying ≠ eating.
- An invalid quantity or shelf life blocks the whole confirmation with a per-line reason.

### 7.6 Demo boundary

The whole region is explicitly demo: catalog, prices, stock quantities, orders and receiving are
illustrative. The catalog's nutrition references **never** feed a menu (ADR-0005).

---

## 8. Optional 1-day experience flow

An optional entry on the plan page: `先体验 1 天轻食流程`, confirmed as a single
`kind = experience` stage. It is separate from the ordinary 1-day health cycle.

- **Stock only.** No 补菜, no simulated restocking, no purchase inside the experience.
- Generation produces the **unconfirmed** meals only (already-confirmed slots are preserved and
  returned in the input as `confirmed`), always three slots 早/中/晚. Reserving is cumulative: the
  three unconfirmed meals together may not exceed each batch's stock, and their total kcal may not
  exceed the day's remaining budget.
- Each meal is confirmed individually through the normal actual-consumption flow (§10).
- Progress = confirmed slots ÷ 3. At 100% the app shows a recap and offers follow-on plans.
- Repeated confirmation of the same slot is refused.
- Missing nutrition references are estimated online with provenance (§9). If estimation fails, the
  reference stays unknown and one menu request still proceeds.
- On upgrade, old unconfirmed gap/free-text menus are cleared; already-confirmed recommendations and
  execution snapshots are kept.

---

## 9. Nutrition estimation

- Missing references are estimated by the online model and written back to the batch as
  `source: "AI 估算"` with `basis`, `estimated_at`, and `model` (ADR-0004). Estimation does not
  change quantity, shelf life, or history, and does not trigger an inventory operation.
- User-entered and package data always win; the UI must label AI-derived values
  `（含 AI 估算，仅作参考）`.
- Estimation failure, timeout or malformed output leaves the reference **unknown** — never zero,
  never a fabricated value — and does not block menu generation or confirmation.
- Requests are triggered: before menu generation when references are missing; when adding an actual
  food; and for actual items during confirmation. A nutrition request sends **only** the item names
  and ids — no body data.
- Estimated values are used for kcal-budget checks and are approximated per 100 g with stated
  assumptions. Estimation is independent of the menu auto-generation switch.
- Confirmed history copies the reference it used at confirmation time; later re-estimation or stock
  edits never rewrite confirmed history (ADR-0002).

---

## 10. Actual consumption and execution records

### 10.1 Flow

1. An accepted, deduction-ready menu exposes `扣减` (consume). The user reviews the actual food.
2. The user may: change quantities, switch to another batch of the same ingredient, remove a
   recommended item, add another in-stock batch, or add a food eaten outside the pantry (never
   deducted).
3. Missing references are estimated if possible.
4. Only the **final confirmation** writes anything. Cancelling or editing a draft writes nothing.
5. An experience slot can be confirmed only once.

### 10.2 Confirmation validation

- Actual meal title non-empty.
- 1–60 actual items.
- Per item: quantity > 0; grams ≥ 0 and ≤ 100000; piece quantities integer.
- In-stock items must be coverable from the eligible pool (earliest expiry first, no batch spent
  twice — the same allocation as §5.5). If a batch expired or stock ran out, the user is told to
  change the amount.
- Each item's nutrition reference is either entirely absent (record as unknown) or complete and
  valid; a partially filled reference is rejected.
- Extra calories (cooking oil, drinks) are optional, integer 0–2000; empty means 0.
- Repeated confirmation of an already-confirmed slot is refused.

### 10.3 Record contents

Confirmation appends an execution record containing: its id; the stage (`program_id`) and slot
(0/1/2 for experience, −1 for a normal day menu); the stage's snapshot (name, days, kind, started,
targets); the confirmation time and civil day; the actual meal title; a **snapshot of the
recommended menu**; the `actual[]` list (batch id, name, grams, native quantity+unit, nutrition
reference, origin `确认库存` / `库存外食用`); the meal kcal (including extra), protein, fiber and
vitamin C; `extra_kcal`; `nutrition_complete`; and `nutrition_unknown` (the names that stayed
unknown).

In the same atomic save it: appends the same snapshot to the stage's `intake`; adds the values to
the stage's consumed totals; marks the source menu `consumed` with its recorded kcal; deducts stock;
updates today's consumed kcal, meal count, last meal and last goal; and updates the rolling weekly
health counters. If the save fails, everything rolls back.

### 10.4 Honest recording

- Exceeding the calorie target is recorded as fact, with a note; the app never edits reality to look
  on target.
- Incomplete nutrition is recorded as incomplete with the known partial total; an unknown part is
  never counted as zero.
- Deleting a menu, regenerating, switching or archiving a plan never deletes execution records.
- Historical records lacking actual detail (older versions) are left as-is; they are never
  reconstructed from a recommended menu.

### 10.5 Weekly health counters

Rolling 7-day window. When the window elapses, the window restarts and today's/week's counters reset.
On a civil-day change, today's meal count resets. Meals always increment the meal counts; protein
and fiber only accumulate for numerically-quantified meals.

---

## 11. Settings, AI switches and image intake

### 11.1 Settings surface

| Control | Rule |
| --- | --- |
| `管家暂停` | Pauses reminders and cancels a waiting generation. Does not undo anything. |
| `主动提醒` | On/off for expiry reminders. |
| Load demo data | Backs up current data, installs the demo dataset, and **keeps existing execution records** while remapping the demo stage to a fresh id. Refuses if the demo dataset is invalid. |
| Restore before demo | Restores the pre-demo backup and merges back any execution records created in the meantime. |
| Advance demo time 12 h | Demo only; moves the app clock, not the system clock, then re-checks risks. |
| Online AI master switch | Off means **no** menu, nutrition or image model requests are sent. An explicit "off" choice in an old save is preserved across upgrades and restarts. Turning it off invalidates a waiting model result. |
| Automatic menu generation | **Off by default.** Manual generation is always allowed. Turning it off invalidates pending automatic flows, including an in-flight nutrition→menu continuation; late successes, failures and timeouts must not continue. Turning it back on does not revive the old flow. |
| Re-read AI configuration | Re-probes provider configuration. |

AI configuration state must distinguish: *checking*, *service available*, *not configured*,
*configuration error*, *host unsupported*. **A budget/usage query succeeding does not mean a
provider is configured.** On failure the app says to check the provider settings and retry; there is
**no local menu fallback** for a user-requested menu.

Already-dispatched requests may still be processed and billed after the user stops waiting.

### 11.2 Image intake

- Enabled only when the host reports image capability. Otherwise show a manual-entry fallback:
  no image picker is opened and no recognition request is sent.
- The user chooses camera capture or an existing image. One image per attempt, JPEG/PNG/WebP,
  ≤ 10 MiB. The image content (not a local path) is sent to the configured provider.
- The result is an **editable draft**: rows of text fields — name, quantity, unit, package size,
  package size unit, average item grams, remaining days, printed expiry date, kcal, protein, fiber,
  vitamin C. Uncertain fields must be filled by the user before confirmation.
- Recognition output never writes stock by itself. Only user-confirmed rows become batches, tagged
  `receipt_image` or `food_image`.
- Images may contain receipt or label information; the privacy notice must say so.

### 11.3 Voice

**Unavailable.** The icon shows an unavailable notice; no microphone permission is requested, no
audio is recorded or uploaded, and the app must not claim otherwise.

### 11.4 Privacy rules (business-observable)

- All data is local. Only the configured online provider receives model requests.
- Menu requests carry the goal, profile (including allergies/avoidances/preferences), eligible batch
  names with units/quantities/grams, nutrition references and remaining budget. Nutrition requests
  carry only item names/ids. Images carry image content.
- No credential input exists in the product. No ads, no analytics, no background execution after
  the app closes.

---

## 12. Cross-cutting business invariants

1. Nothing but a confirmed actual meal changes stock, intake or progress.
2. Snapshots (stages, execution records, recommendations inside records) are never rewritten by later
   edits.
3. The gap list is always recomputed from the menu and stock; no stored gap number may survive a
   restart.
4. The mock market never changes stock or progress except through explicit `确认收货`, which creates
   only new batches.
5. Unknown nutrition is never zero and never blocks a required action.
6. Model output is always locally validated; a model never writes stock directly.
7. Every state write is atomic and validated as a whole; a rejected write leaves the previous state.
8. The product never claims allergy safety, medical advice, exact nutrition, or a live provider
   response it did not verify.

---

## 13. Acceptance criteria

Business-level end-to-end scenarios. A rebuild is acceptable when every one of these passes on a
fresh install with synthetic data (never a real person's inventory).

**A. Onboarding and planning**
1. A fresh install walks scene → basics → candidates → plan → inventory → normal app.
2. Basics outside the validation ranges are rejected with a specific message.
3. With a non-ready profile (e.g. `special` filled, or age 18), candidates show zero targets and
   cannot be confirmed.
4. Regenerating one cycle changes only that cycle's candidates; other cycles and any confirmed stage
   are unchanged.
5. Composing 7+15+30 yields 52 total days, three stages with meeting date ranges and per-stage
   targets; more than 12 stages is impossible.
6. Confirming copies targets into the stages; later editing a candidate does not change a confirmed
   stage.

**B. Inventory**
7. A batch with an invalid unit/quantity/expiry combination is rejected with a reason; a valid
   `kg`, `ml`, `个` and package (`盒` + per-package amount) batch is accepted and shows its canonical
   amount.
8. Editing a batch recomputes its canonical amount; discarding a batch removes it without counting
   as eaten.
9. Replaying the same inventory operation id twice changes stock once.
10. Freshness labels follow the unknown/expired/临期/normal rules.

**C. Day menu and gap list**
11. With sufficient, reference-complete stock, generation produces exactly three meals and the menu
    can be accepted.
12. Accepting a menu changes no stock, no progress and no execution records.
13. A menu containing an excluded ingredient, an incomplete reference, or a total above the day's
    remaining budget cannot be accepted, each with its own reason.
14. Generating a menu and then removing/expiring stock leaves the menu acceptable and shows per-item
    gaps; the gap list equals `用量 − 可用` merged per ingredient.
15. Editing a gap row rewrites the menu usage; if a meal would become empty or the menu becomes
    invalid, the edit is rolled back and nothing is stored.
16. Restarting the app re-derives the same gap list (nothing stale survived).

**D. Simulated market**
17. `一键补齐` is inert before the menu is accepted and active after.
18. Entering the region prefills a cart from the gap list, flags unmatched/over-stock lines, and
    offers same-category replacements with excluded ingredients omitted.
19. Cart edits, replacements and deletions survive a restart and are never overwritten by auto-fill.
20. Submitting an order changes no stock, progress or execution record; the order survives a restart
    with `待收货`.
21. Receiving an order with a reduced quantity and an edited shelf life creates new batches, leaves
    the unchecked remainder in the gap list, writes no execution record, and moves no progress.
22. A line whose product spec has no convertible measure cannot be confirmed, with a reason.

**E. Actual consumption**
23. Confirming an actual meal deducts stock, appends an execution record, adds to the stage's intake
    and consumed totals, and advances progress by exactly one meal.
24. Cancelling the draft leaves stock, progress and records unchanged.
25. An amount exceeding stock or an expired batch blocks confirmation with a reason.
26. A meal exceeding the calorie target is still recorded, with an honest note.
27. Unknown nutrition is recorded as unknown, not zero, and does not block confirmation.
28. Confirming the same experience slot twice is refused.

**F. Experience flow**
29. The experience stage generates only unconfirmed meals, reserves stock across the three meals, and
    reaches 100% after three confirmations, then shows the recap.

**G. Stages and progress**
30. A stage locks on its start day; future stages can still be reordered/replaced/removed/retargeted,
    recalculating later ranges and totals without touching confirmed progress or records.
31. The overview shows the stage whose dates cover today, and its target against actuals.

**H. Online behaviour**
32. With online AI off, no menu, nutrition or image request is sent; the switch state survives a
    restart.
33. A malformed model menu is rejected locally, the specific error is fed back once, and a second
    failure leaves the previous menu intact and asks the user to retry.
34. A model result that arrives after stock/goal/stage changed is discarded.
35. Turning off automatic generation invalidates a pending automatic flow; a late success does not
    create a menu.

**I. Image intake**
36. Without image capability, image buttons show the manual fallback and no picker or request occurs.
37. With capability, a recognised image becomes an editable draft; only confirmed rows become
    batches, tagged with the image source.

**J. Persistence and honesty**
38. Reloading after each of the above restores the same user-visible facts, including cart, orders,
    plans, records and gaps.
39. The four current screenshots' flows (dashboard, three-meal menu with gap list, editable gap,
    inventory) are reproducible.

---

## 14. Out of scope / non-goals

- Real purchasing, payment, delivery or any external shopping integration.
- Voice input, microphone use, audio upload.
- Real-time camera recording beyond one still image.
- "Cook what I want, then buy the ingredients": a menu may cite only stock that already existed at
  generation time (ADR-0005).
- Medical, allergy-safety, or food-safety guarantees; the prototype's numeric targets are estimates.
- Cross-device sync, accounts, sharing, analytics.
- Reconstructing history from old recommendations, or fabricating facts to look on target.

---

## 15. Open questions

These are genuine ambiguities in the shipped behaviour; decide them before or during the rebuild.

1. **`健身减脂` scene.** The target formula handles `健身减脂` (factor 0.96, training intensity) but
   the onboarding screen offers only `减脂控卡 / 日常健康 / 健身增肌 / 轻负担饮食`. Is the fourth
   option missing on purpose or an oversight?
2. **Piece ingredients and multi-batch allocation.** A piece use must fit in one batch because
   average item weight is per-batch. Is a split-piece menu ever needed?
3. **Today's budget for a normal day menu.** The day's remaining budget is `min(stage remaining,
   daily target − today's recorded kcal)`; confirm this is the intended meaning of "today's".
4. **Two stale-menu rules.** Replacing/expiring stock leaves an accepted menu acceptable but possibly
   not executable; the exact wording of `库存够 / 还差 N 克` should be fixed by the rebuild.
5. **Reminder cadence.** The original checks about once a minute; pick a sensible platform-equivalent
   cadence and keep the once-per-risk-set rule.
6. **Experience day boundary.** Generation is limited to day 1 of the experience stage
   (`program_day_index ≤ 1`); confirm what happens on day 2 with unconfirmed meals.
7. **`模拟商品` nutrition references.** They are display-only and must not feed menus; confirm no
   future feature may change that without a new ADR.
