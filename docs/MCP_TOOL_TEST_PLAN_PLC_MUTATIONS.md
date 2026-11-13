# PLC mutation slice — test plan

## Scope and implementation boundary

This plan covers four bounded operations implemented in
`TiaManager.PlcMutations.cs`:

1. `create_plc_tag_table`
2. `create_tagtable_group`
3. `create_user_constant`
4. `update_user_constant` (value only)

No live TIA Portal or PLC tests were run. The bundled
`src/lib/Siemens.Engineering.Step7.dll` metadata exposes the relevant V21
collection `Create` overloads: tag tables and user groups take a name;
user constants take name, datatype, and value. `PlcUserConstant.Value` has a
public string setter, and its datatype/name properties are readable. The
implementation uses those typed collection methods and that setter; it does
not infer mutability from reflection or an attribute name.

The relevant typed signatures are `PlcTagTableComposition.Create(string)`,
`PlcTagTableUserGroupComposition.Create(string)`,
`PlcUserConstantComposition.Create(string, string, string)`, and
`PlcUserConstant.set_Value(string)`. Their returned CLR object types are,
respectively, `PlcTagTable`, `PlcTagTableUserGroup`, and `PlcUserConstant`.

The result reports the PLC name and scoped object path. The APIs used here do
not provide a stable object ID, revision, or transaction boundary. Engineering
or COM exceptions and failed post-mutation read-back are reported as potentially
partial; inspect the target before deciding whether to retry.

## Registered handler calls and input schemas

The handlers and strict JSON schemas are registered in `ToolRegistry` and
`schemas/`.

```csharp
TiaManager.Instance.CreatePlcTagTable(
    string? plcName, string tableName, string? groupPath = null);

TiaManager.Instance.CreateTagTableGroup(
    string? plcName, string groupName, string? parentGroupPath = null);

TiaManager.Instance.CreateUserConstant(
    string? plcName, string tableName, string constantName,
    string dataType, string value, string? groupPath = null);

TiaManager.Instance.UpdateUserConstant(
    string? plcName, string tableName, string constantName,
    string value, string? groupPath = null);
```

Suggested tool-schema required/optional fields:

| Tool | Required | Optional |
|---|---|---|
| `tia_create_plc_tag_table` | `table_name` | `plc_name`, `group_path` |
| `tia_create_tagtable_group` | `group_name` | `plc_name`, `parent_group_path` |
| `tia_create_user_constant` | `table_name`, `constant_name`, `data_type`, `value` | `plc_name`, `group_path` |
| `tia_update_user_constant` | `table_name`, `constant_name`, `value` | `plc_name`, `group_path` |

`plc_name` may be omitted only when there is exactly one PLC. Group paths are
relative, slash-separated paths; omitted paths mean the root group. Resolution
uses exact ordinal names, and never picks the first matching PLC/group/table.
Name collision checks are case-insensitive to avoid creating an ambiguous
same-scope target.

Handlers return the full `PlcMutationResult` JSON object using the default
PascalCase serializer: `Success`, `Operation`, `Status`, `PlcName`,
`ObjectName`, `TargetPath`, optional `DataType`/`Value`, `Partial`, `Message`,
and `Warnings`. Expected non-success statuses include `invalid_input`,
`not_found`, `ambiguous`, `collision`, `api_failure`, and
`verification_failed`. API errors are classified as partial only when the V21
engineering or COM API throws after the mutation call starts.

## Case-by-case verification

Before each case, call `tia_get_status` and confirm the connected V21 instance
and intended disposable project. Run each case against a project with at least
one PLC, and record the TIA Portal build, PLC name, project path, and whether
the project was saved/reopened. Do not use a production project.

### `create_plc_tag_table`

| Case | Setup / input | Expected result and checks |
|---|---|---|
| Valid root creation | Unique `table_name`; unique or sole PLC; omit `group_path` | `Success=true`, `Status=created`, exact PLC/name/path; `Partial=false`; warnings disclose no ID/revision/transaction. Exact-name in-memory read-back confirms the new table. Save, close, reopen, and read again. |
| Valid nested creation | Existing exact parent path and unique name | Path is `group_path/table_name`; table appears only in that group; repeat the save/close/reopen read-back. |
| Invalid name | Empty/whitespace name, edge whitespace, slash, backslash, or control character | `invalid_input`; no API create call; no change in table count. |
| Invalid path | Absolute, trailing slash, repeated slash, or edge whitespace | `invalid_input`; no change in table/group counts. |
| Missing target | Unknown PLC name or missing group segment | `not_found`; no change. |
| Ambiguous target | Omitted PLC when multiple PLCs exist; or duplicate exact matching child groups in one path segment | `ambiguous`; no first-match behavior and no change. |
| Collision | Existing table or group name in selected group, including case-only difference | `collision`; existing object remains unchanged; no new table. A same-named object in a different group does not collide. |
| API/read-back failure | In a disposable project, provoke a V21 engineering/COM exception after prechecks or use an API test double to simulate missing/duplicate read-back | `api_failure` or `verification_failed`; `Partial=true` because the call began; warning directs read-back. Inspect the group before retrying. Do not interpret this as rollback. |
| Result shape | Exercise one success and one precondition failure through the registered tool handler | All PascalCase fields serialize; success includes warnings; precondition failure has `Partial=false`; no fabricated ID/revision. |
| Persisted read-back | Successful root and nested creations | Save via the existing project-save flow, close/reopen, then locate each exact group/table. Verify the operation did not silently overwrite an existing table. |

### `create_tagtable_group`

