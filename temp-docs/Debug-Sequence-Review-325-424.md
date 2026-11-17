# Debug sequence review: #325 and #424

Reviewed: 9 November 2026. Covers both opening posts, all six comments on #325, and #424 (no comments). Specification baseline: local checkout `6410abc`, schema version 1.7.62. The review order and recommendations below are proposed conclusions, not agreed specification changes.

## Condensed summary

| Thread | Problem and proposals | Position in the discussion |
| --- | --- | --- |
| [Issue #325: observable access errors](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/issues/325) | A sequence that suppresses access errors cannot reliably distinguish a failed read from valid register data, particularly during reset or secure boot. Proposed solutions include reserved `__errorcontrol` bits, a dedicated error variable, or opt-in updates to `__Result`. | Open. Comments develop the opt-in `__Result` approach and capability detection, but leave function coverage and control semantics unresolved. The latest maintainer response, on 18 March 2026, confirms the problem remains under consideration. |
| [Discussion #424: state and sequence interfaces](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/discussions/424) | Add writable globals lasting for one debugger connection and positional sequence parameters. Also consider writable parameters and returning data through `Sequence()`, using `__Result` or a new `__Return`. Use cases include cached device state and reusable register or trace-pin configuration. | An initial proposal with open choices about syntax, scope, outputs, atomic blocks, and compatibility. No follow-up comments or agreed design are recorded. |

The existing specification makes user-defined debug access variables read-only inside sequences. `Sequence("name")` returns zero; `__Result` carries sequence success/error. `__dp`, `__ap`, `__apid`, and `__errorcontrol` are saved/restored across nested calls; `__Result` is not saved/restored. These rules form the compatibility baseline. See [syntax and variables](doxygen/src/debug_description.txt) and [debugvars, sequence, and block definitions](doxygen/src/devices_schema.txt).

## Overlap and boundaries

The central overlap is **how sequences communicate status and data**. #325 considers making `__Result` reflect individual access failures, while #424 considers using it to return sequence data. Combining those roles could overwrite an error, mistake data for failure, or lose information across nested calls. The [#325 opt-in proposal](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/issues/325#issuecomment-2335110277) therefore needs to be reviewed alongside #424's return-value alternatives.

Both threads also require precise state lifetimes, call behavior, and feature detection. Their capabilities remain distinct: writable globals do not automatically expose debugger-detected access failures, and an access-error indicator does not provide reusable sequence arguments or persistent device state. #325 can progress independently once its status contract is settled.

Periodic DHCSR polling is a related debugger-interference concern. The #325 comments explicitly [split it into #327](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/issues/325#issuecomment-2335113105); it should retain separate tracking.

## Recommended review order

1. **Agree the roles of status and data.** Distinguish the last access outcome, the overall sequence result, returned sequence data, and state retained between debug events. Preserve existing `__Result` success/error behavior by default. Use this model to assess every proposed variable or parameter mechanism.

2. **Choose the access-error reporting mechanism (#325).** Compare a dedicated status variable, reserved `__errorcontrol` fields, and opt-in updates to `__Result`. Decide whether a Boolean outcome is sufficient or detailed error codes are needed. Keep failed-read detection independent of the returned data: neither zero nor all ones is a universally safe failure sentinel.

3. **Define error updates and execution behavior.** Specify the combinations of skip-errors and report-errors controls; whether success clears a previous error or errors remain latched; which functions update status; and who clears handled errors. The comments disagree about functions already returning error codes. Cover nested calls and `__Result` initialization so a subsequent call cannot silently erase needed status. See the [unresolved alternatives](https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/issues/325#issuecomment-2368615564).

4. **Set compatibility and capability rules.** Define how a sequence or tool discovers support and what unsupported implementations must do. Writing and reading back reserved bits is insufficient: #325 reports that existing implementations may retain those writes. Review a read-only capability indication and version/requirement gating. The [current schema](schema/PACK.xsd) permits lax additional sequence attributes and stores debugvars as text, so schema validation alone cannot guarantee rejection of unsupported syntax.

5. **Define writable global state (#424, proposal A).** Confirm connection lifetime, initialization and `.dbgconf` overrides, visibility to predefined sequences, and family/device inheritance. Specify behavior on reset, reconnect, failed setup, and multiple processors sharing a device. In particular, decide how cached silicon, boot, or lock state becomes invalid; connection lifetime alone does not ensure its continued accuracy.

6. **Define sequence inputs (#424, proposal B).** Review positional, by-value, unsigned 64-bit parameters; naming and scope; argument evaluation; and count/type validation. Define how parameters interact with inherited or processor-specific sequence implementations. Keep debugger-invoked predefined sequence entry points usable without arguments.

7. **Choose how sequence outputs work.** Compare a `Sequence()` return value, `__Return`, and writable parameters against the status model from steps 1 to 3. Decide which mechanisms are needed and whether they complement globals. For writable parameters, specify eligible arguments, aliasing, copy-back timing, and behavior on abort; reference and copy-back implementations are not interchangeable in every case. Preserve legacy zero-return behavior.

8. **Resolve atomic execution and shared-state effects.** Define global writes and error-status updates inside atomic blocks, including partial failure. Clarify visibility and ordering if processor sequences can execute concurrently. Preserve the existing prohibition on `Sequence()` inside atomic blocks unless a separate change is justified.

9. **Finalize syntax and specification changes.** After the semantics are settled, choose names such as `__rwvar`/`__gvar`, `params`, and `rwparams` or `in`/`out`. Update the expression language, variable/function descriptions, `SequenceType`, and any affected `DebugVarsType` rules together. Separate text/parser changes from actual XSD changes, and obtain debugger-implementer feedback before fixing the contract.

10. **Agree acceptance examples and delivery stages.** Cover failed polling reads with both zero- and one-valued success conditions, recovery followed by successful access, nested calls, reusable pin configuration, globals across events/reset, output handling on abort, and an older debugger encountering new features. Deliver the access-error contract, writable globals, and parameter/output support as separately reviewable changes with consistent shared semantics.

Recommended starting point: settle steps 1 to 4 jointly, then advance #325 while reviewing #424's state and call-interface extensions. This closes the concrete polling gap without making it depend on the broader language extension.
