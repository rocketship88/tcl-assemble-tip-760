# Tcl's Assembler and the Dual-Path Compile System

*A working document on `tcl::unsupported::assemble` and how Tcl's bytecode
compiler builds a `CompileEnv` two different ways for the same underlying
machinery.*

---

## Chapter 0: Outline

This chapter is the table of contents — expanded with enough detail that
someone skimming can find what they need without reading start to finish.
Each numbered item below is its own full chapter later in the document.

### Chapter 1. Introduction / motivation

- **Why this exists:** writing a fast Tcl-level DSL depends directly on
  understanding `assemble` and the dual-path compile system — not incidental
  detail, but the mechanism the DSL's correctness and performance actually
  rest on.
- **The gap:** that mechanism isn't documented anywhere coherent. What
  exists is scattered across old wiki pages, a decades-old paper, and the
  understanding of a small number of core maintainers.
- **Scope:** mechanics of source-to-bytecode via these two paths, not a
  tutorial on writing TAL itself.
- **Audience:** comfortable reading Tcl core C; wants to either use
  `assemble`/write a compile-proc, or understand what they're looking at in
  a debugger.

### Chapter 2. Foundations: CompileEnv and the emit macros

- **What a `CompileEnv` owns:** the growing bytecode buffer, the literal
  table, the exception-range list, and running stack-depth accounting —
  without yet saying how one comes to exist (that's chapter 3; this chapter
  is deliberately indifferent to origin).
- **The raw emit primitives:** `TclEmitOpcode`, `TclEmitInstInt1/4/14/44/41`,
  `TclEmitPush`, `TclEmitInvoke` — always take `envPtr` explicitly, nothing
  hidden.
- **The shorthand layer:** `OP`, `OP1`, `OP4`, `PUSH`, `PUSH_OBJ`,
  `BACKJUMP`/`FWDJUMP`, catch-range helpers, `STKDELTA`, and others, defined
  in `tclCompile.h`. These hide `envPtr` by assuming a variable of that
  exact name is already in scope, and paste `INST_` onto the opcode name at
  compile time. Used pervasively across the many per-command compile-procs.
- **The sharp edge:** shorthand macros only work when the opcode is a
  compile-time literal, since `OP(name)` pastes `INST_` onto `name` at
  preprocessor time; a runtime-computed opcode forces a drop back to the
  raw layer with `envPtr` spelled out by hand.
- **Debugger note:** why `envPtr` seems to vanish when stepping through
  these macros — no separate call frame for a preprocessor substitution.

### Chapter 3. Two ways to get a CompileEnv: the dual-path system

- **Path A — direct/one-shot eval:** a fresh, throwaway `CompileEnv`,
  built and torn down around a single assembly, producing a standalone
  `ByteCode`.
- **Path B — proc-body/incremental:** reuses the caller's ongoing
  `CompileEnv`, appending into it, with save/restore of command count, code
  offset, stack depth, and exception ranges so a failed compile rolls back
  cleanly. Also called **inlining** elsewhere (e.g. Nadkarni's *The Tcl
  Programming Language*) — both names refer to the same mechanism, and
  this document uses both.
- **Central diagram:** same core assembly logic, two different `CompileEnv`
  lifetimes wrapped around it.

### Chapter 4. How an ordinary but non-trivial command gets compiled: `for`

- **The control group:** a normal per-command compile-proc, registered
  against a specific command, with `envPtr` as an ordinary parameter and
  opcodes known and named at C-compile time.
- **Why `for`, specifically:** most compile-procs (`append`, `incr`, ...)
  only ever emit straight-line code — no branching at all. `for` is
  deliberately chosen as one of the few that isn't trivial: it needs the
  jump macros (`FWDJUMP`/`BACKLABEL`/`BACKJUMP`/`FWDLABEL`) and the
  exception-range macros (`MAKE_LOOP_RANGE`/`CATCH_RANGE`/
  `CONTINUE_TARGET`/`BREAK_TARGET`/`FINALIZE_LOOP`) that most compile-procs
  never touch.
- **Why that matters for what's coming:** those same concerns — arbitrary
  jumps, catch/exception tracking — are exactly what `assemble` (chapter 5)
  has to support in general, for any TAL program, not just one hand-written
  control structure. `for`'s use of them is the closest an ordinary
  compile-proc gets to what `assemble` has to handle generally.
- **Worked example: `TclCompileForCmd`.** Introduced here at the
  structural level — bail-out conditions, overall shape of the function —
  with the line-by-line walk deferred to chapter 6 so this chapter stays
  focused on establishing the comparison point.

### Chapter 5. How `assemble` actually works internally

- **`TalInstructionTable`:** one row per TAL mnemonic (roughly 150), each
  holding its operand-encoding shape (`TalInstType`), its real opcode, and
  operand counts.
- **The actual dispatch:** `AssembleOneLine` switches on the shape
  (`TalInstType`, a much smaller category count), not on the mnemonic
  directly.
- **The three chokepoints:** `BBEmitOpcode`, `BBEmitInstInt1`,
  `BBEmitInstInt4` — the only places the raw emit layer actually gets
  called from assemble's own code.
- **Extra bookkeeping:** `BasicBlock` and `AssemblyEnv`, and why assemble
  needs them where an ordinary compile-proc doesn't.
- **No narrow-operand instructions to choose between anymore:** the old
  1-byte/4-byte jump split (and the backpatching it required) is gone;
  legacy TAL mnemonics like `jump4` map to the same real opcode as their
  bare counterparts, kept only for old TAL source compatibility.
- **The peephole optimizer (`tclOptimize.c`):** the four-pass pipeline
  (`ConvertZeroEffectToNOP`, `BetterEqualityTesting`, `AdvanceJumps`,
  `TrimUnreachable`) that runs over ordinary compiled code — why dead
  push/pop pairs become NOPs rather than being deleted (offset stability
  for jumps, exception ranges, and the command map), and why jumps get
  retargeted around those NOPs so they cost nothing at runtime. Worked
  evidence: two `for`-loop disassembly traces (empty body vs. `puts $i`
  body), plus an 8.6-vs-9.x comparison showing the narrow-operand-retirement
  point independently confirmed by NOP byte counts (3 vs. 6).
- **`assemble` never calls the optimizer itself** — confirmed from source.
  But inlined (Path B) TAL still gets swept by whatever *encloses* it: a
  proc, method, or even a bare top-level script all run their own
  end-of-compile optimizer pass over the whole result, TAL included. Only
  genuine Path A (no enclosing compile) truly escapes it — confirmed
  empirically for both cases.
- **DSL authors can still add their own pass, and it isn't redundant:**
  neither path's cleanup covers optimizations specific to one DSL's own
  generation strategy. Worked example: `:calc`'s `transform=`, rewriting
  `push <name>; loadStk` to `load <name>` via `regsub`, gated on LVT
  availability.

### Chapter 6. Using `assemble` to get inlining (Path B) in practice

- **Terminology note:** this document has called the proc-body/incremental
  path "Path B" since chapter 3. Ashok Nadkarni's *The Tcl Programming
  Language* and other existing material call the same thing **inlining** —
  compiling a command's effect directly into the surrounding bytecode
  rather than deferring it to a runtime call. Both names refer to the
  identical mechanism from chapter 3; this document uses "inlining" from
  here on where it reads more naturally, without retiring "Path B."
- **The practical question this chapter answers:** given that `assemble`
  supports both paths, how does a DSL author actually arrange for their
  generated code to get inlined (Path B) rather than falling back to
  Path A's one-shot compile-and-run on every call?
- **Worked example: `:calc`'s `proc=`.** How `:calc` defines a construct
  that, when used inside an ordinary proc body, gets its TAL assembled
  directly into that proc's own `CompileEnv` — landing on Path B/inlining
  — rather than being evaluated as a standalone `assemble` call. What
  `proc=` has to do differently from a plain call to `assemble` to make
  that happen, and what the resulting bytecode looks like compared to the
  Path A equivalent.
- **The payoff:** connects chapter 3's abstract dual-path description and
  chapter 5's internals to something a DSL author can actually go and do.

---

## Appendices

### Appendix 1. Worked walkthrough: `TclCompileForCmd` in full

- **Line-by-line walk** of the function, building on the structural
  introduction from chapter 4.
- **The bail-out checks** at the top — why `for` declines to compile
  inline (returns `TCL_ERROR`, deferring to runtime) when the test, next,
  or body words require substitution, and what that protects against.
- **Loop rotation:** the jump ordering (start → jump to test → body → next
  → test → conditional jump back) as a genuine optimization — one fewer
  branch taken per iteration than a naive test-first layout, and
  especially valuable here because every bytecode instruction (including a
  jump) costs a full trip through the `TEBC` dispatch loop, unlike a
  native CPU branch a hardware predictor can absorb almost for free.
- **The exception-range machinery in practice:** `MAKE_LOOP_RANGE`,
  `CATCH_RANGE`, `CONTINUE_TARGET`, `BREAK_TARGET`, `FINALIZE_LOOP` as they
  actually get used, including why the `next` clause gets its own
  exception range with `supportsContinue` turned off.
- **The `CATCH_RANGE(range) { ... }` pattern:** a single-iteration C `for`
  used as a scope guard (setup before the block, teardown after, via the
  comma-operator condition) — flagged explicitly since it's easy to misread
  as an actual loop.
