# Package Format

Status: current public package-authoring reference.

## Directory and Manifest

```
<package-id>/
  package.json
  lua/
  ue4ss/
  aa/
  js/
  ui/
```

Only manifest-declared files are runtime inputs. Use package-relative paths and `/` separators.

| Field          | Required | Meaning                               |
| -------------- | -------- | ------------------------------------- |
| `id`           | yes      | Stable ID and directory name          |
| `version`      | yes      | Version used by dependency resolution |
| `displayName`  | no       | Human-readable name                   |
| `description`  | no       | Short English description             |
| `dependencies` | no       | Explicit package dependencies         |
| `runtime`      | yes      | Files, public entries, and runnables  |
| `application`  | no       | Application JavaScript scripts (`finder`, `launcher`) |
| `ue4ss` | no | Support-package distribution: `{"frameworkRoot":"assets/ue4ss"}`; see [selection and validation](../../hexinton-engine-wiki/engine/ue4ss-package-runtime.md) |
| `frontend`     | no       | Hosted UI declarations                |
| `required`     | no       | `true` for a package players cannot switch off (restricted, see below) |

## Runtime Files and Runnables

```json
{
  "runtime": {
    "files": [
      { "path": "lua/main.lua", "runtime": "lua" },
      { "path": "aa/enable.aa", "runtime": "aa" }
    ],
    "runnables": [
      { "id": "enable", "kind": "enable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "enable" },
      { "id": "disable", "kind": "disable", "runtime": "lua", "entryFile": "lua/main.lua", "entrySymbol": "disable" },
      { "id": "hook", "kind": "action", "runtime": "aa", "entryFile": "aa/enable.aa" }
    ]
  }
}
```

Kinds are `enable`, `disable`, `action`, `query`, and `service`. Every package needs one functional enable and disable lifecycle entry. Auto Assembler is action-oriented. Application JavaScript is host-side and is not executed by the native engine runtime.

The native runtime names are `lua`, `aa`, and `ue4ss`. UE4SS uses Lua source and requires `entrySymbol`, just like host Lua; `runtime: "ue4ss"` selects the in-game interpreter. A package can mix runtime declarations, while retaining one shared lifecycle/dependency graph. See the [UE4SS manifest and API](../../hexinton-engine-wiki/engine/ue4ss-runtime.md#manifest-and-entry-functions) and [complete two-package example](../../hexinton-engine-wiki/engine/samples/ue4ss-player-location.md).

## Application Scripts

`application.scripts` declares host-side JavaScript that runs outside the game: a game `finder`
([Game Detection](../../hexinton-engine-wiki/application/game-detection.md)) or a game `launcher`
([Custom Game Launchers](../../hexinton-engine-wiki/application/custom-game-launchers.md)). Each entry has
`id`, `kind` (`finder` or `launcher`), `runtime: "js"`, `entryFile`, `entrySymbol`, and `capabilities`.
Package upload accepts `finder` and `launcher` only in `application.scripts`; any other kind there, or
these kinds in `runtime.runnables`, is rejected as `invalid_executable`. Capabilities are requests:
finder read capabilities and launcher capabilities are granted only to approved game-support creators
and first-party packages, so an ordinary upload that declares them is rejected at publication.

## Required Packages

A top-level `"required": true` marks a package that the game cannot work without, such as a game
launcher. Players cannot unsubscribe from it or switch it off, and it is installed for every player of
the game. The value must be a JSON boolean (`invalid_required_declaration` otherwise). Only first-party
packages and approved game-support creators may publish it; other uploads are rejected with
`required_package_not_authorized`. Omit the field for ordinary packages. See the
[launcher example](../../hexinton-engine-wiki/application/custom-game-launchers.md#a-required-launcher-package).

## Dependencies

```json
{
  "dependencies": [
    { "id": "trainer.shared", "compatible": "^1.0.0", "tested": "1.0.2" }
  ]
}
```

Dependencies enable before dependents and disable in reverse order. Consume only declared public values; private labels, allocations, timers, and hooks are not dependency API.

Native execution currently resolves `compatible` and ignores `tested`; cloud publication requires
both fields and verifies that the tested version satisfies the range. Retain both fields for an
uploadable package. See [script package dependencies](../../hexinton-engine-wiki/engine/script-packages.md#dependencies).

## Validation

The host checks strict JSON, matching directory and ID, declared files, unique runnable IDs, entry-file/runtime consistency, lifecycle entries, dependency constraints, and action parameters. Invalid packages produce structured errors. See [Errors and Results](../../hexinton-engine-wiki/engine/errors-and-results.md).

## User-facing text: six launch locales

Status: current rule, effective 2026-10-21.

Packages can ship the text players see in the six launch locales: English (`en`), Simplified
Chinese (`zh-Hans`), Japanese (`ja`), Korean (`ko`), German (`de`), and Spanish (`es`).

- **Your packages:** translations are optional. Studio and package upload never report a missing or
  partial locale as an error or a warning; players see the English text for anything not
  translated. Studio's AI agent writes the five translation files by default whenever it creates or
  edits player-facing text, and you can edit or delete them like any other file.
- **HexMod's own converted packages:** all six locales are required and complete; the conversion
  pipeline refuses to validate or upload a package with a gap.

**Manifest text.** `package.json` holds the English source text. Next to it, one overlay file per
other locale, `locales/zh-Hans.json`, `locales/ja.json`, `locales/ko.json`, `locales/de.json`, and
`locales/es.json`, maps each exact English string to its translation:

```json
{
  "locale": "zh-Hans",
  "replacements": {
    "Survival": "生存",
    "Unlimited Health": "无限生命",
    "Health never drops below its maximum.": "生命值始终保持满值。"
  }
}
```

A complete file translates every host-rendered string: the display name and description,
group, widget, and surface labels and descriptions, placeholders, button labels, tag labels, native
hotkey labels, and every option label, whatever the list length. Keys are the exact English text
(same case, spacing, and punctuation). An entry of a long list (items, places) may keep its source
name as the translation, but it must be present.

**Custom-interface text.** A custom interface owns its own text. Each surface keeps one flat
`{"key": "text"}` file per locale inside its folder, `frontend/<surface-id>/locales/en.json` through
`frontend/<surface-id>/locales/es.json`, with the same keys in every file it ships. The page loads the file
that matches the host locale and falls back to `en`; see
[Custom Interface API](../../hexinton-engine-wiki/application/custom-interface-api.md#localized-text).

**Writing translations.**

- Write natural, idiomatic interface wording for each language, not word-for-word translation.
- Use an official in-game name (item, character, place, skill) as the game shows it in that
  language when you know it; otherwise keep the English name. A translation may equal the English
  text for exactly this case.
- Never translate IDs, option values, file paths, code, hotkey chords, or catalogue data.

In the conversion pipeline, a missing file, a missing or blank translation, or a surface without
all six locale files is the error `package_locale_missing`. Studio does not run this check.
