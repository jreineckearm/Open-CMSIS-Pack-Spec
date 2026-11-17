***Last updated: 17/11/2026***

Hi @MarianSavchuk , apologies for the long silence on this one. I'd also like to use this thread to continue the discussion started in https://github.com/Open-CMSIS-Pack/Open-CMSIS-Pack-Spec/issues/325 . The topic is closely related.

We have reviewed the above input and share our thoughts below. It picks up a lot of your ideas which are a great starting point.
Please review and let us know what you think.

I _believe_ this answers your questions.
1) No, there is no existing way yet.
2) See "Global declarations"
3) See the combination of all below.
4) Yes. `<debugvars>` gains the boolean attributes `readonly` and `configurable`, both defaulting to `true`. Setting `readonly="false"` enables writes by sequences; `configurable="true"` allows `device-settings:` or `.dbgconf` to set their initial values.
5) See "Atomic blocks" below.
6) Yes. Predefined and user-defined sequences use the same global declarations. See "Global declarations" for persistence.
7) See the compatibility notes below.

Notes up front:
- The below breaks compatibility to some extent. Otherwise we would end up with a few very awkward constructs which certainly would get worse with future enhancements.
- We currently have no plans to implement changes in uVision once they are agreed. Which effectively means that DFPs updated with the below will most likely not work there. We plan to implement changes though in CMSIS Debugger / pyOCD.
- The reason for deviating significantly from your global variables proposal is to have clearer distinction between what global variables are configuration variables, and which are not. This may help in future with offering configuration options to users through [`device-settings:`](https://open-cmsis-pack.github.io/cmsis-toolbox/pyOCD-Debugger/#device-settings) in `*.csolution.yml` without a `.dbgconf` file present. We were considering to introduce a new keyword like `__rwvar` or `__const`. But the `readonly` and `configurable` attributes on `<debugvars>` let us make these distinctions without changing the language itself.
- Future tools should state their supported specification revisions and report a `schemaVersion` outside that range. Unsupported required features should be rejected before sequence execution. How to report those warnings/errors may become tool specific.

Please let us know what you think. I suspect we will also briefly touch the topic during the next [Open-CMSIS-Pack Technical Meeting](https://github.com/Open-CMSIS-Pack/landing/wiki/Open%E2%80%90CMSIS%E2%80%90Pack-Technical-Meetings).

---------------------------------

All values are unsigned 64-bit integers.

## Sequence parameters and arguments

- Declare `<parameter>` children at the top of `<sequence>`, before any `<block>`,  `<control>`, or `<return>`. Both `name` and `default` are mandatory. Names must  be unique; defaults must be constants.
- Call arguments use `name = value`, where the value can be an expression.  Supply arguments for declared parameters in any order, each name at most once.  Unknown or repeated names are errors. Omitted arguments use the corresponding  `<parameter>` default.
- Argument expressions are evaluated left to right in the caller and passed by  value. Parameters are local variables in the called sequence and its child  controls; assignments affect only that invocation's copy. Existing variable naming and no-redefinition rules apply.

```xml
<sequence name="AddValues">
  <parameter name="left" default="0"/>
  <parameter name="right" default="0"/>
  <return value="left + right"/>
</sequence>

<sequence name="ArgumentExample">
  <block>
    __var base = 10;
    __var sum = Sequence("AddValues", right = base + 4, left = 2);
    __var withDefault = Sequence("AddValues", left = 3);
    // sum is 16; withDefault is 3.
  </block>
</sequence>
```

## Sequence return values

- `<return>` has an optional `value` attribute containing an unsigned 64-bit number or an expression resolving to one. If omitted, `value` defaults to `0`.
- When reached, `<return>` evaluates its value and ends the current sequence,  including any enclosing controls. The calling `Sequence()` returns that value,  and the caller continues execution.
- Reaching the end without executing `<return>` yields `0`. Existing abort behavior and save/restore rules for `__dp`, `__ap`, `__apid`, and
  `__errorcontrol` apply.

```xml
<sequence name="SelectValue">
  <parameter name="enabled" default="0"/>
  <parameter name="value" default="1"/>
  <control if="enabled">
    <return value="value"/>
  </control>
</sequence>
```

## Last function error

`__lasterror` is a predefined, read-only variable.

- It clears to `0` when a debug access function begins execution, after argument evaluation. Errors set it to `-1` unless detailed codes are specified for a function.
- Inspect or copy it before another function call clears it. Variable reads and arithmetic without function calls do not clear it.
- For an atomic block, it updates after block execution with the first error in the block, or `0` if no error occurred. Follow-up errors do not replace the first error. It must not be read inside the block.
- `__errorcontrol` bit 0 must be `1` to continue after an error. Otherwise, execution aborts before the sequence can read `__lasterror` again.

Debugger error output should identify the failing function and sequence location. Split complex expressions and nested calls into separate statements when this makes execution easier to follow.

```xml
<sequence name="ProbeRead">
  <parameter name="address" default="0"/>
  <block>
    __var savedErrorControl = __errorcontrol;
    __errorcontrol = __errorcontrol | 1;
    Read32(address);
    __var accessError = __lasterror;
    __errorcontrol = savedErrorControl;
  </block>
  <return value="accessError"/>
</sequence>
```

## Errors, results, and return values

| Mechanism | Purpose |
| --- | --- |
| `__lasterror` | Engine-reported error status of the last debug access function call. |
| `__Result` | Status reported to the debugger: `0` for success, `-1` for error. |
| Sequence return value | Numeric value returned to the caller of `Sequence()`. Its interpretation is defined by the sequence implementation. |

- The debugger initializes `__Result` to `0` before invoking a sequence and uses it as the status when execution returns to the debugger. Nested `Sequence()` calls do not reinitialize or stack it; assignments remain visible to the caller.
- A sequence's return-value format may have no space for an error indicator. It can encode errors only if the format defined by its implementation allows it. `__Result` reports status independently of the returned value.
- For `Sequence()`, `__lasterror` is nonzero only if the call itself fails, for example due to invalid call arguments. It does not propagate errors from functions inside the called sequence. If needed, the called sequence can set `__Result` to `-1` when its `__lasterror` is nonzero.
- Returning a value does not implicitly modify `__lasterror` or `__Result`. Assigning `__Result` does not end sequence execution or supply a return value.

## Atomic blocks

- Do not read `__lasterror` or change `__errorcontrol` inside an atomic block. Set the error-handling mode before the block and inspect its result afterward, if execution continues.
- `<return>` is allowed only as a child of `<sequence>` or `<control>`, never inside a `<block>`. `Query()` and `Sequence()` remain prohibited in atomic blocks.
- A variable assigned a target-access result must not be used as an r-value later in the same atomic block. This also applies to writable globals.

Assignments to writable globals follow the same rules as assignments to local variables, including when an access fails.

## Global declarations

| Declaration | Sequence access | Initialization |
| --- | --- | --- |
| `<debugvars>` | Read-only | Device settings > `.dbgconf` > declared value |
| `<debugvars readonly="false">` | Read-write | Device settings > `.dbgconf` > declared value |
| `<debugvars configurable="false">` | Read-only | Declared value; no configuration overrides |
| `<debugvars readonly="false" configurable="false">` | Read-write | Declared value; no configuration overrides |

- `readonly` and `configurable` are boolean attributes accepting `1`, `0`, `true` and `false`. Each defaults independently to `true`.
- Declarations use `__var` with mandatory constant unsigned 64-bit initializers. Expressions are not allowed.
- `readonly` restricts writes by sequences. Configurable read-only variables can still receive initialization overrides.
- Only variables with `configurable` true can be initialized through [`device-settings:`](https://open-cmsis-pack.github.io/cmsis-toolbox/pyOCD-Debugger/#device-settings) nodes in `csolution.yml` and `.dbgconf` files, with device settings taking precedence.
- At each device-hierarchy level, a declaration block replaces the complete inherited block with the same effective `(readonly, configurable)` values. Contents are not merged. Apply attribute defaults before matching; `1` and `true` are equivalent, as are `0` and `false`.
- Predefined and user-defined sequences access the same global declarations. Sequence calls do not stack these variables; sequence entry does not reapply initializers or configuration overrides.
- Variables declared in `<debugvars>` are initialized once, before the first debug access sequence for a device connection. They are shared by all sequences for all processors within that connection.
- Reset, flash and trace operations, and connecting or disconnecting an individual processor, do not reinitialize them. Sequences explicitly assign writable globals when reinitialization is needed.
- Values are retained through the final device-disconnect sequence and discarded when the device connection closes. A new device connection initializes a new set of variables. Independent debugger sessions have separate contexts.
- Assignments to writable globals are not guaranteed to be atomic.

Configuration-file handling across multiple configurable groups also needs clarification.

```xml
<debugvars>
  __var BoardOption = 0;
</debugvars>

<debugvars configurable="false">
  __var MaxBufferSize = 4096;
</debugvars>

<debugvars readonly="false" configurable="false">
  __var CachedState = 0;
</debugvars>

<sequence name="ClearCachedState">
  <block>
    CachedState = 0;
  </block>
</sequence>
```
