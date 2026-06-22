# ABAP Helper

A knowledge base and prompt set for AI-assisted SAP ABAP S/4HANA migration fixes.

## Project Structure

```text
abap-helper/
├── CLAUDE.md                          # Claude Code instructions (AI runtime config)
├── prompts/
│   └── abap_fixer_prompt.txt          # System prompt for standalone AI web usage
└── docs/
    └── reference/
        ├── rules/                     # Per-error-ID fix rules (primary lookup)
        │   ├── B001.md  B002.md  B014.md  B016.md
        │   ├── T003.md  T007.md  T010.md  T019.md  T020.md  T021.md
        │   ├── D001.md  F014.md
        └── sap_abap_error_reference.md  # General rules: B999, F999, sections 6–11
```

## Usage

### Claude Code (CLI)

Open this repo in Claude Code. Per `CLAUDE.md`, when a developer submits code with an error ID, Claude reads `docs/reference/rules/{ID}.md` directly and applies the fix.

### Standalone AI (Claude.ai, ChatGPT, etc.)

Paste `prompts/abap_fixer_prompt.txt` into the chat before submitting the ABAP code and error ID.

## Supported error IDs

`B001` `B002` `B014` `B016` `T003` `T007` `T010` `T019` `T020` `T021` `D001` `F014`

For other error IDs, consult the Teamlead or the applicable SAP Note first.