| Case | Setup / input | Expected result and checks |
|---|---|---|
| Valid root creation | Unique `group_name`; unique or sole PLC | `created`, root path equals the name, `Partial=false`; exact-name in-memory read-back and subsequent group traversal confirm the group. Save/close/reopen and verify. |
| Valid nested creation | Existing exact `parent_group_path`; unique name | `created` with full parent/name path; parent remains unchanged; save/close/reopen and verify. |
| Invalid name/path | Empty/edge-whitespace/control/path separator name, or malformed parent path | `invalid_input`; no new group. |
| Missing target | Unknown PLC or parent-group segment | `not_found`; no new group. |
| Ambiguous target | Multiple PLCs with omitted PLC selector, or non-unique exact path segment | `ambiguous`; no new group. |
| Collision | Same-scope group or tag table already has this name, including case-only difference | `collision`; neither existing object changes. Same name in an unrelated group does not collide. |
| API/read-back failure | Force a V21 engineering/COM failure or simulate a missing/duplicate read-back after resolution | `api_failure` or `verification_failed`, `Partial=true`, read-back warning. Inspect before retry; no atomicity claim. |
| Result shape | Success, collision, and missing-parent calls through the registered handler | Correct status, resolved PLC, object/path, message, and `Partial`; warnings do not claim IDs, revisions, or rollback. |
| Persisted read-back | Create root and nested groups | Save/close/reopen; traverse exact parent paths and verify the groups remain. |

### `create_user_constant`

| Case | Setup / input | Expected result and checks |
|---|---|---|
| Valid Bool | Existing exact table; `data_type=Bool`; `value=true` | `created`, normalized `DataType=Bool`, `Value=TRUE`; confirm exact `UserConstants` read-back; save/close/reopen and verify. |
| Valid numeric | Create signed/unsigned integer and finite `Real`/`LReal` constants with valid decimal values | Canonical type casing and invariant-culture numeric value returned; confirm the created properties from the collection, then save/close/reopen. |
| Invalid name/value/type | Bad constant name; empty value; malformed bool/numeric; integer outside range; NaN/Infinity; unsupported type (including string, temporal, or structured) | `invalid_input`; no create call and no added constant. |
| Missing target | Unknown PLC, group, or table | `not_found`; no change. |
| Ambiguous target | Omitted PLC with multiple matches, ambiguous group segment, or duplicate exact table in its group | `ambiguous`; no change. |
| Collision | Exact constant name already exists in selected table, including case-only difference | `collision`; no overwrite and no other table is modified. |
| API/read-back failure | Trigger a V21 engineering/COM failure or simulate a mismatching exact-name read-back after prechecks | `api_failure` or `verification_failed`, `Partial=true`; read back the same constant name/table before retry. The client validator is deliberately a strict subset and cannot guarantee TIA acceptance. |
| Result shape | Valid create and each precondition/API/read-back failure | Response contains normalized `DataType`/`Value` when available; `Partial=false` before API call and true for uncertain post-call outcomes; no revision/ID claim. |
| Persisted read-back | Create constants of supported types | Read value/type from the table, save/close/reopen, and read again. Confirm existing constants remain unchanged. |

### `update_user_constant` (value only)

| Case | Setup / input | Expected result and checks |
|---|---|---|
| Valid update | Existing exact constant; valid new literal for its stored datatype | `updated`; returned value is normalized and equals read-back from the same constant; save/close/reopen and read again. |
| Invalid value | Wrong bool/numeric syntax, out-of-range integer, non-finite float, or unsupported stored datatype | `invalid_input`; setter is not invoked; prior value remains unchanged. |
| Missing target | Unknown PLC/group/table or no exact constant in selected table | `not_found`; no change. |
| Ambiguous target | Omitted PLC when multiple PLCs exist, ambiguous group path, duplicate exact table, or duplicate exact constant names within a table | `ambiguous`; no setter call. |
| Collision/dependency | Not applicable: this operation does not create or rename an object and does not change datatype. Verify that same-name constants in other tables are untouched. |
| API/read-back failure | On a disposable project force a V21 engineering/COM error or simulate a mismatch after setter invocation starts | `api_failure` or `verification_failed`, `Partial=true`; read back immediately because the setter may have applied before the failure. |
| Result shape | Successful update and each failure category | `Operation`, `Status`, target, and `Value` are explicit; `Partial=false` for validation/lookup failures and true for uncertain setter failures; no expected-revision field is accepted or implied. |
| Persisted read-back | Update one constant | Verify immediate in-memory read-back, save/close/reopen, and verify persisted value and unchanged datatype. |

## Deferred and unsupported APIs

- `create_plc_tag`: deferred. This slice does not implement address syntax,
  address/datatype compatibility, symbolic tag assignment, or the V21 mapping
  for those inputs.
- Tag-table initial tags, tag comments, and metadata beyond the properties
  used above: deferred.
- `update_user_constant` updates only the value. Datatype, name, and comment
  updates are not exposed by this operation.
- String/character, date/time, enum, and structured user-constant literals are
  rejected. Their exact V21 literal grammar and normalization have not been
  established by this slice.
- Block/UDT rename, block/type/tag-table property writes, and tag-table rename:
  deferred until the local API proves writable setters and safe reference
  behavior.
- Project-text writes and project creation: deferred.
- Deletes, imports/replacements, transfers, and any multi-object workflow:
  explicitly out of scope.
- Stable IDs, revisions, optimistic concurrency, transactions, rollback, and
  live TIA tests are not claimed or implemented.
