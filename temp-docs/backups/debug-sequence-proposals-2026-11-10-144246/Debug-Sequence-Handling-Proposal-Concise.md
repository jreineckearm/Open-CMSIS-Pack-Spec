# Proposal: improved CMSIS debug sequence handling

All values are unsigned 64-bit integers.

## Sequence parameters and arguments

- Declare `<parameter>` children at the top of `<sequence>`, before any `<block>`
  or `<control>`. Both `name` and `default` are mandatory. Names must be unique;
  defaults must be constants.
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
  <block>
    __returnvalue = left + right;  // Alternative 1
  </block>
</sequence>

<sequence name="ArgumentExample">
  <block>
    __var base = 10;
    Sequence("AddValues", right = base + 4, left = 2);
    __var sum = __returnvalue;
    Sequence("AddValues", left = 3);
    __var withDefault = __returnvalue;
    // sum is 16; withDefault is 3.
  </block>
</sequence>
```

## Sequence return values

- `<control return="expression">` evaluates its `if` and `while` conditions
  first. All supplied conditions must be true; omitted conditions impose no
  restriction. A false condition skips the return expression.
- A return ends the current sequence before that control's children execute,
  exits enclosing controls, and supplies the value returned by `Sequence()`.
  The caller continues execution.
- Reaching the end without taking a return yields `0`. Existing abort behavior
  and save/restore rules for `__dp`, `__ap`, `__apid`, and `__errorcontrol` apply.
  Return values do not modify `__Result`.

```xml
<sequence name="SelectValue">
  <parameter name="enabled" default="0"/>
  <parameter name="value" default="1"/>
  <control if="enabled" return="value"/>
</sequence>
```

**Alternative 1 — `__returnvalue`:** add a predefined read-write output variable.
The callee assigns it; the caller reads it after `Sequence()`. Initialization and
nested-call rules need definition.

**Alternative 2 — global read-write variable:** use an output variable declared
with `__rwvar` in `<debugvars>`. The callee writes it; the caller reads it after
`Sequence()`. Calls sharing the variable can overwrite earlier results.

Both leave `Sequence()` returning `0` and provide no early exit.

**Assessment:** explicit returns provide an output per call and early exit, but
require result storage, propagation through nested controls, and state restoration.
64-bit expressions are already supported. Variable alternatives may be simpler
if output at normal completion suffices. `__returnvalue` adds predefined state
and lifecycle rules; a global `__rwvar` uses the proposed global mechanism with
shared state managed by sequences. Prefer explicit returns for reusable functions
and early exits; consider variable alternatives when minimizing engine changes.

## Last function error

`__lasterror` is a predefined, read-only variable that can always be read.

- It clears to `0` when a debug access function begins execution, after argument
  evaluation. Errors set it to `1` unless the function specifies detailed codes.
  It is independent of the function's return value and `__Result`.
- Inspect or copy it before another function call clears it. Variable reads and
  arithmetic without function calls do not clear it.
- In an atomic block, it updates after block execution. Do not use it as input
  to another debug access function within that block.
- After `Sequence()`, it describes that call itself. Successful completion
  reports `0`, including when the called sequence suppressed access errors.
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
    __returnvalue = __lasterror;
    __errorcontrol = savedErrorControl;
  </block>
</sequence>
```

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

## Open question

How should packs declare required features and tools reject unsupported ones?
Existing lax attribute rules and text declarations mean XSD validation alone
cannot establish support for all extensions.
