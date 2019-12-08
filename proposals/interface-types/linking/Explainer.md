# Linking

This document outlines a future proposal that would extend the present 
[Interface Types proposal](../Explainer.md) to allow WebAssembly modules
to define how their own [instances] and their dependencies' instances are
created and linked together. This extension is not proposed as part of the
initial Interface Types release, but rather as a potential follow-up. The
reason to discuss it earlier is that the motivating use cases for this proposal
have come up enough in separate contexts that it's useful to work on the broad
strokes early and align efforts.

1. [Motivation](#motivation)
1. [Is first class instantiation the answer?](#is-first-class-instantiation-the-answer)
1. [Linking via Interface Types](#linking-via-interface-types)
1. [Use Cases Revisited](#use-cases-revisited)
1. [Non-GC / AOT Embeddings](#non-gc--aot-embeddings)
1. [ESM-integration and Linking](#esm-integration-and-linking)


## Motivation

Currently in WebAssembly, if a module `a.wasm` contains an import
`(func (import "b" "x"))`, the way in which the [module name] `b` and the field
name `x` are resolved to a function is entirely host-defined. For example, with
the [JS API], the module name is just used as a property name string, looked up
in an [import object].

If the module name `b` gets resolved by the host to some WebAssembly module
`b.wasm`, the way in which `b.wasm` is instantiated is also entirely
host-defined. This includes what imports are passed to `b.wasm` and whether the
`b.wasm` instance is created anew for `a.wasm` or shared with other.

Overall, this double layer of host-definedness can be visualized with "host"
bubbles on the links between modules and instances:

<p align="center"><img src="./without-linking.svg" width="250"></p>

Without specifying a module namespace, which is outside the scope of this proposal
and, perhaps, the WebAssembly specification in general, we can't expect to
eliminate the host-defined behavior of resolving a module name to a module.
However, after this one host-defined resolution step, there is no reason that
the subsequent instantiation can't be entirely defined by WebAssembly.

Thus, the goal of this "wasm linking" proposal is to allow (but not require)
the replacement of the "host" bubbles between instances with something
WebAssembly-defined:

<p align="center"><img src="./with-linking.svg" width="250"></p>

On first glance, specifying instantiation may seem like a trivial matter of
spec engineering. However, there are 3 interesting use cases, described next,
that suggest that a more general, expressive approach is required.

### Command modules

If, e.g., a C program with a `main()` function is compiled to a wasm module,
and the `main()` function is exported (directly, or indirectly via some
exported wrapper function), the natural way to instantiate the module is once
each time `main()` is called. Otherwise, if a single instance is created and
has `main()` called on it multiple times, fundamental program assumptions may
be broken. In general, programs are often architected to either be run as
single-shot *commands* (run from a command-line or shell script) or 
[daemons]/services (run in the background and responding to a dynamic sequence
of calls), and it can be hard to convert from one architecture to the other.

Moreover, creating and destroying short-lived instances can more eagerly
release memory to the system, reduce fragmentation and avoid a certain class
of leaks and bugs. This matches the "many small Unix processes" style of program
design, but with lower overhead due to wasm instances being lighter-weight than
OS processes.

Lastly, command modules should themselves be able to import and call other
commands modules, similar to how POSIX processes can recursively spawn and
wait on child processes (via `system()` et al). The desired instantiation
pattern is illustrated below with `clang` and `make` as command modules.

<p align="center"><img src="./command.svg" width="600"></p>

This use case rules out any scheme that forces instantiation to happen eagerly
or at most once for a given module.

### "Shared-everything" (dynamic) linking

The principle behind [shared-nothing linking] is that, since it's difficult and
error prone to get unrelated toolchains and languages to share low-level state,
like linear memory and tables, such low-level state should be encapsulated by
the module by default, with data copied at interface boundaries (enabled and
optimized by Interface Types).

However, many C/C++/Rust libraries cannot be factored into shared-nothing
modules, either due to fundamental dependencies on sharing low-level state
with the client or practical reasons like performance or difficulty refactoring.
By default, such libraries are currently [statically-linked] into a containing
wasm module. However, this can potentially lead to significant code duplication
so [dynamic linking]  (or, for symmetry, "shared-everything" linking) can be
useful to the code duplication of static linking.

A challenge with shared-everything linking is to control which instances
share which memory. In particular, it should be possible to write a "main"
module (the analogue of a native `.exe`) that creates its own private memory,
explicitly shares it with N other "shared" modules (the analogue of native
`.dll`/`.so`s), but encapsulates the memory from all other modules, so that the
use of shared-everything linking is kept an implementation detail from the point
of view of the main module's clients.

In particular, it should be possible to use the wasm linking proposal to achieve
the following configuration of instances, where the orange rectangles represent
shared-nothing boundaries maintained by the main module in each rectangle.

<p align="center"><img src="./shared-everything.svg" width="600"></p>

Here, `libc.wasm` is specially chosen by the toolchain to define and export
memory (along with memory-management functions like `malloc`). Both `zip.wasm`
and `img.wasm` are main modules, and make their own new, private intsances of
`libc.wasm` and then each respectively instantiates `libm.wasm`, explicitly
passing the private `libc.wasm` instance to `libm.wasm` as an import.

As seen in this diagram, what we need is to be able to create a single instance
graph which contains encapsulated subgraphs of tightly-coupled shared-everything
modules that are loosely coupled with other shared-nothing modules.

### Link-time virtualization

When using an imperative instantiation API like the [JS API], the imports of the
being-instantiated module appear as explicit parameters supplied by the
instantiating code. This has several useful properties that any wasm
linking proposal should preserve:

First, it supports applications that wish to adhere to the [Principle of Least Authority],
passing modules only the imports necessary to do their job.

Second, it enables client modules to fully or partially virtualize the imports
of their dependencies without extraordinary challenge or performance overhead.
For example, if a module imports a set of file operations to operate on the file
handles it receives in its interface, and a client wants to reuse this module
on a system without builtin file I/O, the file operations and handles can be
virtualized with an in-memory implementation. Thus, if virtualization is
well-supported and efficient, software reusability and composability is
increased.

In fact, virtualization also enhances the ability to implement the Principle of
Least Authority, by allowing a client module to not only pass *subsets* of
capabilities, but also pass capabilities that are dynamically *attenuated*,
applying additional policies at runtime to reduce the granted authority.

To support these use cases, therefore, it's important that a wasm linking
proposal fully supports *link-time virtualization*. For example, it should be
possible to use wasm linking to achieve the following configuration of
instances:

<p align="center"><img src="./virtualization.svg" width="600"></p>

Here, `parent.wasm` imports a `wasi:file` instance (not indicating a specific
module, just importing *some* instance implementing the `wasi:file` interface)
and passes this `wasi:file` instance on to instantiate `attenuate.wasm`.
`attenuate.wasm` applies its attenuation policy to implement the same
`wasi:file` interface which can thus be passed to instantiate `child.wasm`.
Critically, `child.wasm` has no way to get a hold of the original `wasi:file`
instance, without it being explicitly given.


## Is first-class instantiation the answer?

Given the above varied use cases, one might ask if we should simply define
an imperative instantiation API (analogous to the [JS API], but not
JS-specific), which would clearly enable the above use cases and many more.
Indeed, first-class instantiation has already been discussed 
[in the context of the reference-types proposal][host-references] and it seems
quite likely to be added to wasm at some point in the future.

One problem with a first-class instantiation API is that it fundamentally
depends on GC. The reason is that, by containing mutable reference cells
(globals and tables), wasm instances can participate in arbitrary cyclic graphs
that require some form of GC (or cycle collection) to collect. As a general
rule, WebAssembly standardization avoids creating any hard dependencies on GC
(for anything outside the [GC proposal]). This choice is essential to the high
degree of embeddability of wasm, allowing GC support to be optional on hosts
that need a small footprint.

Another goal, specific to linking, is that linking should be declarative enough
to allow:
* webpack-style fusion of multiple linked wasm modules into a single wasm
  module (using [multi-memory], to keep the source modules' memories
  independent);
* full Ahead-of-Time compilation of a linked module graph into a
  native binary, optimizing aggressively based on whole-program knowledge.

On the other end of the spectrum from the JS API, [ESM-integration] 
(WebAssembly module integration into the ECMAScript Module system) illustrates
one possible declarative module-linking scheme: wasm linking could just
specify to "do what ESM-integration would do when all the ESMs happen to
be wasm". This would be especially nice for Web compatibility.

Unfortunately, all 3 of the above use cases fall outside of what is possible
(in general, without hacks) with current ESMs. (We'll return to ESM-integration
[below](#esm-integration).)


## Linking via Interface Types

The Interface Types proposal provides a potential midpoint between a fully
first-class instantiation API and the overly-restrictive ESM loader.
Specifically, interface adapters are programmable, but declarative enough that
all interface value producers can be statically matched with interface value
consumers. In the context of the base Interface Types proposal, this
definitions-match-uses property ensures that compound values can [always][direct-copy]
be copied directly between instances' memories. For the purposes of wasm
linking, though, this property ensures that the link-time relationship between
instances can be known statically. The rest of this section incrementally
introduces the extensions to Interface Types that enable wasm linking.
(Familiarity with the concepts and syntax in the [Interface Types Explainer] is
assumed.)


### Tweaks to the underlying Interface Types proposal

Before starting, this proposal suggests making one tweak to the base Interface
Types proposal. (If we can make that tweak, we can remove this subsection.)

For the purpose of linking, it's useful to pass around whole *instances*,
not individual exports (like functions). Currently, in both core wasm and
interface adapters, every import has two strings: a module name and a field
name. Thus, if we import N exports from a single module name, the module name is
repeated N times:
```wasm
(module
  (@interface func (import "foo" "f1") (param string))
  (@interface func (import "foo" "f2") (result string))
  ...
)
```
However, using the *instance types* defined by the [Module Types] proposal,
we can express the same module type more compactly as taking a single
instance with two exports:
```wasm
(module
  (@interface instance (import "foo")
    (func (export "f1") (param string))
    (func (export "f2") (result string))
  )
)
```
This factored form ends up significantly simplifying a number of aspects of the
rest of the linking proposal.


### Module Imports

What's fundamentally missing in wasm is the ability to import (uninstantiated)
*modules*, as opposed to (already-created) *instances*. Thus, symmetric to
instance imports, wasm linking adds **module imports**:

```wasm
(module
  (@interface module (import "some_module")
    (instance (import "a")
      (func (export "b") (param s32))
    )
    (func (export "c") (param string))
  )
  ...
)
```

Walking through this module import bit by bit:

Module imports are part of the interface adapter, hence the `@interface` and
the allowed use of interface types in the contained function signatures.

Symmetric to instance imports, module imports are imported with a single module
name (here `some_module`). This name is mapped to an actual module by the
host; this proposal doesn't help define how this mapping happens; it might be
resolved using the file system, a `package.json` or something else.

As with all other imports, a module import must declare the import's type. Since
this is in the interface adapter, it imports whole instances (via an `instance`
import) instead of individual fields.

Symmetric to all other kinds of imports, module imports build a *module index
space* containing the module imports. In addition to module imports, the
module index space contains the outer containing module. If nested modules are
added to wasm in the future, they would also be present in the module index
space too.

Before we can instantiate this module, though, we need a new interface type to
describe its parameters and results.


### Instance reference types

Unlike other interface types, which classify values, since instances are
stateful (due to their contained memories, tables and globals) and thus have an
identity, they need to be passed by *reference*.

To avoid duplicating an instance type everywhere we need to refer to it, we'll
often define an instance type once, with a `(@interface type $Name (instance ...))`
definition and reference it elsewhere via `(ref `$Name)` or `(type $Name)`,
depending on the context. This is symmetric to what's possible in wasm today
to define and reuse function types ([example][func-type-example]).

This example shows an adapter function taking an instance reference parameter:
```wasm
(module
  (@interface type $I (instance
    (func (export "run") (param string) (result string))
  ))
  (@interface func (export "foo") (param (ref $I))
    ...
  )
)
```

As with the module imports described above, instance imports and the current
instance form an *instance index space*. To get a reference to any instance
in the instance index space, adapter code can use a new `ref.instance`
adapter instruction.

For example, in this example:
```wasm
(module
  (@interface type $ImportType (instance
    ... some exports
  ))
  (@interface instance $Import (import "some_import") (type $ImportType))
  (@interface func (export "foo") (result (ref $ImportType))
    ref.instance $Import
  )
)
```
the adapter function `foo` returns its imported instance.

Another adapter instruction that takes instance references is
`call-instance-export`, which has the signature:
```
call-instance-export <string> : [(ref $I) T*] -> [U*]
```
where `$I` must be an instance type containing a function export named
`<string>` taking `T*` to `U*`.


### Instantiating modules

The big new instruction, though, is `instantiate`, which creates a new
instance of a given module. The signature of `instantiate` is:
```
instantiate <module> : [(ref (instance Iᵢ))ⁿ] -> [(ref (instance I))]
```
where `<module>` is an index or `$id` identifying a module in the module index
space and the `n` operands are statically required to match the `n` imports
of `<module>`, ignoring the module-name strings of imports and just using the
order in which the imports are declared in `<module>`'s type.

For example, in this example:
```wasm
(module $ThisModule
  (@interface type $I (instance
    ... some exports
  ))
  (@interface instance $Import (import "whatever") (type $I))
  (@interface func (export "spawn") (result (ref $I))
    ref.instance $Import
    instantiate $ThisModule
  )
)
```
the `spawn` adapter function creates a new instance, using the same import as
the original instance. `$ThisModule` has a single import, so `instantiate`
takes a single operand.

However, as this contrived example shows, `instantiate` would be significantly
limited if it could only be called from adapter functions called from
*already-created* instances. Most of the motivating use cases involve creating
and linking instances where there were none to begin with.


### Constructor functions

The final extension to support wasm linking is a new kind of *constructor*
function that is called when a module is intantiated, receiving the instantiation
arguments, allowing a module to control and encapsulate more of how it is
instantiated. In particular, this allows a module to recursively instantiate
its dependencies (module imports) and call exported functions to establish
initial invariants.

A constructor function is similar to a normal adapter function, with a signature
and body containing adapter instructions. The differences are:
* a constructor function starts with the keyword `ctor` instead of `func`
* the parameters of a constructor function are named with strings (for reasons
  below)
* the result type of a constructor function must be a `(ref (instance ...))`,
  where the instance type is that of the containing module, and thus is not
  explicitly declared
* constructor function bodies are allowed to contain a new `instantiate-self`
  instruction which allows the constructor to invoke the default spec-defined
  instantiation (which the constructor is effectively hiding)

For example, the following module is able to both instantiate itself
and immediately call an `init` function to finish initialization. (Notice that
this overcomes the classic limitation of [start functions] being nullary and
their exports not being externally visible.)
```wasm
;; one.wasm
(module
  (func (export "init") (param i32 i32) ...)
  (@interface ctor (param $p "initialValue" string)
    instantiate-self
    dup
    (call-instance-export "init" (string-to-memory (local.get $p)))
  )
  (@interface func (export "use") (result string) ...)
)
```

When a module contains a constructor function, the constructor's signature
modifies the module type, replacing the module imports (which are now internally
supplied by `instantiate-self`) with the constructor's parameters. For example,
in the *absence* of a constructor function, the above `one.wasm` would have the
module type:
```wasm
(module
  (@interface func (export "use") (result string))
)
```
However, *with* the constructor function, the module type becomes:
```wasm
(module
  (@interface string (import "initialValue"))
  (@interface func (export "use") (result string))
)
```
Here, `string` takes the place of `instance` and `module` as the entity that is
being imported. Thus, when a client creates this module with the `instantiate`
instruction, the operand type will need to be `string` instead of the usual
`(ref (instance ...))` type. For example:
```wasm
;; two.wasm
(module
  (@interface module $One (import "./one.wasm")
    (string (import "initialValue"))
    (func (export "use") (result string))
  )
  (@interface instance (import "")
    (func (export "use") (result string))
  )
  (@interface ctor (param $p string)
    (instantiate $One (local.get $p))
    instantiate-self
  )
)
```
Looking at the 3 definitions in this module:
* The first definition imports the above `one.wasm` module.
* The second definition imports an instance with a `one.wasm`-compatible
  instance type. Because `instantiate-self` passes imports positionally,
  the import string is insignificant and so the empty string is used.
* The third definition defines a constructor function which first instantiates
  the dependency (`one.wasm`) and then uses the resulting instance to instantiate
  `two.wasm`.

From another perspective, constructor functions are vaguely similar to a common
C++ pattern of using static member functions to encapsulate construction. For
example, the rough equivalent of the above two wasm modules is:

```c++
class One {
  One() {}
  void init(string);
public:
  static One* ctor(string s) {
    One* one = new One();
    one->init(s);
    return one;
  }
};

class Two {
  One* one_;
  Two(One* one) : one_(one) {}
public:
  static Two* ctor(string s) {
    One* one = One::ctor(s);
    return new Two(one);
  }
};
```


### The New Picture

TODO: show a sweet diagram that subsumes [the original](https://github.com/WebAssembly/interface-types/blob/master/proposals/interface-types/overview.svg).


## Use Cases Revisited

With these 4 additions to interface adapters ([module imports](#module-imports),
[instance reference types](#instance-reference-types),
the [`instantiate`](#instantiating-modules) instruction and
[constructor functions](#constructor-functions)), we can now revisit the 3 use cases
listed in the [Motivation](#motivation) and sketch how they are addressed.

**Command modules**

TODO: need to introduce nested modules above to solve this.

**"Shared-everything" (dynamic) linking**

Any dynamic linking scheme ultimately requires establishing an [ABI] that all
participating modules (and their producing toolchain) have to adhere to.
With dynamic linking, this ABI needs to cover not just the ABI between
functions, but also between entire modules.

To start with, a dynamic linking ABI could define an instance type which
exported the memory, table, and the common operations to allocate and free
memory and elements. Let's call this instance type `$libc`:

```wasm
(type $libc (instance
  (memory (export "memory") 1)
  (memory (export "funcptr_table") funcref 1)
  (func (export "malloc") (param i32) (result i32))
  (func (export "free") (param i32))
  ...
))
```

A given toolchain would also ship with some `libc.wasm` in its sysroot whose
instances match `$libc`.

Next, assume the toolchain makes the distinction between "main" and "shared"
modules, where the distinction is that each main module instance creates its
own new memory while shared modules always expect to import memory. E.g., in
traditional C/C++ toolchain, a default compilation would produce a main module,
while compiling with `-shared` would produce a shared module.

When compiling a shared module, the toolchain would emit a constructor that takes
a `(ref $libc)` parameter. The exports of a shared module would be those explicitly
exported via [`dllexport`]/[`visibility("default")`], and potentially some
ABI-defined special exports, like a `shared_main`:

```wasm
(module
  (@interface ctor (param "libc" (ref $libc)) ...)
  (func (export "shared_main") ...)
  (func (export "my_export") ...)
)
```

When compiling the main module, the toolchain would emit module imports for
`libc.wasm` and each shared module. The main module would also have a
constructor which would take the following steps:
1. `instantiate` `libc.wasm`
2. `instantiate` each shared module, passing them the `libc` instance from step 1.
3. `instantiate-self` the main module, passing `libc` and shared instances created
   by steps 1 and 2.
4. Call the `shared_main` export of shared modules, if defined by the ABI.
5. Call the main module's `main()` function.

Note that this process can be recursive with shared modules themselves importing
and instantiating shared modules. Since memory is shared explicitly through
the `libc` parameter, modules can control which other modules they share memory
with.

The above instantiation scheme ensures that exports of shared modules can be
imports of the main module. If shared modules need to import functions from
the main module, they'll need to call through a `funcref` table, which is
initialized dynamically by the main module. This would be achieved by having
the shared modules import a `(global mut i32)` for each function they want
to import from the main module, with the `i32` being the index of the global
export in the `funcptr_table`.

However, the converse instantiation scheme is also expressible, wherein the main
module is instantiated first, so that its exports can be imported by shared
modules. And other hybrids too, e.g., fusing `libc.wasm` with the main module,
and instantiating this fused main+`libc` first, imported by the shared modules.

Ultimately, the toolchain ABI will determine many of the details above; wasm
linking just provides the necessary building blocks to allow the ABI to be
expressed in pure WebAssembly in a declarative-enough form to be optimized,
as mentioned below.

**Link-time virtualization**

TODO
* module imports: do not grant capabilities
* constructor parameters: instances convey capabilities; must be explicitly
  passed by the parent
* revisit attenuation virtualization example, now with code

## Additional use cases unlocked

* Exports visible during start function
* The daemonize pattern and self-revocation of capabilities


## Non-GC / AOT Embeddings

TODO:
* describe the conditions for there to be no GC required.
* describe what sort of AOT optimizations are possible.


## ESM-integration

TODO: wasm linking could, in fact, be integrated into ESM



[Instances]: https://webassembly.github.io/spec/core/exec/runtime.html#module-instances
[Module Name]: https://webassembly.github.io/spec/core/syntax/modules.html#syntax-import
[start functions]: https://webassembly.github.io/spec/core/syntax/modules.html#start-function

[JS API]: https://webassembly.github.io/spec/js-api/index.html
[Import Object]: https://webassembly.github.io/spec/js-api/index.html#read-the-imports

[Interface Types Explainer]: https://github.com/WebAssembly/interface-types/blob/master/proposals/interface-types/Explainer.md
[Shared-nothing Linking]: https://github.com/WebAssembly/interface-types/blob/linking/proposals/interface-types/Explainer.md#motivation
[direct-copy]: https://github.com/WebAssembly/interface-types/blob/master/proposals/interface-types/Explainer.md#lifting-lowering-and-laziness

[host-references]: https://github.com/WebAssembly/reference-types/blob/master/proposals/reference-types/Overview.md#further-possible-generalisations
[multi-memory]: https://github.com/webassembly/multi-memory
[esm-integration]: https://github.com/WebAssembly/esm-integration
[func-type-example]: https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md#examples
[Module Types]: https://github.com/WebAssembly/module-types/blob/master/proposals/module-types/Overview.md
[GC proposal]: https://github.com/WebAssembly/gc/blob/master/proposals/gc

[daemons]: https://en.wikipedia.org/wiki/Daemon_(computing)
[Statically-linked]: https://en.wikipedia.org/wiki/Static_library
[Dynamic linking]: https://en.wikipedia.org/wiki/Dynamic_loading
[Principle of Least Authority]: https://en.wikipedia.org/wiki/Principle_of_least_privilege
[ABI]: https://en.wikipedia.org/wiki/Application_binary_interface

[`dllexport`]: https://docs.microsoft.com/en-us/cpp/cpp/dllexport-dllimport
[`visibility("default")`]: https://gcc.gnu.org/wiki/Visibility
