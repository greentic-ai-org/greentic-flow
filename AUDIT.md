# AUDIT.md — PR-FLOW-03 Stage 1 Wizard Audit
Date: 2026-03-02

## Scope summary
This audit covers wizard-related behavior in `greentic-flow` CLI and runtime code paths, including menu wizard (`greentic-flow wizard`) and wizard-enabled step commands (`add-step`, `update-step`, `delete-step`).

## A) CLI surface

### Exact help output (captured)

#### `greentic-flow --help`
```text
Flow scaffolding helpers

Usage: greentic-flow [OPTIONS] <COMMAND>

Commands:
  new             Create a new flow skeleton at the given path
  update          Update flow metadata in-place without overwriting nodes
  add-step        Insert a step after an anchor node
  update-step     Update an existing node (re-run config/default with overrides)
  delete-step     Delete a node and optionally splice routing
  doctor          Validate flows
  doctor-answers  Validate answers JSON against a schema
  answers         Emit JSON schema + example answers for a component operation
  bind-component  Attach or repair a sidecar component binding without changing flow nodes
  wizard          Wizard flow helpers (interactive by default)
  help            Print this message or the help of the given subcommand(s)

Options:
      --permissive       Enable permissive schema handling (default: strict)
      --format <FORMAT>  Output format (human or json) [default: human] [possible values: human, json]
      --locale <LOCALE>  Diagnostic locale (BCP47)
      --backup           Backup flow files before overwriting (suffix .bak)
  -h, --help             Print help
  -V, --version          Print version
```

#### `greentic-flow wizard --help`
```text
Wizard flow helpers (interactive by default)

Usage: greentic-flow wizard [OPTIONS] <PACK>

Arguments:
  <PACK>  Pack root directory

Options:
      --answers-file <ANSWERS_FILE>  Write wizard answers to a JSON file without prompting
      --permissive                   Enable permissive schema handling (default: strict)
      --dry-run                      Validate and run doctor, but do not persist flow mutations
      --format <FORMAT>              Output format (human or json) [default: human] [possible values: human, json]
      --locale <LOCALE>              Diagnostic locale (BCP47)
      --backup                       Backup flow files before overwriting (suffix .bak)
  -h, --help                         Print help
```

#### Wizard-enabled flags on step commands (captured)
- `add-step --help`: includes `--wizard-mode`, `--answers`, `--answers-file`, `--answers-dir`, `--overwrite-answers`, `--reask`, `--locale`, `--interactive`, and `--validate-only`.
- `update-step --help`: includes `--wizard-mode`, `--answers`, `--answers-file`, `--answers-dir`, `--overwrite-answers`, `--reask`, `--locale`, `--non-interactive`, `--interactive`.
- `delete-step --help`: includes `--wizard-mode`, `--answers`, `--answers-file`, `--answers-dir`, `--overwrite-answers`, `--reask`, `--locale`, `--interactive`.

### Findings table
| Item | Current behavior | Files/lines |
|---|---|---|
| Wizard command path(s) | Top-level CLI subcommand `wizard` is a menu wizard with no clap subcommands. Separate wizard-enabled flows are embedded in `add-step` / `update-step` / `delete-step` when `component_id` or `--wizard-mode` is provided. | `src/bin/greentic-flow.rs:512-533`, `src/bin/greentic-flow.rs:536-545`, `src/bin/greentic-flow.rs:7189-7197`, `src/bin/greentic-flow.rs:7630-7636`, `src/bin/greentic-flow.rs:7997-8003` |
| Flags for locale | Global `--locale` exists; wizard step commands also expose local `--locale` for prompt localization. | `src/bin/greentic-flow.rs:495-497`, `src/bin/greentic-flow.rs:707-709`, `src/bin/greentic-flow.rs:787-788` |
| Flags for answers import/export | No `--emit-answers` and no AnswerDocument envelope. Menu wizard has `--answers-file` (preload + save path). Step commands support `--answers` and `--answers-file` (JSON/YAML object merge), write per-step artifacts via `answers::write_answers`. | `src/bin/greentic-flow.rs:539-541`, `src/bin/greentic-flow.rs:940-947`, `src/bin/greentic-flow.rs:1325-1419`, `src/bin/greentic-flow.rs:692-700`, `src/bin/greentic-flow.rs:771-779`, `src/bin/greentic-flow.rs:8294-8319`, `src/answers.rs:13-17`, `src/answers.rs:20-64` |
| Validate/apply split | No wizard-level `validate`/`apply` subcommands. `add-step` has `--validate-only`; `update-step`/`delete-step` do not expose equivalent validation-only for wizard mode. | `src/bin/greentic-flow.rs:6970-6971`, `src/bin/greentic-flow.rs:7328-7339`, `src/bin/greentic-flow.rs:7549-7557` |
| Non-zero exit handling | Errors bubble via `Result<()>` and `anyhow::bail!`, producing process failure. Tests assert `.failure()` for invalid inputs/missing required answers. | `src/bin/greentic-flow.rs:846-885`, `src/bin/greentic-flow.rs:6846-6859`, `src/bin/greentic-flow.rs:8299-8317`, `tests/flow_cli_ops.rs:1286-1312`, `tests/flow_cli_ops.rs:1969-2015` |

