# Eager Diagnostic Invalidation for Agentic LSP Clients

## Thesis

For interactive editors, brief diagnostic staleness after `textDocument/didChange` is usually acceptable UX.
For agentic clients, that same staleness is a correctness problem: diagnostics are machine input to decision loops.

This proposal adds an opt-in server behavior to invalidate diagnostics immediately after a change so programmatic clients do not act on stale results.

## Why This Matters

LSP servers are optimized for GUI editors. Debouncing diagnostics reduces flicker while a human is still typing. This default is sensible and should remain unchanged.

Agentic workflows are different:

- an agent edits a file
- immediately reads diagnostics
- decides next action from those diagnostics

If diagnostics are stale in that window, the agent may:

- re-fix an already fixed issue
- miss a newly introduced issue
- loop on diagnostics that no longer apply

So the problem is not "slightly delayed UI." It is "incorrect state observed by an automated consumer."

We implemented this behavior across four widely used language ecosystems:

- `pyright` (Python)
- `rust-analyzer` (Rust)
- `gopls` (Go)
- `typescript-language-server` (TypeScript)

TypeScript and Python usage trends are growing rapidly, as reflected in the GitHub Octoverse 2025 report. This does not directly prove agentic tool distribution, but it supports the inference that improving correctness in these ecosystems has high practical impact.

## Proposed Behavior

Add a per-server, opt-in setting that enables eager diagnostic invalidation on `didChange`.

When enabled:

1. Client sends `textDocument/didChange`.
2. Server ensures diagnostics for that file are cleared before fresh analysis results arrive.
3. Server continues normal analysis/debounce pipeline.
4. Server publishes fresh diagnostics when analysis completes.

When disabled (default):

- Existing behavior is preserved for editor-centric workflows.
- GUI clients avoid extra diagnostic flicker from transient "clear then republish" cycles.

## Design Principle

Default mode remains optimized for humans (stable UI).  
Opt-in mode is optimized for programmatic correctness (no stale diagnostic state between edits).

## Configuration (Current Implementations)

| Server | Setting | Default |
|---|---|---|
| [pyright](https://github.com/Microsoft/pyright) | `pyright.eagerDiagnosticInvalidation` | `false` |
| [rust-analyzer](https://github.com/rust-lang/rust-analyzer) | `diagnostics.eagerInvalidation` | `false` |
| [gopls](https://github.com/golang/tools) | `eagerDiagnosticsClear` | `false` |
| [typescript-language-server](https://github.com/typescript-language-server/typescript-language-server) | `diagnostics.eagerClear` | `false` |

Setting names follow each project's established configuration style.

## Ordering and Versioning Notes

`textDocument/publishDiagnostics` notifications are not globally ordered by the protocol. Implementations should rely on both timing and version semantics, not timing alone.

In practice, eager invalidation is triggered synchronously during `didChange` handling, before asynchronous reanalysis publishes fresh results. `didChange` is a notification, not a request.

Version-aware servers/clients provide extra safety: stale diagnostics tagged to older document versions can be dropped if reordering occurs.

## Concrete Example

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

Some clients incorrectly reuse document version numbers in `didChange`.
That is an independent spec-compliance issue and can cause servers to discard updates.

These are distinct failure modes:

- fixing version compliance alone does not remove stale diagnostics between edits
- fixing eager invalidation alone does not correct non-monotonic version bugs

Reliable agentic workflows need both.

## Implementation Summary

- **pyright**: reuses existing diagnostics tracking (`documentsWithDiagnostics`) and workspace settings.
- **rust-analyzer**: reuses `clear_native_for()` path in `didChange`; config integrated via standard diagnostics config path.
- **gopls**: uses existing version-filtered publish pipeline so stale cached diagnostics are naturally excluded after version bump.
- **typescript-language-server**: reuses close-path invalidation behavior to cancel pending debounced publishes before clear.

## Status

This is active work.

- **pyright**: pending
- **rust-analyzer**: pending
- **gopls**: pending (Gerrit)
- **typescript-language-server**: pending
- **Claude Code client issue**: pending (`didChange` version monotonicity)

## License

Each upstream contribution follows that project's license and contribution process.
