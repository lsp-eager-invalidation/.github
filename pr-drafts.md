# PR and Issue Drafts

Draft descriptions for upstream PRs and issues. These will be used when opening contributions to each project.

---

## 1. typescript-language-server PR

**Title:** feat: add `diagnostics.eagerClear` setting to clear diagnostics on change

**Body:**

AI coding agents and other headless LSP clients edit files rapidly and then immediately read back diagnostics to decide what to do next. Because tsserver reanalysis is asynchronous, there's a window after each edit where the previous diagnostics are still published. The client has no way to distinguish "these diagnostics reflect the current file content" from "these are leftover from before my edit." This causes agents to act on stale errors - sometimes reverting correct fixes or looping on diagnostics that no longer apply.

This adds an opt-in `diagnostics.eagerClear` setting (default `false`). When enabled, `didChangeTextDocument` synchronously publishes empty diagnostics for the changed file before forwarding the change to tsserver. Fresh diagnostics arrive normally once tsserver finishes reanalysis.

The implementation adds `clearDiagnosticsForFile()` to `DiagnosticsManager`, which reuses the existing `onDidClose()` path to cancel any pending debounced publish and emit an empty diagnostic set. The hook in `lsp-server.ts` is a single conditional before the existing `onDidChangeTextDocument` call.

This is a partial answer to #983 - it doesn't add `version` tracking to `publishDiagnostics`, but it solves the immediate practical problem for non-UI clients that need to know when diagnostics are stale. The `version` approach would be a more complete solution and could complement this one.

Context: anthropics/claude-code#17979 tracks users disabling LSP integration entirely because stale diagnostics cause more harm than help in agentic workflows.

Tests cover both the enabled and disabled paths, verifying that diagnostics are (or are not) cleared synchronously after `didChange`.

Relates to #983

