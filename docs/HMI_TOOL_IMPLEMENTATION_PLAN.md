# HMI MCP Tool Implementation Plan

## Goal

Add the missing WinCC Unified HMI MCP capabilities needed to inspect and update
screen-item properties and event scripts, and to create, edit, and read
expression dynamizations. Preserve existing HMI item identities, layouts,
button scripts, and unrelated dynamizations.

This plan concerns **MCP server source code only**. It does not authorize
connecting to, reading from, modifying, compiling, or saving any live TIA
project.

## Execution plan

### 1. Audit the existing server implementation

- Map tool registration, handlers, schemas, Openness helper patterns, tests,
  and tool-reference documentation.
- Inspect the existing HMI JSON patch operation schema and determine whether
  its guarded operations already cover any requested capabilities.
- Confirm whether `tia_save_project` and a JS type-definition resource/tool
  already exist before proposing additions.
- Identify the server-reported version source and project/assembly version
  metadata.

**Exit condition:** A short inventory of reusable implementation and test
patterns, with a precise list of genuinely missing capabilities.

### 2. Verify supported WinCC Unified V21 APIs

Confirm the supported Openness/runtime surface for:

- Reading and updating `Visible`, `Top`, and other requested item properties.
- Creating and reading event scripts on screen items and screens, including
  supported event names and trigger types.
- Creating, editing, and fully reading expression dynamizations, including
  their applicability to `Visible` and `Top`.
- Reading tag values and accessing item properties from runtime scripts.
- Returning JavaScript type definitions through an MCP resource or tool.

Do not implement an API based only on an unverified example. Record version
constraints and return explicit errors for unsupported operations.

**Exit condition:** Each operation is mapped to a verified API, or explicitly
marked unsupported and excluded from implementation.

### 3. Design minimal tool contracts

Prefer a small set of general tools over screen- or button-specific duplicates.
Evaluate these capabilities against the audit:

- Read one item's complete supported property set and dynamizations.
- Update an allowlisted set of item properties in place.
- Read and write event scripts on supported items and screens.
- Read full dynamization state and create/update expression entries without
  replacing unrelated entries.
- Expose JavaScript type definitions using the server's existing MCP resource
  conventions, if available.

Define argument and result schemas, target identification, validation,
readback behavior, and error reporting before implementation. Keep existing
`hmi_get_screen_items` and `hmi_update_button_scripts` behavior compatible.

**Exit condition:** Contracts are consistent with existing MCP naming and
error-handling patterns and specify how existing state is preserved.

### 4. Implement generic item and event-script operations

- Add or extend the item read/update path using Openness objects in place.
- Support only verified, explicitly allowlisted properties; validate value
  types and report unsupported properties clearly.
- Add event-script read/write support for verified screen and item event types.
- Read back each successful update and return the resulting state.
- Do not delete/recreate items or overwrite button scripts unless explicitly
  requested by a future, separately authorized task.

### 5. Implement expression dynamization operations

- Add create, update, and full-read support for expression dynamizations only
  where the V21 API is verified.
- Preserve existing tag, Singlebit, Range, bitmask, and other dynamization
  entries when editing one entry.
- Validate target property, expression text, and expression result type.
- Use `hmi_patch_json` only if its existing guards and operation schema are
  proven to preserve the same invariants; otherwise implement a direct
  Openness path.

### 6. Test, document, and version

- Add focused tests for successful read/write/readback, invalid input, missing
  items, unsupported properties/events, and preservation of existing scripts
  and dynamizations.
- Update tool schemas and the tools reference documentation.
- Run the narrow relevant test suite, then the server build/type checks.
- Before choosing the next MCP server version, check the running server's
  `/status` endpoint when available; otherwise inspect the reported-version
  source. Bump the MCP server version and align assembly/project metadata as
  required by the repository's versioning convention.
- Do not invoke a live TIA save or compile as part of unit tests. Any live
  integration test requires a separately approved target and action.

**Completion criteria:** Tests and build pass; tool schemas and docs match the
implemented behavior; readback confirms writes through the test harness; and
the reported server version is updated.

## Codespace and live-project boundaries

- First verify that the Codespace has the required .NET SDK and can build the
  repository. TIA Openness/COM integration may require a Windows machine with
  TIA Portal and may not be runnable inside a Codespace; use tests or mocks for
  source-level validation when that integration is unavailable.
- Do not connect to or inspect a TIA project, PLC block, or live tag while
  implementing these server-side tools unless separately authorized.
- Do not save or compile a live project without explicit approval.

## Deferred sidebar use case

The previously approved sidebar parameters were a 72 px pitch, visibility bits
0–4 in button order, and a shared `bWithRobot` tag. The inspected HMI tag list
did not contain `bWithRobot`, and the active project was explicitly put on hold.
Therefore this plan does not include applying or validating those settings in
that project.