- **Where the peephole optimizer shows up in this example's own
  disassembly:** brief pointer back to chapter 5 — after `for` generates
  its code, a peephole pass replaces the dead push/pop from an empty body
  (and the command's own empty-string result) with NOPs, and a separate
  part of the same pass retargets jumps around them so they cost nothing
  at runtime. Kept to a sentence or two here; the mechanism itself lives in
  chapter 5, not this walkthrough.
- **The payoff:** ties chapters 2–5's abstractions to a real, complete,
  readable compile-proc.

### Appendix 2. Reference material

- **Existing documentation pointers:** the wiki hub, the original
  bytecode-compiler paper, `tclVM` — with honest notes on currency and
  completeness.
- **Glossary:** CompileEnv, ByteCode, LVT, TalInstType, BasicBlock,
  ExceptionRange.
- **Quick-reference table** of the two macro layers from chapter 2.

---

## Chapter 1: Introduction / Motivation

Writing a fast Tcl-level DSL depends directly on understanding `assemble`
and the dual-path compile system it sits alongside. This isn't incidental
detail — it's the actual mechanism a DSL's correctness and performance
rest on. Get it wrong, and the DSL either runs slower than it should or
breaks in ways that are hard to trace back to a cause.

That mechanism isn't documented anywhere coherent today. What exists is
scattered: old wiki pages that predate much of the current implementation,
a paper describing the bytecode compiler's original design from decades
ago, and an understanding held mostly by a small number of Tcl's core
maintainers rather than written down in one place. Someone starting from
scratch has no single resource that ties these pieces together.

This document's scope is deliberately narrow: the mechanics of how source
becomes bytecode via these two paths — the raw emit machinery, how a
`CompileEnv` comes into being one of two different ways, and how
`assemble` itself is built on top of both. It is not a tutorial on writing
TAL (the assembler's own input language) — that ground is better covered
by existing `assemble` documentation and the wiki pages referenced in
chapter 7.

The intended reader is comfortable reading Tcl core C and wants one of two
things: to use `assemble` or write their own compile-proc with a working
mental model of what's actually happening underneath, or to make sense of
what they're looking at when stepping through this code in a debugger.

---

## Chapter 2: Foundations: CompileEnv and the emit macros

### What a CompileEnv owns

Every piece of code that emits Tcl bytecode — whether it's an ordinary
command's compile-proc or the `assemble` machinery itself — ultimately
writes into a `CompileEnv`: the structure that owns the growing bytecode
buffer, the literal table, the exception-range list, and the running
stack-depth accounting for whatever's currently being compiled.

How a given `CompileEnv` comes into existence, and how long it lives,
varies — that's chapter 3's subject. Nothing in this chapter depends on
that. The macros described here just consume whatever `envPtr` they're
handed, regardless of its origin.

There are two layers of macros built on top of a `CompileEnv`.

### The raw emit primitives

Functions like `TclEmitOpcode`, `TclEmitInstInt1`, `TclEmitInstInt4`,
`TclEmitInstInt14`, `TclEmitInstInt44`, `TclEmitInstInt41`, `TclEmitPush`,
and `TclEmitInvoke` are the actual mechanism that writes bytes into a
`CompileEnv`'s buffer and advances it.

- Every one of these takes `envPtr` as an explicit, ordinary argument.
- Nothing about the pointer is hidden or implicit at this layer.
- Call one of these directly and the pointer is right there in the call,
  the same as any other parameter.

### The shorthand layer

Sitting on top of the raw primitives, `tclCompile.h` defines a family of
convenience macros that exist purely to cut boilerplate at each emit call
site:

- `OP`, `OP1`, `OP4`, `OP14`, `OP44`, `OP41` — issue an instruction, with
  zero, one, or two operands of various widths.
- `INVOKE`, `INVOKE4`, `INVOKE41` — issue an instruction that can generate
  a break/continue.
- `PUSH`, `PUSH_STRING`, `PUSH_SIMPLE_TOKEN`, `PUSH_OBJ`, `PUSH_OBJ_FLAGS`,
  `PUSH_TOKEN`, `PUSH_EXPR_TOKEN` — push a value onto the stack from one
  of several different sources (a string literal, a `Tcl_Obj`, a parse
  token, an expression).
- `BODY` — compile the body of a command like `if` or `while`.
- `BACKLABEL`/`BACKJUMP` and `FWDLABEL`/`FWDJUMP` — mark and emit backward
  and forward jumps.
- `MAKE_CATCH_RANGE`/`MAKE_LOOP_RANGE`, `CATCH_RANGE`, `CATCH_TARGET`,
  `BREAK_TARGET`, `CONTINUE_TARGET`, `FINALIZE_LOOP` — create and finalize
  exception ranges.
- `STKDELTA` — adjust the tracked stack depth directly.

`OP(LOAD_STK)` expands to `TclEmitOpcode(INST_LOAD_STK, envPtr)`.
`OP4(LOAD_SCALAR, idx)` expands to
`TclEmitInstInt4(INST_LOAD_SCALAR, idx, envPtr)`. Two things happen at
once: the `INST_` prefix gets pasted onto the name automatically, and
`envPtr` disappears from what the caller has to type.

That disappearance isn't `envPtr` being passed silently — it isn't passed
at all. The macro doesn't take `envPtr` as one of its own arguments; it
just assumes a variable named exactly `envPtr` already exists in whatever
function the macro gets expanded inside of, and refers to it directly.
This works reliably because the convention of naming that parameter
`envPtr` is followed consistently everywhere these macros are used — it's
a naming convention the macros lean on, not something enforced by the
compiler.

This shorthand layer is used pervasively. Every command that has its own
dedicated compile-proc — `append`, `incr`, `if`, `for`, `foreach`, and many
more, spread across `tclCompCmds.c` and its sibling files — leans on it
repeatedly. Without it, every single emitted instruction in every one of
those functions would otherwise need `INST_` and `, envPtr` typed out by
hand, at every call site.

### The sharp edge

Because the shorthand macros paste `INST_` onto a literal token at
preprocessor time, they only work when the opcode is known at C-compile
time. If the opcode is itself a value computed at runtime — the result of
an array or table lookup, say — `OP(...)`'s trick doesn't apply, since
there's no literal name to paste `INST_` onto. Code in that situation has
to fall back to the raw layer, calling `TclEmitOpcode` (or one of its
siblings) directly, with `envPtr` spelled out by hand.

### Debugger note

This split also explains a common source of confusion when stepping
through this code in a debugger. `TclEmitOpcode(INST_LOAD_STK, envPtr)`
shows the pointer right there in the call. `OP(LOAD_STK)` does not —
syntactically, `envPtr` isn't present at all until the preprocessor
expands the macro. There's no separate call frame for a macro invocation
the way there is for a function call, so single-stepping won't show a
distinct "step into" point where the pointer is passed. Seeing where it
actually comes from means looking at preprocessed source, or simply
knowing the macro's definition well enough to expand it mentally.

---

## Chapter 3: Two ways to get a CompileEnv: the dual-path system

Chapter 2 deliberately stayed silent on how a `CompileEnv` comes into
existence. There are two distinct answers, and `assemble` supports both —
which is the reason it needs two separate compile-time entry points rather
than one.

### Path A — direct/one-shot evaluation

This is the path taken when `tcl::unsupported::assemble` is invoked
directly, as an ordinary command call, rather than appearing inside a
proc or method body that's itself being compiled.

- `CompileAssembleObj` builds a brand-new `CompileEnv` from scratch, with
  `TclInitCompileEnv`.
- The TAL source is assembled into that fresh environment via
  `TclAssembleCode` — the same core assembly logic used by Path B.
- Once assembly finishes, the result is turned into a standalone
  `ByteCode` object with `TclInitByteCodeObj`.
- That `ByteCode` is then actually executed — `assemble`, used this way,
  behaves like a compile-and-run construct (similar in spirit to `eval`,
  but for TAL): it has to produce a result for its caller, not just a
  compiled object sitting unused. This document hasn't directly located
  the specific execution call in source, so treat the exact function name
  as unconfirmed; what's solid is the behavior itself — `assemble`
  evaluates the TAL and returns whatever the assembled code produces.
- The temporary `CompileEnv` is then torn down with `TclFreeCompileEnv` —
  its job was only ever to exist long enough to produce, and then run,
  that one `ByteCode`.

Nothing about this `CompileEnv` is shared with anything else. It's built,
used once, and discarded. The result of executing the `ByteCode` it
produced is what survives — that result is `assemble`'s own return value.

### Path B — proc-body/incremental compilation

*Elsewhere — including Ashok Nadkarni's* The Tcl Programming Language *—
this same mechanism is called* **inlining***: compiling a command's
effect directly into the surrounding bytecode rather than deferring it to
a runtime call. Both names describe the identical thing; this document
keeps "Path B" as its own running label, introduced in contrast to Path A,
but uses "inlining" alongside it from here on, particularly once chapter 6
gets into using it deliberately.*

This is the path taken when `assemble` (or code that behaves like it)
appears as a command inside a proc or method body that's being compiled
as a whole — for example, if `assemble` itself were registered as, or
called from within, a compile-proc for some other command.

- `TclCompileAssembleCmd` is itself a registered compile-proc, so it
  receives an `envPtr` the same automatic way any other compile-proc
  does — except here, that `envPtr` is the *caller's* ongoing
  `CompileEnv`, already partway through compiling the enclosing proc
  body, not a fresh one.
- The TAL source is assembled directly into that shared environment, via
  the same `TclAssembleCode` core used by Path A.
- Because this `CompileEnv` isn't disposable — it belongs to whatever
  proc or method is being compiled as a whole — a failed assembly can't
  simply be discarded the way Path A's can. Before assembling, several
  pieces of the environment's state are saved: the command count, the
  current code offset, the tracked stack depth, and the exception-range
  list. If assembly fails partway through, that saved state is restored,
  rolling the `CompileEnv` back to exactly where it was before the
  attempt — so a failed `assemble` doesn't leave partial, inconsistent
  bytecode sitting in the middle of an otherwise-valid proc body.

### The same core, two different lifetimes

Both paths funnel through the same underlying assembly logic
(`TclAssembleCode`); what differs is entirely how long the surrounding
`CompileEnv` lives and who owns it.

```
Path A: direct/one-shot                Path B: proc-body/incremental
------------------------               ------------------------------
TclInitCompileEnv                      (envPtr already exists —
      |                                 belongs to the enclosing
      v                                 proc/method compile)
  fresh CompileEnv                            |
      |                                       v
      v                                save numCommands, codeNext,
TclAssembleCode  <--------------+      stack depth, exception ranges
      |                         |             |
      v                         |             v
  TclInitByteCodeObj            +----- TclAssembleCode
      |                                       |
      v                              on failure: restore saved state
 execute the ByteCode                on success: state simply advances
      |                                       |
      v                                       v
TclFreeCompileEnv                    bytecode now part of the enclosing
      |                                proc body's CompileEnv (not
      v                                executed yet — it runs later,
result returned to caller             whenever that proc is called)
```

The practical consequence: Path A both compiles and runs the TAL in one
step, at the moment `assemble` is called — its result is what `assemble`
itself returns. Path B's assembled bytecode has no independent existence
and doesn't run at all during compilation; it becomes part of the
enclosing proc or method's `CompileEnv`, indistinguishable afterward from
code emitted by any other compile-proc that ran during the same compile,
and only actually executes later, whenever that proc is called.

### What a failed compile-proc actually means

The rollback in Path B only makes sense once one thing is clear: a
compile-proc returning `TCL_ERROR` is not a script error. It's a private
signal to the surrounding compiler — `TclCompileScript`, the function that
walks a proc body one command at a time during compilation — meaning
"decline to compile this one command inline; fall back to compiling it as
an ordinary runtime call instead."

The fallback itself is mechanical and the same for every compile-proc in
Tcl core, not something specific to `assemble`:

- Push the command name.
- Push each argument word.
- Emit `invokeStk` (or an `invokeStkN` variant, depending on word count).

`TclCompileScript` then simply moves on to the next command in the script.
Nothing about one command's fallback interrupts or poisons compilation of
anything that follows it — the rest of the proc compiles exactly as it
would have otherwise. The actual Tcl-level error, if there is one, is only
raised later, at runtime, when the `invokeStk` executes and the command's
ordinary implementation runs and discovers whatever problem made
compile-time inlining impossible.

Two concrete instances of this same contract, already covered in this
document:

- `TclCompileForCmd`'s bail-out checks (chapter 4) — a non-literal test,
  next, or body word doesn't fail the enclosing proc's compile; it just
  makes that one `for` get compiled as a plain command invocation instead
  of inlined bytecode.
- `TclCompileAssembleCmd`'s *pre-attempt* checks — the wrong number of
  words, or a TAL argument that isn't a literal simple word — return
  `TCL_ERROR` immediately, before any assembly is attempted, triggering
  the identical push-name/push-args/`invokeStk` fallback. This is the one
  case where `assemble`'s compile-proc genuinely follows the general
  contract described above.

### A genuine TAL error doesn't follow this contract at all

Once `TclCompileAssembleCmd` actually attempts assembly — the argument
passed both pre-attempt checks, and `TclAssembleCode` was called — a
failure there (an invalid mnemonic, a stack-depth inconsistency, and so
on) is handled completely differently, confirmed directly from source:

```c
if (TCL_ERROR == TclAssembleCode(envPtr, tokenPtr[1].start,
        tokenPtr[1].size, TCL_EVAL_DIRECT)) {
    Tcl_AppendObjToErrorInfo(interp, ...);
    envPtr->numCommands = numCommands;
    envPtr->codeNext = envPtr->codeStart + offset;
    envPtr->currStackDepth = depth;
    TclCompileSyntaxError(interp, envPtr);
}
return TCL_OK;
```

State is rolled back exactly as described above — but the function then
calls `TclCompileSyntaxError` and returns `TCL_OK` unconditionally, not
`TCL_ERROR`. `TclCompileSyntaxError` bakes a small stub directly into
the bytecode at that point: unconditionally raise the already-known
error message, if this code is ever actually executed. `TclCompileScript`
sees a successful compile — nothing about this triggers the push-name/
push-args/`invokeStk` fallback at all.

This makes sense once the two situations are told apart: a non-literal
`for` test is something the compile-proc genuinely *can't evaluate* at
compile time — deferring entirely to a real runtime attempt is the only
option, and that attempt might well succeed. A malformed TAL mnemonic or
an unbalanced stack, by contrast, is a permanent defect already fully
diagnosed at compile time — re-running the real `assemble` command at
runtime would just redo the same parsing work to rediscover the
identical error. Baking the already-known error in directly is cheaper,
and skips the redundant work entirely.

One consequence worth being aware of: a defect fully known at compile
time can still lie completely dormant for the entire life of a running
program, if the specific bytecode location it was baked into is never
actually reached. This isn't really a downside specific to baking in a
stub, though — a genuine runtime fallback would rediscover the identical
error just as conditionally, only when that same line executes. It's a
property of Tcl's per-command compile-proc model generally, compounded
by the fact that a proc's body isn't even compiled until its first call
(chapter 3) — so the error stays invisible until both the proc has been
called at least once, and execution actually reaches the bad line.

> **Tip:** A dormant error like this is easy to find without ever
> running the code. `TclCompileSyntaxError` bakes in a distinctive,
> three-instruction pattern — `push <error message>`, `push <error
> options>`, `syntax` — and `tcl::unsupported::disassemble proc` forces
> a proc to compile (and therefore reveals any baked-in error) without
> ever calling it. Walking every proc in an application (via `info
> procs`, and similarly for methods) and disassembling each one, then
> checking the output for `syntax`, surfaces every compile-time-known
> TAL defect in one pass — including ones sitting in code paths that
> haven't run yet.

### Path B's failure is a handoff to Path A — but only sometimes

Follow the *pre-attempt* fallback for `assemble` one step further: the
`invokeStk` it triggers, once it actually executes at runtime, calls
`assemble`'s ordinary command procedure — not its compile-proc. That
ordinary command procedure is exactly what runs when `assemble` is
invoked directly at top level in the first place, which is Path A: a
fresh, disposable `CompileEnv`, assembled into via the same
`TclAssembleCode` core, turned into a standalone `ByteCode`, then torn
down.

So a pre-attempt Path B failure doesn't lead to some separate, generic
error path — it hands off directly into Path A, re-running the identical
assembly logic once more, at runtime, in a fresh one-shot environment.

**But this only covers the narrower failure mode.** A genuine TAL syntax
error — the case most people would actually picture when they imagine
"assembling bad TAL fails" — never reaches Path A at all, per the
previous section: it's caught, diagnosed, and baked into a syntax-error
stub during the Path B attempt itself, with the compile reported as
successful. Path A only gets involved when the compile-proc declines to
even attempt assembly in the first place — a non-literal argument, or
the wrong number of words — which is a narrower case than "the TAL is
invalid."

**This is specific to `assemble`, not a universal consequence of the
fallback rule.** The general rule is only that a compile-proc failure
falls back to `invokeStk`, calling that command's ordinary command
procedure — what that procedure actually does varies entirely by command.
For `assemble`'s pre-attempt bail-out, it happens to be Path A, which
still produces and executes real bytecode. For a command like `for`, the
ordinary command procedure (`Tcl_ForObjCmd`) is genuinely different code
with no bytecode in it at all — it directly evaluates the start/test/
next/body scripts in an interpreted loop. `assemble` degrading into
Path A is a property of `assemble`'s particular architecture — there's
no simpler way to "run TAL" than to assemble and execute it — not
something every compiled command does, and not even something every
`assemble` failure does.

### The fallback operates per command, not per script

A script with several commands doesn't get compiled as an all-or-nothing
unit. `TclCompileScript` walks a script's commands once, in order, and
makes one independent decision per command: if it has a registered
compile-proc and that compile-proc succeeds, its instructions are inlined
directly into the surrounding `CompileEnv`. If it has no compile-proc at
all, or its compile-proc returns `TCL_ERROR`, that one command becomes a
bare `invokeStk` to its ordinary command procedure instead. Every other
command in the same script is completely unaffected either way.

The result is a single `ByteCode` for the whole script that's a mix of
inlined instruction sequences and `invokeStk` calls, decided command by
command. Only a limited set of core commands (`for`, `if`, `while`,
`foreach`, `switch`, `set`, `incr`, `append`, and a few dozen others) have
a compile-proc at all; every user-defined proc and every library or
package command was never a candidate for inlining in the first place,
and goes straight to `invokeStk` without anything resembling an attempt
or a failure.

### Compiled-but-not-cached versus never compiled

It's worth keeping two different things distinct: whether a script gets
compiled into bytecode at all, and whether that bytecode is reused across
multiple evaluations of the same script.

By default, any script handed to `Tcl_EvalObjEx` is compiled the first
time it executes, and the resulting bytecode is cached on the script's
own `Tcl_Obj` so that later evaluations of that same object can skip
recompiling. This applies uniformly — a top-level script, a `source`d
file, a proc body, an `eval`'d string, and a callback script registered
for a trace, binding, `fileevent`, or `after` are all compiled the same
way on their first run.

Whether a *later* firing of the same callback benefits from that cached
bytecode depends on whether the callback mechanism retains and reuses the
same `Tcl_Obj` across firings, or reconstructs a fresh string each time
(common where the callback text needs `%`-style substitution before
evaluation). A callback rebuilt fresh on every firing gets recompiled
from scratch every time — but it is still going through the ordinary
compile-then-execute machinery, not the separate, deliberately
bytecode-free `TCL_EVAL_DIRECT` path. "Recompiled every time" and "never
compiled at all" are different things, and callback scripts land in the
first category by default, not the second.

---

## Chapter 4: How an ordinary but non-trivial command gets compiled: `for`

### The control group

Before looking at how `assemble` builds bytecode dynamically from TAL
source, it's worth seeing how an ordinary, individually-registered
compile-proc does the same underlying job — writing instructions into a
`CompileEnv` using the macros from chapter 2 — for one specific, known
command.

A normal compile-proc is registered against exactly one command name. It
receives `envPtr` as an ordinary function parameter, supplied automatically
by whichever machinery is compiling the surrounding script (chapter 3).
Every opcode it might emit is known and named at C-compile time — there's
no equivalent of `assemble`'s runtime mnemonic lookup, because the
compile-proc already knows, by construction, exactly which command it's
compiling and exactly which instructions that command needs.

### Why `for`, specifically

Most compile-procs only ever emit straight-line code. `TclCompileIncrCmd`,
`TclCompileAppendCmd`, and similar functions call a short, fixed sequence
of `OP`/`OP1`/`OP4`/`PUSH`-family macros with no branching involved at
all — there's nothing to jump over or around.

`for` is deliberately chosen here because it isn't one of those. It needs:

- The jump macros — `FWDJUMP`, `BACKLABEL`, `BACKJUMP`, `FWDLABEL` — to
  lay out the loop's control flow.
- The exception-range macros — `MAKE_LOOP_RANGE`, `CATCH_RANGE`,
  `CONTINUE_TARGET`, `BREAK_TARGET`, `FINALIZE_LOOP` — so that `break` and
  `continue` inside the loop body work correctly.

Most compile-procs never touch either of these macro families at all.

### Why that matters for what's coming

Arbitrary jumps and exception-range tracking aren't a `for`-specific
concern — they're exactly what `assemble` (chapter 5) has to support in
general, for any TAL program a person might write, not just one
hand-written control structure. `for`'s use of the jump and exception-range
macros is the closest an ordinary, single-command compile-proc gets to the
general problem `assemble` has to solve for arbitrary TAL. Seeing it here
first, in a small and familiar setting, makes chapter 5's more general
machinery (`BasicBlock`, `AssemblyEnv`, the jump-fixup pass) easier to
recognize as doing the same kind of job at a larger scale.

### `TclCompileForCmd`, structurally

The full function is walked line by line in chapter 6. Here, the goal is
just its overall shape.

`TclCompileForCmd` takes the usual compile-proc signature —
`Tcl_Interp *interp`, `Tcl_Parse *parsePtr`, an unused `Command *`, and
`CompileEnv *envPtr`. Its structure falls into three parts:

1. **Bail-out checks.** Before emitting anything, the function verifies
   that it has exactly the five words a `for` command needs
   (`for start test next body`), and that the `test`, `next`, and `body`
   words are all `TCL_TOKEN_SIMPLE_WORD` tokens — meaning they contain no
   substitutions that could only be resolved at runtime. If any of these
   checks fail, the function returns `TCL_ERROR` immediately, having
   emitted nothing. Per chapter 3's fallback contract, this doesn't fail
   the surrounding script's compile — it just means this particular `for`
   gets compiled as an ordinary runtime command call instead, via
   `Tcl_ForObjCmd`, the separate, non-bytecode interpreted implementation.
2. **The main emission sequence.** Once the checks pass, the function
   emits the `start` command, the loop body wrapped in a catch range, the
   `next` command in its own catch range, and the `test` expression,
   arranged using the jump macros in a specific order — covered in full,
   with the reasoning behind that order, in chapter 6.
3. **Finalization and result.** The exception ranges are finalized, and
   the function pushes an empty string as `for`'s own result (per Tcl
   semantics, `for` always evaluates to the empty string), then returns
   `TCL_OK`.

### The source, with line numbers for reference

The chapters that follow (especially chapter 6's line-by-line walk) cite
specific lines back into this listing, so it's given here in full rather
than in fragments.

```c
  1  /*
  2   *----------------------------------------------------------------------
  3   *
  4   * TclCompileForCmd --
  5   *
  6   *      Procedure called to compile the "for" command.
  7   *
  8   * Results:
  9   *      Returns TCL_OK for a successful compile. Returns TCL_ERROR to defer
 10   *      evaluation to runtime.
 11   *
 12   * Side effects:
 13   *      Instructions are added to envPtr to execute the "for" command at
 14   *      runtime.
 15   *
 16   *----------------------------------------------------------------------
 17   */
 18  int
 19  TclCompileForCmd(
 20      Tcl_Interp *interp,         /* Used for error reporting. */
 21      Tcl_Parse *parsePtr,        /* Points to a parse structure for the command
 22                                   * created by Tcl_ParseCommand. */
 23      TCL_UNUSED(Command *),
 24      CompileEnv *envPtr)         /* Holds resulting instructions. */
 25  {
 26      DefineLineInformation;      /* TIP #280 */
 27      Tcl_Token *startTokenPtr, *testTokenPtr, *nextTokenPtr, *bodyTokenPtr;
 28      Tcl_ExceptionRange bodyRange, nextRange = -1;
 29      Tcl_BytecodeLabel evalBody, testCondition;
 30      if (parsePtr->numWords != 5) {
 31          return TCL_ERROR;
 32      }
 33      /*
 34       * If the test expression requires substitutions, don't compile the for
 35       * command inline. E.g., the expression might cause the loop to never
 36       * execute or execute forever, as in "for {} "$x > 5" {incr x} {}".
 37       */
 38      startTokenPtr = TokenAfter(parsePtr->tokenPtr);
 39      testTokenPtr = TokenAfter(startTokenPtr);
 40      if (testTokenPtr->type != TCL_TOKEN_SIMPLE_WORD) {
 41          return TCL_ERROR;
 42      }
 43      /*
 44       * Bail out also if the body or the next expression require substitutions
 45       * in order to insure correct behaviour [Bug 219166]
 46       */
 47      nextTokenPtr = TokenAfter(testTokenPtr);
 48      bodyTokenPtr = TokenAfter(nextTokenPtr);
 49      if ((nextTokenPtr->type != TCL_TOKEN_SIMPLE_WORD)
 50              || (bodyTokenPtr->type != TCL_TOKEN_SIMPLE_WORD)) {
 51          return TCL_ERROR;
 52      }
 53      /*
 54       * Inline compile the initial command.
 55       */
 56      BODY(                       startTokenPtr, 1);
 57      OP(                         POP);
 58      /*
 59       * Jump to the evaluation of the condition. This code uses the "loop
 60       * rotation" optimisation (which eliminates one branch from the loop).
 61       * "for start cond next body" produces then:
 62       *       start
 63       *       goto A
 64       *    B: body                : bodyCodeOffset
 65       *       next                : nextCodeOffset, continueOffset
 66       *    A: cond -> result      : testCodeOffset
 67       *       if (result) goto B
 68       */
 69      FWDJUMP(                    JUMP, testCondition);
 70      /*
 71       * Compile the loop body.
 72       */
 73      bodyRange = MAKE_LOOP_RANGE();
 74      BACKLABEL(              evalBody);
 75      CATCH_RANGE(bodyRange) {
 76          BODY(                   bodyTokenPtr, 4);
 77      }
 78      OP(                         POP);
 79      /*
 80       * Compile the "next" subcommand. Note that this exception range will not
 81       * have a continueOffset (other than -1) connected to it; it won't trap
 82       * TCL_CONTINUE but rather just TCL_BREAK.
 83       */
 84      CONTINUE_TARGET(        bodyRange);
 85      if (!TclIsEmptyToken(nextTokenPtr)) {
 86          nextRange = MAKE_LOOP_RANGE();
 87          envPtr->exceptAuxArrayPtr[nextRange].supportsContinue = 0;
 88          CATCH_RANGE(nextRange) {
 89              BODY(               nextTokenPtr, 3);
 90          }
 91          OP(                     POP);
 92      }
 93      /*
 94       * Compile the test expression then emit the conditional jump that
 95       * terminates the for.
 96       */
 97      FWDLABEL(               testCondition);
 98      PUSH_EXPR_TOKEN(            testTokenPtr, 2);
 99      BACKJUMP(                   JUMP_TRUE, evalBody);
100      /*
101       * Fix the starting points of the exception ranges (may have moved due to
102       * jump type modification) and set where the exceptions target.
103       */
104      BREAK_TARGET(           bodyRange);
105      FINALIZE_LOOP(bodyRange);
106      if (nextRange != -1) {
107          BREAK_TARGET(       nextRange);
108          FINALIZE_LOOP(nextRange);
109      }
110      /*
111       * The for command's result is an empty string.
112       */
113      PUSH(                       "");
114      return TCL_OK;
115  }
```

The three-part shape described above maps onto this listing as: bail-out
checks at lines 30–52, the main emission sequence at lines 53–113
(with the pushed empty-string result folding into the third part), and
`return TCL_OK` at line 114.

### What triggers the bail-out in practice

Concretely, a `for` loop drops out of `TclCompileForCmd` and into the
interpreted fallback whenever:

- The `test`, `next`, or `body` word contains a variable or command
  substitution — for example, `for {set i 0} $test {incr i} $body`, where
  `test` and `body` are variables holding script text rather than literal
  braced text.
- The command word itself isn't recognizable as `for` at compile time —
  for example, invoking it through a variable holding the command name,
  or via `{*}$someList` expansion.
- `for` has been renamed, or shadowed by a proc of the same name — Tcl
  tracks this with a command epoch that invalidates any cached
  association between the literal word `for` and `TclCompileForCmd`.

None of these are error conditions in the ordinary sense — the loop still
runs correctly either way. They're simply cases the compile-proc has
decided aren't worth the complexity of inlining, deferring instead to the
straightforward interpreted implementation.

---

## Chapter 5: How `assemble` actually works internally

### `TalInstructionTable`: one row per mnemonic

Everything `assemble` knows about the TAL language lives in a single
static table, `TalInstructionTable`, with one row per mnemonic — roughly
150 entries, covering `push`, `load`, `store`, `jump`, `add`, and every
other instruction TAL supports. Each row holds:

- the mnemonic string itself,
- a `TalInstType` value — not the real bytecode opcode, but a much smaller
  category (around 20 values) naming the *shape* of the instruction's
  operand: `ASSEM_1BYTE` (no operand), `ASSEM_LVT` (a local-variable-table
  index), `ASSEM_JUMP` (a jump offset), `ASSEM_PUSH` (a literal to push),
  and so on,
- the real `INST_*` bytecode opcode that mnemonic ultimately emits, and
- operand consumed/produced counts, used for stack-depth bookkeeping as
  the assembler works through a TAL program.

### The dispatch is on shape, not on mnemonic

It would be reasonable to guess that assembling a line of TAL means a
big switch statement with one case per mnemonic — roughly 150 cases. That
isn't how it works. `AssembleOneLine` looks the mnemonic up in
`TalInstructionTable` to find its `TalInstType`, and dispatches on
*that* — around 20 cases, not 150. Mnemonics that share an operand shape
(every `ASSEM_LVT` instruction, for instance) are parsed and validated by
the same code path, even though they differ in which real opcode they
eventually emit. The table is what maps each of the many mnemonics down
to one of the few shapes.

### Three chokepoints for actual emission

Chapter 2 described two layers of emit macros: the raw primitives
(`TclEmitOpcode`, `TclEmitInstInt1`, `TclEmitInstInt4`, and siblings), and
the shorthand layer built on top of them. `assemble`'s own code uses only
the raw layer, and only from three places:

- `BBEmitOpcode` — opcode byte only,
- `BBEmitInstInt1` — opcode plus a one-byte operand,
- `BBEmitInstInt4` — opcode plus a four-byte operand.

These three functions are the *only* call sites for the raw emit macros
anywhere in `assemble`'s own source. Every one of the roughly 150
mnemonics, at runtime, gets routed through one of these three, parameterized
by the table row that mnemonic resolved to. Where an ordinary compile-proc
like `TclCompileForCmd` has macro calls fixed in its own source for each
opcode it might emit, `assemble` has exactly three fixed call sites, and
decides at runtue, via the table, which opcode each one actually writes.

### Extra bookkeeping an ordinary compile-proc doesn't need

`assemble` has to support arbitrary jumps and exception ranges for *any*
TAL program a person might write — not one fixed control structure known
in advance, the way `TclCompileForCmd` only ever needs to build one very
specific loop shape (chapter 4). That generality needs its own supporting
structures:

- `BasicBlock` — TAL is assembled as a graph of basic blocks rather than
  a flat instruction stream, so that jump targets and stack-depth
  consistency can be checked once the whole program's shape is known,
  rather than line by line as each mnemonic is read.
- `AssemblyEnv` — the assembler's own state struct, wrapping the
  `CompileEnv*` (as an ordinary field, not hidden — see chapter 2 and
  chapter 3) alongside TAL-specific state: parse position, the label
  table, and the list of basic blocks under construction.

**What a basic block actually is.** A basic block is a run of
instructions with exactly one entry point, at the top, and exactly one
exit, at the bottom — nothing jumps into its middle, and nothing inside
it jumps out except as the very last instruction.

**The rule for where one starts and ends:** a basic block starts at a
label, the first instruction, or right after a jump — and ends at a
jump, the last instruction, or right before a label.

"Jump" here includes not just an explicit TAL
`jump`/`jumpTrue`/`jumpFalse`, but the out-of-band transfer an exception
range performs — a `catch` target reachable from an error or a `return`,
or a loop range's break/continue targets — since those reach a label
just as surely as an explicit jump does, even with no jump instruction
anywhere in sight.

Take a small hand-written TAL loop:

```
push 0
store i
pop
label loop
load i
push 10
lt
jumpFalse done
load i
push 1
add
store i
pop
jump loop
label done
load i
```

This splits into four basic blocks:

| Block | Instructions | Why it starts/ends here |
|---|---|---|
| BB1 | `push 0` / `store i` / `pop` | Starts at the very top; ends because `label loop` immediately follows — the next instruction is a jump target, so it must begin a new block. |
| BB2 | `load i` / `push 10` / `lt` / `jumpFalse done` | Starts at `label loop`; ends at the conditional jump, which has two possible successors: fall through to BB3, or jump to BB4. |
| BB3 | `load i` / `push 1` / `add` / `store i` / `pop` / `jump loop` | Starts right after the conditional jump (the fall-through target); ends at its own unconditional jump back to BB2. |
| BB4 | `load i` | Starts at `label done`; the final block, falls off the end. |

Each block's own instructions are always executed in sequence — the
branching only ever happens *between* blocks, at the boundaries. That's
what makes the graph useful: `assemble` can check things like "does
every path through this program leave the stack at a consistent depth"
one block at a time, following the edges between them (BB1→BB2,
BB2→BB3, BB2→BB4, BB3→BB2), rather than re-deriving control flow from
scratch by scanning raw jump offsets. It's also what makes computing
final jump offsets a one-time, after-the-fact step (this chapter's next
section) rather than something that has to be guessed at while each
instruction is still being written.

### Stack-depth consistency across the graph

A block is under no obligation to leave the stack at the same depth it
was entered at — depth can rise and fall freely within a block as it
does real work. What has to stay consistent is narrower: whenever a
block has more than one *incoming* edge, every one of those edges has to
agree on the depth it hands off, since the block's own instructions only
make sense relative to one specific starting depth.

BB2 above is exactly this case — it's reached both from BB1 (falling
straight through) and from BB3 (jumping back via `label loop`) — so this
worked example doubles as a live test of whether `assemble` actually
checks that. It does. The version above, with a `pop` after each
`store`, assembles and runs correctly — but leaving those two `pop`
instructions out (as an earlier draft of this very example did) produces
a real, immediate assembly-time error:

```
inconsistent stack depths on two execution paths
```

Since a `store` leaves the stack unchanged, a `pop` is needed afterward
to actually remove the value — there's no fused instruction that stores
and discards in one step. Leaving that `pop` out, as the earlier version
of this example did, leaves the value sitting on the stack at the end of
both BB1 and BB3. BB1→BB2 then hands off at depth 1, but BB3→BB2 hands
off at depth 2, one iteration's worth of accumulated leftovers higher —
exactly the "two execution paths disagree" `assemble` catches. Adding a
`pop` after each `store` brings both edges into BB2 back to the same
depth, which is what makes the corrected version above valid. This is
the same discipline chapter 4 showed `TclCompileForCmd` following
unconditionally via its own `OP(POP)` after every loop body —
`assemble` doesn't do that cleanup automatically; it only checks, after
the fact, that whatever the TAL author wrote is actually self-consistent.

### How the check actually walks the graph

Confirmed directly from `tclAssembly.c`'s own function list and struct
fields, not just inferred: the `BasicBlock` struct stores an
`initialStackDepth` (absolute depth on entry), `minStackDepth` and
`maxStackDepth` (low/high-water marks, relative), and a `finalStackDepth`
(relative depth change across the block) — exactly the split this
section has been describing: each block's own relative delta is a
simple, unambiguous calculation, since there's no branching inside a
block to complicate it.

A dedicated function, `StackCheckBasicBlock`, then walks the graph itself
— following the fall-through edge and any jump targets — propagating an
absolute starting depth forward along each edge. A `BB_VISITED` flag
(with `ResetVisitedBasicBlocks` to clear it between passes) keeps the
walk from looping forever on a graph that contains a cycle, like BB3→BB2
in this chapter's own example. When the walk reaches a block a second
time by a different route, it checks the newly-arriving depth against
whatever was already recorded for that block on the first visit — and
that comparison failing is precisely the "inconsistent stack depths on
two execution paths" error this section just triggered.

### No more choosing between narrow and wide jump encodings

Older Tcl bytecode had separate 1-byte and 4-byte encodings for jump
offsets, and compiling a jump meant guessing which size would be needed,
then going back and patching the instruction (growing it, and shifting
everything after it) if a forward jump turned out to need the wider form.
That guess-and-patch machinery is gone. In the current `TalInstructionTable`,
legacy mnemonics like `jump4`, `jumpTrue4`, and `jumpFalse4` are kept only
for old TAL source compatibility, and map to the identical real opcode as
their bare counterparts (`jump4` → `INST_JUMP`, same as plain `jump`) —
there is no longer a distinct narrow encoding to choose between.

`assemble` sidesteps the original problem differently: because TAL is
built as a graph of `BasicBlock`s first, final jump offsets are only
computed once the whole program's shape is known (`FillInJumpOffsets`
runs after basic-block construction, not interleaved with it). There's
nothing to guess, because nothing is written until the real distance is
already known.

