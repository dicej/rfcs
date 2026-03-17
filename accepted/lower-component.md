# Summary
[summary]: #summary

This document proposes to create a pair of tools to support running components
using any compatible WebAssembly.  These tools could be used either as an
alternative to providing native support for components in a given runtime or as
a temporary polyfill while native support is being built for that runtime.

# Motivation
[motivation]: #motivation

Implementing the complete Component Model specification is a non-trivial task
which entails significant up-front effort as well as ongoing maintenance.  [Jco]
has proven useful as a polyfill for JS-embedded Wasm runtimes which don't yet
have native component support, but it entails a performance penalty and doesn't
support stand-alone Wasm runtimes.

On the other hand, though Wasmtime has performant, native support for
components, that code cannot easily be reused for other runtimes.  In addition,
that implementation significantly expands the trusted compute base which must be
audited for correct and secure behavior beyond what is needed for core Wasm.  It
is closely integrated with the unsafe internals of the runtime, requiring
specialized knowledge and care to maintain and modify.

Ideally, a component implementation would provide the best qualities of both of
those implementations, while addressing or side-stepping their weaknesses:

- Portable to arbitrary runtimes (JS-embedded or standalone)
- Performant
- Secure, e.g. doing as much work (and allocation) as practical in sandboxed
  guest code, minimizing the TCB
- Maintainable without specialized knowledge of the internals of a particular
  runtime
- Compatible with embedded and/or memory-limited scenarios

[Jco]: https://github.com/bytecodealliance/jco

# Proposal
[proposal]: #proposal

This proposal includes four things:

- A `lower-component` tool which takes a component as input and "lowers" it into
  a core module
- A C API representing the intrinsics which a host runtime must provide in order
  to run a module produced by `lower-component`
- A `host-wit-bindgen` tool which takes a WIT world and produces code for a
  chosen target language to instantiate a `lower-component`-generated module and
  invoke its exports
- A C API representing the operations which a host runtime must provide to
  enable instantiation, invocation, access to memories and globals, etc. for
  `host-wit-bindgen`-generated code to make use of

## `lower-component`

The job of this tool is to take an arbitrary component as input and "lower" it
into a core module which may be run using an arbitrary Wasm runtime, assuming
the host provides a small set of intrinsics (covered later) which the module
will call as import functions.  This tool could either be used ahead-of-time or
just prior to instantiation.

In general, a component may include a composition of more than one subcomponent,
each instantiation of which may require its own memory and table.  In that case,
the output module will use multiple memories and tables and include generated
adapter code to "fuse" the imports of one component to the exports of another,
handling validation, cross-memory copies, etc., just as Wasmtime's FACT does
today.

In addition to the generated "fused adapter" code, the output module will
include component model runtime code, separately compiled from Rust source,
which handles, among other things:

- table management for resource and waitable values
- guest-to-guest stream and future I/O
- task and thread bookkeeping

That runtime code will itself make use of intrinsic functions imported from the
host in order to do things only the host can do, e.g. create, suspend, and
resume fibers and collect error backtraces.  See the next section for details.

In the case of component-level interfaces which involve resource, stream and/or
future types, the generated module would include function exports which the host
may call to create new values of those types.  This is necessary because the
tables for such values are managed internally by the guest, not by the host.

### Multiply-instantiated Modules

One challenge with lowering arbitrary components is that a component may
instantiate the same module more than once.  In that case, we have three options:

- Reject the component
- Generate an output module with duplicate copies of each function in a
  multiply-instantiated module, one per instance.
    - Note that leaf functions which do not use memory or globals can be reused
      without duplication.
    - This could lead to significant bloat for "batteries-included" guest
      languages like Python which do not have dead-code-elimination.
- Generate multiple output modules, plus metadata indicating how to instantiate
  and link them together.
    - This would require specifying how that metadata is represented and how the
      whole combination of modules+metadata should be packaged.

Regarding the third option: one possibility would be to define a minimal subset
of the component model which only supports declaring, linking, and instantiating
core modules and use that to package the modules.
  
## Host C API for Lowered Components

This API includes two parts:

- Imported functions called by the guest and provided by the host
- Exported functions called by the host and provided by the guest

### Guest->Host API

As mentioned above, modules produced using `lower-component` can't (yet) express
all operations in core Wasm, and therefore must use intrinsics for certain
things:

- creating, suspending, and resuming fibers
- generating stack traces for component-level errors

