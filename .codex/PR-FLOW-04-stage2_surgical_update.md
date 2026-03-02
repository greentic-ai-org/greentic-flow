# greentic-flow: Stage 2 — Surgical Update (apply audit inputs)

    **Pre-req:** Stage 1 audit completed and its findings copied into the “Audit Inputs” section below.

    ## Objective
    Implement the minimal set of changes to support:
    - Stable AnswerDocument import/export (envelope)
    - Schema identity + version (`schema_id`, `schema_version`) for this wizard
    - Non-interactive execution via `--answers <file>`
    - Optional migration via `--migrate`
    - i18n keys (schema uses keys; labels resolved by locale)
    - Preserve/alias existing CLI paths where required

    ## Repo description
    Owns flow-level wizards (schema/plan/execute/migrate) + i18n prompting

    ## Audit Inputs (paste from Stage 1)
    Fill these **before coding**:

    - Wizard command path(s): `src/bin/greentic-flow.rs:512-533`, `src/bin/greentic-flow.rs:923-1129`, `src/bin/greentic-flow.rs:7189-7197`, `src/bin/greentic-flow.rs:7630-7636`, `src/bin/greentic-flow.rs:7997-8003`
    - Current flags (locale/answers): global/local `--locale`; `--answers`, `--answers-file`, `--answers-dir` on step commands; no prior `--emit-answers` on wizard paths.
    - Schema location/model: scaffold wizard questions in `src/wizard/mod.rs:86-166`; generic question model in `src/questions.rs:15-25`; component QA model in `src/qa_runner.rs:195-253`.
    - Execution model (plan/apply): scaffold provider returns `WizardPlan` (`src/wizard/mod.rs:168-243`) and executes via `execute_plan` (`src/wizard/mod.rs:247-275`); step wizards execute directly in `src/bin/greentic-flow.rs:7189-8205`.
    - Tests to update/add: `tests/flow_cli_ops.rs` wizard command tests + `tests/answers.rs` parsing tests.

    ## Proposed changes (minimal)
    ### 1) Add/standardize AnswerDocument envelope
    - Implement (or adapt) a small struct matching:
      - `wizard_id`, `schema_id`, `schema_version`, `locale`, `answers`, `locks`
    - Ensure read/write JSON is stable and documented.
    - Do **not** centralize in a shared repo unless later desired.

    ### 2) CLI flags + semantics (surgical)
    Implement/alias:
    - `--answers <FILE>`: load AnswerDocument; run non-interactive validate/apply
    - `--emit-answers <FILE>`: write AnswerDocument produced (interactive or merged)
    - `--schema-version <VER>`: pin version for interactive mode
    - `--migrate`: if AnswerDocument version older, migrate (and optionally re-emit)

    **Compatibility rule:** if existing flags already exist, keep them and add aliases.

    ### 3) Schema identity + versioning
    - Define stable identifiers:
      - `wizard_id`: e.g. `greentic-flow.wizard.<purpose>`
      - `schema_id`: e.g. `greentic-flow.<purpose>`
      - `schema_version`: start at `1.0.0` unless audit shows existing versioning
    - Ensure interactive renders and validators emit these IDs/versions into AnswerDocument.

    ### 4) Validate vs apply split
    Prefer separate subcommands:
    - `wizard validate --answers ...`
    - `wizard apply --answers ...`
    If existing model differs, adapt with minimal surface changes but keep semantics.

    ### 5) Migration
    - If breaking changes are present or anticipated, add a migration function:
      - input: old AnswerDocument
      - output: new AnswerDocument
    - If no breaking change yet, implement a stub that returns identity but wires the mechanism.

    ### 6) i18n wiring
    - Ensure schema/question definitions use i18n keys
    - Ensure `--locale` controls resolution only (answers stay stable)

    ## Acceptance criteria
    - [x] `wizard run` interactive still works
    - [x] `wizard validate --answers answers.json` works (no side effects)
    - [x] `wizard apply --answers answers.json` works (side effects)
    - [x] `wizard run --emit-answers out.json` produces AnswerDocument with correct ids/versions
    - [x] `wizard validate --answers old.json --migrate` succeeds (if old version) and can re-emit migrated doc
    - [x] Tests updated/added per audit notes, minimal and focused

    ## Implementation notes (apply from audit)
    - **Files to touch (from audit):**
      - `src/answer_document.rs`
      - `src/lib.rs`
      - `src/bin/greentic-flow.rs`
    - **Tests to touch/add (from audit):**
      - `tests/flow_cli_ops.rs`
      - `tests/answers.rs`

    ## Risk controls
    - No large refactors; keep changes localized
    - Preserve existing UX defaults
    - Avoid schema mega-merges; keep nested docs for composition

    ## Common target behavior (all repos)

**Goal:** Standardize wizard execution and portability via a stable AnswerDocument envelope and consistent CLI semantics, while keeping schema ownership local to each wizard.

### AnswerDocument envelope (portable JSON)
```json
{
  "wizard_id": "greentic.pack.wizard.new",
  "schema_id": "greentic.pack.new",
  "schema_version": "1.1.0",
  "locale": "en-GB",
  "answers": { "...": "..." },
  "locks": { "...": "..." }
}
```

### Required CLI semantics
All wizards should converge on these flags and semantics (names can vary only if you provide compatibility aliases):
- `--locale <LOCALE>`: affects i18n rendering only; **answers must remain stable IDs/values**
- `--answers <FILE>`: non-interactive input (AnswerDocument)
- `--emit-answers <FILE>`: write AnswerDocument produced (interactive or merged)
- `--schema-version <VER>`: pin schema version used for interactive rendering/validation
- `--migrate`: allow automatic migration of older AnswerDocuments (including nested ones where applicable)
- Separate `validate` vs `apply` paths (subcommands or flags), recommended:
  - `wizard validate --answers ...`
  - `wizard apply --answers ...`

### Versioning rules
- Patch/minor: backwards compatible additions (defaults) only
- Major: breaking changes require migration logic
- Avoid flattening composed schemas into one mega-schema; prefer nested AnswerDocuments for composed flows.

### i18n rules
- Schema uses i18n keys; runtime resolves by locale
- Answers never depend on localized labels; only stable values/IDs

Date: 2026-03-02
