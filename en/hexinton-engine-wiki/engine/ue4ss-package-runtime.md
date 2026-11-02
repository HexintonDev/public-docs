# Package-provided UE4SS Runtimes

Status: implemented package runtime selection, reviewed on 2026-11-02.

The client ships a default UE4SS distribution. A game that needs a custom build can deliver it as
assets in a support package, without adding that build to every client installation.

## Declare the distribution

The support package declares a top-level manifest property:

```json
"ue4ss": { "frameworkRoot": "assets/ue4ss" }
```

The path must remain inside the package. That directory contains `.frameworks/ue4ss/profile.json`
and the distribution files referenced by its hashes. Feature packages depend on the support package
through ordinary package dependencies. The support package's host-Lua enable/disable functions can
be empty; selecting a distribution does not itself load UE4SS into the game.

## Selection and validation

Before the first Unreal runnable, the engine resolves declarations in the applied package set,
verifies the selected assets, stages its bridge and loads the selected DLL. Studio resolves the
same declaration from the saved game workspace before inspection. Without a declaration the engine
uses the application-configured default framework root. An explicit Studio inspection framework
path is a development override.

Conflicting declared profile digests fail instead of choosing an arbitrary distribution. Absolute
paths, traversal outside the package and missing profiles fail manifest validation. The old loose
`<scriptsRoot>/.frameworks/ue4ss` layout is not a substitute for a declared support package.

No additional runtime download occurs during execution: the files arrive with package installation.
The selected distribution must still support the native Windows x64 game, and loading requires
sufficient process access. Integrity checks do not prove game compatibility or successful reflected
operations. Qualify live-world behavior and cleanup with the selected build.

## Preparing and publishing

The engine repository supplies `ProcessEngine/scripts/prepare-ue4ss-package.ps1` to prepare a support
package from a release ZIP. It retains the DLL, settings, signature/layout overrides, license and
shared Lua helpers; proxy loaders, debug symbols, tools and example mods are omitted. Server
validation checks declared profiles and hashes without executing them. Native DLL assets remain
subject to the existing first-party native publication policy.

Read [UE4SS loading and prerequisites](ue4ss-runtime.md#loading-and-prerequisites),
[package format](../../start-here/getting-started/package-format.md), and
[testing](testing.md#ue4ss-packages) before publishing a compatibility claim.
