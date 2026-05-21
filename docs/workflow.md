# Creating Generals — Workflow and Skill Sketch

> Originally a Twin Harbour Obsidian vault note. References below to internal memory notes (`project_*`, `reference_*`, `feedback_*`) and to TODO sections point at the author's local notebook and are not reproduced in this repo. They're kept inline because they document *why* a decision was made, even when the *what* is what you actually need.

Capturing the workflow used to author all 60 generals star-tokens (+ ~48 cooldown tokens) in sup-server, so it can be replayed — by the original author, by a teammate, or by a future Claude session that doesn't have the prior context loaded — without redoing the discovery work.

> [!NOTE]
> **Scope of the generated content.** 4 factions × 3 rarities = **12 character lines**. Each line = **5 star ranks + 4 cooldown tokens** = 9 files. Total **108 JSON files** under `sup-server/content-items/510/token/`. Star tokens encode the buffs; cooldown tokens encode the post-death / post-recall lockouts referenced from `armyDeadStrategy` and `manualRemovalStrategy`.

---

## 1. What was actually given to the model

The minimum kit handed over for it to bootstrap from cold:

1. **One finished starter token** — a single complete `*_star_*.json` for one general (cooldown refs included). Acted as the canonical shape; everything else was pattern-matched off it.
2. **A pointer to `tokenConfig`** — the schema / registration code (`content.model.contentitem.TokenContentItem`, the `@type` discriminator) so it could understand the JSON's contract, not just its shape.
3. **The Generals MVP Abilities Confluence page** — the *authoring note* / design spec — [Generals MVP - Abilities](https://bytrolabs.atlassian.net/wiki/spaces/LGCOW/pages/6075973642/Generals+MVP+-+Abilities). Source of truth for the buff table, character flavour, and star scaling rules.
4. **A pointer to `sup-server/content-items/` and "look at how the content editor got the IDs"** — i.e. learn the ID-allocation convention by inspecting existing files. **There is no editor-side auto-allocator** (verified 2026-05-21): `CsvController.createNewItems` at `content-item-editor-presenter/.../csv/CsvController.java:262-290` reads `itemId` directly from a CSV column via `ContentItemId.parse(...)`; no max+1, no gap-fill, no per-type ranges. So the convention is observed-by-archaeology — scan the directory, infer the per-line block layout (§ 3.2), and claim the next free contiguous 9-ID range.

> [!NOTE]
> **Pencil-tracing analogy.** Giving one finished token + the schema + the buff table is like handing an apprentice **one finished pencil drawing**, a **ruler**, and a **photo of the subject**. The drawing tells them the style. The ruler enforces proportions. The photo is the truth they must replicate from. Hand the apprentice all three at once and the rest of the gallery comes out consistent. Hand them only one and you get sixty different drawings.

> [!WARNING]
> **What was NOT handed over explicitly, but the model relied on.** A lot of replication speed came from **prior-session memory** the model already had:
> - The generals epic structure (internal memory: `project_generals_epic`)
> - The `splitStrategy.ORIGINAL_ARMY` decision (internal memory: `project_split_terminology`)
> - The rule that activation conditions stay constant across ranks; only magnitudes scale (internal memory: `project_general_activation_conditions`)
> - The token framework gap tickets (LGCOW-2601…2604) that affect what buffs work today
>
> **A fresh session won't have these.** The skill (`create-general/SKILL.md`) bundles them in explicitly, otherwise day-one Claude will ask 30 clarifying questions or — worse — invent answers.

---

## 2. The mapping rules taught mid-flight

These weren't in the Confluence page or the starter token — they emerged from review feedback and need to be encoded in the skill so the next run doesn't relitigate them.

> [!IMPORTANT]
> **Rule 1: "Damage bonus" is a pair, not a single modifier.**
> Confluence table entry of "Damage bonus" → emit **both** `ATTACK_FACTOR_MODIFIER` and `DEFENCE_FACTOR_MODIFIER` with the **same value** and **same `requirementExpression`**. Description goes on `ATTACK_FACTOR_MODIFIER` only — the pair is one ability line in the UI, not two.

> [!IMPORTANT]
> **Rule 2: Paired-penalty effects share a description too.**
> When the table says e.g. *"Tank units gain X% Defense but lose Y% movement speed"*, emit the primary modifier (`DEFENCE_FACTOR_MODIFIER`) **and** the penalty (`SPEED_FACTOR_MODIFIER`, negative value). Description lives on the primary only. Same logic applies to `STEALTH_LEVEL_FLAT_MODIFIER` + `SPEED_FACTOR_MODIFIER` for the Pan-Asian Epic line.

> [!IMPORTANT]
> **Rule 3: Player and province effects still need their own description.**
> Despite rules 1 & 2 bundling army effects, **each `playerEffects.*` or `provinceEffects.*` entry is its own ability line** and gets its own `metadata.DESCRIPTION`. As of 2026-05-21 the schema doesn't yet support `metadata` on `PlayerEffectContent` / `ProvinceEffectContent` (only `ConditionalArmyEffects` has it) — add the description anyway; follow-up PR wires the schema.

> [!IMPORTANT]
> **Rule 4: Star scaling formula.**
> Per the Confluence "Ability Star Scaling" footer:
> - 1★: Major = 50% of max
> - 2★: Major unchanged, Minor introduced at 50% of max
> - 3★: Major = 75% of max, Minor = 50% of max
> - 4★: Major = 75% of max, Minor = 75% of max
> - 5★: Major = 100%, Minor = 100%, **Rec. Spd cooldown reduced by 30%** (= 5-star uses the dedicated `*_5_star_*_cooldown` tokens, not the line's default cooldown)
>
> "No scaling" in the table = value stays constant across all 5 ranks. "NA" = effect not granted at that rank.

> [!IMPORTANT]
> **Rule 5: Description text mirrors the table's parametrised template, with the numeric value substituted from the JSON, not the rounded value from the table.**
> Confluence shows "Ma: 37%" for Pan-Asian Legendary 3★, but the JSON stores `0.375` → description reads "37.5%", not "37%". The JSON is the source of truth for the displayed number.

---

## 3. The replication recipe (everything a fresh session needs)

> [!NOTE]
> **Cookbook analogy.** Below is the complete recipe. A new chef should be able to walk in cold, follow the steps, and produce the same 108 dishes — no questions to the kitchen owner.

### 3.1 File layout

```
sup-server/content-items/510/token/
  <faction>_<rarity>_<N>_star_<numericId>.json
  <faction>_<rarity>_defeated_cooldown_<numericId>.json
  <faction>_<rarity>_withdrawn_cooldown_<numericId>.json
  <faction>_<rarity>_5_star_defeated_cooldown_<numericId>.json
  <faction>_<rarity>_5_star_withdrawn_cooldown_<numericId>.json
```

- **Factions:** `axis`, `allies`, `comintern`, `pan_asian`
- **Rarities:** `legendary`, `epic`, `free_epic`
- **N (rank):** `1`..`5`

### 3.2 ID allocation pattern

> [!WARNING]
> **The content editor does NOT auto-allocate IDs.** Verified 2026-05-21: `content-item-editor-presenter/.../csv/CsvController.java:262-290` (`createNewItems` + `getItemId`) reads each ID from a `itemId` column in an imported CSV via `ContentItemId.parse(...)`. There is no "give me the next free ID" call anywhere in the editor/presenter/JavaFX modules. **Authoring path = scan `sup-server/content-items/510/token/` for the highest used ID and claim the next contiguous block.**

Each character line gets a contiguous **9-ID block**. Within the block, offsets from `block_start`:

| Offset | File |
| --- | --- |
| +0 | 1-star token |
| +1 | defeated cooldown (1–4★ share this) |
| +2 | withdrawn cooldown (1–4★ share this) |
| +3 | 2-star token |
| +4 | 3-star token |
| +5 | 4-star token |
| +6 | 5-star token |
| +7 | 5-star defeated cooldown |
| +8 | 5-star withdrawn cooldown |

Observed ID blocks (CoW, title 510, sup-server `develop` as of 2026-05-21):

| Line | Block |
| --- | --- |
| `axis_free_epic` | 60723–60731 |
| `axis_epic` | 60734–60742 |
| `axis_legendary` | 60745–60753 |
| `allies_legendary` | 60756–60764 |
| `allies_epic` | 60767–60775 |
| `allies_free_epic` | 60778–60786 |
| `comintern_legendary` | 60789–60797 |
| `comintern_epic` | 60800–60808 |
| `comintern_free_epic` | 60811–60819 |
| `pan_asian_legendary` | 60822–60830 |
| `pan_asian_epic` | 60844–60852 |
| `pan_asian_free_epic` | 60833–60841 |

> [!WARNING]
> **Pan-Asian Free Epic block is out of sequence** (60833–60841 sits between Pan-Asian Legendary and Pan-Asian Epic). If allocating new lines, claim the next free contiguous block — don't assume strict left-to-right ordering by faction.

### 3.3 Star-token JSON template

```json
{
  "@type": "content.model.contentitem.TokenContentItem",
  "goldFeature": false,
  "id": "<Faction Display> <Rarity> <N> Star (<numericId>)",
  "isAbstract": false,
  "name": "<Faction Display> <Rarity> <N> Star",
  "status": 0,
  "versions": {
    "1": {
      "activationStrategy": {
        "activationDelay": "PT30M",
        "strategy": "NONE"
      },
      "affectedUnits": { "factions": [], "upgradeGroups": [] },
      "armyDeadStrategy": {
        "cooldownStrategy": {
          "cooldownToken": "<defeated cooldown name> (<id+1 or id+7>)",
          "cooldownType": "GLOBAL",
          "trigger": "ON_APPLY"
        }
      },
      "effects": {
        "armyEffects":   { /* see § 3.4 */ },
        "playerEffects": { /* see § 3.4 */ },
        "provinceEffects": {}
      },
      "manualRemovalStrategy": {
        "cooldownStrategy": {
          "cooldownToken": "<withdrawn cooldown name> (<id+2 or id+8>)",
          "cooldownType": "GLOBAL",
          "trigger": "ON_APPLY"
        },
        "requirementExpression": "NOT isCloseCombat AND isTokenActivated"
      },
      "mergeStrategy": { "blockingTags": ["general"] },
      "metadata": {
        "NAME":              { "textValue": "<Rarity> <Faction> General" },
        "CURRENT_RANK":      { "intValue": <N> },
        "MAX_RANK":          { "intValue": 5 },
        "MIN_RANK":          { "intValue": 1 },
        "DESCRIPTION":       { "textValue": "Placeholder Description" },
        "RARITY":            { "textValue": "<Rarity Display>" },
        "AFFECTED_UNITS":    { "textValue": "<from Confluence: 'All Units' / 'Infantry' / 'Tank' / 'Ordnance'>" },
        "AFFILIATE_FACTION": { "textValue": "<Faction Display>" }
      },
      "requirementExpression": "hasFaction:<FACTION_ENUM> AND hasPremiumItem:<upgradePremiumId>,<rankLower>,<rankUpper> AND hasAtMostTagsInArmy:general,1 AND NOT isCloseCombat",
      "splitStrategy": { "strategy": "ORIGINAL_ARMY" },
      "tokenTags": ["general"]
    }
  }
}
```

> [!IMPORTANT]
> **`requirementExpression` rank-gating: `hasPremiumItem:<upgradePremiumId>,<lower>,<upper>`** with these bounds per rank:
> - 1★ → `1,2`
> - 2★ → `3,4`
> - 3★ → `5,6`
> - 4★ → `7,8`
> - 5★ → `9,10`
>
> The premium item itself is the line's *upgrade material*; you need 10 copies to reach 5★ (one per upgrade level, two levels per star).

> [!IMPORTANT]
> **`hasFaction` enum values:** `GERMAN` (Axis), `AMERICAN` (Allies), `RUSSIAN` (Comintern), `JAPANESE` (Pan-Asian). Read off the existing tokens to confirm — these are the in-engine enums, not the marketing names.

### 3.4 Effect block — derivation from Confluence

For each Major and Minor buff at each rank:

1. **Map the buff name to its property enum** (see Confluence § "Army Buff Properties" + "Unit Features"). Examples:
   - "Damage bonus" / "% increased damage" → `ATTACK_FACTOR_MODIFIER` (paired with `DEFENCE_FACTOR_MODIFIER`, rule 1)
   - "% Defense multiplier" → `DEFENCE_FACTOR_MODIFIER`
   - "% Hitpoints multiplier" → `HITPOINTS_FACTOR_MODIFIER`
   - "% Movement speed" → `SPEED_FACTOR_MODIFIER`
   - "% Vision range" → `VISION_RANGE_FACTOR_MODIFIER`
   - "Stack size" → `STACK_LIMIT_FLAT_MODIFIER`
   - "Scout level" → `SCOUT_LEVEL_FLAT_MODIFIER`
   - "Stealth level" → `STEALTH_LEVEL_FLAT_MODIFIER`
   - "Attack speed" / "Attack interval reduction" → `ATTACK_INTERVAL_FACTOR_MODIFIER` (**negative value** = faster)
   - "Resource loot from conquering" → `RESOURCE_LOOT_FACTOR_MODIFIER` (player effect, not army)
   - "Not affected by negative terrain" → custom key `NegativeTerrainImmunity` (army effect, no modifier value — references a `UnitFeature`)
2. **Decide the bucket:** army effect / player effect / province effect.
   - Combat stats, vision, stealth, scout, stack limit, terrain immunity → `armyEffects`
   - Resource loot (and anything player-wide) → `playerEffects`
   - Province production (none in MVP) → `provinceEffects`
3. **Scope via `requirementExpression`** on the effect (NOT the token):
   - Per-unit-type filtering (Infantry / Tank / Ordnance) → fully landed as of 2026-05-06 (LGCOW-2601 / PR #6215). Use the per-unit filter syntax in armyEffects.
   - Territory gating → `"territoryType:ENEMY"` / `"territoryType:ALLIED"` / `"territoryType:OWN"`
   - HP threshold → `"unitHpFactorBelow:0.5"` (Axis Legendary)
   - No condition → empty string `""`
4. **Pair-rule check (rule 1 & 2):** If two effects describe one ability line ("damage bonus" = ATTACK+DEFENCE; "X but lose Y" = primary+penalty), put `metadata.DESCRIPTION` on the **primary** only, leave it absent on the partner.
5. **Substitute the numeric value into the description template** from the Confluence row, using the JSON value (rule 5).

### 3.5 Cooldown token shape

Cooldown tokens are tiny — they're effectively named timers referenced by `armyDeadStrategy` / `manualRemovalStrategy`. Open any existing one (e.g. `pan_asian_legendary_defeated_cooldown_60823.json`) and clone the shape. Only the `id`, `name`, and possibly the duration differ between defeated vs withdrawn and 1–4★ vs 5★ (5★ duration is 30% shorter per rule 4).

> [!NOTE]
> If the engine starts to need a third cooldown variant (e.g. per-rank, not just "5★ or not"), this is where it would land. For MVP, the two-tier (default + 5★) split is enough.

### 3.6 Validation pass after generation

- [ ] Every star token's `armyDeadStrategy.cooldownToken` resolves to an existing defeated cooldown file in the same line.
- [ ] Every star token's `manualRemovalStrategy.cooldownToken` resolves to an existing withdrawn cooldown file in the same line.
- [ ] 5★ tokens reference the `5_star_*` cooldown variants, 1–4★ reference the base variants.
- [ ] `CURRENT_RANK.intValue` matches the rank in the filename.
- [ ] `hasPremiumItem:<id>,<lower>,<upper>` matches the per-rank table in § 3.3.
- [ ] Every Major/Minor effect has a description **except** the partner side of a pair (rules 1 & 2).
- [ ] Player effects all carry a `metadata.DESCRIPTION` even though the schema currently drops it (rule 3).
- [ ] Values match the rank's star-scaling tier (rule 4).

---

## 4. Sketch — packaging this as a Claude skill

> [!NOTE]
> **Goal:** a `/create-general` slash-skill that, given a faction + rarity + (optional) star rank, generates the corresponding token files using the recipe above, with the Confluence table and the codebase patterns already loaded as context.
>
> The implementation lives in [`../create-general/SKILL.md`](../create-general/SKILL.md). What follows is the original design sketch.

### 4.1 Skill manifest (rough shape)

```markdown
---
name: create-general
description: |
  Generate generals token JSON files for sup-server. Bundles the Generals MVP
  Abilities spec, the file/ID/naming conventions, the buff-pair mapping rules,
  and the star-scaling formula. Use when the user wants to author a new
  general line or fill in missing ranks/cooldowns.
trigger: |
  - User asks to "create / author / add / generate" a general (legendary / epic /
    free epic) for any of: axis, allies, comintern, pan-asian.
  - User points to a row in the Generals MVP Abilities table and says
    "implement this".
  - User asks to backfill missing star ranks or cooldowns for an existing line.
inputs:
  - faction: axis | allies | comintern | pan_asian
  - rarity: legendary | epic | free_epic
  - ranks: optional, default 1..5 (subset allowed for backfill)
  - id_block_start: optional, default = auto-allocate next free 9-ID block
tools: [Read, Write, Edit, Bash, mcp__atlassian__getConfluencePage]
---
```

### 4.2 Skill body (the instructions Claude follows when the skill fires)

1. **Fetch fresh spec.** Pull the [Generals MVP - Abilities](https://bytrolabs.atlassian.net/wiki/spaces/LGCOW/pages/6075973642/Generals+MVP+-+Abilities) page via the Atlassian MCP. Don't trust cached values in this note — the table is still moving.
2. **Locate the canonical template.** Read one existing complete line for the same rarity (e.g. for a new Legendary, read `pan_asian_legendary_3_star_60826.json`). Use it as the shape reference. Read its two cooldown templates too.
3. **Confirm with user:**
   - Confirm Confluence row interpretation, esp. ambiguous text (e.g. "+50% resource loot" → `RESOURCE_LOOT_FACTOR_MODIFIER` × 0.50).
   - Confirm ID-block start (auto-suggest the next free block; show the user before claiming).
   - Confirm if any **rule-3** player/province effect is in scope (so the user is aware the JSON ships ahead of the schema until the player/province effect metadata follow-up PR lands).
4. **Generate files** (9 per line: 5 star tokens + 4 cooldowns) following § 3.3–3.5. Apply pair rules (1 & 2), scaling (4), and JSON-value-into-description (5).
5. **Self-validate** via § 3.6 checklist before reporting done.
6. **Report:**
   - The 9 new file paths.
   - The pair-rule decisions made (which effects share a description).
   - Any rule-3 player/province effects added that depend on the pending schema PR.
   - Any open questions the Confluence table left ambiguous.

### 4.3 What the skill must NOT do

> [!WARNING]
> **Don't invent buff names.** If the Confluence row says something not in the "Army Buff Properties" or "Unit Features" tables (e.g. a new feature like `NegativeTerrainImmunity` was originally), stop and ask. Pattern-matching to the nearest existing enum will silently misroute the buff.
>
> **Don't reorder ID blocks.** ID blocks are claim-once; don't shuffle existing ones to "tidy up" Pan-Asian's out-of-sequence block. Existing references in player accounts, premium-item content, and DB rows resolve by ID.
>
> **Don't relitigate the pair rule with the user.** Apply rules 1 & 2 silently; only surface the pair decision in the report so the user can audit. Asking "should I add metadata to both ATTACK and DEFENCE?" every run is the kind of friction the skill exists to remove.
>
> **Don't skip the cooldown tokens.** A general line without its 4 cooldowns will load but every defeat/recall will produce a "cooldown token not found" warning. Always generate the cooldowns even when the user only mentions star tokens.

### 4.4 Knowledge the skill bakes in (so a cold session doesn't need memory)

| Domain | What gets bundled |
| --- | --- |
| Spec source | Confluence page ID `6075973642` + the table-to-JSON mapping rules |
| Codebase | Path `sup-server/content-items/510/token/`, `@type` value, `TokenContentItem` field names, `EffectsContent` field bucketing |
| Conventions | Filename pattern, ID block pattern, ID-per-rank pattern, cooldown naming |
| Buff catalogue | The 11 modifier properties + 4 unit features from Confluence § "Army Buff Properties" / "Unit Features" |
| Pair rules | Damage-bonus pair, paired-penalty pair, player-effect description rule |
| Star scaling | The 50/50+/75/75/100 formula + 5★ cooldown-30% mapping to the dedicated cooldown tokens |
| Pending tickets | LGCOW-2601…2604 → which buffs work today vs ship-ahead-of-engine-support |
| Activation rules | Constant across ranks; only magnitudes scale |
| Split behaviour | `ORIGINAL_ARMY` (general tokens stay with the original army on split) |

> [!NOTE]
> **Memory-vs-skill analogy.** Memory is the engineer's notebook — useful, but only present if you've shown up to the same desk before. A skill is the **laminated card pinned to the wall**: anyone walking up to the desk can read it cold. The skill must contain the laminated-card version of everything in the right-hand column above. Don't rely on the reader having the notebook.

### 4.5 Open questions before turning this into a real skill

- [ ] Does the team want one skill that covers all 4 factions × 3 rarities, or a separate skill per rarity? **Resolved 2026-05-21:** one parameterised skill — the per-rarity divergence is just the buff table.
- [x] **Where does the next ID block come from?** Verified 2026-05-21: no editor-side auto-allocator (see § 3.2 warning callout). The skill must scan `sup-server/content-items/510/token/` itself — read every filename's trailing `_<id>.json`, find the max, claim `max+1`..`max+9` as the next line's block. (Could be lifted to a `Bash` one-liner inside the skill: `` ls content-items/510/token/*_star_*.json | grep -oE '[0-9]+(?=\.json)' | sort -n | tail -1 ``.) Worth noting: claim a contiguous block across **all** content types (not just tokens) if other content items share the same ID namespace — verify before assuming token-scoped allocation.
- [ ] Should the skill update `i18n` / English string catalogues at the same time? Deferred to a future v2; see README "Out of scope".
- [ ] Should the skill auto-create the related Jira tickets (parent epic LGCOW-2432 + sub-tickets per general line)? Or stop at the file-generation boundary? **Deferred to v2:** v1 stops at file generation.
- [ ] What's the test/validation story? A small JUnit "every general line has its 4 cooldowns and rank/id consistency" test would catch most of the rule-of-thumb regressions § 3.6 enumerates — and would also serve as the skill's self-check oracle. **Deferred to v2:** v1 validates via the § 3.6 checklist only.

---

## 5. Cross-references

References below point at internal Obsidian / memory notes — not reproduced in this repo.

- `project_generals_epic` (internal memory) — the umbrella epic
- `project_general_activation_conditions` (internal memory) — why activation conditions stay constant across ranks
- `project_split_terminology` (internal memory) — the `ORIGINAL_ARMY` decision
- `reference_generals_tickets` (internal memory) — Jira tickets
- sup-server `TODO.md` § "Token Framework Gaps — Generals MVP" (internal note) — the engine work that gates which buffs actually function today
- `Token System - Configuration Guide` (internal note) — neighbouring note on the broader token system
