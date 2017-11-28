# Wasm-linkage:<br>A subset of wasm-gc without dynamic allocation or gc

This repository is a clone of [github.com/WebAssembly/gc/](https://github.com/WebAssembly/gc/)
which is a clone of [github.com/WebAssembly/spec/](https://github.com/WebAssembly/spec/).

The wasm-linkage we propose here is a stepping stone towards wasm-gc that solves several important problems, but does not yet introduce any need for dynamic allocation of garbage collection. We intend wasm-linkage to be a superset of current wasm and a subset of the full wasm-gc.

See the [overview](https://github.com/erights/wasm-linkage/blob/master/proposals/gc/Overview.md) for a summary of the wasm-linkage proposal.