## B) Schema + questions

### Findings table
| Item | Current approach | Files/lines |
|---|---|---|
| Schema identity | No wizard `schema_id`/`wizard_id` for answer payloads. Only unrelated `schema_id` appears in `FlowBundle::NodeRef` and defaults to `None`. | `src/flow_bundle.rs:24-28`, `src/flow_bundle.rs:163-167` |
| Schema versioning | No wizard answer schema version field. Flow document uses numeric `schema_version` (default 2). QA form built in runtime uses fixed `version: "0.6.0"` for generated `FormSpec`. | `src/bin/greentic-flow.rs:564-566`, `src/wizard/mod.rs:204`, `src/qa_runner.rs:241-245` |
| Question model | Two question systems: (1) generic flow/config questions from manifest or config-flow mapped to `Question` struct; (2) component QA spec (`ComponentQaSpec`) decoded from wizard ops and converted to `FormSpec`. | `src/questions.rs:15-25`, `src/questions.rs:130-187`, `src/wizard_ops.rs:57-62`, `src/wizard_ops.rs:698-737`, `src/qa_runner.rs:195-253` |
| Validation rules | Required-key checks via `validate_required`; qa-spec validation via `qa_spec::validate`; payload validation through component schema when available. | `src/questions.rs:59-67`, `src/qa_runner.rs:141-183`, `src/bin/greentic-flow.rs:7470-7481`, `src/bin/greentic-flow.rs:7921-7933` |
| Defaults | Defaults are merged from question/default definitions before prompting; optional null seeding exists for default wizard mode on component QA. | `src/questions.rs:93-104`, `src/wizard_ops.rs:285`, `src/wizard_ops.rs:724`, `src/bin/greentic-flow.rs:7228-7231`, `src/bin/greentic-flow.rs:7675-7678`, `src/bin/greentic-flow.rs:8322-8333` |
| i18n keys | Wizard UI strings use key-based lookup (`wizard.*`) from embedded `i18n/wizard/*.json`. Component QA labels/help are i18n text resolved by locale. | `src/bin/greentic-flow.rs:19`, `src/bin/greentic-flow.rs:3558-3573`, `src/bin/greentic-flow.rs:1465-1469`, `src/i18n.rs:27-40`, `src/i18n.rs:73-104`, `src/qa_runner.rs:48-60`, `i18n/wizard/en-GB.json` |

## C) Plan / execute / migrate

