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

* seamless interoperability between multiple languages

### Non-goals (other)

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


### Challenges

* Fast but type-safe
* Lean but sufficiently universal
* Language-independent
* Trade-off triangle between simplicity, expressiveness and performance

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


### Efficiency Considerations

Wasm-linkage support should maintain Wasm's efficiency properties as much as possible, namely:

* all operations are very cheap, ideally constant time,
* structures are contiguous, dense chunks of memory,
* accessing fields are single-indirection loads and stores,
* no implicit boxing operations (i.e. no implicit allocation on the heap),
* primitive values should not need to be boxed to be stored in managed data structures,
* allows ahead-of-time compilation and code caching.


### Evaluation

Demonstrate that the Alice-Bob-Carol example works.
Write a trivial defensively consistent module and try to attack it from other modules.

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


# TEXT BELOW THIS LINE NOT YET REVISED



### Type Recursion

Through references, aggregate types can be *recursive*:
```
(type $list (struct (field i32) (field (ref $list))))
```
Mutual recursion is possible as well:
```
(type $tree (struct (field i32) (fiedl (ref $forest))))
(type $forest (struct (field (ref $tree)) (field (ref $forest))))
```

The [type grammar](#type-grammar) does not make recursion explicit. Semantically, it is assumed that types can be infinite regular trees by expanding all references in the type section, as is standard.
Folding that into a finite representation (such as a graph) is an implementation concern.


### Type Equivalence

In order to avoid spurious type incompatibilities at module boundaries,
all types are structural.
Aggregate types are considered equivalent when the unfoldings of their definitions are (note that field names are not part of the actual types, so are irrelevant):
```
(type $pt (struct (i32) (i32) (i32)))
(type $vec (struct (i32) (i32) (i32)))  ;; vec = pt
```
This extends to nested and recursive types:
```
(type $t1 (struct (type $pt) (ptr $t2)))
(type $t2 (struct (type $pt) (ptr $t1)))  ;; t2 = t1
(type $u (struct (type $vec) (ptr $u)))   ;; u = t1 = t2
```
Note: This is the standard definition of recursive structural equivalence for "equi-recursive" types.
Checking it is computationally equivalent to checking whether two FSAs are equivalent, i.e., it is a non-trivial algorithm (even though most practical cases will be trivial).
This may be a problem, in which case we need to fall back to a more restrictive definition, although it is unclear what exactly that would be.


### Subtyping

Subtyping is designed to be _non-coercive_, i.e., never requires any underlying value conversion.

The subtyping relation is the reflexive transitive closure of a few basic rules:

1. The `anyref` type is a supertype of every reference type (top reference type).
2. The ` anyfunc` type is a supertype of every function type.
3. A structure type is a supertype of another structure type if its field list is a prefix of the other (width subtyping).
4. A structure type is a supertype of another structure type if they have the same fields and for each field type:
   - The field is mutable in both types and the storage types are the same.
   - The field is immutable in both types and their storage types are in (covariant) subtype relation (depth subtyping).
5. An array type is a supertype of another array type if:
   - Both element types are mutable and the storage types are the same.
   - Both element types are immutable and their storage types are in
(covariant) subtype relation (depth subtyping).
6. A function type is a supertype of another function type if they have the same number of parameters and results, and:
   - For each parameter, the supertype's parameter type is a subtype of the subtype's parameter type (contravariance).
   - For each result, the supertype's parameter type is a supertype of the subtype's parameter type (covariance).

Note: Like [type equivalence](#type-equivalence), subtyping is *structural*.
The above is the standard (co-inductive) definition, which is the most general definition that is sound.
Checking it is computationally equivalent to checking whether one FSA recognises a sublanguage of another FSA, i.e., it is a non-trivial algorithm (even though most practical cases will be trivial).
Like with type equivalence, this may be a problem, in which case a more restrictive definition might be needed.

Subtyping could be relaxed such that mutable fields/elements could be subtypes of immutable ones.
That would simplify creation of immutable objects, by first creating them as mutable, initialize them, and then cast away their constness.
On the other hand, it means that immutable fields can still change, preventing various access optimizations.
(Another alternative would be a three-state mutability algebra.)


### Casting

To minimize typing overhead, all uses of subtyping are _explicit_ through casts.
The instruction
```
(cast_up <type1> <type2> (...))
```
casts the operand of type `<type1>` to type `<type2>`.
An upcast is always safe.
It is a validation error if the operand's type is not `<type1>`, or if `<type1>` is not a subtype of `<type2>`.

Casting is also possible in the reverse direction:
```
(cast_down <type1> <type2> $label (...))
```
also casts the operand of type `<type1>` to type `<type2>`.
It is a validation error if the operand's type is not `<type1>`, or if `<type2>` is not a subtype of `<type1>`.
However, a downcast may fail at runtime if the operand's type is not `<type2>`, in which case control branches to `$label`, with the operand as argument.

Downcasts can be used to implement runtime type analysis, or to recover the concrete type of an object that has been cast to `anyref` to emulate parametric polymorphism.

Note: Casting could be extended to allow reinterpreting any sequence of _transparent_ (i.e., non-reference) fields of an aggregate type with any other transparent sequence of the same size.
That would require constraining the ability of implementations to reorder or align fields.


### Import and Export

Types can be exported from and imported into a module:
```
(type (export "T") (type (struct ...)))
(type (import "env" "T"))
```

Imported types are essentially parameters to the module.
As such, they are entirely abstract, as far as compile-time validation is concerned.
The only operations possible with them are those that do not require knowledge of their actual definition or size: primarily, passing and storing references to such types.

TODO: The ability to import types makes the type and import sections interdependent.


## Possible Extension: Variants

TODO


## Possible Extension: Closures

TODO


## Possible Extension: Nesting

* Want to represent structures embedding arrays contiguously.
* Want to represent arrays of structures contiguously (and maintaining locality).
* Access to nested data structures needs to be decomposable.
* Too much implementation complexity should be avoided.

Examples are e.g. the value types in C#, where structures can be unboxed members of arrays, or a language like Go.

Example (C-ish syntax with GC):
```
struct A {
  char x;
  int y[30];
  float z;
}

// Iterating over an (inner) array
A aa[20];
for (int i = 0..19) {
  A* a = aa[i];
  print(a->x);
  for (int j = 0..29) {
    print(a->y[j]);
  }
}
```

Needs:

* incremental access to substructures,
* interior references.

Two main challenges arise:

* Interior pointers, in full generality, introduce significant complications to GC. This can be avoided by distinguishing interior references from regular ones. That way, interior pointers can be represented as _fat pointers_ without complicating the GC, and their use is mostly pay-as-you-go.

* Aggregate objects, especially arrays, can nest arbitrarily. At each nesting level, they may introduce arbitrary mixes of pointer and non-pointer representations that the GC must know about. An efficient solution essentially requires that the GC traverses (an abstraction of) the type structure.


### Basic Nesting

* Aggregate types can be field types.
* They are unboxed, i.e., nesting them describes one flat value in memory; references enforce boxing.

```
(type $colored-point (struct (type $point) (i16)))
```
Here, `type $point` refers to the previously defined `$point` structure type.


### Interior References

Interior References are another new form of value type:
```
(local $ip (inref $point))
```
Interior references can point to unboxed aggregates, while regular ones cannot.
Every regular reference can be converted into an interior reference (but not vice versa) [details TBD].


### Access

* All access operators are also valid on interior references.

* If a loaded structure field or array element has aggregate type itself, it yields an interior reference to the respective aggregate type, which can be used to access the nested aggregate:
  ```
  (load_field (load_field (new $colored-point) 0) 0)
  ```

* It is not possible to store to a fields or elements that have aggregate type.
  Writing to a nested structure or array requires combined uses of `load_field`/`load_elem` to acquire the interior reference and `store_field`/`store_elem` to its contents:
  ```
  (store_field (load_field (new $color-point) 0) 0 (f64.const 1.2))
  ```

TODO: What is the form of the allocation instruction for aggregates that nest others, especially wrt field initializers?


### Fixed Arrays

Arrays can only be nested into other aggregates if they have a *fixed* size.
Fixed arrays are a second version of array type that has a size (expressed as a constant expression) in addition to an element type:
```
(type $a (array i32 (i32.const 100)))
```

TODO: The ability to use constant expressions makes the type, global, and import sections interdependent.


### Flexible Aggregates

Arrays without a static size are called *flexible*.
Flexible aggregates cannot be used as field or element types.

However, it is a common pattern wanting to define structs that end in an array of dynamic size.
To support this, flexible arrays could be allowed for the _last_ field of a structure:
```
(type $flex-array (array i32))
(type $file (struct (field i32) (field (type $flex-array))))
```
Such a structure is itself called *flexible*.
This notion can be generalized recursively: flexible aggregates cannot be used as field or member types, except for the last field of a structure.

Like a flexible array, allocating a flexible structure would require giving a dynamic size operand for its flexible tail array (which is a direct or indirect last field).


### Type Structure

With nesting and flexible aggregates, the type grammar generalizes as follows:
```
fix_field_type   ::=  <storage_type> | (mut <storage_type>) | <fix_data_type>
flex_field_type  ::=  <flex_data_type>

fix_data_type    ::=  (struct <fix_field_type>*) | (array <fix_field_type> <expr>)
flex_data_type   ::=  (struct <fix_field_type>* <flex_field_type>) | (array <fix_field_type>)
data_type        ::=  <fix_data_type> | <flex_data_type>
```
However, additional checks need to apply to (mutually) recursive type definitions in order to ensure well-foundedness of the recursion.
For example,
```
(type $t (struct (type $t)))
```
is not valid.
For example, well-foundedness can be ensured by requiring that the *nesting height* of any `data_type`, derivable by the following inductive definition, is finite:
```
|<storage_type>|               = 0
|(mut <storage_type>)|         = 0
|(struct <field_type>*)|       = 1 + max{|<field_type>|*}
|(array <field_type> <expr>?)| = 1 + |<field_type>|
```
