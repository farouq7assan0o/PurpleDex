# command-reference - architecture & rules for any session

Offline pentest/blue-team command library. Static site: `index.html` + `js/` + `css/`.

## THE ONE RULE THAT MATTERS MOST

**`js/commands.js` is a GENERATED BUILD ARTIFACT. NEVER edit it by hand.**

- The source of truth is the per-card JSON under `commands/**/*.json` (one file = one card).
- `js/commands.js` is produced by `node build-commands.js`, which walks `commands/**`, and **overwrites `commands.js` completely from the JSON**.
- Therefore ANY edit made directly to `js/commands.js` - a new card, a new field, a rename, a dash cleanup - is **silently and permanently destroyed** the next time anyone runs the build.
- This has already caused a large, real data loss (15 cards + structured detection + interview Q&A + ~35 CRTP variations, all written only to the bundle, all wiped on the next rebuild). Do not repeat it.

**To add or change a card: edit its JSON file under `commands/`, then run `node build-commands.js`.**
If a card's JSON file does not exist yet, create it under the appropriate `commands/` subfolder.

## Guardrails (already wired - use them)

- `node build-commands.js --check` - builds in memory and **fails (exit 1) if `js/commands.js` does not match a fresh build from the JSON source.** Run this before committing. If it fails, either your bundle was hand-edited (move the change into the JSON source or lose it) or the bundle is stale (rebuild).
- `npm run guard` - runs the drift check + schema validation together.
- `.githooks/pre-commit` - blocks commits on drift/schema failure. Enable once with: `git config core.hooksPath .githooks`
- `node validate.js` - schema validator (reads the JSON source, not the bundle). Must PASS.

## Verify-your-writes discipline

When editing card JSON with a script, do NOT trust that a write "said success":
read -> `JSON.parse` -> mutate the object -> `JSON.stringify(obj, null, 2) + "\n"` -> write -> **re-read, re-parse, and assert the change is actually present** (e.g. array length grew). Prefer object mutation over regex/string insertion into JSON. The prior data loss involved insertions that reported success but landed in the wrong place or only in the bundle.

## Durability

Work only survives if it is **committed**. Large edits left uncommitted (especially inside a forked/sandboxed session) are invisible to every other session and are lost on rebuild or fork. Commit after each batch; if in a fork, push to a branch.

## Content conventions

- **SIEM detection fields are FLAT siblings inside each card's `defense` object:** `splunk_spl`, `elastic_kql`, `sentinel_kql`, `sigma_rules` (all strings), placed right after `artifacts`. `sentinel_kql` is Microsoft Sentinel/Defender **Kusto** (over `SecurityEvent`, `DeviceProcessEvents`, `W3CIISLog`, etc.) - a different language from `elastic_kql` (Kibana filter syntax); do not duplicate them.
- **No en-dashes or em-dashes (or other non-ASCII punctuation) anywhere in card content. Plain ASCII hyphen `-` only.**
- Placeholders use canonical tokens from `js/vars.js` (`<ip>`, `<dc_ip>`, `<domain>`, `<target>`, ...). `canonVar()` is case-insensitive. A per-command local value that should NOT be filled from the global context bar must use a token that is NOT in the registry (e.g. `<target_account>`).
- Defense-eligible card types: `command`, `payload`, `attack-chain`. `reference`/`cheatsheet`/`script`/`resource` are exempt. Cards retired in place are set to `{"_ignore": true}` (the build skips them).
- **Structured detections (Phase 0+):** `defense.detections[]` is an optional array of typed detection objects. Each detection requires: `platform` (splunk|elastic|sentinel|sigma), `name`, `logic` (the actual query/rule), `data_source` (from the canonical list in validate.js), `attack[]` (MITRE IDs - must match the card's `mitre[]`), `fidelity` (behavioral|signature|telemetry). Optional: `confidence`, `log_ids[]`, `false_positives`, `tuning`. `defense.visibility` is an optional object with `requires[]` (mandatory log sources) and `better_with[]` (enhancement sources). The flat SIEM fields (`splunk_spl` etc.) remain - detections are additive. The build emits `js/coverage.json` (technique-to-detection and data-source-to-technique indices) from the structured detections.
- **Purple-team validation:** `defense.validation` is an optional object with: `expected_events[]` (array of `{source, event_id?, description}` - what logs SHOULD appear when the attack runs), `success_criteria` (string - when does the detection pass), `test_command` (string - command to trigger the detection), `response_steps[]` (array of strings - investigation playbook steps). Rendered in the Defend tab under "Purple Team Validation".

## Build / validate loop

```
node build-commands.js        # regenerate bundle from JSON (auto-runs validate)
node build-commands.js --check  # fail if bundle != source (pre-commit gate)
node validate.js              # schema check on the JSON source
```
