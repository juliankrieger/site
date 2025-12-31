---
title: Annoying your Rust binarie's reverse engineer
date: '2025-07-07'
draft: true
---

- turn off println / stack unwinding, because the IDA toolset from Intezer and Sentinel One can find the original file from debugger information included in the macro for unwinding
- look into BYREFs potentially from closures? -> Write a lot of closures lol (25:29)
- Above may also be the cause of &mut
- what is the decompilation status for &dyn impls (each with vtable lol)
- a lot of the work in RIFT is done via compiler metadata that identifies its toolchain and hash as well as crates.
- what about SYN heavy code?
