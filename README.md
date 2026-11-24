# development-guidelines

A Claude Code plugin with coding guidelines that Claude loads when writing, refactoring or reviewing code.

| Skill | Loads when | Holds |
| --- | --- | --- |
| `core-guidelines` | any code work | language-neutral rules and the review checklist |
| `go-guidelines` | Go code | Go rules; Do and Don't examples in `references/examples.md` |

## Install

```sh
claude plugin marketplace add chrisguiney/development-guidelines
claude plugin install development-guidelines@development-guidelines
```

Update after a new release:

```sh
claude plugin marketplace update development-guidelines
```

## Layout

```text
.claude-plugin/marketplace.json
plugins/development-guidelines/
  .claude-plugin/plugin.json
  skills/
    core-guidelines/SKILL.md
    go-guidelines/SKILL.md
    go-guidelines/references/examples.md
```

## Adding a language

1. Create `skills/<language>-guidelines/SKILL.md` with a `description` naming the language, its file extensions and the topics it covers.
2. State how each core rule applies in that language, plus rules that exist only there. Keep `SKILL.md` to rules; put every Do and Don't pair in `references/examples.md`, one section per rule, and link to it.
3. Add the skill to the table above and bump `version` in `plugin.json`.

## License

Apache License 2.0; see [LICENSE](LICENSE).
