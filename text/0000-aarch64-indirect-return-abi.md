- Feature Name: `abi_aarch64_indirect_return`
- Start Date: 2025-08-24
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Introduce a new target-specific calling convention `aarch64-indirect-return`, which will pass the first argument as if it was an indirect result location pointer (that is, through the `x8` register). This would make it possible for Rust to interoperate with C++ functions that return non-trivial C++ objects on `aarch64`.

# Motivation
[motivation]: #motivation

Currently, thanks to C++ ABI being very similar to C ABI on most platforms, it is possible to interoperate with C++ code in Rust via carefully-crafted `#[repr(C)]` types and `extern "C"` functions.

One tricky case to handle is functions returning non-trivial C++ objects (trivial objects have C ABI).
For the purposes of ABI, C++ considers any object that has a non-trivial copy or move constructor or a non-trivial destructor to be non-trivial:

```cpp
struct NonTrivialObject {
    int data = 42;
    ~NonTrivialObject();
};

NonTrivialObject get_object();
```

In case such an object is returned from a function, Itanium ABI requires the caller to allocate a location for the returned object to be stored in and then pass a pointer to that location to the callee. Most platforms pass this pointer as a hidden argument before all other arguments and before `this`, if any. Therefore, it's possible to make an ABI-compatible definition for such a function in Rust:

```rust
unsafe extern "C" {
    fn get_object(result_location: *mut NonTrivialObject);
}
```

`aarch64`, however, uses register `x8` for this, which otherwise does not participate in argument passing. Because of this, it is not possible to call or implement `get_object` on `aarch64` in Rust.

It is possible to use either C++ or assembly wrappers conforming to the `extern "C"` calling convention.

C++ shims add a dependency on C++ compiler and complicate the build process. Assembly shims are quite tricky to get right. And both of those approaches are quite unergonomic and become more complex once the user wants to implement such a function or call it through a function pointer.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This ABI is only useful for interfacing with C++ on `aarch64` platform.

When a non-trivial C++ object is being returned from a function, [Itanium ABI mandates](https://itanium-cxx-abi.github.io/cxx-abi/abi.html#non-trivial-return-values) that the caller must pass a pointer for the target object to be constructed at as a hidden parameter before all other parameters or `this`. On top of that, [`aarch64` ABI mandates](https://github.com/ARM-software/abi-aa/blob/main/cppabi64/cppabi64.rst#summary-of-differences-from-and-additions-to-the-generic-c-abi) (see section "GC++ABI §3.1.3 Return Values") that the pointer parameter must be passed using Indirect Result Location Register (x8), which is otherwise unused for parameter passing.

The `extern "aarch64-indirect-return"` calling convention makes the compiler pass the first function parameter as if it was that hidden result location pointer, causing it to be allocated to the `x8` register. It is otherwise equivalent to `extern "C"` calling convention.

This ABI requires the function to return `()` and for the first argument to be a mutable pointer.

This calling convention also has an unwinding equivalent: `extern "aarch64-indirect-return-unwind"`.

## Example Usage

For the following declaration of function `get_object`:

```cpp
struct NonTrivialObject {
    int data = 42;
    ~NonTrivialObject();
};

NonTrivialObject get_object();
```

Use these rust definitions to call it:

```rust
#[repr(C)]
struct NonTrivialObject {
    data: c_int,
}

unsafe extern "aarch64-indirect-return" {
    fn get_object(result_location: *mut NonTrivialObject);
}
```

It is also possible to implement a function compatible with this declaration:

```rust
unsafe extern "aarch64-indirect-return" get_object(result_location: *mut NonTrivialObject) {
    unsafe {
        *result_location = NonTrivialObject {
            data: 42,
        };
    }
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

`extern "aarch64-indirect-return"` calling convention has requirements that are checked by the Rust compiler:
- The function must not be an async function or a generator
- The function must be marked unsafe
- The return value has to be `()`
- The first argument has to be a thin mutable pointer
- The calling convention must only be used when targeting `aarch64`

The first argument of `extern "aarch64-indirect-return"` has to be passed through the `x8` register, as-if it was an indirect result location pointer.

It can be implemented by marking the first argument to the function with an `sret` LLVM attribute, or using another equivalent mechanism provided by the backend.

# Drawbacks
[drawbacks]: #drawbacks

This feature does add work for alternative Rust implementations/code generators.

Due to limited applicability (C++ FFI without shims on `aarch64`), it is going to benefit a small fraction of Rust users.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Alternative: marking types as non-trivial

An alternative approach would be to introduce an attribute `#[repr(non_trivial)]` that you would put on types in addition to `#[repr(C)]` to signify that they should be passed through a pointer regardless of their size. This would take effect both in return value and argument positions, leading to function signatures closer to those of C++:

```cpp
struct NonTrivialObject {
    int data = 42;
    ~NonTrivialObject();
};

NonTrivialObject get_object();
void pass_object(NonTrivialObject arg);
```

```rust
#[repr(C, non_trivial)]
struct NonTrivialObject {
    data: c_int,
}

unsafe extern "C" {
    fn get_object() -> NonTrivialObject;
    fn pass_object(object: NonTrivialObject);
}
```

This approach, however, may lead to unsoundness when the object is not trivially movable.
There is no mechanism in Rust to call the move constructor on object move, but calling the above-defined bindings does require moving the objects.

The `aarch64-indirect-return`-based approach is explicit about the placement of the returned object and does not rely on Rust moves.
This provides *a* way to soundly work with non-trivially moveable objects.

## Alternative: a marker for an argument instead of whole function

Another alternative might be to mark the function argument with a marker that would be effectively equivalent to LLVM's `sret`:

```rust
#[repr(C)]
struct NonTrivialObject {
    data: c_int,
}

unsafe extern "C" {
    fn get_object(#[sret] return_storage: *mut NonTrivialObject);
}
```

The attribute has to be a new part of function's signature, requiring sweeping changes throughout the compiler for the benefit of a single target.

## Alternative: a target-independent "C-indirect-return" calling convention

Since the `sret` LLVM attribute (and their equivalents in other backends) exist not only for `aarch64`, but for all targets, it is possible to implement this calling convention as a target-independent one.

This can, in theory, allow users to make API bindings without `#[cfg(target_arch = "aarch64")]` conditional blocks, since on other platforms the convention will be equivalent to `extern "C"`.

However, as per https://github.com/rust-lang/rust/issues/137018, there seems to be a negative sentiment towards target-specific calling conventions with fallbacks on other platforms.

# Prior art
[prior-art]: #prior-art

[`bindgen`](https://github.com/rust-lang/rust-bindgen) supports C++ FFI to some extent. It does this by generating ABI-compatible function definitions and linking to them. 
Supporting this calling convention (or some other way to pass non-trivial C++ objects) is a prerequisite for this approach to work on `aarch64`.
Although `bindgen` will not be able to utilize it right away, because it can't currently determine if a type is non-trivial because `libclang` does not expose it (https://github.com/rust-lang/rust-bindgen/issues/778).

[`cxx`](https://github.com/dtolnay/cxx) crate also provides C++ interop, but instead of directly linking to the C++ functions, it generates C wrappers around them.
It also does not allow passing non-trivial objects by-value, instead either requiring indirection or conversion to C types.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is the name `aarch64-indirect-return` a good fit?

# Future possibilities
[future-possibilities]: #future-possibilities

This feature is relatively isolated in limited in scope, so it is not expected that this feature will be extended in the future.
