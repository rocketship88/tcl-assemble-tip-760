# tcl::unsupported::assemble (TAL) Opcode Reference


> Draft manual entries for TIP 760. Generated against a Tcl 9.x source tree
> (`tclAssembly.c`, `tclExecute.c`, `tclCompile.c/.h`, `tclInt.h`,
> `tclOOInt.h`) supplied for this pass. Organized by `TalInstType` "shape"
> — the assembler's classification of an opcode's *operand and stack-effect
> pattern*, declared as an enum near the top of `tclAssembly.c` — in the
> enum's declaration order, then alphabetically by mnemonic within each
> shape. "LVT" below means Local Variable Table, the per-procedure slot
> array TAL source references local variables through.

Each entry below follows the pattern shown here for `add`, the
mnemonic you'd actually write in TAL source (`add` in a `tcl::unsupported::assemble`
script compiles to one `INST_ADD` bytecode):

> ### `add`
>
> • **Underlying instruction:** `INST_ADD` — the actual Tcl bytecode opcode this mnemonic assembles to; the same name `tcl::unsupported::disassemble` prints.
>
> • **Compile-time operand(s):** none — arguments written after the mnemonic in the TAL source line itself (a variable name, a jump label, a literal count...), fixed when the code is assembled, as distinct from the values read off the stack when it runs.
>
> • **Stack effect:** consumes 2, produces 1 — how many values this instruction pops ("consumes") off the runtime evaluation stack and pushes ("produces") back on; `add` pops two operands and pushes their sum. `INT_MIN` in this field means "a variable count, read from a compile-time operand" rather than a fixed number.
>
> Pops two operands, pushes their sum — then the instruction's actual
> runtime behavior continues here, as implemented in `tclExecute.c`.

`add` happens to take no operands. Where a mnemonic does take
one or more, written directly after it on the TAL source line, they're
named and italicized right in the heading the same way — e.g. `jump` *label*,
or `dictSet` *n* *varName* for two at once.


## Shapes summary


29 shapes, 156 opcode mnemonics total (150 distinct underlying instructions — some mnemonics alias the same instruction, e.g. `concat`/`strcat`).


| Shape | Count | What it covers |
| --- | ---: | --- |
| `ASSEM_1BYTE` | 93 | Fixed-arity, no-operand instructions: a single opcode byte, nothing else in the bytecode stream. Stack effect is fixed and given by the table. |
| `ASSEM_BEGIN_CATCH` | 1 | Marks the start of a `catch`-protected region. Takes one operand: a jump target used by the compiler/assembler to build the exception-range table (it is not read at runtime as a jump — the runtime instruction only records the stack depth). |
| `ASSEM_BOOL` | 3 | One boolean operand, assembled as a literal 0/1 byte (`GetBooleanOperand`), consumed at runtime as a flags byte. |
| `ASSEM_BOOL_LVT` | 2 | One boolean operand plus one variable reference resolved against the assembly's Local Variable Table (LVT) at compile time. |
| `ASSEM_CLOCK_READ` | 1 | One 1-byte unsigned case selector (0-3) choosing which clock to sample. |
| `ASSEM_CONCAT1` | 2 | One 1-byte unsigned operand count N (must be > 0); consumes N operands, produces 1. |
| `ASSEM_DICT_GET` | 2 | One operand: key count N (must be > 0); consumes N+1 operands (the dict plus N keys), produces 1. |
| `ASSEM_DICT_SET` | 1 | Two operands: key count N (> 0) and an LVT index; consumes N+1 operands (N keys plus the value), produces 1 (the updated dict, or nothing if the assembler folds in a following `pop`). |
| `ASSEM_DICT_UNSET` | 1 | Two operands: key count N (> 0) and an LVT index; consumes N key operands, produces 1. |
| `ASSEM_END_CATCH` | 1 | No operands. Pops the exception-range/catch stack and resets the interpreter result. |
| `ASSEM_EVAL` | 2 | One operand — a script or expression, either a compile-time literal (compiled directly inline by the assembler) or a run-time value (pushed then evaluated by the underlying instruction). |
| `ASSEM_INDEX` | 1 | One 4-byte operand: a list index, which may be a plain integer or an `end`-relative index expression, pre-encoded at assembly time. |
| `ASSEM_INVOKE` | 1 | One operand: an argument count (must be > 0); consumes that many operands, produces 1. |
| `ASSEM_JUMP` | 6 | One operand: a label naming the jump target; the assembler resolves it to a 4-byte relative pc offset once basic-block layout is finalized. |
| `ASSEM_JUMPTABLE` | 2 | One operand: a flat `{key targetLabel ...}` list, turned into a `JumptableInfo`/`JumptableNumInfo` auxiliary-data record referenced by a 4-byte index. |
| `ASSEM_LABEL` | 1 | Not a runtime instruction at all — an assembler directive. One operand names a label at the current code address; it emits no bytecode. |
| `ASSEM_LINDEX_MULTI` | 1 | One operand: an index-argument count N (must be > 0); consumes N+1 operands (list plus N indices), produces 1. |
| `ASSEM_LIST` | 2 | One operand: an element/operand count N (must be >= 0); consumes N operands, produces 1. |
| `ASSEM_LSET_FLAT` | 1 | One operand: total operand count N (must be >= 3); consumes N operands, produces 1. |
| `ASSEM_LVT_N` | 8 | One 4-byte LVT index operand. (Functionally like ASSEM_LVT; the assembler notes it separately because it doesn't advance the compiler's notion of the current source line for debug info.) |
| `ASSEM_LVT_SINT1` | 2 | One 4-byte LVT index plus one 1-byte signed integer operand. |
| `ASSEM_LVT` | 14 | One 4-byte operand referencing a slot in the procedure's Local Variable Table. |
| `ASSEM_OVER` | 1 | One operand: a depth count N; consumes N+1 operands and produces N+2 (it duplicates one, leaving the original N+1 in place below the copy). |
| `ASSEM_PUSH` | 1 | One operand: a literal value, registered in the code's literal pool and referenced by a 4-byte index. Must be table slot 0 (the assembler relies on this to synthesize PUSH internally, e.g. for literal `eval`/`expr` operands). |
| `ASSEM_REGEXP` | 1 | One boolean (`-nocase`) operand which the assembler pre-combines with `TCL_REG_ADVANCED` into a single 1-byte regexp-compile-flags operand — not stored as a plain 0/1 the way ASSEM_BOOL is. |
| `ASSEM_REVERSE` | 1 | One operand: a count N (must be >= 0); consumes and produces the same N operands, reversing their order in place. |
| `ASSEM_SINT1` | 2 | One 1-byte signed integer operand. |
| `ASSEM_SINT4_LVT` | 1 | One 4-byte signed integer operand plus one 4-byte LVT index. |
| `ASSEM_DICT_GET_DEF` | 1 | One operand: key count N (must be > 0); consumes N+2 operands (the dict, N keys, and a default value), produces 1. |


## Operand types


11 distinct kinds of compile-time operand appear across all 29 shapes (a shape may combine two of them, e.g. `dictSet` *n* *varName*); this is every word that shows up italicized in a heading elsewhere in this document.


| Type | Meaning | Written in TAL as | Example | Encoded in bytecode as | Used by |
| --- | --- | --- | --- | --- | ---: |
| *n* | A count — how many things (keys, list elements, stack values...) the instruction should act on. | a plain non-negative integer literal | `list 3` | 1 or 4 bytes, depending on the instruction — see its own entry | 19 mnemonics |
| *label* | The name of a `label` directive elsewhere in the same assembly. | a label name | `jump loopTop` | 4-byte relative pc offset, resolved once code layout is final | 7 mnemonics |
| *varName* | A local variable's name. | a variable name | `load x` | 4-byte index into the procedure's Local Variable Table (LVT), resolved at assembly time | 29 mnemonics |
| *boolean* | An on/off switch. | `0`/`1` (or another token `GetBooleanOperand` accepts) | `strmatch 1` | 1 byte | 6 mnemonics |
| *value* | A literal value to push. | any single word/braced/quoted value | `push {hello}` | 4-byte index into the code's literal-object pool | 1 mnemonic |
| *name* | A label's own name, as declared. | a label name | `label loopTop` | not encoded into the bytecode at all — it's purely a key into the assembler's own label table | 1 mnemonic |
| *table* | A jump table: alternating match-value/target-label pairs. | a flat Tcl list, e.g. `{a lbl1 b lbl2}` | `jumpTable {a lbl1 b lbl2}` | 4-byte index into a `JumptableInfo`/`JumptableNumInfo` AuxData record built at assembly time | 2 mnemonics |
| *index* | A list index, possibly `end`-relative. | an integer or an `end`/`end-N` expression | `listIndexImm end-1` | 4-byte pre-encoded index, resolved at assembly time | 1 mnemonic |
| *case* | A small selector among a fixed, short menu of choices (only used by `clockRead`, choosing which of 4 clocks to read). | an integer 0-3 | `clockRead 2` | 1 byte | 1 mnemonic |
| *script* | A Tcl script (only used by `eval`). | a script body, braced or not | `eval {set x 1}` | literal form: compiled inline at assembly time, no runtime operand at all; non-literal form: pushed as a literal and compiled/run at runtime by `INST_EVAL_STK` | 1 mnemonic |
| *expression* | A Tcl expression (only used by `expr`). | an expression body, braced or not | `expr {$x + 1}` | same literal/non-literal split as `script`, but compiled as an `expr`, run by `INST_EXPR_STK` | 1 mnemonic |