### The peephole optimizer: what actually gets skipped, and what doesn't

Tcl's ordinary compiled code goes through a peephole optimization pass —
`TclOptimizeBytecode` in `tclOptimize.c` — after compilation, a four-step
pipeline: `ConvertZeroEffectToNOP` (dead push/pop pairs, among other
zero-effect patterns, become NOPs), `BetterEqualityTesting` (collapses
`push ""` followed by an equality test into a single `IS_EMPTY`
instruction), `AdvanceJumps` (retargets any jump that would land on a NOP
or a chained jump, so it points at the next real instruction instead),
and `TrimUnreachable` (physically shrinks the code, rather than NOPing it,
for dead code at the very end of the buffer where nothing after it needs
its offsets preserved).

Bytes are overwritten with same-length NOPs, rather than deleted, because
every jump target, exception-range boundary, and command-map entry
elsewhere in the same bytecode is stored as a raw numeric offset —
deleting bytes would require re-deriving all of those; overwriting in
place preserves them for free. `AdvanceJumps` then makes sure this costs
nothing at runtime: a NOP an execution path would otherwise have to step
through gets jumped straight past instead, once a jump exists nearby to
retarget.

**`assemble` itself never calls this pass.** Directly from source:

```c
static ByteCode *
CompileAssembleObj(...)
{
    ...
    TclInitCompileEnv(interp, &compEnv, source, sourceLen, NULL, 0);
    status = TclAssembleCode(&compEnv, source, sourceLen, TCL_EVAL_DIRECT);
    ...
    TclEmitOpcode(INST_DONE, &compEnv);
    codePtr = TclInitByteCodeObj(objPtr, &assembleCodeType, &compEnv);
    TclFreeCompileEnv(&compEnv);
    ...
    return codePtr;
}
```

