# Eager Diagnostic Invalidation for Agentic LSP Clients

## Why

LSP diagnostic debounce is a good default for human-driven editors: it reduces flicker while users are typing.

For programmatic clients, diagnostics are decision input. After `textDocument/didChange`, stale diagnostics can cause incorrect actions: re-fixing resolved issues, missing newly introduced issues, or looping on obsolete errors.

This proposal adds an opt-in server behavior that clears diagnostics for the changed file immediately, then publishes fresh diagnostics when analysis completes. The default remains `false` to preserve editor UX.

## Behavior

When eager invalidation is enabled:

1. Client sends `textDocument/didChange`.
2. Server clears diagnostics for that file (or equivalent behavior via its version-aware publish path).
3. Server continues its normal analysis/debounce pipeline.
4. Server publishes fresh diagnostics when analysis completes.

When disabled (`false`, default), existing editor-oriented behavior is unchanged.

## Configuration (Current Upstream Work)

| Server | Setting | Default |
|---|---|---|
| [pyright](https://github.com/Microsoft/pyright) | `pyright.eagerDiagnosticInvalidation` | `false` |
| [rust-analyzer](https://github.com/rust-lang/rust-analyzer) | `diagnostics.eagerInvalidation` | `false` |
| [gopls](https://github.com/golang/tools) | `eagerDiagnosticsClear` | `false` |
| [typescript-language-server](https://github.com/typescript-language-server/typescript-language-server) | `diagnostics.eagerClear` | `false` |

Setting names follow each project's existing configuration style.

## Ordering and Version Notes

`textDocument/publishDiagnostics` notifications are not globally ordered by the protocol.

In these implementations, invalidation is triggered during `didChange` handling before async reanalysis publishes fresh results. Clients should still treat version semantics as authoritative.

Version-aware servers/clients can drop stale diagnostics tagged to older document versions if reordering occurs.

## Example

Without eager invalidation:

```
T0: Agent edits file to fix error E
T1: Agent reads diagnostics -> still sees E (stale)
T2: Agent applies another edit for E (possibly harmful)
T3: Reanalysis finishes -> E was already fixed at T0
```

With eager invalidation:

```
T0: Agent edits file to fix error E
T1: Server clears diagnostics for that file
T2: Agent reads diagnostics -> stale state removed
T3: Reanalysis finishes -> fresh diagnostics published
```

## Related but Separate: Client Version Compliance

Some clients incorrectly reuse document version numbers in `didChange`. That is a separate spec-compliance bug and can cause servers to discard updates.

Both are required for reliable agentic workflows:

- version compliance alone does not remove stale diagnostics between edits
- eager invalidation alone does not correct non-monotonic version bugs

## Implementation Summary

- **pyright**: reuses diagnostics tracking (`documentsWithDiagnostics`) and workspace settings.
- **rust-analyzer**: reuses `clear_native_for()` in `didChange`; config integrated via diagnostics settings.
- **gopls**: uses version-filtered publish behavior so stale cached diagnostics are excluded after version bump.
- **typescript-language-server**: reuses close-path invalidation behavior to cancel pending debounced publishes before clear.

## Status

This is active upstream work.

- **pyright**: [target repo](https://github.com/Microsoft/pyright) | [draft PR text](./pr-drafts.md#2-pyright-pr) | upstream PR: not opened
- **rust-analyzer**: [target repo](https://github.com/rust-lang/rust-analyzer) | [draft PR text](./pr-drafts.md#3-rust-analyzer-pr) | upstream PR: not opened
- **gopls**: [target repo](https://github.com/golang/tools) | [draft PR text](./pr-drafts.md#4-gopls-pr) | Gerrit CL: not opened
- **typescript-language-server**: [target repo](https://github.com/typescript-language-server/typescript-language-server) | [draft PR text](./pr-drafts.md#1-typescript-language-server-pr) | upstream PR: not opened
- **Client issue (`didChange` version monotonicity)**: [target repo](https://github.com/anthropics/claude-code) | [draft issue text](./pr-drafts.md#5-claude-code-github-issue) | issue: not opened

## License

Each upstream contribution follows that project's license and contribution process.