Fiber management could be expressed using the Stack Switching proposal, and
indeed `lower-component` will likely have an option to use those instructions,
but that proposal is not yet widely implemented, so we use intrinsics for
maximum portability.  Hopefully all of the above features will eventually be
covered by widely-supported core Wasm instructions.

Note that these intrinsics need not be implemented in C, nor a language with
native support for the Wasm C ABI; we simply use C as a way to represent an ABI
in a familiar, human-readable format.

Finally, note that the exact set of functions imported by a given lowered
component will depend on which intrinsics it actually needs, which stream and
future types are used in the world it targets, and which interfaces and
functions are imported by the world.  For example, imagine we've lowered a
component which targets the following WIT world:

```
package example:package;

interface foo {
  resource thing {
    constructor(v: u32);
    get: func() -> u32;
  }
  
  bar: func(v: string, s: stream<u32>) -> stream<thing>;
}

world target {
  import foo;
  export foo;
}
```

The following is a sketch of the API imported by such a lowered component as a
set of C functions.  Here we assume that the component explicitly creates and
switches between thread, and thus must import intrinsics from the host to do so.

```
// Creates a new thread, initially in a "suspended" state.
//
// - `thread`: The identifier used to refer to the thread hereafter
// - `context`: The value to pass to the function in the new thread
// - `table`: The table in which to find the function
// - `func`: The function to call, of type `(func (param i32))`
__attribute__((__import_module__("env"), __import_name__("thread.new")))
void thread_new(uint32_t thread, void *context, uint32_t table, uint32_t func);

// Switch to the specified thread, suspending the current one.
//
// - `thread`: The identifier of the thread to which to switch
__attribute__((__import_module__("env"), __import_name__("thread.switch-to")))
void thread_switch_to(uint32_t thread);

// The remaining functions listed here are (eventually) intended to match the
// imports a C binding generator would generate per the proposed
// [Guest C ABI](https://github.com/WebAssembly/component-model/pull/378), with
// possible exceptions as noted.

// Constructor for the _imported_ resource `thing`.
//
// - `handle`: The identifier used to refer to the object hereafter
// - `v`: The constructor's `u32` parameter
//
// Note that the signature here differs slightly from the Guest C ABI since in
// this scenario the guest is responsible for allocating resource handles.  We
// _could_ make it match the Guest C ABI by having it return a handle instead of
// taking it as a parameter, but that would require the host to reenter the
// guest to allocate a handle before returning.
__attribute__((__import_module__("example:package/foo"), __import_name__("[constructor]thing")))
void import_example_package_foo_constructor_thing(uint32_t handle, uint32_t v);

// `get` method for the _imported_ resource `thing`.
//
// - `handle`: The identifier of the object
//
// Returns the `u32` result.
__attribute__((__import_module__("example:package/foo"), __import_name__("[method]thing.get")))
uint32_t import_example_package_foo_thing_get(uint32_t handle);

// Drop a borrow or own handle to an instance of the _imported_ resource `thing`.
//
// - `handle`: The identifier of the object
__attribute__((__import_module__("example:package/foo"), __import_name__("[resource-drop]thing")))
void import_example_package_foo_thing_drop(uint32_t handle);

// Imported `bar` function.
//
// - `v_memory`: The memory to which `v_ptr` points
// - `v_ptr`: A pointer to the UTF-8-encoded string representing the `v` parameter
// - `v_len`: The length, in bytes of the encode string
// - `s`: `stream<u32>` parameter
//
// Returns the `stream<thing>` result where `thing` is the _imported_ resource.
__attribute__((__import_module__("example:package/foo"), __import_name__("bar")))
uint32_t import_example_package_foo_bar(uint32_t v_memory, uint8_t *v_ptr, size_t v_len, s: uint32_t);
```

### Host->Guest API

As mentioned earlier, the guest is responsible for maintaining tables to track
resources, streams, futures, and tasks, and threads.  This means the host must
call into the guest to allocate or dispose of such things.  In addition, the
guest must export functions for reading from and writing to streams and futures
where the guest holds one and and the host holds the other.

As with the Guest->Host API, the exact set of functions exported by a given
lowered component will depend on which resource, stream, and future types are
used by the world targeted by the component, as well as which interfaces and
functions are exported.  The following is a sketch of the API exported by the
hypothetical lowered component we presented in the previous section:

