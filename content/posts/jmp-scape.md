+++
title = "Rust bindings for setjmp"
date = "2026-04-10"

[taxonomies]
tags=["rust", "asm", "setjmp", "system-programming"]
+++

About a year ago, I was in talks with a certain engineer at AWS to take over and/or contribute to [cee-scape](https://github.com/pnkfelix/cee-scape). The reason being,
the original developer had left AWS and apparently it was not clear if the project was being actively maintained or not.

I didn't get a chance to do much there in this regard, and soon, [I too left AWS](https://cafeaffe.substack.com/p/goodbye-amazon).

Recently, I started looking into low level stuff all over again, starting with the [`RISC-V` reader](http://www.riscvbook.com/). And that reminded me of `cee-scape`.

Turns out, the PRs were still pending, and not much updated in last couple of years.

So, I created a fork of it. Tried to understand what and how. And updated a few things. I will work on supporting RISC-V too. Meanwhile, I released this as a different
crate [`jmp-scape`](https://crates.io/crates/jmp-scape) with the existing license unchanged.

Almost 100% of the credit goes to the original authors. I am just building over the strong foundational setup done by them. It has been more of a personal learning project for me. 
If anyone uses it, and stumbles across this, feel free to drop me a message.