### Findings table
| Item | Current approach | Files/lines |
|---|---|---|
| Plan representation | Active CLI wizard paths are mostly direct execution with in-memory transforms then write. There is an older scaffold planner (`WizardPlan`) in `src/wizard/mod.rs`, used by tests but not wired to CLI command dispatch. | `src/wizard/mod.rs:24-27`, `src/wizard/mod.rs:168-241`, `src/wizard/mod.rs:247-278`, `src/bin/greentic-flow.rs:869-885`, `tests/wizard_scaffold.rs:22-37` |
| Apply/execution | Wizard-enabled step commands resolve component, gather answers, call `apply_wizard_answers`, mutate flow IR, and write flow + sidecar + answers artifacts. Menu wizard stages pack edits in temp dir and writes on save. | `src/bin/greentic-flow.rs:7238-7264`, `src/bin/greentic-flow.rs:7302-7371`, `src/bin/greentic-flow.rs:7685-7779`, `src/bin/greentic-flow.rs:8052-8102`, `src/bin/greentic-flow.rs:933-1021`, `src/bin/greentic-flow.rs:1221-1262` |
| Validation-only path | `add-step` supports `--validate-only` (wizard and non-wizard path). Menu wizard runs doctor before save. No dedicated `wizard validate` command. | `src/bin/greentic-flow.rs:6970-6971`, `src/bin/greentic-flow.rs:7328-7339`, `src/bin/greentic-flow.rs:7549-7557`, `src/bin/greentic-flow.rs:1240-1250` |
| Migration | No AnswerDocument migration mechanism. Existing compatibility only for legacy mode name `upgrade` -> `update` in state and fallback answer artifact lookup. | `src/bin/greentic-flow.rs:6793-6825`, `src/wizard_state.rs:43-47`, `src/wizard_state.rs:59-60` |
| Locks/reproducibility | No explicit `locks` object for answers. Reproducibility approximated via sidecar source metadata and optional digests, plus persisted per-step answer artifacts (`*.answers.json` + `*.answers.cbor`). | `src/answers.rs:13-17`, `src/answers.rs:20-64`, `src/bin/greentic-flow.rs:7355-7370`, `src/bin/greentic-flow.rs:7764-7779`, `src/bin/greentic-flow.rs:9075-9134` |

## D) Delegation model (repo -> external systems)
- Component wizard operations are delegated to component WASM exports (`component-qa` / descriptor / apply-answers) via `wizard_ops`.
  - `src/wizard_ops.rs:698-737`
- Component reference resolution can delegate to distributor APIs (`HttpDistributorClient`) for `component_id` resolution and remote artifact lookup.
  - `src/bin/greentic-flow.rs:9075-9134`, `src/bin/greentic-flow.rs:8757-8786`
- Translation generation in menu wizard delegates to `greentic_i18n_translator` CLI module in-process (not a shell-out).
  - `src/bin/greentic-flow.rs:3070-3088`

## E) Tests touching wizard paths

| Test | What it covers | Command |
|---|---|---|
| `tests/flow_cli_ops.rs::wizard_help_renders_with_pack_entrypoint` | Wizard command surface/help | `cargo test --test flow_cli_ops wizard_help_renders_with_pack_entrypoint -- --nocapture` |
| `tests/flow_cli_ops.rs::wizard_menu_allows_exit_from_main_menu` | Menu wizard interactive entry/exit | `cargo test --test flow_cli_ops wizard_menu_allows_exit_from_main_menu -- --nocapture` |
| `tests/flow_cli_ops.rs::add_step_wizard_uses_fixture_resolver` | Wizard-mode add-step with fixture resolver and apply flow mutation | `cargo test --test flow_cli_ops add_step_wizard_uses_fixture_resolver -- --nocapture` |
| `tests/flow_cli_ops.rs::add_step_default_fixture_prompts_and_applies_answers` | Prompting + answer application in add-step flow | `cargo test --test flow_cli_ops add_step_default_fixture_prompts_and_applies_answers -- --nocapture` |
| `tests/flow_cli_ops.rs::update_step_non_interactive_missing_required_reports_template` | Non-interactive required-answer failure behavior | `cargo test --test flow_cli_ops update_step_non_interactive_missing_required_reports_template -- --nocapture` |
| `tests/wizard_scaffold.rs::*` | Legacy scaffold provider plan generation/execution (`WizardPlan`) | `cargo test --test wizard_scaffold -- --nocapture` |
| `src/bin/greentic-flow.rs` internal tests around lines ~4960-6383 | Menu wizard save/export/update/delete flows and wizard mode alias behavior | `cargo test --bin greentic-flow -- --nocapture` |

## Constraints / compatibility requirements for Stage 2
1. Existing CLI behavior heavily centers on `add-step/update-step/delete-step` wizard flags plus the separate menu wizard command; introducing `wizard run/validate/apply` needs compatibility handling.
2. Existing answer ingestion is raw object merge (`--answers` / `--answers-file`) and persisted per-step files, not a stable document envelope.
3. There is already backward-compatibility logic for legacy mode `upgrade`; new schema/version migration must preserve this behavior.
4. i18n is already key-based for wizard UI and component QA texts; answers are value-based and not localized labels.
5. There is no current explicit locks model in answer artifacts, so adding one should avoid breaking existing read paths.
