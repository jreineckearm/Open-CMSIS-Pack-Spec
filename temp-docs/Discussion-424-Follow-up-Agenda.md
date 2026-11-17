# Discussion #424: follow-up agenda

Discussion reviewed through 16 November 2026. This agenda covers the five replies
after [comment 19542088](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19542088).
It records discussion agreements, the attribute choices made for this revision,
and recommendations for resolving the remaining questions. These are proposal
decisions, not implemented specification changes.

The [fetched comment](Discussion-424-Comment-19542088.md) remains the source snapshot.

## Settled direction

### Declaration model

Use `<debugvars>` with two boolean attributes. The attribute approach and legacy
defaults were agreed in the discussion; the names and boolean forms below were
selected for this revision.

| Attribute | Meaning when true | Default |
| --- | --- | --- |
| `readonly` | Sequences cannot modify the variables. | `true` |
| `configurable` | User configuration can override initial values. | `true` |

Both attributes accept `1`, `0`, `true`, and `false`. Each omitted attribute
defaults independently to `true`. All four combinations are supported:

| `readonly` | `configurable` | Sequence access | Initialization |
| --- | --- | --- | --- |
| `true` | `true` | Read-only | Configurable; legacy behavior |
| `false` | `true` | Read-write | Configurable |
| `true` | `false` | Read-only | Declared value; no user configuration overrides |
| `false` | `false` | Read-write | Declared value; no user configuration overrides |

Read-only protection applies during sequence execution. A configurable read-only
variable can still receive configuration overrides during initialization.
Omitting both attributes preserves the existing configurable, read-only behavior.
Keep `__var` declarations and replace the earlier `<debugconsts>` and
`<debugglobals>` proposal with these attributes.

Sources: [attribute discussion](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19653624),
[proposed legacy defaults](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19653985),
[defaults agreement](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19659450).

### Other agreements to retain

- Named parameters with mandatory defaults, return values, and persistent globals
  cover the original use cases; `rwparams` is no longer needed.
- The trace-pin example may be reused in the specification update.
- Declaration inheritance replaces a complete block rather than merging its
  contents, separately for each attribute combination. The most specific
  applicable device-hierarchy level wins. The precise grouping rules are covered
  by item 2 below.
- In atomic blocks, prohibit reading `__lasterror`, changing `__errorcontrol`,
  and returning. `__lasterror` updates after block execution. Remaining atomic
  semantics are covered by item 3 below.

Sources: [initial feedback](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19594954),
[maintainer response](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19653624),
[replacement agreement and reuse permission](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19653985).
The [other reply](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19653629)
contains no technical action item.

## 1. Lifetime and multicore ownership

**Decision: Open.**

**Sources:** [feedback, point 1](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19594954)
and [response, point 1](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19653624).

**Agreed position:** Persistent global state is part of the accepted direction.
The engine-lifetime wording does not yet establish a sufficient guarantee across
debugger events or define ownership when several processors are connected.

**Questions to resolve:**

- Which logical lifetime guarantees persistence across connect, reset, flash,
  and trace operations? When are initial values and configuration applied again?
- Is storage owned by a connection, a device, or a debugger session? What do
  separate connections to the same device share, including connections for
  different `Pname` values?
- What happens on disconnect and reconnect, including reconnecting one processor
  while another remains connected?

**Recommended starting position:** Define observable persistence independently
of engine-object lifetime. Start with preservation across debugger events while
the logical connection remains open. Review CMSIS Debugger and pyOCD ownership
models before selecting the sharing boundary; do not infer shared storage merely
from the absence of `Pname` on a declaration.

**Acceptance examples:** Record a value in `DebugPortSetup` or
`DebugDeviceUnlock`, then use it in `ResetSystem`, `FlashInit`, and `TraceStart`.
Specify the expected value after reset, flash, and reconnect. Repeat with two
processor connections to one device and with two independent connections,
including closing and reopening only one connection.

## 2. Inheritance and configuration

**Decision: Replacement principle agreed; detailed rules open.**

**Sources:** [feedback, points 2 and 5](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19594954),
[response, points 2 and 5](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19653624),
and [replacement/defaults follow-up](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19653985).

**Agreed position:** Use the settled two-attribute model and complete-block
replacement per attribute combination. Do not reopen the default values.

**Questions to resolve:** The following are derived details needed to make that
agreement implementable:

- How are equivalent boolean spellings and omitted defaults matched when
  selecting the block that replaces an inherited block?
- May a level contain more than one block for the same effective combination?
  How are duplicate variable names within a block or across effective groups
  handled?
- How do `configfile` and `version` apply when several configurable groups are
  present? Which variables may each `.dbgconf` file initialize?
- How do device settings and `.dbgconf` overrides apply across configurable
  groups, including read-only ones? How are overrides of non-configurable
  variables handled?