```
// (Re)allocates from the specified guest memory.
//
// - `memory`: The index of the memory from which to allocate
// - `ptr`: The previous allocation, or `NULL`
// - `old_size`: The size of the previous allocation, if applicable
// - `align`: The minimum alignment of the new allocation
// - `new_size`: The minimum size of the new allocation
//
// Returns the new allocation, or traps on failure.
//
// Note that this function may become unnecessary once
// [Lazy Lowering](https://github.com/WebAssembly/component-model/issues/383)
// arrives.
__attribute__((__export_name__("cabi_realloc")))
void *cabi_realloc(uint32_t memory, void *ptr, size_t old_size, size_t align, size_t new_size);

// Represents the write- and read-ends of a `stream` or `future`.
//
// Note that we use a currently-hypothetical `__multivalue_return__` attribute
// here to indicate that functions returning this type should compile to a core
// Wasm type of e.g. `(func ... (result i32 i32))`.
__attribute((__multivalue_return__))
typedef struct {
  uint32_t writer;
  uint32_t reader;
} writer_reader_pair_t;

// Constructs a new `stream<u32>`.
//
// Returns the (writer, reader) pair.
__attribute__((__export_name__("stream<u32>.new")))
writer_reader_pair_t stream_u32_new();

// Constructs a new `stream<thing>`, where `thing` is the _imported_ resource.
//
// Returns the (writer, reader) pair.
__attribute__((__export_name__("stream<import example:package/foo#[constructor]thing>.new")))
writer_reader_pair_t stream_import_example_package_foo_thing_new();

// Constructs a new `stream<thing>`, where `thing` is the _exported_ resource.
//
// Returns the (writer, reader) pair.
__attribute__((__export_name__("stream<export example:package/foo#[constructor]thing>.new")))
writer_reader_pair_t stream_export_example_package_foo_thing_new();

// Represents the result of a stream read or write.
//
// Note that we use a currently-hypothetical `__multivalue_return__` attribute
// here to indicate that functions returning this type should compile to a core
// Wasm type of e.g. `(func ... (result i32 i32))`.
__attribute((__multivalue_return__))
typedef struct {
  uint32_t result;
  size_t count;
} result_and_count_t;

// Reads from a `stream<u32>` whose write end is owned by the guest.
//
// - `stream`: The identifier of the stream from which to read
// - `memory`: The memory in which the buffer resides
// - `buffer`: The buffer to receive the items
// - `length`: The maximum number of items which may be received
//
// The return value indicates the result of the operation in the same format as
// the return value of the `stream.read` CM intrinsic.
__attribute__((__export_name__("stream<u32>.read")))
result_and_count_t stream_u32_read(uint32_t stream, uint32_t memory, uint32_t* buffer, size_t length);

// As above, but for writing.
__attribute__((__export_name__("stream<u32>.write")))
result_and_count_t stream_u32_write(uint32_t stream, uint32_t memory, uint32_t* buffer, size_t length);

// As above, but for `stream<thing>`, where `thing` is the _imported_ resource
// type.
__attribute__((__export_name__("stream<import example:package/foo#thing>.read")))
result_and_count_t stream_import_example_package_foo_thing_read(
  uint32_t stream, uint32_t memory, uint32_t* buffer, size_t length
);

// As above, but for writing.
__attribute__((__export_name__("stream<import example:package/foo#thing>.write")))
result_and_count_t stream_import_example_package_foo_thing_write(
  uint32_t stream, uint32_t memory, uint32_t* buffer, size_t length
);

// As above, but for `stream<thing>`, where `thing` is the _exported_ resource
// type.
__attribute__((__export_name__("stream<export example:package/foo#thing>.read")))
result_and_count_t stream_export_example_package_foo_thing_read(
  uint32_t stream, uint32_t memory, uint32_t* buffer, size_t length
);

// As above, but for writing.
__attribute__((__export_name__("stream<export example:package/foo#thing>.write")))
result_and_count_t stream_export_example_package_foo_thing_write(
  uint32_t stream, uint32_t memory, uint32_t* buffer, size_t length
);

// The remaining functions listed here are (eventually) intended to match the
// imports a C binding generator would generate per the proposed
// [Guest C ABI](https://github.com/WebAssembly/component-model/pull/378).

// Constructor for the _exported_ resource `thing`
//
// - `v`: The constructor's `u32` parameter
//
// Returns the identifier of the newly-constructed object.
__attribute__((__export_name__("example:package/foo#[constructor]thing")))
uint32_t example_package_foo_constructor_thing(uint32_t v);

// `get` method for the _exported_ resource `thing`
//
// - `handle`: The identifier of the object to be dropped.
//
// Returns the `u32` result.
__attribute__((__export_name__("example:package/foo#[method]thing.get")))
uint32_t example_package_foo_thing_get(uint32_t handle);

// Destructor for the _exported_ resource `thing`
//
// - `handle`: The identifier of the object to be dropped.
__attribute__((__export_name__("example:package/foo#[dtor]thing")))
void example_package_foo_thing_dtor(uint32_t handle);

// Exported `bar` function.
//
// - `v_memory`: The memory to which `v_ptr` points
// - `v_ptr`: A pointer to the UTF-8-encoded string representing the `v` parameter
// - `v_len`: The length, in bytes of the encode string
// - `s`: `stream<u32>` parameter
//
// Returns the `stream<thing>` result where `thing` is the _exported_ resource.
__attribute__((__export_name__("example:package/foo#bar")))
uint32_t example_package_foo_bar(uint32_t v_memory, uint8_t *v_ptr, size_t v_len, s: uint32_t);
```

