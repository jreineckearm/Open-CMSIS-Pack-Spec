# Proposal: improved CMSIS debug sequence handling

Status: initial draft, 9 November 2026.

This proposal extends debug access sequences with named arguments, return values,
observable function errors, and global read-write variables. All values use the
existing unsigned 64-bit expression model.

The baseline is the local specification at `6410abc` (schema version 1.7.62):
[debug access syntax, functions, and variables](doxygen/src/debug_description.txt),
[debug description elements](doxygen/src/devices_schema.txt), and
[PACK.xsd](schema/PACK.xsd). See also the
[published debug description](https://open-cmsis-pack.github.io/Open-CMSIS-Pack-Spec/main/html/debug_description.html).

## Sequence parameters and arguments

A `<sequence>` element can contain zero or more `<parameter>` children declaring
parameters accepted by that sequence. These declarations must appear at the top
of the sequence, before any `<block>` or `<control>` children. Both `name` and
`default` are mandatory attributes.

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
  <control return="left + right"/>
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

A `<control>` element can have a `return` attribute containing an unsigned 64-bit
number or an expression resolving to one.

The existing `if` and `while` conditions are evaluated before `return`. The return
takes effect only when all supplied conditions are true; an omitted condition
places no restriction on it. If a condition prevents entry, the return expression
is not evaluated.

An active return evaluates its expression and immediately ends the current
sequence, including any enclosing controls. It takes effect before any children
of that control execute. The calling `Sequence()` function returns the resulting
value and the caller continues execution.

```xml
<sequence name="SelectValue">
  <parameter name="enabled" default="0"/>
  <parameter name="value" default="1"/>
  <control if="enabled" return="value"/>
</sequence>
```

If execution reaches the end of the sequence without taking a return,
`Sequence()` returns `0`. This also applies when return controls are present but
their conditions are false. An error that aborts execution retains the existing
abort behavior.

The returned value is separate from `__Result`. Returning a value does not
implicitly modify `__Result`; its existing sequence success/error role remains.
The existing save/restore rules for `__dp`, `__ap`, `__apid`, and `__errorcontrol`
also apply when a sequence exits through `return`.

**Alternative:** reuse `__Result` for sequence output instead of introducing
`return`. It already carries a result, but currently only `0` (success) and `-1`
(error) are defined; other values are reserved. It is not saved/restored across
nested calls, and assignments do not end execution or change `Sequence()`'s
current zero return value. Reusing it would require defining how output data
coexists with success/error status and whether early exit is needed.

**Assessment:** explicit returns give each call its own output and support early
exit, but require the engine to carry a result per invocation, propagate returns
through nested controls, and restore saved state on that exit. Arbitrary 64-bit
values add relatively little complexity because the expression engine already
supports them. Reusing `__Result` may be simpler if output at normal sequence
completion is sufficient, provided its success/error role and nested-call
behavior are resolved. Explicit returns favor clear semantics for reusable
functions and early exits; `__Result` is worth considering when minimizing engine
changes is the priority.

## Last function error

Introduce the read-only predefined debug access variable `__lasterror`.

| Value | Meaning |
| --- | --- |
| `0` | No error reported by the function call. |
| `1` | An error occurred, unless that function specifies a detailed error code. |
| Function-specific nonzero value | A detailed error code specified for that function. |

`__lasterror` is cleared to `0` when a debug access function begins execution,
after its argument expressions have been evaluated. If the function encounters
an error, it sets `__lasterror` to `1`, or to a detailed code defined for that
function. A later function call clears the previous value, so a sequence must
inspect or copy it before making another call.

In an atomic block, `__lasterror` is updated after the block has executed. Do not
use it as input to another debug access function within the same atomic block.

After a `Sequence()` call, `__lasterror` refers to the `Sequence()` call itself.
A successfully completed call reports `0`, including when errors from debug
access functions were suppressed within the called sequence.

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
  <control return="accessError"/>
</sequence>
```

This example returns the access error code independently of the read data.
The new variable does not replace a function's existing return value or the
sequence result in `__Result`.

## Global read-write variables

Extend `<debugvars>` with the declaration keyword `__rwvar`:

```xml
<debugvars>
  __var BoardOption = 0;
  __rwvar CachedState = 0;
</debugvars>
```

`__rwvar` defines a global unsigned 64-bit variable that sequences can read and
write. Its declaration must include an initialization value. The keyword is
permitted only in `<debugvars>`, and is not permitted in the sequence
implementation language. Existing `__var` declarations in `<debugvars>` retain
their read-only behavior within sequences.

As with existing `<debugvars>` declarations, initializers are constant unsigned
numbers. For initial values, device settings (`device-settings` in `csolution.yml`
for the CMSIS Debugger) take precedence over `.dbgconf`, which takes precedence
over the `<debugvars>` section.

In general, the sequence engine is initialized before the first sequence
execution of a session and its state is discarded after the last sequence
execution. The exact timing and session boundaries depend on the debugger's
strategy and use case.

After initialization, a `__rwvar` variable is global within the sequence engine.
Its storage is not bound to a PDSC hierarchy level, sequence invocation, or
control scope. All sequences using that engine access the same value. A sequence
call does not save and restore it, and leaving a sequence or control does not
destroy it.

```xml
<sequence name="ClearCachedState">
  <block>
    CachedState = 0;
  </block>
</sequence>
```

Entering a sequence does not reapply the initializer or configuration overrides.
If a sequence needs a global variable reinitialized, its implementation must do
so explicitly, as above.

Updates to global read-write variables are not guaranteed to be atomic. Debug
sequences are not expected to execute concurrently, so this is not expected to
be an issue.

## Specification integration

| Area | Required change |
| --- | --- |
| `SequenceType` | Add `<parameter>` declarations directly under `<sequence>`, before its executable children. Keep them out of the shared control/block child group. |
| `SequenceControlType` | Add optional `return` with expression syntax. |
| Expression language and `Sequence()` | Parse and validate `name = value` arguments, bind local parameter values, and return the sequence's value. |
| Predefined debug access variables | Define `__lasterror` and its interaction with error suppression and function-specific codes. |
| `<debugvars>` language | Recognize `__rwvar`, initialization overrides, global storage, and sequence assignments. `DebugVarsType` already stores declarations as text. |

Existing sequences without arguments or returns retain their zero return value.
Existing read-only debug variables retain their behavior. This draft describes
the proposal; schema and normative specification updates follow agreement on
the details below.

## Details to settle

- **Feature support:** how packs declare these requirements and tools reject
  unsupported features. The existing lax attribute rules and text declarations
  mean XSD validation alone cannot establish support for all extensions.
