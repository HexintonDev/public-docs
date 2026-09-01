# Custom Interface Security

Status: current security guidance.

Custom surfaces are untrusted presentation code. Keep the host boundary least-authority and validate
all package-provided data.

- Treat package assets and JavaScript as untrusted input.
- Expose only declared commands, queries, ViewModels, and feeds.
- Do not expose CLR reflection, arbitrary native calls, credentials, or unrestricted bridge methods.
- Review `filesystem.write` and `winapi` as dangerous application-JavaScript capabilities.
- Validate message shape, request IDs, runnable IDs, parameters, and feed scope.
- Avoid placing secrets or sensitive process data in rendered state or diagnostic output.
- Use a restrictive content policy and package-local assets when hosting a surface.

Capability declaration is authorization input, not proof that a request is safe. The host must still
validate paths, arguments, package ownership, and session state.
