# Proposal: improved CMSIS debug sequence handling

Status: initial draft, 9 November 2026.

This proposal extends debug access sequences with named arguments, return values,
observable function errors, and global declarations. All values use the
existing unsigned 64-bit expression model.

The baseline is the local specification at `6410abc` (schema version 1.7.62):
[debug access syntax, functions, and variables](doxygen/src/debug_description.txt),
[debug description elements](doxygen/src/devices_schema.txt), and
[PACK.xsd](schema/PACK.xsd). See also the
[published debug description](https://open-cmsis-pack.github.io/Open-CMSIS-Pack-Spec/main/html/debug_description.html).

## Sequence parameters and arguments

A `<sequence>` element can contain zero or more `<parameter>` children declaring
parameters accepted by that sequence. These declarations must appear at the top
of the sequence, before any `<block>`, `<control>`, or `<return>` children. Both
`name` and `default` are mandatory attributes.

| Attribute | Meaning |
| --- | --- |
| `name` | Parameter name. It must be unique within the sequence's parameter declarations. |
| `default` | Constant unsigned 64-bit value used when the caller omits the argument. Expressions are not allowed. |

The `Sequence()` debug access function accepts named arguments after the sequence
name, using `name = value`:

```text
Sequence("name", parameterName = value, anotherName = expression)
```

Here, `=` binds a value to a parameter of the called sequence; it does not assign
to a variable in the caller.

Each supplied name must match a `<parameter>` declaration of the selected sequence
and must occur only once in the call. Arguments can be supplied in any order.
Unknown or repeated argument names are errors. Arguments omitted from the call's
argument list use the value of the `default` attribute in their corresponding
`<parameter>` definition.

Each supplied value is an unsigned 64-bit number or an expression resolving to
one. Argument expressions are evaluated in the caller's context from left to
right, in the order written in the call, and passed by value. The parameter name
is available as a local variable in the called sequence, including its child
controls. Assigning to it changes only that invocation's copy. Parameter names
follow the existing variable naming and no-redefinition rules.

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

`<return>` has an optional `value` attribute containing an unsigned 64-bit number
or an expression resolving to one. If omitted, `value` defaults to `0`.

When execution reaches `<return>`, its value is evaluated and the current
sequence ends immediately, including any enclosing controls. The calling
`Sequence()` function returns that value and the caller continues execution.

```xml
<sequence name="SelectValue">
  <parameter name="enabled" default="0"/>
  <parameter name="value" default="1"/>
  <control if="enabled">
    <return value="value"/>
  </control>
</sequence>
```

If execution reaches the end of the sequence without executing `<return>`,
`Sequence()` returns `0`. An error that aborts execution retains the existing
abort behavior.

The existing save/restore rules for `__dp`, `__ap`, `__apid`, and `__errorcontrol`
also apply when a sequence exits through `<return>`.

## Last function error

Introduce the read-only predefined debug access variable `__lasterror`.

| Value | Meaning |
| --- | --- |
| `0` | No error reported by the function call. |
| `-1` | An error occurred, unless detailed codes are specified for the function. |
| Function-specific nonzero value | A detailed error code specified for that function. |

`__lasterror` is cleared to `0` when a debug access function begins execution,
after its argument expressions have been evaluated. Errors set it to `-1` unless
detailed codes are specified for a function. A later function call clears the
previous value, so a sequence must inspect or copy it before making another call.

In an atomic block, `__lasterror` is updated after the block has executed. Do not
use it as input to another debug access function within the same atomic block.

A sequence can always read `__lasterror`. Setting `__errorcontrol` bit 0 to `1`
allows execution to continue after a function error so the sequence can inspect
the error value. Otherwise, the error aborts sequence execution before the value
can be read again. Reading a variable or evaluating arithmetic alone does not
constitute a debug access function call.

Debugger error output should identify the failing function and its location in
the sequence to help determine the root cause. Sequence authors should split
complex expressions and nested function calls into separate statements when
this makes execution easier to follow.

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

This example returns the access error code independently of the read data.

## Errors, results, and return values

| Mechanism | Purpose |
| --- | --- |
| `__lasterror` | Engine-reported error status of the most recent debug access function call. |
| `__Result` | Sequence result reported to the debugger: `0` for success, `-1` for error. |
| Sequence return value | Data or an implementation-defined error code returned to the caller of `Sequence()`. |

The debugger initializes `__Result` to `0` before invoking a sequence. Sequence
implementations can assign it; nested `Sequence()` calls do not stack the
variable.

For `Sequence()`, `__lasterror` is nonzero only if the call itself fails, for
example due to invalid call arguments. Errors from debug access functions within
the called sequence are not propagated into the caller's `__lasterror`. Sequence
implementations report their own errors through `<return value="..."/>`; the
caller interprets the returned value according to that sequence's definition.

Returning a value does not implicitly modify `__lasterror` or `__Result`.
Assigning `__Result` does not end sequence execution or supply a return value.

## Global declarations

Global constants and variables use the existing `__var` declaration syntax.
Each declaration requires a constant unsigned 64-bit initializer; expressions
are not allowed.

| Element | Sequence access | Initialization |
| --- | --- | --- |
| `<debugvars>` | Read-write | Device settings > `.dbgconf` > declared value |
| `<debugconsts>` | Read-only | Declared value; no configuration overrides |
| `<debugglobals>` | Read-write | Declared value; no configuration overrides |

```xml
<debugvars>
  __var BoardOption = 0;
</debugvars>

<debugconsts>
  __var MaxBufferSize = 4096;
</debugconsts>

<debugglobals>
  __var CachedState = 0;
</debugglobals>
```

Existing `<debugvars>` declarations become read-write within sequences. Only
variables declared in `<debugvars>` can be configured through device-settings
nodes in `csolution.yml` and `.dbgconf` files. Device settings take precedence
over `.dbgconf`, which takes precedence over the declared initialization value.

In general, the sequence engine is initialized before the first sequence
execution of a session and its state is discarded after the last sequence
execution. The exact timing and session boundaries depend on the debugger's
strategy and use case.

After initialization, all three groups are global within the engine and shared
by all its sequences. Sequence calls do not stack these variables. Writable
values persist across calls. Leaving a sequence or control does not destroy
them.

```xml
<sequence name="ClearCachedState">
  <block>
    CachedState = 0;
  </block>
</sequence>
```

Entering a sequence does not reapply the initializer or configuration overrides.
If a sequence needs a variable from `<debugvars>` or `<debugglobals>` reinitialized,
its implementation must assign it explicitly, as above.

Updates to variables in `<debugvars>` and `<debugglobals>` are not guaranteed to
be atomic. Debug sequences are not expected to execute concurrently, so this is
not expected to be an issue.

## Specification integration

| Area | Required change |
| --- | --- |
| `SequenceType` | Add `<parameter>` declarations directly under `<sequence>`, before its executable children. Keep them out of the shared block/control/return child group. |
| `SequenceElementGroup` and a new return element type | Add empty `<return>` elements under `<sequence>` and `<control>`, with optional expression attribute `value` defaulting to `0`. |
| Expression language and `Sequence()` | Parse and validate `name = value` arguments, bind local parameter values, and return the sequence's value. |
| Predefined debug access variables | Define `__lasterror` and its interaction with error suppression and function-specific codes. |
| Global declaration elements | Add `<debugconsts>` and `<debugglobals>` alongside `<debugvars>`, using existing `__var` syntax with constant unsigned 64-bit initializers and shared global storage. |
| Declaration access and configuration | Make `<debugvars>` read-write, `<debugconsts>` read-only, and `<debugglobals>` read-write. Allow device-settings and `.dbgconf` overrides only for `<debugvars>`, with device settings taking precedence. |

Existing sequences without arguments or returns retain their zero return value.
This draft describes the proposal; schema and normative specification updates
follow agreement.
