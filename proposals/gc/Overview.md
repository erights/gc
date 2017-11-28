# GC Extension

## Introduction

### Motivation
   * Enable modular inter-compartment linkage in an ocap-safe and
     ocap-expressive manner.
   * Be a superset of (current) wasm and a subset of wasm-gc
   * Do not create any need for dynamic allocation or
     garbage collection.
   * Provide the core mechanism needed for host bindings,
     hiding the differences between host functions and functions from other
     compartments. From the perspective of wasm code, the host is as-if
     just another compartment.
   * Omit needless statefulness, in the sense raised in the 
     [WASM and ocap](https://groups.google.com/forum/#!topic/e-lang/3A6zYWF6u5E) thread.
   * Enable defensively consistent modules
     at reasonable effort, given the current wasm and wasm-gc limits on shared state concurrency.
   * Little overhead compared to current intra-compartment practice for passing function pointers,
     such as passing three words on the stack per function pointer, rather than one. This enables
     compilers to use this call mechanism for dynamic calls in general without making a special case for
     intra-compartment calls.

### Non-goals (of wasm-gc goals)
   * Efficient support for high-level languages
      - faster execution
      - smaller deliverables
      - the vast majority of modern languages need it
   * Efficient interoperability with embedder heap
      - for example, DOM objects on the web
      - no space leaks from mapping bidirectional tables
   * Provide access to industrial-strength GCs
      - at least on the web, VMs already have high performance GCs

### Non-goals (of wasm-gc non-goals)

* Seamless interoperability between multiple languages

### Non-goals (at the moment. Would be nice to have)

   * Unlike the [first inter-compartment linkage proposal](https://groups.google.com/d/msg/e-lang/3A6zYWF6u5E/KH2Jf39fBgAJ) or [Counter-proposals #1 and #2](https://groups.google.com/d/msg/e-lang/3A6zYWF6u5E/2RpcpcviBgAJ) this wasm-linkage
proposal does not enable passing slices of either memory or tables.
   * As discussed in the [WASM and ocap](https://groups.google.com/forum/#!topic/e-lang/3A6zYWF6u5E) thread, depending on how concurrency is introduced, wasm modules may no longer be able to be defensively consistent at reasonable effort. The key question is whether module instance A may enter module B in thread T2 while B is already executing in T1. This is the pervasive shared state concurrency of Java, which has proven incompatible with defensive programming.
   * abstract opaque data types, to type non-function arguments to typed functions.
   * dynamic type testing beyond that already in wasm.
   * subtyping.
   * pass-by-copy aggregate types.

### Background

   * [Wasm itself](https://github.com/WebAssembly/spec/) of course
   * [The wasm-gc proposal](https://github.com/WebAssembly/gc/blob/master/proposals/gc/Overview.md)
   * The [host bindings proposal](https://github.com/WebAssembly/host-bindings/blob/master/proposals/host-bindings/Overview.md) ([slides](https://docs.google.com/presentation/d/10vz6pldVOA8N3guv2jf4DCUujqz6jFmDnp37ax4SCc0/edit?usp=sharing)).
   * The [first inter-compartment linkage proposal](https://groups.google.com/d/msg/e-lang/3A6zYWF6u5E/KH2Jf39fBgAJ)
     of the [WASM and ocap](https://groups.google.com/forum/#!topic/e-lang/3A6zYWF6u5E) thread.
   * [Counter-proposals #1 and #2](https://groups.google.com/d/msg/e-lang/3A6zYWF6u5E/2RpcpcviBgAJ)

### Approach

* Introduce only the typed-function and ref-to-typed-function types from wasm-gc.
* Allow ref-to-typed-function to be passed in parameters and stored in local variables.
* Introduce the `call_ref` opcode from wasm-gc
* Allow new tables (as the wasm spec already anticipates) where each new table is typed
  as a uniform array of refs-to-typed function.
* Provide instructions for loading and storing between table entries and stack variables.
* Add a new ref-to-typed-closure that is like a ref-to-typed-function but with an additional
  field that enables the code of the creating compartment to create an indivisble
  closure using its own memory manager, which wasm remains ignorant of.
* The ref-to-typed-closure is passed by copy just as ref-to-typed function is.
* The ref-to-typed-closure is likely to be a subset of the anticipated
  but not-yet-speced ref-to-typed-closure from wasm-gc.

### An Implementation Approach

The concrete representation of a typed-ref-to-closure is a three word record passed by copy
in parameter and local variables and in typed table entries. We depend on the type safety of
wasm to keep this record opaque and indivisible. The three words are:
   * A pointer to the machine code of the function
   * A pointer to the module instance containing this function
   * An i32 (perhaps i64) facet id that the code in the containing compartment can use, if it
     wishes, to implement closures in the memory it manages.
     
On invocation, the pointed-to-machine-code is given access to 
   * the pointer to the module instance
   * the facet id
   * whatever arguments were passed, according to the type of the function.

The passing of references-to-closures on the stack parallels the passing of numbers on the stack. If a function wishes to remember a passed number after the function returns, it writes it somewhere in memory, according to whatever the memory management discipline is of that language in that compartment. If a function wishes to remember a passed ref-to-closure after the function returns, it writes it somewhere into a table of functions of that type, according to whatever the table-memory management discipline is of that language in that compartment.

If the memory management within a compartment goes haywire, it fouls its own nest --- it destroys the integrity of its own compartment. But it does not threaten the integrity of defensively consistent compartments that it interacts with.


### Efficiency Considerations

Wasm-linkage support should maintain Wasm's efficiency properties as much as possible, namely:

* all operations are very cheap, ideally constant time,
* structures are contiguous, dense chunks of memory,
* accessing fields are single-indirection loads and stores,
* no implicit boxing operations (i.e. no implicit allocation on the heap),
* primitive values should not need to be boxed to be stored in managed data structures,
* allows ahead-of-time compilation and code caching.


### Evaluation

* Demonstrate that the Alice-Bob-Carol example works.
* Demonstrate the variant with chained delegation.
* Write a trivial defensively consistent module and try to attack it from other modules.

## Use Cases


### Closures

* Want to associate a code pointer and its "environment" in a pass-by-copy reference-to-typed-closure
* Want to be able to allow compiler of source language to choose appropriate environment representation
  including the same freedom of manual memory management that current compilers to plain wasm have.

Needs:
* function pointers
* (mutually) recursive function types


### Type Export/Import

* Want to allow type definitions to be imported from other modules.


### Function References

References can also be formed to function types, thereby introducing the notion of _typed function pointer_.

Function references can be called through `call_ref` instruction:
```
(type $t (func (param i32))

(func $f (param $x (ref $t))
  (call_ref (i32.const 5) (get_local $x))
)
```
Unlike `call_indirect`, this instruction is statically typed and does not involve any runtime check.

Values of function reference type are formed with the `ref_func` operator:
```
(func $g (param $x (ref $t))
  (call $f (ref_func $h))
)

(func $h (param i32) ...)
```


## Type Structure

### Type Grammar

The type syntax can be captured in the following grammar:
```
num_type       ::=  i32 | i64 | f32 | f64
ref_type       ::=  (ref <def_type>)
value_type     ::=  <num_type> | <ref_type>

func_type      ::=  (func <value_type>* <value_type>*)
def_type       ::=  <func_type>
```
where `value_type` is the type usable for parameters, local variables and the operand stack, and `def_type` describes the types that can be defined in the type section.


### Type Recursion

Through references, function types can be *recursive*.

The [type grammar](#type-grammar) does not make recursion explicit. Semantically, it is assumed that types can be infinite regular trees by expanding all references in the type section, as is standard.
Folding that into a finite representation (such as a graph) is an implementation concern.


### Type Equivalence

In order to avoid spurious type incompatibilities at module boundaries,
all types are structural.
Aggregate types are considered equivalent when the unfoldings of their definitions are.

Note: This is the standard definition of recursive structural equivalence for "equi-recursive" types.
Checking it is computationally equivalent to checking whether two FSAs are equivalent, i.e., it is a non-trivial algorithm (even though most practical cases will be trivial).
This may be a problem, in which case we need to fall back to a more restrictive definition, although it is unclear what exactly that would be.

