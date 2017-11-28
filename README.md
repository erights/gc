# Wasm-linkage:<br>A subset of wasm-gc without dynamic allocation or gc

This repository is a clone of [github.com/WebAssembly/gc/](https://github.com/WebAssembly/gc/)
which is a clone of [github.com/WebAssembly/spec/](https://github.com/WebAssembly/spec/).

The wasm-linkage we propose here is a stepping stone towards wasm-gc that solves several important problems, but does not yet introduce any need for dynamic allocation of garbage collection. We intend wasm-linkage to be a superset of current wasm and a subset of the full wasm-gc.

See the [overview](https://github.com/erights/gc/blob/master/proposals/gc/Overview.md) for a summary of the wasm-linkage proposal.

The claims:
   * Wasm-linkage enables modular inter-compartment linkage in an ocap-safe and
     ocap-expressive manner.
   * Wasm-linkage is a superset of (current) wasm and a subset of wasm-gc
   * Wasm-linkage does not create any need for dynamic allocation or
     garbage collection.
   * Wasm-linkage provides the core mechanism needed for host bindings,
     hiding the differences between host functions and functions from other
     compartments. From the perspective of wasm code, the host is as-if
     just another compartment.
   * Wasm-linkage is not excessively stateful in the sense raised in the 
     [WASM and ocap](https://groups.google.com/forum/#!topic/e-lang/3A6zYWF6u5E) thread.
   * The ***current*** wasm as enhanced by wasm-linkage enables defensively consistent modules
     at reasonable effort. The qualification is that currently, the only observable 
     shared state concurrency is via shared array buffers. The case for defensive consistency relies on that.

Non-claims:

   * Unlike [first inter-compartment linkage proposal](https://groups.google.com/d/msg/e-lang/3A6zYWF6u5E/KH2Jf39fBgAJ) or [Counter-proposals #1 and #2](https://groups.google.com/d/msg/e-lang/3A6zYWF6u5E/2RpcpcviBgAJ) this wasm-linkage
one does not enable passing slices or either memory or tables.
   * As discussed in the [WASM and ocap](https://groups.google.com/forum/#!topic/e-lang/3A6zYWF6u5E) thread, depending on how concurrency is introduced, wasm modules may no longer be able to be defensively consistent at reasonable effort. The key question is whether module instance A may enter module B in thread T2 while B is already executing in T1. This is the pervasive shared state concurrency of Java, which has proven incompatible with defensive programming.

Background:
   * [Wasm itself](https://github.com/WebAssembly/spec/) of course
   * [The wasm-gc proposal](https://github.com/WebAssembly/gc/blob/master/proposals/gc/Overview.md)
   * The [host bindings proposal](https://github.com/WebAssembly/host-bindings/blob/master/proposals/host-bindings/Overview.md) ([slides](https://docs.google.com/presentation/d/10vz6pldVOA8N3guv2jf4DCUujqz6jFmDnp37ax4SCc0/edit?usp=sharing)).
   * The [first inter-compartment linkage proposal](https://groups.google.com/d/msg/e-lang/3A6zYWF6u5E/KH2Jf39fBgAJ)
     of the [WASM and ocap](https://groups.google.com/forum/#!topic/e-lang/3A6zYWF6u5E) thread.
   * [Counter-proposals #1 and #2](https://groups.google.com/d/msg/e-lang/3A6zYWF6u5E/2RpcpcviBgAJ)
