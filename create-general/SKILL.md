---
name: create-general
description: |
  Generate generals token JSON files for sup-server (CoW, title 510). Authors a complete
  character line — 5 star-rank tokens + 4 cooldown tokens (9 files) — under
  `sup-server/content-items/510/token/` following the Generals MVP spec.

  USE WHEN: user asks to create, author, add, generate, or scaffold a general (legendary
  / epic / free-epic) for any of axis, allies, comintern, pan-asian; or asks to backfill
  missing star ranks / cooldowns for an existing line; or points at a row in the
  Generals MVP Abilities Confluence table and says "implement this".

  SKIP WHEN: user is editing an existing general (just edit the file directly), or asks
  about generals behaviour at runtime (engine code, not content authoring).
---

# create-general

Author a complete general character line as JSON tokens in sup-server, matching the
[Generals MVP - Abilities](https://bytrolabs.atlassian.net/wiki/spaces/LGCOW/pages/6075973642/Generals+MVP+-+Abilities)
spec.

## Inputs

Required:
- `faction`: `axis` | `allies` | `comintern` | `pan_asian`
- `rarity`: `legendary` | `epic` | `free_epic`

Optional:
- `ranks`: subset of `1..5` (default: all 5). Use for backfill of missing ranks.
- `id_block_start`: numeric ID for the first file in the 9-ID block. Default:
  auto-allocate by scanning the directory (see § "ID allocation").

## What you generate

For each character line, **9 JSON files** in `sup-server/content-items/510/token/`:

| Filename pattern | Purpose |
| --- | --- |
| `<faction>_<rarity>_1_star_<id+0>.json` | 1★ token |
| `<faction>_<rarity>_defeated_cooldown_<id+1>.json` | Cooldown shared by 1–4★ on death |
| `<faction>_<rarity>_withdrawn_cooldown_<id+2>.json` | Cooldown shared by 1–4★ on recall |
| `<faction>_<rarity>_2_star_<id+3>.json` | 2★ token |
| `<faction>_<rarity>_3_star_<id+4>.json` | 3★ token |
| `<faction>_<rarity>_4_star_<id+5>.json` | 4★ token |
| `<faction>_<rarity>_5_star_<id+6>.json` | 5★ token |
| `<faction>_<rarity>_5_star_defeated_cooldown_<id+7>.json` | 5★-specific death cooldown (30% shorter) |
| `<faction>_<rarity>_5_star_withdrawn_cooldown_<id+8>.json` | 5★-specific recall cooldown (30% shorter) |

## Procedure

### Step 1 — Fetch the spec fresh

Pull the Confluence page each run; the table moves:
- Use `mcp__atlassian__getConfluencePage` with `cloudId: bytrolabs.atlassian.net`,
  `pageId: 6075973642`, `contentFormat: markdown`.
- Locate the row for the requested `(faction, rarity)` in the "Character Buffs Overview"
  table. Note the Major and Minor buff names, descriptions, and per-rank value cells.

### Step 2 — Read the canonical template

For shape reference, read an existing complete line of the same rarity. Suggested
templates (any 3★ token works; 3★ is the most fully-populated rank):
- Legendary: `content-items/510/token/pan_asian_legendary_3_star_60826.json`
- Epic: `content-items/510/token/allies_epic_3_star_60771.json`
- Free Epic: `content-items/510/token/axis_free_epic_3_star_60727.json`

Also read the corresponding cooldown templates from the same line — `*_defeated_cooldown_*`
and `*_withdrawn_cooldown_*` files share the same shape, only durations and IDs differ.

### Step 3 — Allocate the ID block

**There is no editor-side auto-allocator.** Verified at
`content-item-editor-presenter/src/main/java/com/bytro/content/editor/controller/csv/CsvController.java:262-290`:
`createNewItems` reads IDs from a CSV `itemId` column via `ContentItemId.parse(...)`.
No max+1, no gap-fill.

So scan the directory yourself:

```bash
ls sup-server/content-items/510/token/*.json \
  | grep -oE '_[0-9]+\.json$' \
  | grep -oE '[0-9]+' \
  | sort -n \
  | tail -1
```

Claim the next contiguous **9-ID block** starting at `(max + 1)`. Confirm the block
with the user before writing files — ID claims are sticky once references exist.

>[!warning] **Caveat:** the ID namespace may be shared across all content types
> (`unit/`, `mod/`, `premium/`, `token/`, etc.), not just tokens. If the directory scan
> in this skill is restricted to `token/`, a collision with another content type is
> possible. Defensive option: scan `sup-server/content-items/510/` recursively for the
> max ID across all subdirectories. Not yet verified at skill-write time
> (2026-05-21) — flag this to the user when running.

### Step 4 — Build each star token

Template (substitute `<placeholders>`):

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
      "activationStrategy": { "activationDelay": "PT30M", "strategy": "NONE" },
      "affectedUnits": { "factions": [], "upgradeGroups": [] },
      "armyDeadStrategy": {
        "cooldownStrategy": {
          "cooldownToken": "<defeated cooldown name with ID>",
          "cooldownType": "GLOBAL",
          "trigger": "ON_APPLY"
        }
      },
      "effects": {
        "armyEffects":     { /* see step 5 */ },
        "playerEffects":   { /* see step 5 */ },
        "provinceEffects": {}
      },
      "manualRemovalStrategy": {
        "cooldownStrategy": {
          "cooldownToken": "<withdrawn cooldown name with ID>",
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
        "AFFECTED_UNITS":    { "textValue": "<from Confluence>" },
        "AFFILIATE_FACTION": { "textValue": "<Faction Display>" }
      },
      "requirementExpression": "hasFaction:<FACTION_ENUM> AND hasPremiumItem:<upgradePremiumId>,<rankLower>,<rankUpper> AND hasAtMostTagsInArmy:general,1 AND NOT isCloseCombat",
      "splitStrategy": { "strategy": "ORIGINAL_ARMY" },
      "tokenTags": ["general"]
    }
  }
}
```

**`hasFaction` enum values** (sourced from existing tokens):
- Axis → `GERMAN`
- Allies → `AMERICAN`
- Comintern → `RUSSIAN`
- Pan-Asian → `JAPANESE`

**Per-rank `hasPremiumItem` ranges** (one upgrade item gates the line; 2 levels per star):
- 1★ → `1,2`
- 2★ → `3,4`
- 3★ → `5,6`
- 4★ → `7,8`
- 5★ → `9,10`

**1–4★ tokens reference the line's base cooldowns. 5★ tokens reference the dedicated
`5_star_*_cooldown` tokens** (Confluence: "Rec. Spd: 30%" — the 5★ cooldown is 30%
shorter, encoded as a separate cooldown token).

### Step 5 — Derive effects from the Confluence row

Map each Major / Minor buff from the table to its property enum:

| Buff phrasing in table | Property | Bucket |
| --- | --- | --- |
| "Damage bonus", "% increased damage" | `ATTACK_FACTOR_MODIFIER` **paired with** `DEFENCE_FACTOR_MODIFIER` (see Rule 1) | armyEffects |
| "% Defense multiplier" | `DEFENCE_FACTOR_MODIFIER` | armyEffects |
| "% Hitpoints multiplier" | `HITPOINTS_FACTOR_MODIFIER` | armyEffects |
| "% Movement speed" | `SPEED_FACTOR_MODIFIER` | armyEffects |
| "% Vision range" | `VISION_RANGE_FACTOR_MODIFIER` | armyEffects |
| "Stack size", "active combat units" | `STACK_LIMIT_FLAT_MODIFIER` | armyEffects |
| "Scout level" | `SCOUT_LEVEL_FLAT_MODIFIER` | armyEffects |
| "Stealth level" | `STEALTH_LEVEL_FLAT_MODIFIER` | armyEffects |
| "Attack speed", "attack interval reduction" | `ATTACK_INTERVAL_FACTOR_MODIFIER` (**negative value** = faster) | armyEffects |
| "Resource loot from conquering" | `RESOURCE_LOOT_FACTOR_MODIFIER` | **playerEffects** |
| "Not affected by negative terrain" | custom key `NegativeTerrainImmunity` (references a `UnitFeature`, no scalar value) | armyEffects |

**`requirementExpression` on the effect** (NOT on the token) is how you scope it:
- Territory gating: `"territoryType:ENEMY"` / `"territoryType:ALLIED"` / `"territoryType:OWN"`
- HP threshold: `"unitHpFactorBelow:0.5"` (Axis Legendary)
- Unconditional: `""`
- Per-unit-type filtering (Infantry / Tank / Ordnance): use the per-unit filter syntax
  on `armyEffects`. **Note (LGCOW-2601):** this filter is fully landed as of 2026-05-06
  (PR #6215) — author the filter; the engine will enforce it.

### Step 6 — Apply the 5 mapping rules

Apply these silently. Do **not** ask the user to relitigate them at runtime — they were
agreed during the original generals authoring pass. Surface the decisions in the final
report so the user can audit.

>[!important] **Rule 1 — "Damage bonus" is a pair, not a single modifier.**
> Confluence "Damage bonus" → emit **both** `ATTACK_FACTOR_MODIFIER` and
> `DEFENCE_FACTOR_MODIFIER`, same value, same `requirementExpression`.
> `metadata.DESCRIPTION` goes on `ATTACK_FACTOR_MODIFIER` only.

>[!important] **Rule 2 — Paired-penalty effects share a description.**
> Patterns like *"Tank units gain X% Defense but lose Y% movement speed"* or
> *"Infantry units gain 1 Stealth level but lose Y% movement speed"* emit the primary
> (`DEFENCE_FACTOR_MODIFIER` or `STEALTH_LEVEL_FLAT_MODIFIER`) **and** the penalty
> (`SPEED_FACTOR_MODIFIER`, negative value). Description on the primary only.

>[!important] **Rule 3 — Player and province effects need their own description.**
> Despite Rules 1 & 2 bundling army effects, every entry in `playerEffects` /
> `provinceEffects` is its own ability line and needs its own `metadata.DESCRIPTION`.
> **Schema caveat (2026-05-21):** `PlayerEffectContent` / `ProvinceEffectContent` do
> not yet have a `metadata` field — only `ConditionalArmyEffects` does. Author the
> description anyway; Jackson will drop it on load until the follow-up schema PR lands.
> Flag this in the final report.

>[!important] **Rule 4 — Star scaling formula.**
> From Confluence "Ability Star Scaling":
> - 1★: Major = 50% of max
> - 2★: Major unchanged, Minor introduced at 50% of max
> - 3★: Major = 75%, Minor = 50%
> - 4★: Major = 75%, Minor = 75%
> - 5★: Major = 100%, Minor = 100%, **Rec. Spd: cooldown −30%**
>   (= 5★ uses dedicated `_5_star_*_cooldown` tokens)
>
> "No scaling" in a cell = value stays constant across ranks. "NA" = effect absent at
> that rank.

>[!important] **Rule 5 — JSON value, not table value, drives description text.**
> Confluence shows e.g. "Ma: 37%" for Pan-Asian Legendary 3★ but the JSON stores
> `0.375` → description reads "37.5%", not "37%". The JSON is the source of truth for
> the displayed number.

### Step 7 — Validate before reporting done

Two phases. Both must pass before Step 8.

#### Phase 7a — JSON-shape checklist (manual self-review)

- [ ] Every star token's `armyDeadStrategy.cooldownToken` resolves to an existing
      defeated cooldown file in the same line.
- [ ] Every star token's `manualRemovalStrategy.cooldownToken` resolves to an existing
      withdrawn cooldown file in the same line.
- [ ] 5★ tokens reference `5_star_*` cooldowns; 1–4★ reference the base cooldowns.
- [ ] `CURRENT_RANK.intValue` matches the rank in the filename for each star token.
- [ ] `hasPremiumItem:<id>,<lower>,<upper>` matches the per-rank table (Step 4).
- [ ] Every Major/Minor effect has a description **except** the partner side of a pair
      (Rules 1 & 2).
- [ ] Player and province effects all carry `metadata.DESCRIPTION` (Rule 3) — flag
      schema-ahead-of-code in report.
- [ ] Numeric values match the rank's star-scaling tier (Rule 4).
- [ ] All 9 files in the block exist (5 stars + 4 cooldowns).

#### Phase 7b — Run the content validators (close the loop)

The shape checklist above only catches what you remembered to check. The codebase has
three mod-level validators that run over all authored content and surface schema /
reference errors you'd otherwise miss. Run them from the sup-server repo root:

```bash
./gradlew :sup-server:test \
  --tests "com.bytro.sup.token.content.TokenContentTest" \
  --tests "ultshared.UltModTest" \
  --tests "com.bytro.sup.requirement.mod.RequirementModTest"
