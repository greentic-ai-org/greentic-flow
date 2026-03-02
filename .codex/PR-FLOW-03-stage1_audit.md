# greentic-flow: Stage 1 — Audit (Wizard schema/plan/execute/migrate + delegation)

**Purpose:** Produce a factual snapshot of the current wizard implementation so Stage 2 can be surgical.
**Output artifacts:** `AUDIT.md` + filled tables below + exact file/line references.

## Scope
- Identify existing wizard entrypoints (CLI, subcommands, modules)
- Identify current schema representation (if any) and how questions are defined
- Identify execution model: plan/apply/side effects; validate vs apply
- Identify i18n handling and locale resolution
- Identify any existing non-interactive answer support (flags/files/env)
- Identify how this repo delegates to other binaries (if applicable)
- Identify tests/snapshots covering wizard paths

## Repo description
Owns flow-level wizards (schema/plan/execute/migrate) + i18n prompting

## Audit checklist

### A. CLI surface
- [x] List `wizard` subcommands and flags (copy exact help output)
- [x] Note any existing `--locale`, `--answers`, `--emit-answers` or equivalents
- [x] Note default mode (interactive vs non-interactive) and how it decides

**Findings table (fill in):**
| Item | Current behavior | Files/lines |
|---|---|---|
| Wizard command path(s) | `greentic-flow wizard` is menu-based and has no clap subcommands. Wizard execution also exists in `add-step` / `update-step` / `delete-step` when `component_id` or `--wizard-mode` is supplied. | `src/bin/greentic-flow.rs:512-533`, `src/bin/greentic-flow.rs:536-545`, `src/bin/greentic-flow.rs:7189-7197`, `src/bin/greentic-flow.rs:7630-7636`, `src/bin/greentic-flow.rs:7997-8003` |
| Flags for locale | Global `--locale` plus wizard-step `--locale`. | `src/bin/greentic-flow.rs:495-497`, `src/bin/greentic-flow.rs:707-709`, `src/bin/greentic-flow.rs:787-788` |
| Flags for answers import/export | No `--emit-answers`. Supports `--answers`/`--answers-file` for step commands, plus menu-wizard `--answers-file`; writes per-step artifacts to `*.answers.json` and `*.answers.cbor`. | `src/bin/greentic-flow.rs:539-541`, `src/bin/greentic-flow.rs:692-700`, `src/bin/greentic-flow.rs:771-779`, `src/bin/greentic-flow.rs:8294-8319`, `src/answers.rs:13-17`, `src/answers.rs:20-64` |
| Validate/apply split | No wizard `validate`/`apply` command split today. `add-step` has `--validate-only`; `update-step` and `delete-step` do not expose equivalent validate-only flow. | `src/bin/greentic-flow.rs:6970-6971`, `src/bin/greentic-flow.rs:7328-7339`, `src/bin/greentic-flow.rs:7549-7557` |
| Non-zero exit handling | Errors return via `Result` + `anyhow::bail!`; tests verify failure exit for invalid/non-interactive missing answers. | `src/bin/greentic-flow.rs:846-885`, `src/bin/greentic-flow.rs:6846-6859`, `tests/flow_cli_ops.rs:1286-1312`, `tests/flow_cli_ops.rs:1969-2015` |

### B. Schema + questions
- [x] Where are questions defined (data-driven JSON, Rust structs, macros, etc.)?
- [x] Is there a schema_id/schema_version today? If yes, where?
- [x] Is there an i18n key system already (en.json tags)?

**Findings table (fill in):**
| Item | Current approach | Files/lines |
|---|---|---|
| Schema identity | No wizard answer `schema_id`/`wizard_id` envelope exists. (`schema_id` appears in `FlowBundle::NodeRef` only, set `None`.) | `src/flow_bundle.rs:24-28`, `src/flow_bundle.rs:163-167` |
| Schema versioning | No wizard answer schema version field. Flow docs use numeric `schema_version`; QA form hardcodes `version: "0.6.0"`. | `src/bin/greentic-flow.rs:564-566`, `src/wizard/mod.rs:204`, `src/qa_runner.rs:241-245` |
| Question model | Two models: generic `Question` from manifest/config-flow graph and component wizard `ComponentQaSpec` decoded from component exports. | `src/questions.rs:15-25`, `src/questions.rs:130-187`, `src/wizard_ops.rs:57-62`, `src/wizard_ops.rs:698-737`, `src/qa_runner.rs:195-253` |
| Validation rules | Required validation + QA form validation + payload schema validation (when schema available). | `src/questions.rs:59-67`, `src/qa_runner.rs:141-183`, `src/bin/greentic-flow.rs:7470-7481`, `src/bin/greentic-flow.rs:7921-7933` |
| Defaults | Defaults merged from question specs; optional null-seeding used for default wizard mode in component QA. | `src/questions.rs:93-104`, `src/wizard_ops.rs:724`, `src/bin/greentic-flow.rs:7228-7231`, `src/bin/greentic-flow.rs:8322-8333` |
| i18n keys | Yes: key-based wizard strings (`wizard.*`) in embedded `i18n/wizard/*.json`; locale resolution/fallback in `i18n.rs`; QA labels resolved by locale. | `src/bin/greentic-flow.rs:19`, `src/bin/greentic-flow.rs:3558-3573`, `src/i18n.rs:27-40`, `src/i18n.rs:73-104`, `src/qa_runner.rs:48-60`, `i18n/wizard/en-GB.json` |

