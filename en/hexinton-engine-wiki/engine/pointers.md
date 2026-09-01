# Pointers and Address Chains

Status: current public address-resolution subset.

The engine supports a useful CE-style subset, not a complete pointer scanner or full Auto Assembler
expression language.

## Supported Expressions

- module names
- registered symbols
- hexadecimal literals
- `+` and `-` offsets
- nested bracket dereferences such as `[[player_base+0x10]+0x30]`

Bare numeric tokens are hexadecimal. Resolution tries a known whole symbol/module before interpreting
`-` as subtraction, so hyphenated names remain usable.

```lua
local address = getAddressSafe("[[game.exe+0x1234]+0x18]+0x30")
if not address then
    return { ok = false, reason = "pointer chain unavailable" }
end
return { value = readInteger(address) }
```

## Chain Semantics

For a base address and offsets, each step reads a target-sized pointer, adds the next offset, and
returns the final address. Use a typed read after resolution to obtain the value at that address.
Pointer width follows the target process architecture, not the host architecture.

## Failure Rules

Use `getAddressSafe` when a missing module, symbol, pointer, or invalid dereference is expected. Treat
failure as a normal result for optional features and never write until the complete chain resolves.
This syntax does not provide pointer scanning, arbitrary expression evaluation, or automatic safety
proof for a guessed chain.
