# sup-server-generals-skill

A [Claude Code](https://docs.claude.com/en/docs/claude-code) skill for authoring **generals** in [sup-server](https://github.com/bytro/sup-server) — Bytro's CoW (Call of War) server codebase, title 510.

Generates a complete character line (5 star-rank tokens + 4 cooldown tokens = **9 JSON files**) under `sup-server/content-items/510/token/`, following the Generals MVP spec and the lived-in conventions from the original authoring pass.

## What this skill does

Given a `(faction, rarity)` pair — for example `(pan_asian, legendary)` — the skill:

1. Pulls the current [Generals MVP - Abilities](https://bytrolabs.atlassian.net/wiki/spaces/LGCOW/pages/6075973642/Generals+MVP+-+Abilities) Confluence table.
2. Reads an existing canonical token file as a shape reference.
3. Scans `sup-server/content-items/510/token/` for the next free contiguous 9-ID block.
4. Writes the 9 JSON files with all the right fields, requirement expressions, cooldown wiring, and effect descriptions — applying 5 silent mapping rules to translate the Confluence row into JSON.
5. Self-validates against a 9-item checklist.
6. Reports back: what got written, where descriptions landed, what depends on the pending schema PR.

## Prerequisites

- **Claude Code** installed (see [docs](https://docs.claude.com/en/docs/claude-code/setup))
- **Atlassian MCP** configured — the skill fetches the Confluence buff table at runtime via `mcp__atlassian__getConfluencePage`. Without it, the skill cannot derive buff values.
- **A working copy of `sup-server`** checked out locally. The skill reads canonical token templates and writes new files under `content-items/510/token/`.
- **`gh` CLI authenticated** (only if you also want the skill to auto-create Jira / GitHub artifacts — not in v1).

## Install

Skills live under `~/.claude/skills/`. Drop the `create-general/` directory into that path:

```bash
# clone this repo to a working location
git clone https://github.com/JPSharrock-THI/sup-server-generals-skill.git
cd sup-server-generals-skill

# install the skill into your Claude Code skills directory
mkdir -p ~/.claude/skills
cp -r create-general ~/.claude/skills/
```

Verify:

```bash
ls ~/.claude/skills/create-general/SKILL.md
```

Claude Code picks up new skills automatically — no restart needed for the CLI. In an interactive session you may need to type `/help` or start a new conversation for the new skill to appear in the available list.

## Usage

In a Claude Code session inside your sup-server working copy, ask for a general:

> "Create the allies epic general"

Or, more specifically:

> "Author the Comintern Legendary line — all 5 stars + cooldowns"

Or for a backfill:

> "Fill in the missing 4★ and 5★ ranks for the Axis Free Epic general, the others are already done"

The skill kicks in automatically when the request matches its trigger description. It will:

- Confirm the next ID block before claiming it.
- Apply the 5 mapping rules silently and report the pair decisions in the final summary.
- Flag any rule-3 player/province effects whose `metadata.DESCRIPTION` ships ahead of the schema PR.

## The 5 mapping rules (preview)

These rules drive how Confluence-table rows map to the JSON structure. The full reasoning lives in [`create-general/SKILL.md`](create-general/SKILL.md); preview here:

1. **"Damage bonus" is a pair.** ATTACK + DEFENCE modifiers, same value, description on ATTACK only.
2. **Paired-penalty effects share a description.** "X but lose Y" → primary modifier holds the description, penalty does not.
3. **Player/province effects still need their own description.** Even though the schema doesn't yet support `metadata` on those buckets — ships ahead of code.
4. **Star scaling: 50 / 50+ / 75 / 75 / 100 + cooldown −30% at 5★.**
5. **JSON value, not Confluence-rounded value, drives description text.** `0.375` → "37.5%", not "37%".

## Out of scope (v1)

The v1 skill is deliberately file-generation-only. Future work could extend it:

- **i18n string catalogues.** Descriptions currently ship as raw English; localisation files must be updated separately.
- **Jira ticket creation.** No parent epic / per-line sub-tickets created automatically.
- **JUnit consistency test emission.** Self-validation is the manual checklist in step 7 of `SKILL.md`. A future v2 could emit a JUnit test that asserts ID/rank/cooldown consistency across all generals.
- **Cross-content-type ID-namespace verification.** Skill scans `token/` only. If the numeric ID namespace turns out to be shared with `unit/`, `mod/`, `premium/`, etc., a chosen block could collide — not yet verified.

PRs welcome.

## How the skill was developed

The skill was extracted from the original [authoring pass](https://bytrolabs.atlassian.net/browse/LGCOW-2432) that generated all 108 generals JSON files for sup-server. The conventions captured here are the lived-in ones, not the design-doc ones — they include corrections and decisions that emerged from PR review and content-team feedback during that pass.

See the workflow note in the [Twin Harbour Obsidian vault](#) (local, not public): *Creating Generals — Workflow and Skill Sketch*.

## License

Internal Bytro tooling — not licensed for external use.