**Recommended starting position:** Identify groups by the effective boolean pair
`(readonly, configurable)` after applying defaults and normalizing values:
`1` equals `true`, and `0` equals `false`. Within a group, the most specific
applicable block replaces the complete inherited block. `configfile` and
`version` should not create additional group identities. Start with rejecting
duplicate effective groups at one level and duplicate names across effective
declarations. Retain the proposed initialization precedence of device settings,
then `.dbgconf`, then the declared value for configurable variables; resolve file
ownership and invalid-override diagnostics explicitly.

**Acceptance examples:** Compare an attribute-free block with an explicit
`readonly="1" configurable="true"` block; they identify the same group. Exercise
both omitted attributes independently and all four boolean spellings. Define two
different groups at family level and replace only one at device level; variables
from the replaced block must not be merged back. Include duplicate groups/names,
multiple configuration files, competing overrides, and a configurable read-only
variable that accepts an initial override but rejects a sequence assignment.

## 3. Atomic execution

**Decision: Three restrictions agreed; assignment and failure details open.**

**Sources:** [feedback, point 3](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19594954),
[response, point 3](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19653624),
and the [current block definition](../doxygen/src/devices_schema.txt).

**Agreed position:** Prohibit reading `__lasterror`, changing `__errorcontrol`,
and returning inside an atomic block. Preserve the existing prohibition on
`Query()` and `Sequence()` there. Also preserve the existing rule that a variable
cannot be used as an r-value later in the same block after it receives a
target-access result.

**Questions to resolve:**

- Which global assignments are permitted, when do their results become visible,
  and what effects remain if the block completes only partially?
- What does `__lasterror` report after several accesses, including a failure
  followed by success or several failures?
- How does the `__errorcontrol` value selected before the block affect partial
  execution, abort, and the opportunity to inspect the block result afterward?

**Recommended starting position:** Express the contract in terms of batched
execution and deferred results, without mandating CMSIS-DAP or another probe
protocol. Define variable effects and error reporting for the whole block, with
explicit treatment of partial execution. Do not imply rollback. Select a
deterministic error-reporting rule during this review, rather than assuming that
ordinary per-call clearing gives the intended block result.

**Acceptance examples:** Assign a target-access result to a writable global and
read it after the block. Reject a dependent read inside the block and each of the
three prohibited constructs. Cover all-success, failure followed by success,
multiple failures, and partial execution, with errors ignored and with abort
enabled. State the resulting global values and whether the following sequence
statement executes.

## 4. Compatibility reporting

**Decision: Direction supported; reporting and failure policy open.**

**Sources:** [feedback, point 4](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19594954)
and [response, point 4](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424#discussioncomment-19653624).

**Agreed position:** Improve future tools' handling of supported specification
revisions. The legacy declaration defaults preserve existing DFP behavior on
newer engines; they do not establish support for new constructs in older tools.
The discussion records no planned uVision implementation update.

**Questions to resolve:**

- Where should a tool state its supported revision range, and where should a
  mismatch be reported? Which mismatches require an error rather than a warning?
- How are unsupported required sequence features identified and rejected before
  execution? How does this relate to the pack's `schemaVersion`?
- What guidance is needed for DFP authors and users of tools that will not
  implement the new constructs?

**Recommended starting position:** Treat supported-version reporting and required
feature support as separate checks. Require unsupported required features to be
rejected before executing affected sequences. Review revision-range diagnostics
with implementers; schema acceptance alone is insufficient, particularly where
unknown attributes can be accepted without their semantics being implemented.
Track the suggested uVision knowledge-base article as a separate documentation
action; do not make the proposal depend on a uVision update.

**Acceptance examples:** Run through a legacy DFP on a newer engine, a DFP using
new constructs on an older engine, and a pack outside the stated revision range.
Include an engine that accepts an attribute syntactically but does not implement
it. State the diagnostic, whether execution is blocked, and when the check occurs.

## Working method and completion checks

1. Work through items 1 to 4 in order. For each question, record the chosen rule,
   rationale, relevant implementation evidence, and expected acceptance-example
   outcomes. Mark the decision as open, proposed, or decided; recommendations
   above remain proposals until decided.
2. After the decisions are settled, make verified backups before substantial
   edits, prepare the comment revision, and synchronize the full and concise
   proposals. Preserve the fetched comment as the source snapshot.
3. Update terminology, declaration tables, examples, and integration notes
   together. Remove the unconditional read-write treatment of legacy
   `<debugvars>`, replace the earlier three-element declaration model, and replace
   engine-lifetime wording with the chosen persistence contract.
4. Remove contradictory wording: `__lasterror` cannot be described as always
   readable once reads inside atomic blocks are prohibited. Keep debugger-facing
   `__Result`, last-function error status, and sequence return data distinct.
5. Check coverage of legacy declarations, all four attribute combinations, every
   boolean spelling, omitted versus explicit defaults, inheritance, configuration
   overrides, reset/flash persistence, two processor connections, reconnect, and
   atomic partial failure. Keep discussion agreements, local choices, and new
   recommendations distinguishable throughout the revision.