---

## `ASSEM_1BYTE`  (93 opcodes)

Fixed-arity, no-operand instructions: a single opcode byte, nothing else in the bytecode stream. Stack effect is fixed and given by the table.

> ### `add`
>
> • **Underlying instruction:** `INST_ADD`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two operands, pushes their sum. Operands must already be (or convert to)
> numbers; non-numeric operands raise an error identifying which side ("left "/
> "right ") was illegal. Wide-integer operands are added with overflow checking
> (`Overflowing` macro on the two's-complement sum); on overflow the result is
> promoted to bignum arithmetic. Floating-point and mixed operands fall through
> to the general numeric-promotion path shared with `expr`.
>
>
> ### `appendArrayStk`
>
> • **Underlying instruction:** `INST_APPEND_ARRAY_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 3, produces 1
>
> Pops a value, an element name, and an array name (in that order, value on
> top); appends the value (as a plain string, via `TCL_APPEND_VALUE`, no list
> formatting) to the array element `arrayName(elementName)`, creating the
> element if needed, and pushes the element's new value. This is the
> "name looked up from the stack" counterpart of `appendArray`.
>
>
> ### `appendStk`
>
> • **Underlying instruction:** `INST_APPEND_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a value and a variable name; appends the value as a plain string to that
> scalar variable (creating it if undefined) and pushes the new value.
> Stack-name counterpart of `append`.
>
>
> ### `arrayExistsStk`
>
> • **Underlying instruction:** `INST_ARRAY_EXISTS_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a variable name, pushes 1 if a variable of that name exists and is
> currently an array, else 0. Array traces on the variable are consulted (via
> `TclCheckArrayTraces`) before the check; a triggered trace that raises an
> error aborts execution. Followed directly by a conditional jump, this
> instruction folds into it via the interpreter's peephole optimizer.
>
>
> ### `arrayMakeStk`
>
> • **Underlying instruction:** `INST_ARRAY_MAKE_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 0
>
> Pops a variable name and turns that variable into an empty array if it is
> currently undefined. If the variable already exists as a scalar (or as an
> array *element*), this raises a "variable isn't array" error. If it is
> already an array, this is a no-op. Produces nothing.
>
>
> ### `bitand`
>
> • **Underlying instruction:** `INST_BITAND`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two operands, pushes their bitwise AND. Both operands must be integers
> (not doubles, not NaN) or the instruction raises an illegal-operand-type
> error; integer operands are ANDed directly as `Tcl_WideInt`, with bignum
> operands routed through the general extended math-operator path.
>
>
> ### `bitnot`
>
> • **Underlying instruction:** `INST_BITNOT`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops one integer operand, pushes its bitwise complement (`~x`). Raises an
> error for non-integer (double/NaN) operands.
>
>
> ### `bitor`
>
> • **Underlying instruction:** `INST_BITOR`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two integer operands, pushes their bitwise OR. Same type restrictions and bignum fallback as `bitand`.
>
>
> ### `bitxor`
>
> • **Underlying instruction:** `INST_BITXOR`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two integer operands, pushes their bitwise XOR. Same type restrictions and bignum fallback as `bitand`.
>
>
> ### `coroName`
>
> • **Underlying instruction:** `INST_COROUTINE_NAME`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 0, produces 1
>
> Pushes the fully qualified command name of the coroutine currently executing
> this bytecode, or the empty string if not running inside a coroutine (or if
> the coroutine's command is in the process of being deleted).
>
>
> ### `currentNamespace`
>
> • **Underlying instruction:** `INST_NS_CURRENT`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 0, produces 1
>
> Pushes the fully qualified name of the namespace that is current at this point in execution.
>
>
> ### `dictExpand`
>
> • **Underlying instruction:** `INST_DICT_EXPAND`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 3, produces 1
>
> Pops a key/value list and a dict; for each key in the list not present as an
> actual dict key, sets a variable of that name from the corresponding dict
> entry (`dict with`-style expansion via `TclDictWithInit`), and pushes the
> resulting dict. Consumes 2 (dict, list), produces 1.
>
>
> ### `dictPut`
>
> • **Underlying instruction:** `INST_DICT_PUT`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 3, produces 1
>
> Pops a value, a key, and a dict (dict lowest); sets `dict[key] = value`,
> copy-on-write duplicating the dict object first if it was shared, and pushes
> the (possibly new) dict object.
>
>
> ### `dictRecombineStk`
>
> • **Underlying instruction:** `INST_DICT_RECOMBINE_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 3, produces 0
>
> Pops a key list, a variable name, and a key/value list (key list on top);
> performs the `dict with`-exit recombination that writes the (possibly
> modified) key/value list back into the dict stored in the named variable,
> restricted to the given path of keys. Produces nothing — used for the "leave
> scope of `dict with`" step where variable name comes from the stack rather
> than the LVT.
>
>
> ### `dictRemove`
>
> • **Underlying instruction:** `INST_DICT_REMOVE`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a key and a dict; removes that key from the dict (copy-on-write
> duplicating first if shared) and pushes the resulting dict.
>
>
> ### `div`
>
> • **Underlying instruction:** `INST_DIV`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two operands, pushes their quotient using Tcl's floor-division rule
> (result rounds toward negative infinity, not toward zero, so the sign of the
> remainder always matches the divisor). Integer division by zero raises a
> "divide by zero" error. Wide-integer operands are divided directly with
> overflow/bignum fallback; floating operands go through the general numeric
> path.
>
>
> ### `dup`
>
> • **Underlying instruction:** `INST_DUP`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 2
>
> Duplicates the top stack value, pushing a second reference to the same object (no copy is made; the object's refcount is simply increased).
>
>
> ### `eq`
>
> • **Underlying instruction:** `INST_EQ`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two operands, pushes 1 if they are numerically/string-equal per `expr`'s `==` semantics (`TclCompareObjs`), else 0.
>
>
> ### `evalStk`
>
> • **Underlying instruction:** `INST_EVAL_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a script value, compiles it (via `TclCompileObj`, caching the compiled
> `ByteCode` on the object for reuse), and yields control to run it as a nested
> bytecode execution; when that finishes, its result replaces the popped value
> on the stack. Unlike `eval` (whose *literal* form is spliced into the
> surrounding code at assembly time), `evalStk` always compiles-then-runs the
> popped object at instruction-execution time.
>
>
> ### `existArrayStk`
>
> • **Underlying instruction:** `INST_EXIST_ARRAY_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops an element name and an array name; pushes 1 if that array element
> exists (as a *defined* variable, triggering read traces along the way) or 0
> otherwise. Directly followed by a conditional jump, this folds via the
> peephole optimizer.
>
>
> ### `existStk`
>
> • **Underlying instruction:** `INST_EXIST_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a variable name; pushes 1 if a variable of that name exists and is defined, else 0. Same peephole-jump folding as `existArrayStk`.
>
>
> ### `expon`
>
> • **Underlying instruction:** `INST_EXPON`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops base and exponent, pushes base**exponent. Integer base and non-negative
> integer exponent are computed directly with overflow detection; a negative
> integer exponent on an integer base produces 0 (except base ±1), matching
> `expr`'s integer-power truncation rule. Non-integer operands or exponents,
> or results that would overflow, are routed through the double/bignum
> extended-math path.
>
>
> ### `exprStk`
>
> • **Underlying instruction:** `INST_EXPR_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops an expression string, compiles it (`CompileExprObj`, cached on the
> object) and yields to run it as nested bytecode; its result replaces the
> popped value. `expr`'s literal-operand form compiles the text inline at
> assembly time instead; `exprStk` always compiles the popped value at
> run time.
>
>
> ### `ge`
>
> • **Underlying instruction:** `INST_GE`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two operands, pushes 1 if the first is `>=` the second per `expr` comparison rules, else 0.
>
>
> ### `gt`
>
> • **Underlying instruction:** `INST_GT`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two operands, pushes 1 if the first is `>` the second per `expr` comparison rules, else 0.
>
>
> ### `incrArrayStk`
>
> • **Underlying instruction:** `INST_INCR_ARRAY_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 3, produces 1
>
> Pops an increment amount, an element name and an array name (increment on
> top); adds the increment to that array element (creating it as 0 first if
> undefined) and pushes the new value. Non-integer increments/values fall
> through to the general `TclIncrObj` bignum path.
>
>
> ### `incrStk`
>
> • **Underlying instruction:** `INST_INCR_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops an increment amount and a variable name; adds the increment to that scalar (auto-vivifying it as 0 if undefined) and pushes the new value.
>
>
> ### `infoLevelArgs`
>
> • **Underlying instruction:** `INST_INFO_LEVEL_ARGS`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a stack-level number (as accepted by `info level`: non-positive counts
> relative to the current level); pushes the full command-invocation word list
> (`objv`) of that call frame. Raises a "bad level" error if the level doesn't
> correspond to any frame currently on the Tcl call stack.
>
>
> ### `infoLevelNumber`
>
> • **Underlying instruction:** `INST_INFO_LEVEL_NUM`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 0, produces 1
>
> Pushes the current Tcl call-stack level number (`[info level]` with no arguments) as an integer.
>
>
> ### `isEmpty`
>
> • **Underlying instruction:** `INST_IS_EMPTY`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a value, pushes 1 if it is the empty string, else 0.
>
>
> ### `lappendArrayStk`
>
> • **Underlying instruction:** `INST_LAPPEND_ARRAY_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 3, produces 1
>
> Pops a value, element name and array name; list-appends the value (as a
> proper list element, i.e. quoted/braced if necessary — unlike
> `appendArrayStk`'s plain-string append) to that array element and pushes the
> new value.
>
>
> ### `lappendListArrayStk`
>
> • **Underlying instruction:** `INST_LAPPEND_LIST_ARRAY_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 3, produces 1
>
> Pops a list, an element name and an array name; appends every element of the
> popped list to the array element's own list value (`lappend` semantics with
> multiple new elements at once) and pushes the result.
>
>
> ### `lappendListStk`
>
> • **Underlying instruction:** `INST_LAPPEND_LIST_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a list and a variable name; appends every element of the popped list to that scalar's list value and pushes the result.
>
>
> ### `lappendStk`
>
> • **Underlying instruction:** `INST_LAPPEND_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a value and a variable name; list-appends the value to that scalar variable and pushes the new value.
>
>
> ### `le`
>
> • **Underlying instruction:** `INST_LE`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two operands, pushes 1 if the first is `<=` the second per `expr` comparison rules, else 0.
>
>
> ### `listConcat`
>
> • **Underlying instruction:** `INST_LIST_CONCAT`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two lists, pushes their concatenation (`Tcl_ListObjAppendList`). If the
> first (lower) list is unshared it is extended and reused in place rather
> than copied, as an optimization.
>
>
> ### `listIn`
>
> • **Underlying instruction:** `INST_LIST_IN`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a list and a value (value below); pushes 1 if the value occurs as an
> element of the list (compared as strings), else 0. Custom list types with an
> "in"-operator hook (e.g. abstract/lazy lists) can override the membership
> test. Directly-following conditional jumps are folded via the peephole
> optimizer.
>
>
> ### `listIndex`
>
> • **Underlying instruction:** `INST_LIST_INDEX`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops an index (or list of indices, for nested `lindex ... {i j}` addressing)
> and a list; pushes the addressed element, or the empty string if any index
> is out of range. Delegates most of the work to `TclLindexList`/
> `TclLindexFlat`; abstract-list types are dispatched through their own index
> callback.
>
>
> ### `listLength`
>
> • **Underlying instruction:** `INST_LIST_LENGTH`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a value, pushes its length as a list (errors if it cannot be interpreted as a well-formed list).
>
>
> ### `listNotIn`
>
> • **Underlying instruction:** `INST_LIST_NOT_IN`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a list and a value; pushes 1 if the value is *not* an element of the list, else 0 — the logical negation of `listIn`, with the same membership test and peephole-jump folding.
>
>
> ### `loadArrayStk`
>
> • **Underlying instruction:** `INST_LOAD_ARRAY_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops an element name and an array name; pushes the value of that array element (raising an error if it is undefined).
>
>
> ### `loadStk`
>
> • **Underlying instruction:** `INST_LOAD_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a variable name; pushes that scalar's current value (raising an error if it is undefined).
>
>
> ### `lsetList`
>
> • **Underlying instruction:** `INST_LSET_LIST`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 3, produces 1
>
> Pops a replacement value, an index-path list, and the old list value
> (replacement on top); returns the list with the element addressed by the
> index path replaced. Equivalent to `lset list indices value` where `indices`
> is a single list argument rather than pre-flattened arguments (compare
> `lsetFlat`).
>
>
> ### `lshift`
>
> • **Underlying instruction:** `INST_LSHIFT`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two integer operands (value, shift-amount), pushes value shifted left
> by that many bits. A negative shift amount is a domain error. Integer
> results that would overflow a machine word are promoted to bignum.
>
>
> ### `lt`
>
> • **Underlying instruction:** `INST_LT`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two operands, pushes 1 if the first is `<` the second per `expr` comparison rules, else 0.
>
>
> ### `mod`
>
> • **Underlying instruction:** `INST_MOD`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two integer operands, pushes the remainder using Tcl's floor-division
> rule: the result's sign always matches the divisor's (e.g. `-7 % 3` is `2`,
> not `-1`). Division by zero raises an error; both operands must be integers
> (this instruction does not accept doubles) or a type error is raised.
>
>
> ### `mult`
>
> • **Underlying instruction:** `INST_MULT`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two operands, pushes their product. Integer operands multiply with
> overflow detection and bignum promotion on overflow; NaN operands (when
> `ACCEPT_NAN` is enabled) propagate as NaN without raising an error.
>
>
> ### `neq`
>
> • **Underlying instruction:** `INST_NEQ`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two operands, pushes 1 if they are *not* equal per `expr`'s `!=`, else 0.
>
>
> ### `nop`
>
> • **Underlying instruction:** `INST_NOP`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 0, produces 0
>
> Consumes and produces nothing; the only opcode-1 instruction that isn't
> handled inside the main dispatch `switch` at all — the interpreter's
> peephole loop special-cases it (skipping a run of consecutive NOPs in one
> step) before the switch is ever reached.
>
>
> ### `not`
>
> • **Underlying instruction:** `INST_LNOT`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a value, pushes its logical negation as a boolean (1/0). The operand must be interpretable as a boolean or an error is raised.
>
>
> ### `numericType`
>
> • **Underlying instruction:** `INST_NUM_TYPE`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a value, pushes an integer code identifying its internal numeric
> representation (0 if it isn't recognized as a number at all; otherwise the
> `TCL_NUMBER_*` tag — int, double, bignum, or NaN — that `Tcl_GetNumberFromObj`
> assigned it). Never raises an error on non-numeric input; it reports type 0
> instead.
>
>
> ### `originCmd`
>
> • **Underlying instruction:** `INST_ORIGIN_COMMAND`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a command name; resolves it to a `Tcl_Command`, then follows any
> import-chain (`TclGetOriginalCommand`) to the ultimate original command and
> pushes its fully qualified name. Raises an "invalid command name" error if
> the name doesn't resolve to a command, or if the resolved name is empty
> (e.g. a deleted command).
>
>
> ### `pop`
>
> • **Underlying instruction:** `INST_POP`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 0
>
> Pops and discards the top stack value (decrementing its refcount).
>
>
> ### `pushResult`
>
> • **Underlying instruction:** `INST_PUSH_RESULT`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 0, produces 1
>
> Pushes the interpreter's current result object, then resets `iPtr->objResultPtr`
> to a fresh empty object (so the pushed value is the caller's own reference,
> not shared state that a later command could mutate out from under it).
>
>
> ### `pushReturnCode`
>
> • **Underlying instruction:** `INST_PUSH_RETURN_CODE`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 0, produces 1
>
> Pushes the numeric return code (e.g. `TCL_OK`, `TCL_ERROR`, ...) most recently produced within this frame, as an integer.
>
>
> ### `pushReturnOpts`
>
> • **Underlying instruction:** `INST_PUSH_RETURN_OPTIONS`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 0, produces 1
>
> Pushes the return-options dictionary (`Tcl_GetReturnOptions`) associated with the most recent result/return code in this frame — the same dict `catch ... opts` would capture.
>
>
> ### `resolveCmd`
>
> • **Underlying instruction:** `INST_RESOLVE_COMMAND`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a command name; resolves it against the current namespace path and
> pushes its fully qualified name, or the empty string if it does not resolve
> to any command. (Unlike `originCmd`, this does not follow import chains to
> find an ultimate origin, and never raises an error for an unresolved name.)
>
>
> ### `rshift`
>
> • **Underlying instruction:** `INST_RSHIFT`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two integer operands (value, shift-amount), pushes value shifted right
> by that many bits (arithmetic shift — sign-extending). A negative shift
> amount is a domain error.
>
>
> ### `storeArrayStk`
>
> • **Underlying instruction:** `INST_STORE_ARRAY_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 3, produces 1
>
> Pops a value, an element name and an array name; stores the value into that array element (creating it if needed) and pushes the stored value.
>
>
> ### `storeStk`
>
> • **Underlying instruction:** `INST_STORE_STK`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a value and a variable name; stores the value into that scalar (creating it if needed) and pushes the stored value.
>
>
> ### `strcaseLower`
>
> • **Underlying instruction:** `INST_STR_LOWER`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a string, pushes its lowercase form (`Tcl_UtfToLower`). Mutates the popped object in place when it is unshared, as an optimization.
>
>
> ### `strcaseTitle`
>
> • **Underlying instruction:** `INST_STR_TITLE`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a string, pushes its title-cased form (`Tcl_UtfToTitle` — first letter of each word capitalized). Mutates in place when unshared.
>
>
> ### `strcaseUpper`
>
> • **Underlying instruction:** `INST_STR_UPPER`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a string, pushes its uppercase form (`Tcl_UtfToUpper`). Mutates in place when unshared.
>
>
> ### `strcmp`
>
> • **Underlying instruction:** `INST_STR_CMP`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two strings, pushes -1, 0, or 1 per `string compare`'s three-way lexical comparison.
>
>
> ### `streq`
>
> • **Underlying instruction:** `INST_STR_EQ`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two strings, pushes 1 if they are identical, else 0 — a dedicated string-equality test distinct from `eq` (which allows numeric comparison).
>
>
> ### `strfind`
>
> • **Underlying instruction:** `INST_STR_FIND`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a needle and a haystack (needle on top; haystack below); pushes the character index of the needle's first occurrence in the haystack, or -1 if absent (`string first`).
>
>
> ### `strge`
>
> • **Underlying instruction:** `INST_STR_GE`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two strings, pushes 1 if the first is lexically `>=` the second, else 0.
>
>
> ### `strgt`
>
> • **Underlying instruction:** `INST_STR_GT`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two strings, pushes 1 if the first is lexically `>` the second, else 0.
>
>
> ### `strindex`
>
> • **Underlying instruction:** `INST_STR_INDEX`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops an index and a string; pushes the single character at that index (an
> `end`-relative index expression is accepted), or the empty string if the
> index is out of range. Byte-array-typed strings get a fast path that indexes
> raw bytes directly rather than decoding Unicode.
>
>
> ### `strle`
>
> • **Underlying instruction:** `INST_STR_LE`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two strings, pushes 1 if the first is lexically `<=` the second, else 0.
>
>
> ### `strlen`
>
> • **Underlying instruction:** `INST_STR_LEN`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a string, pushes its length in characters (`Tcl_GetCharLength`).
>
>
> ### `strlt`
>
> • **Underlying instruction:** `INST_STR_LT`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two strings, pushes 1 if the first is lexically `<` the second, else 0.
>
>
> ### `strmap`
>
> • **Underlying instruction:** `INST_STR_MAP`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 3, produces 1
>
> Pops a target string, a "from" string and a "to" string (`main`, from
> `source`/`target` naming in the source: the "main" string is on top, then
> the *replacement* string, then the *pattern* string below that); replaces
> every non-overlapping occurrence of the pattern in the main string with the
> replacement and pushes the result. This is the single-pair core used by the
> `string map` compiler when the mapping has exactly one from/to pair;
> multi-pair maps fall back to a library call instead of this instruction.
>
>
> ### `strneq`
>
> • **Underlying instruction:** `INST_STR_NEQ`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two strings, pushes 1 if they are *not* identical, else 0 — the negation of `streq`.
>
>
> ### `strrange`
>
> • **Underlying instruction:** `INST_STR_RANGE`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 3, produces 1
>
> Pops a `to` index, a `from` index, and a string (indices on top, in that
> order — `from` below `to`); pushes the substring between them inclusive
> (`end`-relative indices accepted). An empty result is produced, rather than
> an error, when the range is invalid (e.g. `to` before `from`).
>
>
> ### `strreplace`
>
> • **Underlying instruction:** `INST_STR_REPLACE`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 4, produces 1
>
> Pops a replacement string, a `to` index, a `from` index and a string (in
> that stack order, replacement on top); pushes the string with the
> `from..to` (inclusive, `end`-relative) span replaced by the replacement
> text. If the requested range is empty or inverted, the original string is
> returned unchanged and the replacement text is discarded.
>
>
> ### `strrfind`
>
> • **Underlying instruction:** `INST_STR_FIND_LAST`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a needle and a haystack; pushes the character index of the needle's *last* occurrence in the haystack, or -1 if absent (`string last`).
>
>
> ### `strtrim`
>
> • **Underlying instruction:** `INST_STR_TRIM`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a set of characters to trim and a string; pushes the string with any of those characters trimmed from both ends (`string trim`).
>
>
> ### `strtrimLeft`
>
> • **Underlying instruction:** `INST_STR_TRIM_LEFT`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a set of characters to trim and a string; pushes the string with those characters trimmed only from the left/start (`string trimleft`).
>
>
> ### `strtrimRight`
>
> • **Underlying instruction:** `INST_STR_TRIM_RIGHT`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a set of characters to trim and a string; pushes the string with those characters trimmed only from the right/end (`string trimright`).
>
>
> ### `sub`
>
> • **Underlying instruction:** `INST_SUB`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops two operands, pushes their difference (first minus second). Integer
> operands subtract with overflow detection (computed as addition of the
> bitwise complement to reuse the same overflow test as `add`) and bignum
> promotion on overflow.
>
>
> ### `swap`
>
> • **Underlying instruction:** `INST_SWAP`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 2
>
> Exchanges the top two stack values in place.
>
>
> ### `tclooClass`
>
> • **Underlying instruction:** `INST_TCLOO_CLASS`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a value expected to name/reference a TclOO object; pushes the fully qualified name of that object's class. Raises an error if the value doesn't reference a live object.
>
>
> ### `tclooId`
>
> • **Underlying instruction:** `INST_TCLOO_ID`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a value expected to reference a TclOO object; pushes that object's creation epoch (a per-interpreter unique integer identity), as used by `info object` introspection.
>
>
> ### `tclooIsObject`
>
> • **Underlying instruction:** `INST_TCLOO_IS_OBJECT`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a value; pushes 1 if it references a live TclOO object, else 0 — the
> only one of the `tcloo*` accessors that reports failure as a boolean instead
> of raising an error. Directly-following conditional jumps fold via the
> peephole optimizer.
>
>
> ### `tclooNamespace`
>
> • **Underlying instruction:** `INST_TCLOO_NS`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a value expected to reference a TclOO object; pushes the name of that object's private namespace.
>
>
> ### `tclooSelf`
>
> • **Underlying instruction:** `INST_TCLOO_SELF`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 0, produces 1
>
> Pushes the fully qualified name of the object currently executing (the TclOO
> equivalent of `[self]`), looked up from the current TclOO call-frame context
> rather than from the stack. Raises an error if executed outside a
> method/object context.
>
>
> ### `tryCvtToBoolean`
>
> • **Underlying instruction:** `INST_TRY_CVT_TO_BOOLEAN`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 2
>
> Pops a value; pushes a copy of it back plus a following boolean flag (1 if
> the value is or can be converted to a valid boolean, 0 otherwise) — it
> *replaces* the type check that would otherwise raise an error, letting the
> caller branch on validity instead. Net stack effect: consumes 1, produces 2
> (value, then flag).
>
>
> ### `tryCvtToNumeric`
>
> • **Underlying instruction:** `INST_TRY_CVT_TO_NUMERIC`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a value; if it is not already numeric, attempts to convert it in place
> (`ExecuteExtendedUnaryMathOp`-style path via `INST_UPLUS`'s shared code) and
> pushes the (possibly-converted) result. Errors on values that cannot be
> interpreted as numbers.
>
>
> ### `uminus`
>
> • **Underlying instruction:** `INST_UMINUS`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a numeric value, pushes its arithmetic negation. Integer operands
> negate directly except at the most negative representable wide integer,
> which is routed through the extended math path to promote to bignum (since
> its negation doesn't fit back in a `Tcl_WideInt`). NaN negates to NaN.
>
>
> ### `uplus`
>
> • **Underlying instruction:** `INST_UPLUS`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a value, pushes it unchanged if already numeric, or converts and
> pushes the numeric interpretation if not (unary `+` in `expr`, which is
> mainly used to force numeric-type validation/coercion). Errors on
> non-numeric input.
>
>
> ### `verifyDict`
>
> • **Underlying instruction:** `INST_DICT_VERIFY`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 0
>
> Pops nothing and pushes nothing on success: peeks the top-of-stack value and
> raises an error if it cannot be interpreted as a well-formed dictionary,
> otherwise falls through leaving the stack unchanged (consumes 1 conceptually
> per the table, but the value is left in place — the table's "consumes=1"
> reflects that this is the terminal use of that value along this compiled
> path, not a pop).
>
>
> ### `yield`
>
> • **Underlying instruction:** `INST_YIELD`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops the value to yield; if not currently executing inside a coroutine, this
> raises a "yield can only be called in a coroutine" error. Otherwise it
> suspends the running coroutine, delivering the popped value as the result of
> the `coroutine ... resume`/`[coroName]` call that will resume it, and later
> resumes with whatever value is fed back in.


---

## `ASSEM_BEGIN_CATCH`  (1 opcodes)

Marks the start of a `catch`-protected region. Takes one operand: a jump target used by the compiler/assembler to build the exception-range table (it is not read at runtime as a jump — the runtime instruction only records the stack depth).

> ### `beginCatch` *label*
>
> • **Underlying instruction:** `INST_BEGIN_CATCH`
>
> • **Compile-time operand(s):** one jump-target label, converted at assembly time into an exception-range entry
>
> • **Stack effect:** consumes 0, produces 0
>
> Pushes the current stack depth onto the interpreter's internal catch stack
> and marks the start of a protected (`catch`-covered) code region for
> exception-range bookkeeping. It does *not* itself catch anything at runtime
> — catching happens when an error unwinds into `checkForCatch`/`gotError`
> handling and finds this range registered; `beginCatch` only records where the
> stack was so it can be restored to that depth if that happens.


---

## `ASSEM_BOOL`  (3 opcodes)

One boolean operand, assembled as a literal 0/1 byte (`GetBooleanOperand`), consumed at runtime as a flags byte.

> ### `strmatch` *boolean*
>
> • **Underlying instruction:** `INST_STR_MATCH`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 2, produces 1
>
> See the ASSEM_BOOL section — the boolean operand selects case-insensitive
> matching. Pops a pattern and a string; pushes 1 if the string matches the
> glob-style pattern (`string match`), else 0. Chooses among a Unicode-aware
> matcher, a raw-byte-array matcher, or a plain UTF-8 matcher depending on the
> two operands' internal representations. Directly-following conditional
> jumps are folded via the peephole optimizer.
>
>
> ### `unsetArrayStk` *boolean*
>
> • **Underlying instruction:** `INST_UNSET_ARRAY_STK`
>
> • **Compile-time operand(s):** one boolean (0/1) operand
>
> • **Stack effect:** consumes 2, produces 0
>
> Pops an element name and an array name; unsets that array element. The
> boolean operand controls whether a missing/undefined element is silently
> ignored (0) or raises an error (1) — mirroring `unset`'s `-nocomplain`
> switch, but with the sense of the flag inverted (a *true* flag means
> "leave an error message").
>
>
> ### `unsetStk` *boolean*
>
> • **Underlying instruction:** `INST_UNSET_STK`
>
> • **Compile-time operand(s):** one boolean (0/1) operand
>
> • **Stack effect:** consumes 1, produces 0
>
> Pops a variable name and unsets that variable. Same boolean
> error-vs-silent-failure semantics as `unsetArrayStk`.


---

## `ASSEM_BOOL_LVT`  (2 opcodes)

One boolean operand plus one variable reference resolved against the assembly's Local Variable Table (LVT) at compile time.

> ### `unset` *boolean* *varName*
>
> • **Underlying instruction:** `INST_UNSET_SCALAR`
>
> • **Compile-time operand(s):** one boolean operand plus an LVT index
>
> • **Stack effect:** consumes 0, produces 0
>
> Unsets the local scalar variable named by the LVT index. The boolean
> controls whether an already-undefined variable is an error (true) or
> silently ignored (false). Uses a fast direct-unset path when the variable
> has no traces and isn't stored in a hash table; otherwise falls back to the
> general `TclPtrUnsetVarIdx` path.
>
>
> ### `unsetArray` *boolean* *varName*
>
> • **Underlying instruction:** `INST_UNSET_ARRAY`
>
> • **Compile-time operand(s):** one boolean operand plus an LVT index
>
> • **Stack effect:** consumes 1, produces 0
>
> Pops an element name; unsets `arrayVar(elementName)` where `arrayVar` is the
> LVT-indexed local array. Same boolean error/silent semantics as `unset`. A
> fast path handles the common case of an untraced element with no active
> search on the array; otherwise falls back to the general variable-lookup and
> unset machinery.


---

## `ASSEM_CLOCK_READ`  (1 opcodes)

One 1-byte unsigned case selector (0-3) choosing which clock to sample.

> ### `clockRead` *case*
>
> • **Underlying instruction:** `INST_CLOCK_READ`
>
> • **Compile-time operand(s):** one 1-byte case selector: 0=clicks, 1=microseconds, 2=milliseconds, 3=seconds, 4=monotonic
>
> • **Stack effect:** consumes 0, produces 1
>
> Pushes a fresh timestamp read directly from the platform clock, selected by
> the case number: high-resolution "clicks" (`TclpGetWideClicks`, a
> platform-defined fast counter, ideally but not necessarily monotonic),
> wall-clock microseconds/milliseconds/seconds since the epoch (`Tcl_GetDayTime`),
> or a monotonic clock (`Tcl_GetMonotonicTime`). Backs the compiled fast path
> for `clock microseconds`/`clock milliseconds`/`clock seconds` and internal
> timing. An unrecognized case number is a `Tcl_Panic` (a compiler/assembler
> bug, not a user-reachable error).


---

## `ASSEM_CONCAT1`  (2 opcodes)

One 1-byte unsigned operand count N (must be > 0); consumes N operands, produces 1.

> ### `concat` *n*
>
> • **Underlying instruction:** `INST_STR_CONCAT1`
>
> • **Compile-time operand(s):** one 1-byte operand count N (> 0)
>
> • **Stack effect:** consumes N (variable, from the operand), produces 1
>
> Pops N string values and pushes their direct concatenation with no separator
> (`TclStringCat`) — this is the compiled fast path behind adjacent string
> literals/substitutions in a word, *not* the `concat` command (which
> list-joins with spaces and is compiled to `INST_CONCAT_STK`/`concatStk`
> instead). The assembler exposes the identical instruction under two
> mnemonics, `concat` and `strcat`; they are indistinguishable at runtime.
>
>
> ### `strcat` *n*
>
> • **Underlying instruction:** `INST_STR_CONCAT1`
>
> • **Compile-time operand(s):** one 1-byte operand count N (> 0)
>
> • **Stack effect:** consumes N (variable, from the operand), produces 1
>
> Identical to `concat` above — both mnemonics assemble to `INST_STR_CONCAT1`.


---

## `ASSEM_DICT_GET`  (2 opcodes)

One operand: key count N (must be > 0); consumes N+1 operands (the dict plus N keys), produces 1.

> ### `dictExists` *n*
>
> • **Underlying instruction:** `INST_DICT_EXISTS`
>
> • **Compile-time operand(s):** one operand: key count N (> 0)
>
> • **Stack effect:** consumes N (variable, from the operand), produces 1
>
> Same key-path walk as `dictGet`, but reports success/failure as a boolean
> (1/0) instead of raising an error for a missing key or non-dict value —
> the compiled form of `dict exists`. Directly-following conditional jumps are
> folded via the interpreter's peephole optimizer.
>
>
> ### `dictGet` *n*
>
> • **Underlying instruction:** `INST_DICT_GET`
>
> • **Compile-time operand(s):** one operand: key count N (> 0)
>
> • **Stack effect:** consumes N (variable, from the operand), produces 1
>
> Pops N keys and a dict (dict at the bottom of that span); walks the key path
> through nested dicts (`TclTraceDictPath`) and pushes the value at the final
> key. Raises an error — with a `TCL LOOKUP DICT` error code — if any key
> along the path is missing or if the initial value isn't a dict.


---

## `ASSEM_DICT_SET`  (1 opcodes)

Two operands: key count N (> 0) and an LVT index; consumes N+1 operands (N keys plus the value), produces 1 (the updated dict, or nothing if the assembler folds in a following `pop`).

> ### `dictSet` *n* *varName*
>
> • **Underlying instruction:** `INST_DICT_SET`
>
> • **Compile-time operand(s):** key count N (> 0) and an LVT index
>
> • **Stack effect:** consumes N (variable, from the operand), produces 1
>
> Reads the dict currently stored in the LVT-indexed variable (duplicating it
> first — copy-on-write — if the stored object is shared elsewhere), pops N
> keys and a value from the stack, sets that key path to the value
> (`Tcl_DictObjPutKeyList`), writes the updated dict back into the variable,
> and pushes the new dict value. If the variable didn't yet hold a dict, a
> fresh empty one is created first. An update failure (e.g. the variable holds
> something that isn't a dict, or a non-terminal key path segment isn't a
> dict) discards the working copy and raises an error, leaving the original
> variable untouched.


---

## `ASSEM_DICT_UNSET`  (1 opcodes)

Two operands: key count N (> 0) and an LVT index; consumes N key operands, produces 1.

> ### `dictUnset` *n* *varName*
>
> • **Underlying instruction:** `INST_DICT_UNSET`
>
> • **Compile-time operand(s):** key count N (> 0) and an LVT index
>
> • **Stack effect:** consumes N (variable, from the operand), produces 1
>
> Same variable-lookup, copy-on-write and write-back pattern as `dictSet`, but
> removes the given key path (`Tcl_DictObjRemoveKeyList`) instead of setting
> it, and pushes the resulting dict.


---

## `ASSEM_END_CATCH`  (1 opcodes)

No operands. Pops the exception-range/catch stack and resets the interpreter result.

> ### `endCatch`
>
> • **Underlying instruction:** `INST_END_CATCH`
>
> • **Compile-time operand(s):** none
>
> • **Stack effect:** consumes 0, produces 0
>
> Pops the innermost entry off the catch stack and resets the interpreter's
> result to empty (`Tcl_ResetResult`), returning `TCL_OK` as the "current"
> result code for whatever follows. Marks the end of the code protected by the
> matching `beginCatch`.


---

## `ASSEM_EVAL`  (2 opcodes)

One operand — a script or expression, either a compile-time literal (compiled directly inline by the assembler) or a run-time value (pushed then evaluated by the underlying instruction).

> ### `eval` *script*
>
> • **Underlying instruction:** `INST_EVAL_STK`
>
> • **Compile-time operand(s):** one operand: a script, either a compile-time literal or a substituted value
>
> • **Stack effect:** consumes 1, produces 1
>
> If the operand is a literal (no embedded substitutions), the assembler
> compiles it *inline*, at assembly time, via the same machinery used for a
> literal script body elsewhere in Tcl (`CompileEmbeddedScript`) — the
> generated bytecode is spliced directly into the surrounding code, with no
> `INST_EVAL_STK` instruction involved at all. If the operand instead contains
> substitutions, the assembler falls back to pushing it as a literal (via the
> table's PUSH slot) followed by a plain `INST_EVAL_STK`, making it behave
> identically to `evalStk` at that point. Consumes 1 conceptual operand
> (the script), produces 1 (its result) either way.
>
>
> ### `expr` *expression*
>
> • **Underlying instruction:** `INST_EXPR_STK`
>
> • **Compile-time operand(s):** one operand: an expression, either a compile-time literal or a substituted value
>
> • **Stack effect:** consumes 1, produces 1
>
> Same inline-vs-runtime split as `eval`, but for expressions: a literal
> operand is compiled inline by the assembler using the expression compiler
> (no `INST_EXPR_STK` emitted); a non-literal operand is pushed and evaluated
> at runtime with `INST_EXPR_STK`, identically to `exprStk`.


---

## `ASSEM_INDEX`  (1 opcodes)

One 4-byte operand: a list index, which may be a plain integer or an `end`-relative index expression, pre-encoded at assembly time.

> ### `listIndexImm` *index*
>
> • **Underlying instruction:** `INST_LIST_INDEX_IMM`
>
> • **Compile-time operand(s):** one 4-byte pre-encoded list index (integer or `end`-relative)
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a list; pushes the element at the given index (decoded relative to the
> list's actual length, so `end`-relative indices are resolved here), or the
> empty string if the index is out of range. Lists backed by an abstract/lazy
> list type dispatch through that type's own index callback instead of
> materializing element pointers. This is the compiled fast path for
> `lindex $list <constant-index>`.


---

## `ASSEM_INVOKE`  (1 opcodes)

One operand: an argument count (must be > 0); consumes that many operands, produces 1.

> ### `invokeStk` *n*
>
> • **Underlying instruction:** `INST_INVOKE_STK`
>
> • **Compile-time operand(s):** one operand: argument count N (> 0)
>
> • **Stack effect:** consumes N (variable, from the operand), produces 1
>
> Pops N values from the stack, treats them as `objv[0..N-1]` of an ordinary
> Tcl command invocation (`objv[0]` is the command name), and invokes it,
> pushing its result. This performs a full command dispatch (name resolution,
> argument passing, going through the interpreter's normal command-invocation
> machinery), exactly as if that many words had been given to `eval`, except
> the words are supplied pre-substituted from the stack rather than by parsing
> a script.


---

## `ASSEM_JUMP`  (6 opcodes)

One operand: a label naming the jump target; the assembler resolves it to a 4-byte relative pc offset once basic-block layout is finalized.

> ### `jump` *label*
>
> • **Underlying instruction:** `INST_JUMP`
>
> • **Compile-time operand(s):** one label operand, resolved to a 4-byte relative pc offset
>
> • **Stack effect:** consumes 0, produces 0
>
> Unconditional jump to the target label. Consumes and produces nothing.
>
>
> ### `jump4` *label*
>
> • **Underlying instruction:** `INST_JUMP`
>
> • **Compile-time operand(s):** one label operand
>
> • **Stack effect:** consumes 0, produces 0
>
> Legacy alias for `jump`, kept for old assembly source; identical at runtime.
>
>
> ### `jumpFalse` *label*
>
> • **Underlying instruction:** `INST_JUMP_FALSE`
>
> • **Compile-time operand(s):** one label operand
>
> • **Stack effect:** consumes 1, produces 0
>
> Pops a boolean value; jumps to the target label if it is false (0),
> otherwise falls through to the next instruction. The popped value must be
> interpretable as a boolean or an error is raised.
>
>
> ### `jumpFalse4` *label*
>
> • **Underlying instruction:** `INST_JUMP_FALSE`
>
> • **Compile-time operand(s):** one label operand
>
> • **Stack effect:** consumes 1, produces 0
>
> Legacy alias for `jumpFalse`; identical at runtime.
>
>
> ### `jumpTrue` *label*
>
> • **Underlying instruction:** `INST_JUMP_TRUE`
>
> • **Compile-time operand(s):** one label operand
>
> • **Stack effect:** consumes 1, produces 0
>
> Pops a boolean value; jumps to the target label if it is true (nonzero), otherwise falls through. Same boolean-conversion requirement as `jumpFalse`.
>
>
> ### `jumpTrue4` *label*
>
> • **Underlying instruction:** `INST_JUMP_TRUE`
>
> • **Compile-time operand(s):** one label operand
>
> • **Stack effect:** consumes 1, produces 0
>
> Legacy alias for `jumpTrue`; identical at runtime.


---

## `ASSEM_JUMPTABLE`  (2 opcodes)

One operand: a flat `{key targetLabel ...}` list, turned into a `JumptableInfo`/`JumptableNumInfo` auxiliary-data record referenced by a 4-byte index.

> ### `jumpTable` *table*
>
> • **Underlying instruction:** `INST_JUMP_TABLE`
>
> • **Compile-time operand(s):** one operand: a flat key/label list turned into a hash-table auxiliary-data record
>
> • **Stack effect:** consumes 1, produces 0
>
> Pops a string value; looks it up as an exact-match key in the table built at
> assembly time and jumps to the associated label if found. If the key isn't
> present, execution falls through to the next instruction instead of
> jumping. Backs the compiled fast path for `switch -exact` with all-literal
> patterns.
>
>
> ### `jumpTableNum` *table*
>
> • **Underlying instruction:** `INST_JUMP_TABLE_NUM`
>
> • **Compile-time operand(s):** one operand: a flat key/label list, keys interpreted as integers
>
> • **Stack effect:** consumes 1, produces 0
>
> Same as `jumpTable`, but the popped value is converted to a wide integer
> first and looked up by integer key rather than by string; a value that
> cannot be parsed as an integer raises an error rather than simply falling
> through.


---

## `ASSEM_LABEL`  (1 opcodes)

Not a runtime instruction at all — an assembler directive. One operand names a label at the current code address; it emits no bytecode.

> ### `label` *name*
>
> • **Underlying instruction:** `*(no runtime opcode — assembler directive only)*`
>
> • **Compile-time operand(s):** one operand: the label's name
>
> • **Stack effect:** consumes 0, produces 0
>
> Not a runtime instruction — a pure assembler directive that records the
> current bytecode address under the given name in the assembler's label
> table, so that later `jump`/`jumpTrue`/`jumpFalse`/`jumpTable`/`beginCatch`
> operands naming this label can be resolved to a concrete pc offset once
> basic-block layout is finalized. Emits no bytecode and has no runtime stack
> effect.


---

## `ASSEM_LINDEX_MULTI`  (1 opcodes)

One operand: an index-argument count N (must be > 0); consumes N+1 operands (list plus N indices), produces 1.

> ### `lindexMulti` *n*
>
> • **Underlying instruction:** `INST_LIST_INDEX_MULTI`
>
> • **Compile-time operand(s):** one operand: index-argument count N (> 0)
>
> • **Stack effect:** consumes N (variable, from the operand), produces 1
>
> Pops N index values and a list (list at the bottom); walks the indices
> through successively nested lists (`TclLindexFlat` — this is the flattened,
> multi-index form of `lindex`, e.g. `lindex $l $i $j $k`) and pushes the
> final addressed element, or the empty string if any index along the way is
> out of range.


---

## `ASSEM_LIST`  (2 opcodes)

One operand: an element/operand count N (must be >= 0); consumes N operands, produces 1.

> ### `concatStk` *n*
>
> • **Underlying instruction:** `INST_CONCAT_STK`
>
> • **Compile-time operand(s):** one operand: element count N (>= 0)
>
> • **Stack effect:** consumes N (variable, from the operand), produces 1
>
> Pops N values and pushes their `concat`-style joining (`Tcl_ConcatObj`):
> each value is treated as a list, elements are extracted and re-joined with
> single spaces, and leading/trailing whitespace within each value is trimmed
> at the join points. This is the compiled form of the `concat` *command*
> (compare `concat`/`strcat`, ASSEM_CONCAT1, which do plain string
> concatenation with no such list-aware joining).
>
>
> ### `list` *n*
>
> • **Underlying instruction:** `INST_LIST`
>
> • **Compile-time operand(s):** one operand: element count N (>= 0)
>
> • **Stack effect:** consumes N (variable, from the operand), produces 1
>
> Pops N values and pushes them combined into a single new list object, in order (`list` command's compiled form).


---

## `ASSEM_LSET_FLAT`  (1 opcodes)

One operand: total operand count N (must be >= 3); consumes N operands, produces 1.

> ### `lsetFlat` *n*
>
> • **Underlying instruction:** `INST_LSET_FLAT`
>
> • **Compile-time operand(s):** one operand: total operand count N (>= 3)
>
> • **Stack effect:** consumes N (variable, from the operand), produces 1
>
> Pops a replacement value and N-2 flattened index arguments and an original
> list value (replacement on top, then indices, list at the bottom); returns a
> new list with the addressed element replaced (`TclLsetFlat`, or the
> abstract-list type's own set-element hook when the list value has one). This
> is the flattened multi-index `lset` form, e.g. `lset l i j k value`; compare
> `lsetList`, which takes a single pre-built index list instead of separate
> flattened index arguments.


---

## `ASSEM_LVT_N`  (8 opcodes)

One 4-byte LVT index operand. (Functionally like ASSEM_LVT; the assembler notes it separately because it doesn't advance the compiler's notion of the current source line for debug info.)

> ### `append` *varName*
>
> • **Underlying instruction:** `INST_APPEND_SCALAR`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a value; appends it as a plain string (`TCL_APPEND_VALUE`, no list
> quoting) to the LVT-indexed local scalar, creating the variable if
> undefined, and pushes the new value.
>
>
> ### `appendArray` *varName*
>
> • **Underlying instruction:** `INST_APPEND_ARRAY`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops an element name and a value (value on top); appends the value as a
> plain string to `arrayVar(elementName)`, where `arrayVar` is the
> LVT-indexed local array, creating the element if needed, and pushes the new
> value.
>
>
> ### `lappend` *varName*
>
> • **Underlying instruction:** `INST_LAPPEND_SCALAR`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a value; list-appends it (proper list-element quoting) to the LVT-indexed local scalar and pushes the new value.
>
>
> ### `lappendArray` *varName*
>
> • **Underlying instruction:** `INST_LAPPEND_ARRAY`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops an element name and a value; list-appends the value to `arrayVar(elementName)` (LVT-indexed array) and pushes the new value.
>
>
> ### `load` *varName*
>
> • **Underlying instruction:** `INST_LOAD_SCALAR`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 0, produces 1
>
> Pushes the current value of the LVT-indexed local scalar. A direct fast path
> handles the common case (variable has a value and no traces); traced or
> otherwise irregular variables fall through to the general variable-read
> machinery. Raises an error if the variable is undefined.
>
>
> ### `loadArray` *varName*
>
> • **Underlying instruction:** `INST_LOAD_ARRAY`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops an element name; pushes the value of `arrayVar(elementName)`, where
> `arrayVar` is the LVT-indexed local array. Raises an error if the element is
> undefined.
>
>
> ### `store` *varName*
>
> • **Underlying instruction:** `INST_STORE_SCALAR`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 1, produces 1
>
> Stores the top-of-stack value into the LVT-indexed local scalar (creating it
> if needed) and leaves the value on the stack (the table's "consumes=1,
> produces=1" reflects that the same object stays on top — it is not
> popped-then-repushed, so there is no extra refcount churn). A direct fast
> path is used when the variable is writable with no traces.
>
>
> ### `storeArray` *varName*
>
> • **Underlying instruction:** `INST_STORE_ARRAY`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops an element name (the value being stored stays on the stack below it,
> then becomes the new top); stores that value into `arrayVar(elementName)`
> (LVT-indexed array), creating the element if needed, and leaves the stored
> value on top of the stack.


---

## `ASSEM_LVT_SINT1`  (2 opcodes)

One 4-byte LVT index plus one 1-byte signed integer operand.

> ### `incrArrayImm` *varName* *n*
>
> • **Underlying instruction:** `INST_INCR_ARRAY_IMM`
>
> • **Compile-time operand(s):** one 4-byte LVT index and one 1-byte signed increment
>
> • **Stack effect:** consumes 1, produces 1
>
> Adds the signed byte increment to `arrayVar(elementName)`, where `arrayVar`
> is the LVT-indexed local array and the element name is popped from the
> stack; pushes the new value. Non-integer current values route through the
> general `TclIncrObj` bignum-aware increment path.
>
>
> ### `incrImm` *varName* *n*
>
> • **Underlying instruction:** `INST_INCR_SCALAR_IMM`
>
> • **Compile-time operand(s):** one 4-byte LVT index and one 1-byte signed increment
>
> • **Stack effect:** consumes 0, produces 1
>
> Adds the signed byte increment directly to the LVT-indexed local scalar and
> pushes the new value. A fast path adds two wide integers with overflow
> detection and shares the result object in place when it isn't shared
> elsewhere and the addition doesn't overflow; overflow, non-integer values,
> or a shared object fall back to allocating a new result via the general
> increment path.


---

## `ASSEM_LVT`  (14 opcodes)

One 4-byte operand referencing a slot in the procedure's Local Variable Table.

> ### `arrayExistsImm` *varName*
>
> • **Underlying instruction:** `INST_ARRAY_EXISTS_IMM`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 0, produces 1
>
> Pushes 1 if the LVT-indexed local variable currently exists and is an array,
> else 0, after first giving any array-existence traces on it a chance to run
> (and to raise an error, which aborts execution instead of producing a
> boolean).
>
>
> ### `arrayMakeImm` *varName*
>
> • **Underlying instruction:** `INST_ARRAY_MAKE_IMM`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 0, produces 0
>
> Turns the LVT-indexed local variable into an empty array if it is currently
> undefined; raises an error if it already holds a scalar value or is itself
> an array element; is a no-op if it is already an array. Produces nothing.
>
>
> ### `dictAppend` *varName*
>
> • **Underlying instruction:** `INST_DICT_APPEND`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a value; appends it as a plain string to the value stored at a
> (separately, from the stack) supplied key within the dict held in the
> LVT-indexed variable, copy-on-write duplicating the dict first if shared,
> and writes the updated dict back to the variable. (Shares its case-block
> with `dictLappend`; see there for the exact key/value stack layout.)
>
>
> ### `dictLappend` *varName*
>
> • **Underlying instruction:** `INST_DICT_LAPPEND`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a key and a value (value on top); list-appends the value to the list
> currently stored at that key within the dict held in the LVT-indexed
> variable (creating the key with a one-element list if absent, or
> copy-on-write duplicating the existing value if it's shared), writes the
> updated dict back into the variable, and pushes the updated dict. This is
> the compiled `dict lappend` fast path.
>
>
> ### `dictRecombineImm` *varName*
>
> • **Underlying instruction:** `INST_DICT_RECOMBINE_IMM`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 2, produces 0
>
> Pops a key list and a value list (key list on top); performs the `dict
> with`-exit recombination, writing the (possibly modified) values back into
> the corresponding keys of the dict stored in the LVT-indexed variable.
> Produces nothing.
>
>
> ### `exist` *varName*
>
> • **Underlying instruction:** `INST_EXIST_SCALAR`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 0, produces 1
>
> Pushes 1 if the LVT-indexed local scalar currently exists and is defined,
> else 0, after allowing any read trace on it to fire first. Directly
> following conditional jumps are folded via the peephole optimizer.
>
>
> ### `existArray` *varName*
>
> • **Underlying instruction:** `INST_EXIST_ARRAY`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops an element name; pushes 1 if `arrayVar(elementName)` (LVT-indexed
> array) exists and is defined, else 0, after allowing read traces to fire.
> Same peephole-jump folding as `exist`.
>
>
> ### `incr` *varName*
>
> • **Underlying instruction:** `INST_INCR_SCALAR`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops an increment amount; adds it to the LVT-indexed local scalar
> (auto-vivifying it as 0 if undefined) and pushes the new value. Non-integer
> values/increments go through the general bignum-aware increment path.
>
>
> ### `incrArray` *varName*
>
> • **Underlying instruction:** `INST_INCR_ARRAY`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops an increment amount and an element name; adds the increment to
> `arrayVar(elementName)` (LVT-indexed array), creating the element if needed,
> and pushes the new value.
>
>
> ### `lappendList` *varName*
>
> • **Underlying instruction:** `INST_LAPPEND_LIST`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a list; appends every element of it to the LVT-indexed local scalar's
> own list value (multi-element `lappend` in one step) and pushes the result.
> A direct fast path is used when the variable is writable with no traces.
>
>
> ### `lappendListArray` *varName*
>
> • **Underlying instruction:** `INST_LAPPEND_LIST_ARRAY`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a list and an element name; appends every element of the popped list to
> `arrayVar(elementName)`'s list value (LVT-indexed array) and pushes the
> result.
>
>
> ### `nsupvar` *varName*
>
> • **Underlying instruction:** `INST_NSUPVAR`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a variable name and a namespace name (name on top, namespace below);
> links the LVT-indexed local variable to that variable resolved inside the
> named namespace (`upvar`'s `#namespace` form / the `nsupvar` command's
> mechanism), and pushes the *namespace* operand back unchanged (it is not
> consumed — only the two name operands used to locate the target variable are
> popped, per the "consumes 2" in the table meaning the namespace and variable
> name operands together select the target, with the local slot itself
> supplied by the fixed LVT operand). Raises an error if the namespace or
> target variable can't be resolved.
>
>
> ### `upvar` *varName*
>
> • **Underlying instruction:** `INST_UPVAR`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a variable name and a call-frame specifier (level number or `#0`
> absolute-global marker); links the LVT-indexed local variable to that
> variable in the specified caller's frame, and produces 1 (leaves a value on
> the stack, matching `nsupvar`'s convention of not fully consuming its
> frame-selector operand). Raises an error if the frame or target variable
> can't be resolved.
>
>
> ### `variable` *varName*
>
> • **Underlying instruction:** `INST_VARIABLE`
>
> • **Compile-time operand(s):** one 4-byte LVT index
>
> • **Stack effect:** consumes 1, produces 0
>
> Pops a variable name; resolves/creates that variable in the current
> namespace and links the LVT-indexed local slot to it, marking it as a
> namespace variable (`variable` command's per-name compiled step). Raises an
> error if the namespace-scoped variable can't be resolved/created.


---

## `ASSEM_OVER`  (1 opcodes)

One operand: a depth count N; consumes N+1 operands and produces N+2 (it duplicates one, leaving the original N+1 in place below the copy).

> ### `over` *n*
>
> • **Underlying instruction:** `INST_OVER`
>
> • **Compile-time operand(s):** one operand: depth N
>
> • **Stack effect:** consumes N (variable, from the operand); table's raw operandsProduced field is -1-1 (negative encodes "net stack effect = -1 - operandsProduced" = +1) — see description for the actual push/pop counts
>
> Duplicates the stack value N slots below the current top (N=0 duplicates the
> top itself, same as `dup`) and pushes that copy on top, leaving the original
> N+1 values undisturbed underneath. Net effect: consumes N+1, produces N+2 —
> a net gain of one stack slot, which is how `TalInstructionTable` encodes it
> (the table's `operandsProduced` field is negative for `over`/`reverse`;
> by convention there it means "net stack effect = -1 - operandsProduced",
> i.e. +1 here and 0 for `reverse`, rather than a literal produced-item count).


---

## `ASSEM_PUSH`  (1 opcodes)

One operand: a literal value, registered in the code's literal pool and referenced by a 4-byte index. Must be table slot 0 (the assembler relies on this to synthesize PUSH internally, e.g. for literal `eval`/`expr` operands).

> ### `push` *value*
>
> • **Underlying instruction:** `INST_PUSH`
>
> • **Compile-time operand(s):** one operand: a literal value, registered in the code's literal pool
>
> • **Stack effect:** consumes 0, produces 1
>
> Pushes the literal object at the given literal-pool index. This is the only
> instruction the assembler is allowed to synthesize on its own behalf (e.g.
> to push a literal script/expression operand for `eval`/`expr`), which is why
> the table requires `push` to occupy slot 0 of `TalInstructionTable`.


---

## `ASSEM_REGEXP`  (1 opcodes)

One boolean (`-nocase`) operand which the assembler pre-combines with `TCL_REG_ADVANCED` into a single 1-byte regexp-compile-flags operand — not stored as a plain 0/1 the way ASSEM_BOOL is.

> ### `regexp` *boolean*
>
> • **Underlying instruction:** `INST_REGEXP`
>
> • **Compile-time operand(s):** one boolean operand, pre-combined at assembly time into a 1-byte `TCL_REG_ADVANCED | (nocase ? TCL_REG_NOCASE : 0)` flags byte
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops a pattern and a string (string on top, pattern below); compiles the
> pattern as an ARE (advanced regular expression, always with `TCL_REG_ADVANCED`
> set, plus `TCL_REG_NOCASE` if the boolean operand was true) and pushes 1 if
> the string matches anywhere within it, else 0. Note the operand is *not* a
> raw boolean at runtime — unlike `strmatch`'s plain 0/1 nocase byte, the
> assembler folds it together with `TCL_REG_ADVANCED` into one combined flags
> byte before emitting it, so the mnemonic's single boolean source argument
> does not correspond 1:1 with the runtime operand's bit pattern.
> Directly-following conditional jumps are folded via the peephole optimizer.
> Compilation failure or an internal match error raises a Tcl error.


---

## `ASSEM_REVERSE`  (1 opcodes)

One operand: a count N (must be >= 0); consumes and produces the same N operands, reversing their order in place.

> ### `reverse` *n*
>
> • **Underlying instruction:** `INST_REVERSE`
>
> • **Compile-time operand(s):** one operand: count N (>= 0)
>
> • **Stack effect:** consumes N (variable, from the operand); table's raw operandsProduced field is -1-0 (negative encodes "net stack effect = -1 - operandsProduced" = +0) — see description for the actual push/pop counts
>
> Reverses the order of the top N stack values in place (no objects are
> popped or newly pushed — refcounts are untouched, only their stack slots are
> permuted). Used by the compiler/assembler wherever a small run of values
> needs reordering without going through a full pop/push cycle.


---

## `ASSEM_SINT1`  (2 opcodes)

One 1-byte signed integer operand.

> ### `incrArrayStkImm` *n*
>
> • **Underlying instruction:** `INST_INCR_ARRAY_STK_IMM`
>
> • **Compile-time operand(s):** one 1-byte signed increment
>
> • **Stack effect:** consumes 2, produces 1
>
> Pops an element name and an array name; adds the signed byte increment to
> `arrayName(elementName)` (names taken from the stack, not the LVT — compare
> `incrArrayImm`) and pushes the new value.
>
>
> ### `incrStkImm` *n*
>
> • **Underlying instruction:** `INST_INCR_STK_IMM`
>
> • **Compile-time operand(s):** one 1-byte signed increment
>
> • **Stack effect:** consumes 1, produces 1
>
> Pops a variable name; adds the signed byte increment to that variable (name
> from the stack, compare `incrImm`) and pushes the new value.


---

## `ASSEM_SINT4_LVT`  (1 opcodes)

One 4-byte signed integer operand plus one 4-byte LVT index.

> ### `dictIncrImm` *n* *varName*
>
> • **Underlying instruction:** `INST_DICT_INCR_IMM`
>
> • **Compile-time operand(s):** one 4-byte signed increment and one 4-byte LVT index
>
> • **Stack effect:** consumes 1, produces 1
>
> Reads the dict in the LVT-indexed variable (copy-on-write duplicating if
> shared, or creating a fresh empty dict if the variable was empty), pops a
> key, and adds the signed 4-byte increment to the value stored at that key
> within the dict (treating a missing key as 0), copy-on-write duplicating
> that inner value too if it's shared; writes the updated dict back to the
> variable and pushes it. Shares its case block with `dictSet`/`dictUnset` in
> `tclExecute.c` — all three read-duplicate-write the same LVT-indexed dict.


---

## `ASSEM_DICT_GET_DEF`  (1 opcodes)

One operand: key count N (must be > 0); consumes N+2 operands (the dict, N keys, and a default value), produces 1.

> ### `dictGetDef` *n*
>
> • **Underlying instruction:** `INST_DICT_GET_DEF`
>
> • **Compile-time operand(s):** one operand: key count N (> 0)
>
> • **Stack effect:** consumes N (variable, from the operand), produces 1
>
> Pops a default value, N keys and a dict (default on top); walks the key
> path as `dictGet` does, but instead of raising an error for a missing key it
> pushes the supplied default value (`dict getwithdefault`/`dict get ... default`
> semantics). A broken (non-dict) intermediate value along the path is still
> an error.