## `host-wit-bindgen`

This tool takes as input a WIT world and produces source code for a given target
language which may be used to define component-level host functions, instantiate
`lower-component`-produced modules, and invoke its exports, etc.

Similar to `wit-bindgen`, this could be packaged either as a single tool
supporting multiple target languages or as separate tools, one per language.  In
either case, the generated code would bottom out in calls to the runtime as
defined by the API described in the next section.

Alternatively, the functionality of `host-wit-bindgen`-generated code could be
provided by a library providing a general-purpose, dynamic API for creating
component values, defining host functions, and calling functions.  This would be
useful in scenarios where the shape of the component is not known ahead of time,
and/or the target language is already so dynamic that code generation is
redundant.

## Host C API for Embedder Bindings

In theory, `host-wit-bindgen` could support multiple front-ends (e.g. Rust,
Python, C#, Go, etc.) _and_ multiple back-ends (Wasmtime, WAMR, Wazero, JS,
etc.), but it's probably easier to define a runtime-agnostic C API which each
runtime can implement to support the low-level operations required by
`host-wit-bindgen`-generated code.  Those operations include:

- creating a "store" in which one or more modules may be instantiated
- defining host functions
- instantiating a module
- calling a module's exports
- reading from and writing to a module's memories, tables, and globals
- creating, suspending, and resuming fibers

Given that a C API doesn't make sense in e.g. a web browser, this could be
mirrored as a JS API for use in JS-embedded runtimes.

```
// Represents a runtime store in which one or more modules may be instantiated.
typedef struct {
  void *ptr;
} store_t;

// Create a new store.
//
// - `data`: Application-specific data to be associated with the store
//
// Returns the newly-created store.
store_t new_store(void *data);

// Get the application-specific data from the store.
//
// - `store`: The store from which the data should be retrieved
//
// Returns the associated data.
void *store_data(store_t store);

// Dispose of the specified store.
void store_drop(store_t store);

// tbc
```

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

See the `Motivation` section above for rationale.

At a high level, the alternative to creating a runtime-agnostic component model
implementation is to implement a native one for each runtime, possibly with some
parts factored out into reusable libraries.  This is the approach we've taken so
far with Wasmtime, although not necessarily the one we'd choose with the benefit
of hindsight.  In any case, a runtime-agnostic implementation would be useful as
a temporary polyfill for use in a given runtime until a native implementation is
complete.

## Prior art

There are already a few projects which polyfill the component model:

- [Jco](https://github.com/bytecodealliance/jco) for JS-embedded runtimes
- [Gravity](https://github.com/arcjet/gravity) for Wazero on Go
- [Meld](https://github.com/pulseengine/meld) for arbitrary runtimes

# Open questions
[open-questions]: #open-questions

- How to handle multiply-instantiated modules, and how common are the components
  which do that?
- Who should be responsible for type checking during host->guest and guest->host
  invocations?
    - If `host-wit-bindgen`, where will in the flattened module will it find the
      type metadata it needs?
    - If `lower-component`, it could be harder to optimize (e.g. doing it on
      every call, whereas `host-wit-bindgen` could lean on the target language's
      static type guarantees, as `wasmtime-wit-bindgen` does today)
- How much thread-local state management can be handled by the
  `lower-component`-generated module vs. by the host?