Neither `CompileAssembleObj` (Path A) nor `TclCompileAssembleCmd`
(Path B) contains a call to `TclOptimizeBytecode`. But this fact is
narrower than "assembled code is never optimized," and the difference
matters:

- **Genuine Path A — a standalone, dynamically-assembled call with no
  enclosing compile — really does escape optimization entirely.**
  Confirmed by hand-assembling a TAL block with deliberate dead
  push/pop pairs and passing it to `assemble` as a variable (forcing
  Path A, since the compile-proc can't inline a non-literal argument):
  the dead pairs survive untouched in the disassembly of the *calling*
  context, and nothing in `assemble`'s own path could have touched them.
- **Inlined (Path B) TAL does get optimized — just not by `assemble`.**
  Once TAL is inlined into a surrounding `CompileEnv`, its instructions
  are indistinguishable from anything else in that `CompileEnv` (chapter
  3), and whatever compiled the *enclosing* code runs its own
  end-of-compile optimizer pass over the whole thing, TAL included. This
  holds for a proc body, and — confirmed identically via
  `tcl::unsupported::disassemble script` — for a bare top-level script
  too. It isn't proc-specific; it's a property of any enclosing compile.
- **The clearest evidence of this is a case where the optimizer merges
  bytes from two different origins into one eliminated pair.** A minimal
  TAL block (`push "deadlit"; pop; push 1`) placed as the body of a `for`
  loop produced six NOP bytes in the disassembly, not five. `for`'s own
  compile-proc (chapter 4) unconditionally emits its own `pop` right
  after any loop body, to discard the body's result. So the instruction
  stream, before optimization, was: `push`/`pop` (ours) immediately
  followed by `push`/`pop` (the second `push` ours, the `pop` `for`'s
  own). The optimizer caught *both* pairs — including the second one,
  which spans the exact boundary between TAL-inlined code and an
  ordinary compile-proc's own output — with no awareness that two
  different code generators were ever involved. That's about as direct a
  confirmation as this document has that inlined bytecode is genuinely
  fungible with everything around it, from the optimizer's point of view.

### Why the optimizer doesn't insert jumps over dead NOPs

A natural follow-up question: since `AdvanceJumps` already retargets
*existing* jumps around NOPs, why not also *insert* a new jump to skip
over a longer dead stretch, saving the dispatch cost of stepping through
each NOP individually?

Nothing rules this out structurally — a NOP costs one `TEBC` dispatch
trip just like any other instruction (chapter 6), so replacing several
consecutive NOPs with one jump instruction would, past a certain length,
be a net win. But the smallest jump encoding is itself several bytes, so
the win only materializes once a dead stretch is long enough to be worth
it, and deciding that would add real complexity: `ConvertZeroEffectToNOP`
currently does the simplest possible thing — a same-length overwrite,
no size or offset arithmetic required — where "sometimes insert a jump
instead" would need to reason about jump encoding size, whether a given
dead region is even large enough to hold one, and how that interacts
with `AdvanceJumps`' own retargeting pass. The implementation favors
that simplicity and provably-safe offset preservation over squeezing out
the last few dispatch cycles.

### DSL authors can still add their own pass — and it isn't redundant

Since Path A gets no cleanup at all, and Path B only gets whatever the
generic four-pass pipeline happens to recognize, neither path covers
optimizations specific to how one particular DSL generates its own TAL.
`:calc` is a concrete case: its TAL generator always produces a
pessimistic `push <name>; loadStk` pair for every variable reference —
safe whether or not an LVT is available — and a separate step then
rewrites that exact pattern, as plain text, before the TAL ever reaches
`assemble`:

```tcl
regsub -all {push ([[:alpha:]_][^:;\s]*); loadStk} $tal {load \1} tal
```

This collapses the pessimistic pair down to a single `load <name>`, but
only when the surrounding context is known in advance to have a real
LVT — a fact no generic bytecode-level pass could infer on its own,
since it depends on how `:calc`'s *own* generator behaves, not on any
pattern general Tcl bytecode exhibits. Chapter 6 covers the full
mechanism this is part of, and how `:calc` gets TAL onto Path A or
Path B in the first place.

---

## Chapter 6: Using `assemble` to get inlining (Path B) in practice

Chapters 3 and 5 established what the two paths are and how `assemble`
builds bytecode internally. This chapter is about the practical question
a DSL author actually faces: given that `assemble` supports both paths,
how do you arrange for your generated code to land on Path B — inlining
— rather than paying Path A's one-shot compile-and-run cost on every
single call?

### The mechanism is simpler than it looks: literalness alone decides it

Chapter 3 already showed this directly, in the very first live test run
against this document's own examples: a proc whose body is nothing but

```tcl
tcl::unsupported::assemble {push 1}
```

compiles down to just `push1 0` / `done` — the TAL is inlined
automatically, with no special API call, no registration step, and no
DSL machinery involved at all. The deciding factor is exactly what
chapter 3 already established for `TclCompileAssembleCmd`'s own
pre-attempt check: **is the TAL argument a compile-time-known literal.**

A literal argument gets inlined, automatically, by the same generic
mechanism that inlines any other compile-proc's output. A non-literal
argument (a variable, as in `assemble $tal`) can't be inlined at all —
the compile-proc can't even attempt it — and falls back to Path A.

This reframes what a DSL author's tooling actually needs to accomplish.
It isn't "how do I invoke inlining" — that's automatic and free the
moment the TAL is literal. It's **"how do I get my DSL's higher-level
syntax turned into literal TAL text, embedded directly in the generated
Tcl source, before that source is ever compiled."** That's a source-to-
source translation problem, not a runtime API problem.

`:calc` solves it with a group of procs this chapter will call **the
transformers**: `transform=`, and the three thin wrappers around it —
`proc=`, `method=`, and `calc=`. Their job, as a group, is to recognize
source text that would otherwise be an ordinary runtime call to `:calc`'s
`:` command (introduced below) and replace it, before compilation, with
a literal `assemble {...}` call — so that ordinary compilation inlines it
for free. Anything left as an actual, un-transformed call to `:` falls
back to being interpreted on every invocation instead.

### Worked example: `:calc`'s `transform=`

`transform=` walks a DSL-flavored script body line by line, recognizing
several syntactic forms for a calc statement (`= expr`, `: expr`, and
bracketed inline variants), and for each one (error handling not shown
for brevity):

```tcl
set tal [::Calc::compile0 $expr $inproc]
...
if {$inproc} {
    regsub -all {push ([[:alpha:]_][^:;\s]*); loadStk} $tal {load \1} tal
}
```

- `::Calc::compile0` translates the DSL expression into TAL text.
  `compile0` always generates the pessimistic, portable
  `push <name>; loadStk` form for every variable reference — safe
  whether or not an LVT will be available at the eventual call site.
  `compile0` also does a similar optimization on the store side,
  internally, collapsing a trailing `push <name>; ...; storeStk` down to
  `...; store <name>` when `$inproc` is true.
- The `regsub`, applied only when `$inproc` is true, is `:calc`'s own
  optimization pass (chapter 5) — collapsing that pessimistic pair down
  to a single `load <name>` wherever the destination is known in advance
  to have a real LVT. This is precisely the case chapter 5 pointed out no
  generic bytecode-level pass could safely perform on its own, since it
  depends on knowledge specific to `compile0`'s own generation pattern.

The resulting TAL is then spliced back into the generated source as a
literal `tcl::unsupported::assemble {...}` call — which is what actually
triggers inlining, per the mechanism above, the moment the surrounding
code is compiled.

### Three wrapper procs, three different destinations

`transform=` is the engine; three thin wrappers around it decide *where*
the transformed body ends up, and that choice is what actually
determines whether inlining pays off once or repeatedly:

- **`proc=`** calls `transform=` with `inproc 1` (LVT-optimized TAL), then
  `uplevel 1 [list proc $name $arglist $newbody]` — it defines a real,
  ordinary proc. The literal `assemble` calls just sit in that proc's
  body as source text. The first time the proc is ever called, its body
  compiles the normal way, and those literal calls inline automatically.
  Every later call reuses the same cached bytecode (chapter 3) — the
  translation and the inlining both happen exactly once, ever.
- **`method=`** is the identical structure for TclOO methods, registering
  via `uplevel 1 [list method $name $arglist $newbody]` instead.
- **`calc=`** is designed for top-level use, so it calls `transform=`
  with `inproc 0` instead — the only significant difference from
  `proc=`/`method=`.

If a top-level block wrapped by `calc=` contains a loop, the calc
statements inside that loop still only get translated and inlined once
— `calc=`'s `uplevel` compiles the whole block a single time, and the
loop then runs over that already-assembled bytecode on every iteration,
exactly as chapter 5's `for`-loop example showed. Repeated `uplevel`
calls to `calc=` itself don't share cached bytecode with each other, but
if the code in question only runs once anyway, that distinction doesn't
much matter in practice.

### A second route to the same destination, without the wrapper at all

Since inlining only ever depends on the TAL being literal at compile
time, nothing requires a special defining command like `proc=` in the
first place. An ordinary, already-written proc can be retrofitted after
the fact: reconstruct its argument list (`info args`, `info default` for
each argument's default value) and body (`info body`), run that body
through the same `::Calc::transform= $arglist $body 1 $preserve` call
`proc=` itself uses, and redefine the proc in place with the transformed
result — all before the proc is ever called for the first time. This
gets identical inlining behavior to `proc=`, via the identical
underlying mechanism, without the DSL needing to expose its own special
proc-defining command at all.

### The baseline case: an un-transformed call to `:`

Not every piece of `:calc`'s history took the transformer route, and
seeing what came before them is instructive. `:` — the DSL's short
command-name front end — is an ordinary runtime command, not a special
defining wrapper, and calc-DSL source written directly as `: expr`,
without ever passing through a transformer, is genuinely calling it at
runtime, every time that line executes:

```tcl
proc : {arg args} {
    if { [llength $args] != 0 } {
        set arg [join "$arg $args"]
    }
    if {[info exist ::Calc::cache($arg)]} {
        tailcall ::tcl::unsupported::assemble $::Calc::cache($arg)
    }
    tailcall ::tcl::unsupported::assemble [set ::Calc::cache($arg) [::Calc::compile0 $arg]]
}
```

Every call to `:` looks up a *global* cache, keyed on the raw DSL
statement text, and tail-calls `assemble` with whatever TAL it finds or
just produced. Because that TAL is held in a variable
(`$::Calc::cache($arg)`), this is unconditionally Path A — `assemble`'s
compile-proc can't inline a non-literal argument, so every single
un-transformed call to `:` re-enters `assemble`'s real runtime command,
every time. The cache means the DSL-to-TAL translation step
(`compile0`) only ever happens once per distinct statement — but
`assemble`'s own dispatch and execution still happen on every call. This
is exactly the gap the transformers close: recognizing this same source
text ahead of time and replacing it with a literal `assemble` call, so
that translation and inlining both happen once, at compile time, instead
of translation being cached but dispatch repeating forever.

`:`'s TAL never uses the LVT-optimized `load`/`store` form, for two
reasons. Chronologically, the load/store optimization wasn't understood
yet when `:` was first designed — that came later, with `proc=`. And
once it was understood, nobody went back to retrofit it, because `:`
genuinely has no way to know whether a given call originates from inside
a proc or from top level. `proc=`/`method=` can pass `inproc 1` to
`compile0` precisely because *they* know, by construction, that the code
they're generating is destined for a proc or method body; `:` calls
`::Calc::compile0 $arg` with a single argument, so it just falls through
to `compile0`'s own default —
`proc compile0 {exp {inproc 0}}` — the portable form, unconditionally.
The load/store optimization exists only inside `transform=`, applied via
`proc=`/`method=` — a code path `:` never goes through at all — which is
exactly why `:`'s cache is safe to share across arbitrary call sites: it
can never end up holding an LVT-relative instruction in the first place.

And because the DSL only allows bare variable
names (no `$var` substitution) in its statements, the same statement
text always arrives at `:` unchanged regardless of what any variable
currently holds — without that discipline, a statement like `a = $b + 5`
would produce a distinct, permanent cache entry for every value `b` ever
took, exactly the runaway-cache failure mode the discipline exists to
prevent. `compile0` also enforces a hard size limit on the cache as a
backstop, in case that discipline is ever violated. In the latest version,
`compile0` removes any `$` it finds - this causes only barename variables
to remain in the expression, and doesn't change the semantics of the expression.

The cache's original purpose was narrower than inlining, worth stating
precisely: it exists to avoid re-running `compile0` — the DSL-to-TAL
translation — which measured roughly 20x slower than a plain array
lookup when redone on every call, in pure Tcl. It was never expected to
avoid re-running `assemble` itself; that only became possible once
inlining was understood as a separate, later development. 

`:calc` also went through a C-extension implementation of this same cache, motivated
by C being able to do a find lookup in one operation where Tcl
needs two (`info exists` then if true, get the value), plus faster string handling for
the lookup itself — a real, specific micro-optimization of the cache
mechanism as it stood, not a structural change to what was being cached.

### Two different kinds of improvement

The arc from the original Tcl-level cache, through the C-extension
version, to `proc=`-based inlining is worth naming as two genuinely
different categories of improvement, not one continuous line of "getting
faster":

- **Optimizing within a fixed architecture.** The C extension made the
  *same* operation — runtime cache lookup, then assemble, then execute —
  cheaper per call. This is real, valuable engineering, but it doesn't
  change what's fundamentally happening on every call.
- **Removing the operation entirely, for the common case.** Once
  `proc=`/inlining was understood, there is no runtime lookup left to
  optimize at all — the DSL-to-bytecode translation happened once, at
  definition time, and every subsequent call just runs the already-
  compiled instructions directly. The C extension became unnecessary not
  because it got outcompeted by a faster cache, but because the entire
  category of problem it was solving stopped applying to the common case
  it was written for.

`:`'s cache remains genuinely useful for what it was actually built for
— dynamic, top-level, or otherwise-not-proc-shaped uses of the DSL, where
there is no proc body available to inline into in the first place. It
was the wrong tool only for the case `proc=` now covers directly.

`:` as an interpretive version also has the benefit of being debugged with a
tool like tclPro debugger that instruments the original source code and so
remains available for the user until a compiled fast version is needed.

---

## Appendix 1: Worked walkthrough: `TclCompileForCmd` in full

Chapter 4 introduced `TclCompileForCmd` at the structural level — its
three-part shape, and why `for` was chosen as the ordinary-but-non-trivial
example instead of something purely straight-line. This appendix walks
the whole function in order, against the numbered listing in chapter 4.

### The bail-out checks (lines 30–52)

The word-count check (line 30) and the three `TCL_TOKEN_SIMPLE_WORD`
checks (lines 40, 49–50) are chapter 4's territory in full — what
triggers them, and why `for` declines rather than guesses, is covered
there. Nothing more to add here beyond noting where they sit: everything
from line 53 onward only runs once all four checks have already passed,
so the rest of this walkthrough can assume `startTokenPtr`, `testTokenPtr`,
`nextTokenPtr`, and `bodyTokenPtr` are all known-good literal tokens.

### The start command (lines 56–57)

```c
BODY(startTokenPtr, 1);
OP(POP);
```

`start` is compiled exactly once, unconditionally, before any loop
structure exists at all — it's ordinary sequential code, not part of the
loop. `OP(POP)` discards its result immediately, since `for`'s own result
is fixed elsewhere (line 113), not derived from `start`.

### Loop rotation (lines 58–69)

The comment block here is the one this document has already spent real
time on: the jump ordering — `start` → jump to test → body → next → test
→ conditional jump back — trades one unconditional jump, paid once, for
removing a jump from every subsequent iteration. This matters more here
than it would in compiled native code, because every bytecode
instruction, jump included, costs a full trip through the `TEBC`
dispatch loop — there's no hardware branch predictor absorbing the cost
of a taken jump the way there is at the CPU level. `FWDJUMP(JUMP,
testCondition)` on line 69 is that one paid-once jump, landing on the
test (line 97) the first time through, before the loop body has run at
all.

### The loop body, in its own catch range (lines 70–78)

```c
bodyRange = MAKE_LOOP_RANGE();
BACKLABEL(evalBody);
CATCH_RANGE(bodyRange) {
    BODY(bodyTokenPtr, 4);
}
OP(POP);
```

`MAKE_LOOP_RANGE` creates a `LOOP_EXCEPTION_RANGE` entry — the kind that
intercepts `TCL_BREAK` and `TCL_CONTINUE` (chapter 3's material on how an
exception range is reached without any bytecode jump instruction
pointing at it applies directly here). `BACKLABEL(evalBody)` records the
current byte offset as the destination the loop's own back-edge
(line 99) will jump to — this is "B" in the loop-rotation diagram.

`CATCH_RANGE(bodyRange) { ... }` is worth flagging on its own: it's not
an actual loop. `CATCH_RANGE` expands to a single-iteration C `for`,
using the comma operator to run setup (marking the exception range's
start) before the block and teardown (marking its end) after, with the
loop variable itself never used for iteration — just as a scope guard
ensuring the start/end bracketing always happens together, even if the
block inside contains an early `return`. Reading it as an actual
repeating loop is an easy misread on first exposure to this macro
family.

`OP(POP)` after the block discards whatever the body left on the stack —
the same unconditional cleanup chapter 5's stack-depth material already
covered in detail (this is exactly the `pop` whose necessity, absence,
and interaction with the peephole optimizer chapter 5 worked through at
length).

### The `next` subcommand, in its own catch range (lines 79–92)

```c
CONTINUE_TARGET(bodyRange);
if (!TclIsEmptyToken(nextTokenPtr)) {
    nextRange = MAKE_LOOP_RANGE();
    envPtr->exceptAuxArrayPtr[nextRange].supportsContinue = 0;
    CATCH_RANGE(nextRange) {
        BODY(nextTokenPtr, 3);
    }
    OP(POP);
}
```

`CONTINUE_TARGET(bodyRange)` marks this exact point — right after the
body, right before `next` — as where a `continue` inside the body lands.
`next` then gets compiled inside a *second*, separate exception range,
not folded into the body's own range. The reason is in the source
comment: this second range's `supportsContinue` is explicitly set to 0
— a `continue` occurring *while `next` itself is running* isn't given
special handling; only `TCL_BREAK` is trapped here. A `continue` there
simply propagates further outward, exactly like an uncaught error would
— caught by whatever loop encloses this one, if any, or raising
`invoked "continue" outside of a loop"` if there isn't one. Two loop
ranges exist specifically because `break` and `continue` need to behave
differently depending on whether they occur during the body or during
the increment step, and one shared range couldn't express that
distinction.

The `if (!TclIsEmptyToken(nextTokenPtr))` guard means an empty `next`
token (`for {...} {...} {} {...}`) skips this whole block — no wasted
exception range or catch machinery for a step that does nothing.

### The test, and the back-edge (lines 93–99)

```c
FWDLABEL(testCondition);
PUSH_EXPR_TOKEN(testTokenPtr, 2);
BACKJUMP(JUMP_TRUE, evalBody);
```

`FWDLABEL(testCondition)` resolves the forward jump from line 69 — this
is where execution actually lands the first time through, and where it
falls to naturally every iteration thereafter, right after `next`.
`PUSH_EXPR_TOKEN` is the identical macro `if`'s condition compiles
through (chapter 6's `PUSH_EXPR_TOKEN` discussion applies here without
modification — `for`'s test and `if`'s condition are compiled by the
same shared expression machinery). `BACKJUMP(JUMP_TRUE, evalBody)` is
the loop's only per-iteration jump — back to "B," the body — closing the
rotated structure.

### Finalizing the exception ranges (lines 100–109)

```c
BREAK_TARGET(bodyRange);
FINALIZE_LOOP(bodyRange);
if (nextRange != -1) {
    BREAK_TARGET(nextRange);
    FINALIZE_LOOP(nextRange);
}
```

`BREAK_TARGET` marks where a `break` — from either range — actually
lands: right after the loop, past the back-edge, at the point the loop
would naturally exit anyway. `FINALIZE_LOOP` closes out the bookkeeping
for each range once its start, end, and targets are all fixed. The
`nextRange != -1` guard mirrors the earlier `TclIsEmptyToken` check —
nothing to finalize if `next` was empty and no second range was ever
created.

### The result (lines 110–114)

```c
PUSH("");
return TCL_OK;
```

`for` always evaluates to the empty string, unconditionally, regardless
of how the loop terminated (test failed, or `break` was hit) — this
final push is unconditional and sits outside every exception range, so
nothing about loop termination affects it.

### Where the peephole optimizer shows up in this example's own disassembly

Two real disassembly traces from earlier in this document make the
connection to chapter 5 concrete. `for {set i 0} {$i < $n} {incr i} {}`
— an empty body — showed a run of NOPs exactly where the dead
`push ""`/`pop` from the empty body sat (the body still has to evaluate
to *something*, per Tcl's empty-script-evaluates-to-empty-string rule,
and `OP(POP)` on line 78 immediately discards it), plus a second,
unconditional run of NOPs at the very end, from `for`'s own line 113
`push ""` colliding with the *enclosing* script's own implicit `pop`
(since `for` wasn't the last command in that particular test proc).
The second version, with a real body (`{puts $i}`), showed the *first*
NOP run vanish entirely — real code, no longer dead — while the second,
unconditional run remained identical in both traces, exactly as this
appendix's account of lines 110–114 predicts: that push is unconditional
and has nothing to do with what the body contains.

### The payoff

Every macro family chapter 2 catalogued shows up somewhere in this one
function: the plain shorthand macros (`OP`, `BODY`), the jump macros
(`FWDJUMP`, `BACKLABEL`, `BACKJUMP`, `FWDLABEL`), and the full
exception-range family (`MAKE_LOOP_RANGE`, `CATCH_RANGE`,
`CONTINUE_TARGET`, `BREAK_TARGET`, `FINALIZE_LOOP`). Chapter 5's
`BasicBlock` graph, built for arbitrary TAL, is solving the identical
kind of problem `TclCompileForCmd` solves by hand here, for one fixed
control-flow shape — which is exactly why chapter 4 introduced `for` in
the first place, as the ordinary compile-proc that comes closest to
needing what `assemble` has to provide generally.

---

## Appendix 2: Reference material

### Existing documentation

What exists elsewhere on this material, with honest notes on how
current and complete each source actually is — this is what chapter 1
described as scattered rather than coherent, and this is the scatter:

- **The wiki hub**, at `wiki.tcl-lang.org/page/bytecode`, links to a set
  of sub-pages including "Parsing, Bytecodes and Execution," "Proc to
  bytecodes: when, how does it happen," "The anatomy of a bytecoded
  command," "Commands affecting Bytecoding," and a disassembly page.
  These sound directly relevant to several chapters of this document —
  particularly the proc-to-bytecodes page, given chapters 3 and 6's
  material — but this document was built primarily from direct source
  reading and live experimentation rather than from these pages, so
  their current accuracy and depth relative to what's written here
  hasn't been verified.
- **The original bytecode-compiler paper** (Ousterhout et al., the
  USENIX paper describing Tcl's on-the-fly bytecode compiler) is the
  true origin of `CompileEnv`, dual-ported objects, and the literal
  table — genuinely foundational, but describing a design roughly three
  decades old, predating `assemble` entirely. Useful for the origin of
  concepts this document takes for granted; not a source for anything
  `assemble`-specific.
- **`tclVM`** is a wiki-documented introspection tool (`compile`,
  `disasm`, `literals`, `instTable`) — a debugging aid for poking at
  compiled objects, not a tutorial or reference in its own right.
- **`:/Calc`** the calc module discussed above https://github.com/rocketship88/colin-parser

None of these ties `assemble` into the wider bytecode-compiler picture
the way this document attempts to. That gap is this document's actual
reason for existing, per chapter 1.

### Glossary

- **`CompileEnv`** — the C structure that owns a bytecode buffer, literal
  table, exception-range list, and stack-depth accounting for whatever
  is currently being compiled (chapter 2). Every compile-proc, and
  `assemble` itself, writes into one.
- **`ByteCode`** — the finished, executable object a `CompileEnv` is
  converted into once compilation completes (chapter 3). What a proc's
  body caches after its first compile; what Path A produces standalone.
- **LVT (local variable table)** — the compiled-locals slot table a proc
  or method body has, letting variable references resolve to a fixed
  numeric index instead of a runtime name lookup. `load`/`store` (TAL)
  and `loadScalar`/`storeScalar` (ordinary bytecode) both depend on one
  existing; `push <name>; loadStk`/`storeStk` are the name-based
  fallback when it doesn't (chapters 2, 5, 6).
- **`TalInstType`** — the small (~20-value) enum categorizing a TAL
  mnemonic's operand *shape* (no operand, an LVT index, a jump offset,
  and so on), distinct from the mnemonic itself and from the real
  `INST_*` opcode it maps to. What `AssembleOneLine` actually dispatches
  on (chapter 5).
- **`BasicBlock`** — a maximal run of instructions with one entry point
  and one exit; the unit `assemble` builds its internal graph from,
  used for stack-depth consistency checking and jump-offset resolution
  (chapter 5, Appendix 1).
- **`ExceptionRange`** — the side-table entry (not a bytecode
  instruction) that lets `TEBC` intercept a non-`TCL_OK` completion code
  occurring within a given PC range and redirect control accordingly —
  what `catch`, loop `break`/`continue` targets, and `for`'s own two
  separate loop ranges (chapter 4, Appendix 1) are all built from.

### Quick reference: the two emit-macro layers (chapter 2)

| Raw primitive | Shorthand built on it | What it does |
|---|---|---|
| `TclEmitOpcode` | `OP` | Opcode, no operand |
| `TclEmitInstInt1` | `OP1` | Opcode + 1-byte operand |
| `TclEmitInstInt4` | `OP4` | Opcode + 4-byte operand |
| `TclEmitInstInt14` | `OP14` | Opcode + 1-byte operand + 4-byte operand |
| `TclEmitInstInt44` | `OP44` | Opcode + two 4-byte operands |
| `TclEmitInstInt41` | `OP41` | Opcode + 4-byte operand + 1-byte operand |
| `TclEmitInvoke` | `INVOKE` / `INVOKE4` / `INVOKE41` | A break/continue-capable instruction, 0–2 args |
| `TclEmitPush` | `PUSH` / `PUSH_STRING` / `PUSH_SIMPLE_TOKEN` / `PUSH_OBJ` / `PUSH_OBJ_FLAGS` / `PUSH_TOKEN` | Push a value from a literal, token, or `Tcl_Obj` |
| *(compiles a sub-expression)* | `PUSH_EXPR_TOKEN` | Compile and push an expression token — shared by `for`'s test and `if`'s condition |
| *(compiles a sub-script)* | `BODY` | Compile a command's body (e.g. `if`, `while`) |
| — | `BACKLABEL` / `BACKJUMP` | Mark and emit a backward jump |
| — | `FWDLABEL` / `FWDJUMP` | Mark and emit a forward jump |
| — | `MAKE_CATCH_RANGE` / `MAKE_LOOP_RANGE` | Create an exception range |
| — | `CATCH_RANGE` | Scope-guard macro bracketing a range's start/end (not a real loop) |
| — | `CATCH_TARGET` / `BREAK_TARGET` / `CONTINUE_TARGET` | Mark where a caught exception/break/continue lands |
| — | `FINALIZE_LOOP` | Close out a loop range's bookkeeping |
| `TclAdjustStackDepth` | `STKDELTA` | Manually correct tracked stack depth |

Every entry marked `—` has no single raw-layer equivalent; the
shorthand macro composes more than one raw operation itself.