**Branch:** [`feature/eager-diagnostic-invalidation`](https://github.com/lsp-eager-invalidation/typescript-language-server/tree/feature/eager-diagnostic-invalidation)

---

## 2. pyright PR

**Title:** Add opt-in `eagerDiagnosticInvalidation` setting to clear stale diagnostics on edit

**Body:**

## Summary

Adds a new `pyright.eagerDiagnosticInvalidation` workspace setting (default `false`) that publishes empty diagnostics for a file immediately on `textDocument/didChange`, before reanalysis completes. This prevents stale diagnostics from persisting between edits.

## Problem

When an LSP client applies edits programmatically (e.g., agentic coding tools that write code and then read diagnostics), the client sees diagnostics from the previous file version until reanalysis finishes. In rapid-edit workflows this causes stale errors to be treated as current, leading users to disable the language server entirely (ref: anthropics/claude-code#17979).

Interactive editors are unaffected because users tolerate brief staleness, but headless clients that poll diagnostics between edits need a reliable signal that previous results are no longer valid.

## Solution

- Add `eagerDiagnosticInvalidation` to `ServerSettings` and `Workspace`
- Read the setting from the `pyright` section of LSP workspace configuration in `server.ts`
- In `onDidChangeTextDocument`, if the setting is enabled and the file has existing diagnostics (tracked via `documentsWithDiagnostics`), send an empty `publishDiagnostics` notification before the analysis cycle runs
- The `documentsWithDiagnostics` check avoids publishing unnecessary empty notifications for files that have no diagnostics
- Add VS Code extension schema entry
- Default is `false` so existing behavior is unchanged

## Test plan

- New test in `languageServer.test.ts`: opens a file with a type error, waits for diagnostics, sends `didChange`, and asserts that the first `publishDiagnostics` after the change is an empty clear
- Updated 3 test files (`envVarUtils.test.ts`, `testLanguageService.ts`, `testState.ts`) to include the new `Workspace` field

**Branch:** [`feature/eager-diagnostic-invalidation`](https://github.com/lsp-eager-invalidation/pyright/tree/feature/eager-diagnostic-invalidation)

---

## 3. rust-analyzer PR

**Title:** feat: Add `diagnostics.eagerInvalidation` setting to clear stale diagnostics on edit

**Body:**

Closes #20865

## Problem

When automated coding tools (or any rapid-edit workflow) write multiple functions or make several changes in quick succession, native diagnostics from a previous analysis pass can linger until the next pass completes. This causes false errors to appear for symbols that have already been defined in the buffer but have not yet been re-analyzed. Users working with agentic tools report restarting rust-analyzer to clear these phantom diagnostics, or disabling the language server entirely.

## Solution

Add an opt-in `rust-analyzer.diagnostics.eagerInvalidation` setting (default: `false`). When enabled, the `didChange` notification handler clears native diagnostics for the changed file immediately, before reanalysis begins. This removes stale diagnostics from the gap between an edit and the next analysis pass.

The implementation is minimal:

- **`config.rs`**: New `diagnostics_eagerInvalidation: bool = false` config entry with accessor.
- **`handlers/notification.rs`**: In `handle_did_change_text_document`, after updating VFS contents, call `diagnostics.clear_native_for(file_id)` when the setting is enabled. This reuses the same method already used in `didClose`.
- **`tests/slow-tests/main.rs`**: Integration test that opens a file with a type error, confirms diagnostics arrive, sends a correcting `didChange`, and asserts diagnostics are cleared immediately.

The setting defaults to `false` so existing editor workflows are completely unaffected. The intended audience is headless/agentic LSP clients where the brief flash of stale diagnostics between edits is more disruptive than the momentary absence of diagnostics during reanalysis.

## Testing

- New slow test `test_eager_diagnostic_invalidation_on_did_change` covers the end-to-end flow.
- Existing tests pass without modification since the default is `false`.

## AI Disclosure

This contribution was developed with assistance from Claude (Anthropic). All code was reviewed and validated by a human contributor.

**Branch:** [`feature/eager-diagnostic-invalidation`](https://github.com/lsp-eager-invalidation/rust-analyzer/tree/feature/eager-diagnostic-invalidation)

---

## 4. gopls PR

**Title:** gopls/internal/server: add eagerDiagnosticsClear setting to publish empty diagnostics on didChange

**Body:**

When a headless or agentic LSP client edits a file, there is a window
between the edit and the completion of reanalysis during which the client
still sees the previous set of diagnostics. For interactive editors this
is rarely a problem because the user can see the code changing, but for
automated tools that consume diagnostics programmatically (e.g. AI coding
agents), stale diagnostics cause the tool to attempt fixes for errors
that no longer exist.

This CL adds an opt-in `eagerDiagnosticsClear` setting (advanced, default
false) that publishes empty diagnostics for changed files immediately on
`textDocument/didChange`, before reanalysis completes. The real diagnostics
are published as usual once analysis finishes.

The implementation hooks into `didModifyFiles` for the `FromDidChange`
cause. It leverages the existing `publishFileDiagnosticsLocked` pipeline,
which filters view diagnostics by file version. Since the file version
was just incremented by the change, cached diagnostics from the previous
version are naturally excluded, resulting in an empty published set
without requiring any special clearing logic.

Only files that already have tracked diagnostics are processed, so there
is no overhead for files that have never had diagnostics published.

This is complementary to the `chattyDiagnostics` behavior added in
golang/go#54983, which ensures diagnostics are always republished even
when unchanged. `eagerDiagnosticsClear` addresses the opposite side of
the timing window: clearing stale diagnostics eagerly rather than
waiting for fresh ones.

The setting defaults to false so existing editor workflows are
unaffected.

Includes:
- New `EagerDiagnosticsClear` field in `DiagnosticOptions` (status: advanced)
- Setting parser in `Options.setOne`
- Documentation in `gopls/doc/settings.md`
- `ListenToDiagnostics` test helper that records the full history of
  diagnostic notifications for a file, including intermediate clears
- Integration test `TestEagerDiagnosticInvalidation` covering both the
  enabled case (first notification after edit is empty) and the disabled
  case (first notification retains diagnostics)

**Branch:** [`feature/eager-diagnostic-invalidation`](https://github.com/lsp-eager-invalidation/tools/tree/feature/eager-diagnostic-invalidation)

---

## 5. Claude Code GitHub Issue

**Repository:** https://github.com/anthropics/claude-code

**Title:** [BUG] LSP client sends constant document version on textDocument/didChange (never increments)

**Body:**

### Preflight Checklist

- [x] I have searched [existing issues](https://github.com/anthropics/claude-code/issues?q=is%3Aissue%20state%3Aopen%20label%3Abug) and this hasn't been reported yet
- [x] This is a single bug report (please file separate reports for different bugs)
- [x] I am using the latest version of Claude Code

### What's Wrong?

Claude Code's built-in LSP client always sends the same `version` number (typically `1`) in `textDocument/didChange` notifications. Per the [LSP specification](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#versionedTextDocumentIdentifier), the version in `VersionedTextDocumentIdentifier` must be monotonically increasing after each change, including undos/redos:

> The version number of a document will increase after each change, **including undo/redo**. The number doesn't need to be consecutive.

This is a spec compliance bug, not a feature request.

**Why this matters:** Many language servers use the document version to correlate diagnostic responses with the document state that produced them. When every `didChange` carries the same version, version-aware servers have no way to determine whether a diagnostic response corresponds to the current document state or a previous one. Servers that filter or discard stale diagnostics based on version comparison will malfunction.

**Relationship to #17979:** This is distinct from the stale diagnostics problem reported in #17979. That issue describes the symptom (stale diagnostics shown to the model after edits). This issue reports one root cause on the client side: the version field is never incremented, which breaks the protocol contract that servers rely on for ordering. Fixing this bug alone will not fully resolve #17979 (there are timing and invalidation concerns as well), but it is a necessary correctness fix regardless.

### What Should Happen?

The LSP client should maintain a per-document version counter and increment it on every `textDocument/didChange` notification sent to the server. The version should start at 0 (or 1) on `didOpen` and increase by at least 1 on each subsequent `didChange`.

### Steps to Reproduce

1. Enable an LSP plugin in Claude Code (e.g., `typescript-language-server` or `pyright`)
2. Have Claude edit a file multiple times in one session
3. Observe the `textDocument/didChange` notifications sent to the language server (via server-side logging or a proxy)
4. Note that every notification carries the same `version` value

For example, with debug logging enabled on the server side, you will see something like:

```
didChange: file:///project/foo.ts version=1  (first edit)
didChange: file:///project/foo.ts version=1  (second edit)
didChange: file:///project/foo.ts version=1  (third edit)
```

All three carry `version=1`. The spec requires these to be monotonically increasing, e.g. `1, 2, 3`.

### Suggested Fix

Maintain a `Map<string, number>` (keyed by document URI) that tracks the current version for each open document. Initialize to 0 on `textDocument/didOpen`, increment before each `textDocument/didChange`, and remove the entry on `textDocument/didClose`. Use the current counter value as the `version` field in `VersionedTextDocumentIdentifier`.

This is a small, low-risk change.

### Complementary Server-Side Work

Independently of this client bug, we have been working on server-side "eager diagnostic invalidation" patches for several language servers. These patches cause servers to immediately clear (invalidate) diagnostics for a file when they receive a `didChange`, before recomputation finishes. This addresses the stale diagnostics symptom from the server side and helps even with clients that do manage versions correctly, since there is always a window between "change sent" and "new diagnostics computed."

The server-side patches and this client-side fix are complementary and independent. The version bug should be fixed regardless of server-side behavior, because it violates the protocol specification.

### Claude Model

Sonnet (default)

### Is this a regression?

No - the version has never been incremented as far as I can tell.

### Claude Code Version

2.1.63

### Platform

Anthropic API

### Operating System

macOS

### Terminal/Shell

kitty

### Additional Information

Relevant LSP spec sections:

- [VersionedTextDocumentIdentifier](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#versionedTextDocumentIdentifier) - defines the version semantics
- [textDocument/didChange](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didChange) - requires VersionedTextDocumentIdentifier

Related issues:

- #17979 - stale diagnostics symptom (this bug is one contributing cause)
