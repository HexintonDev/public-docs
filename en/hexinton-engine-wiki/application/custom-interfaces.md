# Custom Interfaces

Status: current public overview for hosted trainer surfaces.

A custom interface is a package-owned UI surface hosted by the application. It renders controls and
state while native work remains behind declared package commands, queries, services, and feeds.

## Roles

- The package author declares the surface, bindings, and required capabilities.
- The application validates the package and hosts the surface.
- The player invokes commands and observes projected state.

A surface is not a direct Lua bridge. It must use the supported host protocol.

## Supported Flow

```text
manifest -> surface bootstrap -> view model/feed state -> user action -> command result -> state update
```

## Where surfaces appear

Surfaces appear on the desktop trainer page and in the in-game overlay. The trainer search also filters
surfaces by their label and description. Surfaces are not shown on the phone remote. Ordinary hosted
widgets are shared with the phone by default; see
[phone remote sharing](../engine/trainer-controls.md#phone-remote-sharing).

See the [quickstart](custom-interface-quickstart.md), [manifest](custom-interface-manifest.md), and
[API](custom-interface-api.md) pages for the authoring contract.
