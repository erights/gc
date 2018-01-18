# Wasm-linkage:<br>A subset of wasm-gc without dynamic allocation or gc

This repository is a clone of [github.com/WebAssembly/gc/](https://github.com/WebAssembly/gc/)
which is a clone of [github.com/WebAssembly/spec/](https://github.com/WebAssembly/spec/).

The wasm-linkage we propose here is a stepping stone towards wasm-gc that solves several important problems, but does not yet introduce any need for dynamic allocation of garbage collection. We intend wasm-linkage to be a superset of current wasm and a subset of the full wasm-gc.

See the [baseline](https://github.com/erights/wasm-linkage/blob/master/proposals/wasm-linkage/Baseline.md) for a summary of the baseline wasm-linkage proposal. This baseline introduces only the refs-to-typed functions from wasm-gc, solving some important problems.

This baseline solves a special case of a more general problem, motivating extensions of the baseline proposal:

Today, wasm allows you to freely address your memory in your own [wasm compartment](https://github.com/erights/wasm-linkage/blob/master/proposals/wasm-linkage/Baseline.md#instances-vs-compartments). But there is no built-in, direct way to address memory of another wasm compartment. These proposals create new kind of opaque "fat pointer" that allow one wasm compartment to directly refer to a value in another wasm compartment. These fat pointers can be freely passed around, but only code in the the wasm compartment that created the fat pointer can see what it points at. Additionally, JavaScript values can be turned into these kinds of opaque pointers, and are always opaque in all wasm compartments.

Talk: [Wasm as an OCap Machine](wasm-ocap.pdf)

The extensions (TODO):
   * [Unmanaged Closures](UnmanagedClosures.md)
   * [Memory-range capabilities](MemCaps.md)
   * [Table-range capabilities](TableCaps.md)
   * [General fat pointers](FatPointers.md)
   * [Unmanaged Fat Closures](UnmanagedFatClosures.md)
   * [Host bindings](HostBindings.md)
   * [wasm-ocap](WasmOCap.md)
