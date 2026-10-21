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
| `dependencies` | no       | Explicit package dependencies         |
| `runtime`      | yes      | Files, public entries, and runnables  |
| `frontend`     | no       | Hosted UI declarations                |

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

The host checks strict JSON, matching directory and ID, declared files, unique runnable IDs, entry-file/runtime consistency, lifecycle entries, dependency constraints, and action parameters. Invalid packages produce structured errors. See [Errors and Results](../../hexinton-engine-wiki/engine/errors-and-results.md). Studio and package upload also enforce the [user-facing text rule](#user-facing-text-no-source-brands).

## User-facing text: no source brands

Status: current rule, effective 2026-10-21.

Text that players see describes the effect only. It never names a cheat source, trainer brand,
cheat site, cheat tool, or cheat-table author.

| Bad | Good |
| --- | --- |
| `Infinite EXP (FLiNG)` | `Infinite EXP` |
| `One-hit kills from the CE table` | `One-hit kills` |
| `Teleport (WeMod map)` | `Teleport` |

The rule covers, in every locale:

- package display names and descriptions;
- widget and group labels, descriptions, tooltips, and option labels;
- custom-interface visible text, including strings in surface HTML and JavaScript;
- publication metadata: package profile title, short description, description, instructions,
  changelog, and catalog descriptions.

Internal provenance is allowed in notes, code comments, Lua code, and tests.

A violation is a hard error everywhere. Studio's **Problems** panel reports it as an error, and
package uploads are rejected with error code `source_brand_reference`.

Matching is normalized, not exact. Before comparing, the checker folds case, removes accents,
zero-width and other format characters, converts full-width letters, and treats every punctuation
mark or space as a single separator. A name therefore also matches when it is spaced, hyphenated,
or joined: `We-Mod`, `we mod`, `cheatengine`, and `FLiNGTrainer` are all rejected. There is no
capitalization exception, so an ordinary game term that collides with a name, such as a "Fling to
the Finish" option, must be reworded. A short allowlist in the deny list exempts proven false
positives from shipped games, such as a "Dagger Fling" skill, a "Ring of Flinging", or a
"Fearless Cape". A "Sunbeam" spell is not matched; only phrases such as "SunBeam table" are.

Checked names include FLiNG (flingtrainer.com), WeMod / Wand, Cheat Engine (including "CE table",
"CE script", "from CE", cheat tables, and CT files), Cheat Happens (CoSMOS), MrAntiFun, PLITCH,
Infinity trainer, FearLess Revolution (FRF), Trainer Manager, GameCopyWorld, MegaDev / MegaGames,
palmods.gg, OpenCheatTables, other cheat, trainer, and mod download sites (for example GuidedHacking,
MPGH, XModHub, VGTimes, Nexus Mods, and Thunderstore), and trainer makers and cheat-table authors
(for example Zanzer, Recifense, Csimbi, DrummerIX, SunBeam, and Abolfazl.k). The authoritative list
is the deny list
[`source_brand_denylist.json`](https://github.com/HexintonDev/HexModTools/blob/main/production/game-ingestion-worker/source_brand_denylist.json).
