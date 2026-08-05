+++
title = "jmp-scape now enabled for RISC-V"
date = "2026-08-05"

[taxonomies]
tags=["rust", "riscv", "setjmp", "jmp-scape", "system-programming"]
+++

**[jmp-scape](https://crates.io/crates/jmp-scape)** - a Rust wrapper around `setjmp` / `sigsetjmp` that keeps LLVM from miscompiling non-local jumps, now supports **Linux on riscv64**. That matters for embedded boards, RISC-V servers, and anyone cross-compiling to `riscv64gc-unknown-linux-gnu` who needs safe Rust↔C FFI around `longjmp`.