```

What each one covers:

| Test | Catches |
| --- | --- |
| `TokenContentTest` | Token shape — invalid combat restrictions, metadata round-trip through `TokenConfig`. Touches every `ITokenContent` item, so any new generals JSON in `content-items/510/token/` is exercised. |
| `UltModTest` | Broad mod consistency across all content items. |
| `RequirementModTest` | Every `requirementExpression` parses and references known IDs / enums (this is where `hasPremiumItem:<bad-id>,...` or a typo'd `hasFaction:GREMAN` blows up). |

**On failure** — read the gradle output, identify the offending file(s), fix, re-run.
Treat this as a self-correcting loop, not a one-shot:

1. Parse the failed test's stack/message to find which content item failed and why.
2. Apply the minimal fix to the offending JSON file(s).
3. Re-run the same gradle command.
4. Repeat up to **3 attempts**. If still failing after 3, **stop and surface the failure
   to the user** — do not keep guessing or paper over with broader edits.

**On `BUILD SUCCESSFUL`** — proceed to Step 8.

>[!warning] **Don't skip Phase 7b even when the checklist passes.** The checklist is
> what you thought to check; the validators are what the codebase enforces. Several
> classes of error (unknown requirement-expression name, malformed cooldown reference,
> metadata key not in the enum) won't show up until the validator runs.

### Step 8 — Report

End the turn with a short report covering:
1. The 9 new file paths.
2. The pair-rule decisions made (which effects share a description, where the
   description landed).
3. Any Rule 3 player/province effects added that depend on the pending schema PR.
4. Any open questions the Confluence table left ambiguous.
5. The next free ID block scanned at start, and confirmation of the block you claimed.
6. **Validator status** — `BUILD SUCCESSFUL` from Phase 7b, plus a one-line note if
   any fix-and-retry iterations were needed (what failed, what you changed).

## Hard rules

>[!warning] **Don't invent buff names.** If the Confluence row contains something
> not listed in Step 5's mapping table or in Confluence's own "Army Buff Properties" /
> "Unit Features" sections, **stop and ask the user**. Pattern-matching to the nearest
> existing enum will silently misroute the buff.

>[!warning] **Don't reorder existing ID blocks.** Even if Pan-Asian Free Epic
> (60833–60841) looks "out of sequence" relative to Pan-Asian Legendary / Epic, leave
> it alone. References to those IDs exist in player accounts, premium items, and DB
> rows; renumbering breaks them.

>[!warning] **Don't relitigate the 5 rules with the user at runtime.** Apply them
> silently; surface decisions in the final report (Step 8) so they can be audited.
> Asking "should I add metadata to both ATTACK and DEFENCE?" every run defeats the
> purpose of the skill.

>[!warning] **Don't skip the cooldown tokens.** A general line without its 4 cooldowns
> loads, but every defeat/recall produces a "cooldown token not found" warning. Always
> generate all 9 files even when the user only mentions star tokens.

>[!warning] **Don't change activation conditions across ranks within a line.** Per the
> design memory: a general's activation conditions (HP threshold, terrain, etc.) stay
> constant across all 5 ranks; only the buff magnitudes scale.

## Out of scope (v1)

The skill does NOT do these things — flag them in the final report so the user can
follow up manually:

- **i18n / string catalogues.** Descriptions ship as raw English. Localisation files
  must be updated separately. See sup-server TODO.md § "i18n / Translations —
  Generals, Stats & Requirement-Check Reasons".
- **Jira ticket creation.** No parent epic / per-line sub-tickets created. File via
  the Atlassian MCP separately if needed.
- **Dedicated JUnit consistency test for the new line.** No bespoke test emitted —
  Step 7b runs the existing mod-level validators (`TokenContentTest`, `UltModTest`,
  `RequirementModTest`) which already exercise every content item. A future v2 could
  still write a generals-specific test under
  `sup-server/sup-server/src/test/java/.../GeneralTokenConsistencyTest.java` for
  invariants the mod validators don't catch (e.g. star-scaling formula adherence).
- **Cross-content-type ID-namespace verification.** Skill scans `token/` only. If the
  ID namespace turns out to be shared across all content types
  (`unit/`, `mod/`, `premium/`, etc.), the chosen ID block may collide. See Step 3
  caveat.
- **`splitStrategy.ORIGINAL_ARMY` schema verification.** Skill assumes the strategy is
  supported in the current codebase. Verified for sup-server `develop` as of
  2026-05-21 (JST-131 landed).

## Context the skill assumes (bundled here so a cold session needs no prior memory)

- **Generals are tokens.** They live under `sup-server/content-items/510/token/` and
  use `@type: content.model.contentitem.TokenContentItem`.
- **A "character line"** is the combination `(faction, rarity)`. There are 12 lines
  total (4 factions × 3 rarities).
- **Each line spans 5 star ranks** plus 4 cooldown tokens (2 default + 2 for 5★).
- **`splitStrategy: ORIGINAL_ARMY`** means general tokens stay with the original
  (left-side) army on stack split — not the new right-side army (JST-131 decision).
- **`mergeStrategy.blockingTags: ["general"]`** + token-level `tokenTags: ["general"]`
  enforces "at most one general per stack".
- **`hasAtMostTagsInArmy:general,1`** in the activation requirement enforces the same
  constraint at activation time.
- **`NOT isCloseCombat`** in both activation and removal requirements means generals
  can't be deployed or recalled mid-fight.
- **Source of truth for buff values** = Confluence page 6075973642. Refetch each run.
- **Effect description rule (the 5 rules above)** = the lived-in convention from the
  original authoring pass; not in any other document.