### C. Plan/execute/migrate
- [x] How does wizard output become actions? (plan object? direct side-effects?)
- [x] Where are execution side effects performed?
- [x] Is there any migration capability today?

**Findings table (fill in):**
| Item | Current approach | Files/lines |
|---|---|---|
| Plan representation | Active CLI wizard paths do direct mutation/write; legacy `WizardPlan` exists in `src/wizard/mod.rs` and is test-covered, but not CLI-wired. | `src/wizard/mod.rs:24-27`, `src/wizard/mod.rs:168-241`, `src/wizard/mod.rs:247-278`, `src/bin/greentic-flow.rs:869-885`, `tests/wizard_scaffold.rs:22-37` |
| Apply/execution | Wizard modes call `apply_wizard_answers`, mutate flow IR, then write flow + sidecar + answers artifacts; menu wizard stages edits then saves. | `src/bin/greentic-flow.rs:7238-7264`, `src/bin/greentic-flow.rs:7302-7371`, `src/bin/greentic-flow.rs:7685-7779`, `src/bin/greentic-flow.rs:8052-8102`, `src/bin/greentic-flow.rs:933-1021`, `src/bin/greentic-flow.rs:1221-1262` |
| Validation-only path | `add-step --validate-only` exists; no dedicated `wizard validate` command today. | `src/bin/greentic-flow.rs:6970-6971`, `src/bin/greentic-flow.rs:7328-7339`, `src/bin/greentic-flow.rs:7549-7557` |
| Migration | No AnswerDocument migration. Only legacy `upgrade` mode compatibility in persisted state and answers-file lookup fallback. | `src/bin/greentic-flow.rs:6793-6825`, `src/wizard_state.rs:43-47`, `src/wizard_state.rs:59-60` |
| Locks/reproducibility | No answer `locks` envelope today. Reproducibility uses sidecar/source metadata + optional digest + persisted `answers` JSON/CBOR artifacts. | `src/answers.rs:20-64`, `src/bin/greentic-flow.rs:7355-7370`, `src/bin/greentic-flow.rs:7764-7779`, `src/bin/greentic-flow.rs:9075-9134` |

### D. Tests
- [x] Identify unit/snapshot/e2e tests that touch wizard paths
- [x] Capture how to run them locally

**Findings table (fill in):**
| Test | What it covers | Command |
|---|---|---|
| `tests/flow_cli_ops.rs::wizard_help_renders_with_pack_entrypoint` | Wizard CLI help shape | `cargo test --test flow_cli_ops wizard_help_renders_with_pack_entrypoint -- --nocapture` |
| `tests/flow_cli_ops.rs::wizard_menu_allows_exit_from_main_menu` | Menu wizard interactive entry/exit | `cargo test --test flow_cli_ops wizard_menu_allows_exit_from_main_menu -- --nocapture` |
| `tests/flow_cli_ops.rs::add_step_wizard_uses_fixture_resolver` | Wizard-mode add-step + fixture resolver/apply behavior | `cargo test --test flow_cli_ops add_step_wizard_uses_fixture_resolver -- --nocapture` |
| `tests/flow_cli_ops.rs::add_step_default_fixture_prompts_and_applies_answers` | Prompting + answer application in add-step path | `cargo test --test flow_cli_ops add_step_default_fixture_prompts_and_applies_answers -- --nocapture` |
| `tests/flow_cli_ops.rs::update_step_non_interactive_missing_required_reports_template` | Non-interactive missing-required behavior | `cargo test --test flow_cli_ops update_step_non_interactive_missing_required_reports_template -- --nocapture` |
| `tests/wizard_scaffold.rs::*` | Legacy scaffold wizard planner/apply API | `cargo test --test wizard_scaffold -- --nocapture` |
| `src/bin/greentic-flow.rs` internal tests around ~4960-6383 | Menu wizard save/export/delete/update + aliasing coverage | `cargo test --bin greentic-flow -- --nocapture` |

### Ripgrep audit patterns
Run these from repo root (adjust paths if needed):

```bash
rg -n "wizard|qa|question|prompt|interactive|inquirer|dialoguer|clap.*wizard|subcommand.*wizard" .
rg -n "schema(_id|_version)?|AnswerDocument|answers\.json|emit-answers|--answers|--locale|i18n" .
rg -n "apply_answers|setup|default|install|provision|plan|execute|validate|migrate" .
```

## Deliverables
1) `AUDIT.md` with:
   - Exact CLI help output (copy/paste)
   - File/line references for all key paths
   - Completed tables above
2) Notes on any constraints or compatibility requirements

## Stop condition
**Do not implement changes in Stage 1.** Only gather facts needed for Stage 2.
