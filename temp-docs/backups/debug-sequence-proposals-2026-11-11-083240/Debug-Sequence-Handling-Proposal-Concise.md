# Proposal: improved CMSIS debug sequence handling

All values are unsigned 64-bit integers.

## Sequence parameters and arguments

- Declare `<parameter>` children at the top of `<sequence>`, before any `<block>`,
  `<control>`, or `<return>`. Both `name` and `default` are mandatory. Names must
  be unique; defaults must be constants.
- Call arguments use `name = value`, where the value can be an expression.
  Supply arguments for declared parameters in any order, each name at most once.
  Unknown or repeated names are errors. Omitted arguments use the corresponding
  `<parameter>` default.
- Argument expressions are evaluated left to right in the caller and passed by
  value. Parameters are local variables in the called sequence and its child
  controls; assignments affect only that invocation's copy. Existing variable
  naming and no-redefinition rules apply.

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

- `<return>` has an optional `value` attribute containing an unsigned 64-bit
  number or an expression resolving to one. If omitted, `value` defaults to `0`.
- When reached, `<return>` evaluates its value and ends the current sequence,
  including any enclosing controls. The calling `Sequence()` returns that value,
  and the caller continues execution.
- Reaching the end without executing `<return>` yields `0`. Existing abort
  behavior and save/restore rules for `__dp`, `__ap`, `__apid`, and
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

`__lasterror` is a predefined, read-only variable that can always be read.

- It clears to `0` when a debug access function begins execution, after argument
  evaluation. Errors set it to `-1` unless the function specifies detailed codes.
- Inspect or copy it before another function call clears it. Variable reads and
  arithmetic without function calls do not clear it.
- In an atomic block, it updates after block execution. Do not use it as input
  to another debug access function within that block.
- `__errorcontrol` bit 0 must be `1` to continue after an error. Otherwise,
  execution aborts before the sequence can read `__lasterror` again.

Debugger error output should identify the failing function and sequence
location. Split complex expressions and nested calls into separate statements
when this makes execution easier to follow.

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
| `__Result` | Sequence result reported to the debugger: `0` for success, `-1` for error. |
| Sequence return value | Data or an implementation-defined error code returned to the caller of `Sequence()`. |

- The debugger initializes `__Result` to `0` before invoking a sequence.
  Sequences can assign it; nested `Sequence()` calls do not stack the variable.
- For `Sequence()`, `__lasterror` is nonzero only if the call itself fails, for
  example due to invalid call arguments. It does not propagate errors from
  functions inside the called sequence. Implementations report their own errors
  through `<return value="..."/>`; callers interpret the returned value.
- Returning a value does not implicitly modify `__lasterror` or `__Result`.
  Assigning `__Result` does not end sequence execution or supply a return value.

## Global read-write variables

- `__rwvar` declares a global read-write variable in `<debugvars>` with a
  mandatory constant initializer. The keyword is not allowed in sequence
  implementations. Existing `<debugvars>` declarations using `__var` remain
  read-only within sequences.
- Initialization precedence: device settings > `.dbgconf` > `<debugvars>`.
  CMSIS Debugger device settings use `device-settings` in `csolution.yml`.
- The engine is generally initialized before the first sequence execution of a
  session; its state is discarded after the last. Exact timing and session
  boundaries depend on the debugger's strategy and use case.
- After initialization, a `__rwvar` variable is shared by all sequences in the
  engine, independent of PDSC hierarchy, sequence, or control scope. Calls do
  not save/restore it. Leaving a sequence or control does not destroy it;
  entering a sequence does not reapply initializers or configuration overrides.
  Sequences must explicitly reinitialize it when needed.
- Updates are not guaranteed to be atomic. Debug sequences are not expected to
  execute concurrently.

```xml
<debugvars>
  __var BoardOption = 0;
  __rwvar CachedState = 0;
</debugvars>

<sequence name="ClearCachedState">
  <block>
    CachedState = 0;
  </block>
</sequence>
```
