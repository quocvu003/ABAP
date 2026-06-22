# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A knowledge base for AI-assisted SAP ABAP S/4HANA migration. There is no build system, no test suite, and no runnable code. All work is editing Markdown documentation and plain-text prompt files.

## How to use this repo when helping a developer fix ABAP code

1. Read `docs/reference/rules/{ERROR_ID}.md` directly (e.g. `docs/reference/rules/T019.md`). One file = one rule = one Read call.
2. Apply the fix rule **exactly** — naming conventions, comment markers, pseudo-comments, and output format are all mandatory.
3. **Never guess** when information is missing (field type/length, table definition, business logic, ATC detail, SAP Note). Ask the user first.

## Key files

| File | Purpose |
| --- | --- |
| `prompts/abap_fixer_prompt.txt` | Vietnamese-language system prompt. Paste this into the AI context before the user's code. |
| `docs/reference/rules/{ID}.md` | **Primary lookup** — one file per error ID (B001, B002, B014, B016, T003, T007, T010, T019, T020, T021, D001, F014). Read the exact file needed. |
| `docs/reference/sap_abap_error_reference.md` | Full reference (sections 6–11: naming conventions, coding rules, migration best practices). Fallback if rule file not found. |

## Mandatory output format (every code fix)

Every response must have exactly two parts — no exceptions:

1. **Line-by-line mapping table** (Markdown): `ECC code | S/4HANA code | Reason / Rule`
2. **Complete ABAP source block** (` ```abap ``` `) with all migration markers in place.

Never use Git-style diff format (`-`/`+` lines).

## Migration comment markers

All code changes must be wrapped with these markers (Japanese language for inline logic comments):

```
* S/4HANA Migration ADD START   ← new logic
* S/4HANA Migration ADD END

* S/4HANA Migration UPD START   ← changed logic (old lines commented out above new lines)
* S/4HANA Migration UPD END

* S/4HANA Migration DEL START   ← deleted logic (commented out)
* S/4HANA Migration DEL END
```

Add the program header **only** when the user provides a complete program (contains `REPORT` or `PROGRAM` statement):
```abap
*----------------------------------------------------------------------*
* 修正履歴
* 日付         ユーザ    内容
* yyyy/mm/dd  KSC VU.NQ    S/4HANA Migration
*----------------------------------------------------------------------*
```

## Naming conventions (Section 6 of the reference)

All variable names UPPERCASE. Required prefixes:

| Scope | Variable | Structure | Table | Constant |
|-------|----------|-----------|-------|----------|
| Local | `LV_` | `LS_` | `LT_` | `CNS_` |
| Global | `GV_` | `GS_` | `GT_` | `CNS_` |

Before declaring a new variable in an `ADD START` block, check existing DATA declarations in the same FORM/program — reuse a variable of the same type if possible instead of declaring a new one.

## Supported error IDs

`B001` `B002` `B014` `B016` `T003` `T007` `T010` `T019` `T020` `T021` `D001` `F014`

For any other error ID, do not attempt a fix — ask the user for the applicable rule or SAP Note first.

