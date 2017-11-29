# Wasm-linkage Baseline Proposal

## Introduction

We first present a baseline proposal for wasm-linkage that introduces only pass-by-copy opqaue references-to-typed-functions as inter-compartment object-capabilities. This baseline system is 
   * a superset of current wasm
   * a subset of wasm-gc as currently proposed
   * does not need any dynamic allocation or collection
   * repairs wasm's current linkage confusion at only marginal expense
   * enables linked defensive wasm compartments
   * turns wasm into a formally adequate ocap machine
   * provides the core mechanism needed for host bindings
   
However, this baseline by itself has various practical problems. Separate pages (TODO) then explore various extensions of this baseline for addressing these practical problems, while still avoiding dynamic allocation or garbage collection. Some of these extensions will be outside wasm-gc as currently proposed, but only make sense if future wasm-gc adopts these same extensions, and so remains a superset.

### Motivations

* Modular linkage

A wasm function (i.e., the wasm value that wasm code can invoke) is currently speced to take only numbers as arguments and to return only numbers as results. It may at first seem puzzling why this is sufficient, given that wasm is the target of compilation from languages in which functions can be passed as parameters to functions. The answer is that, in the dominant pattern of use, both calling function `f` and called function `g` are assumed to be share the same indexable spaces, so a table index that `f` uses to refer to a function `h1` can be used by `g` to refer to the same function `h1`, by virtue of indexing into the same table.

If `f` and `g` are in the same instance, then this assumption is necessarily true. However, many uses of wasm cross module instance boundaries: `g` might have been exported from `g`'s module `G` and imported into `f`'s module `F`. In this case, the assumption does not necessarily hold. The index `f` uses to index into `F`'s tables to designate `h1`, if transmitted to `g` and used by `g` to index into `G`'s tables, might designate instead completely unrelated function `h2`.

Language compilers targeting wasm may map intermodule linkage of their source language to inter-instance linkage of their wasm target. They do not encounter this confusion in practice because they use wasm's ability to import and export memories and tables so that all instances linked together share the same memories and the same tables. Among all instances linked together in this way, a table index as used by any function in any of these instances will designate the same thing as that clist index used by any other function in any of those instances. Likewise for numbers to be interpreted as addresses: they can simply be passed as numbers because they index into the same memory.

* Instances vs Compartments

This linkage pattern is so common that we need a name for it. Because there is no isolation between a bunch of instances linked together in this manner, let's call the whole bunch a *compartment*. The wasm module linkage mechanism is clearly designed to enable export/import of functions across compartments. However, the restriction that parameters and return results can only be numbers only works coherently within a compartment. At the same time, wasm does not make visible any difference between calling within a compartment vs calling between them. Wasm code currently cannot reasonably avoid this confusion between `h1` and `h2`.


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
  
The passing of references-to-typed-function on the stack parallels the passing of numbers on the stack. If function `f` wishes to remember passed number `i` after `f` returns, `f` writes `i` somewhere in memory, according to whatever the memory management discipline is of `f`'s language in `f`'s compartment. If function `f` wishes to remember a passed ref-to-typed-function `g` after `f` returns, `f` writes `g` somewhere into a table of functions of `g`'s type, according to whatever the table-memory management discipline is of `f`'s language in `f`'s compartment.

If the memory management within a compartment goes haywire, it fouls its own nest --- it destroys the integrity of its own compartment. But it does not threaten the integrity of defensively consistent compartments that it interacts with. In this sense, a compartment is analogous to an OS process with its own address space. An inter-compartment call is like an IPC. Unlike a conventional OS, a `call_ref` call site can often be ignorant of whether the call is intra- or inter-compartment with little cost.

### An Implementation Approach

The concrete representation of a ref-to-typed-function is a two word record passed by copy
on the stack (parameter and local variables, operand stack entries) and in typed table entries. We depend on the type safety of wasm to keep this record opaque and indivisible. The two words are:
   * A pointer to the machine code of the function
   * A pointer to the module instance containing this function
     
On invocation, the pointed-to-machine-code is given access to 
   * the pointer to the module instance
   * whatever arguments were passed, according to the type of the function.

It is likely that the current representation of a function in a table entry includes these two words, and that `call_indirect` likely already pays at least these costs. Thus, the *only* additional runtime overhead is using a two-word representation of a ref-to-typed-function on the stack, rather than the current practice of a one-word number used as a table index.

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
