# TIP 760: Supporting `assemble` 

This repository supports [TIP 760](https://core.tcl-lang.org/tips/doc/trunk/tip/760.md),
which proposes promoting `tcl::unsupported::assemble` to a fully supported command.

## Contents

- **`tip-760.md`** — the TIP text itself is simply linked above.
- **`assemble-dual-path-tutorial.md`** — a technical document explaining
  how `assemble` works internally, how it fits into Tcl's bytecode
  compiler, and how to use it effectively — including the dual-path
  (direct vs. inlined) compilation model, the peephole optimizer's
  interaction with assembled code, and worked examples from a real
  DSL built on top of it.

## Why this exists

`assemble` is the load-bearing tool for anyone writing a
fast Tcl-level DSL, but the mechanics of how it works — and how to get
the most out of it — aren't documented anywhere coherent. The
tutorial in this repository is an attempt to close that gap, developed
through direct source reading and hands-on experimentation with a real
Tcl build constructed by using Claude.ai.

## License

See [LICENSE](LICENSE).
