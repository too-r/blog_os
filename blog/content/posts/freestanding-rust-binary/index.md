+++
title = "A Freestanding Rust Binary"
order = 0
path = "freestanding-rust-binary"
date = "0000-01-01"
+++

This post describes how to create a Rust executable that does not link the standard library. This makes it possible to run Rust code on the [bare metal] without an underlying operating system.

[bare metal]: https://en.wikipedia.org/wiki/Bare_machine

<!-- more -->

TODO github, issues, comments, etc

This tutorial doesn't require that you're familiar with Rust, but it doesn't explain all the basics. For an introduction to Rust and an installation guide take a look at the [official Rust book].

[official Rust book]: https://doc.rust-lang.org/stable/book/second-edition/

## Disabling the Standard Library
By default, all Rust crates link the [standard library], which dependends on the operating system for features such as threads, files, or networking. It also depends on the C standard library `libc`, which closely interacts with OS services. Since our plan is to write an operating system, we can not use any OS-dependent libraries. So we have to disable the automatic inclusion of the standard library through the [`no_std` attribute].

[standard library]: https://doc.rust-lang.org/std/
[`no_std` attribute]: https://doc.rust-lang.org/book/first-edition/using-rust-without-the-standard-library.html

We start by creating a new cargo application project. The easiest way to do this is through the command line:

```
> cargo new blog_os --bin
```

I named the project `blog_os`, but of course you can choose your own name. The `--bin` flag specifies that we want to create an executable binary (in contrast to a library). When we run the command, cargo creates the following directory structure for us:

```
blog_os
├── Cargo.toml
└── src
    └── main.rs
```

The `Cargo.toml` contains the crate configuration, for example the crate name, the author, the [semantic version] number, and dependencies. The `src/main.rs` file contains the root module of our crate and our `main` function. You can compile your crate through `cargo build` and then run the compiled `blog_os` binary in the `target/debug` subfolder.

[semantic version]: http://semver.org/

### The `no_std` Attribute

Right now our crate implicitly links the standard library. Let's try to disable this by adding the [`no_std` attribute]:

```rust
// main.rs

#![no_std]

fn main() {
    println!("Hello, world!");
}
```

When we try to build it now (by running `cargo build`), the following error occurs:

```
error: cannot find macro `println!` in this scope
 --> src/main.rs:4:5
  |
4 |     println!("Hello, world!");
  |     ^^^^^^^
```

The reason for this error is that the [`println` macro] is part of the standard library, which we no longer include. So we can no longer print things. This makes sense, since `println` writes to [standard output], which is a special file descriptor provided by the operating system.

[`println` macro]: https://doc.rust-lang.org/std/macro.println.html
[standard output]: https://en.wikipedia.org/wiki/Standard_streams#Standard_output_.28stdout.29

So let's remove the printing and try again with an empty main function:

```rust
// main.rs

#![no_std]

fn main() {}
```

```
> cargo build
error: language item required, but not found: `panic_fmt`
error: language item required, but not found: `eh_personality`
```

Now the compiler is missing some _language items_. Language items are special pluggable functions that the compiler invokes on certain conditions, for example when the application [panics]. Normally, these items are provided by the standard library, but we disabled it. So we need to provide our own implementations.

[panics]: https://doc.rust-lang.org/stable/book/second-edition/ch09-01-unrecoverable-errors-with-panic.html

### Enabling Unstable Features

Implementing language items is unstable and protected by a so-called _feature gate_. A feature gate is a special attribute that you have to specify at the top of your `main.rs` in order to use the corresponding feature. By doing this you basically say: “I know that this feature is unstable and that it might stop working without warning. I want to use it anyway.”

To limit the use of unstable features, the feature gates are not available in the stable or beta Rust compilers, only [nightly Rust] makes it possible to opt-in. This means that you have to use a nightly compiler for OS development for the near future (since we need to implement unstable language items). To install a nightly compiler using [rustup], you just need to run `rustup default nightly` (for more information check out [rustup's documentation]).

[nightly Rust]: https://doc.rust-lang.org/book/first-edition/release-channels.html
[rustup]: https://rustup.rs/
[rustup's documentation]: https://github.com/rust-lang-nursery/rustup.rs#rustup-the-rust-toolchain-installer

After installing a nightly Rust compiler, you can enable the unstable `lang_items` feature by inserting `#![feature(lang_items)]` right at the top of `main.rs`.

### Implementing the Language Items

To create a `no_std` binary, we have to implement the `panic_fmt` and the `eh_personality` language items. The `panic_fmt` items specifies a function that should be invoked when a panic occurs. This function should format an error message (hence the `_fmt` suffix) and then invoke the panic routine. In our case, there is not much we can do, since we can neither print anything nor do we a panic routine. So we just loop indefinitely:

```rust
#![feature(lang_items)]
#![no_std]

fn main() {}

#[lang = "panic_fmt"]
#[no_mangle]
pub extern fn rust_begin_panic(_msg: core::fmt::Arguments,
                               _file: &'static str,
                               _line: u32,
                               _column: u32) -> ! {
    loop {}
}
```

The function signature is taken from the [unstable Rust book]. The signature isn't verified by the compiler, so implement it carefully. TODO: https://github.com/rust-lang/rust/issues/44489

[unstable Rust book]: https://doc.rust-lang.org/unstable-book/language-features/lang-items.html#writing-an-executable-without-stdlib

Instead of implementing the second language item, `eh_personality`, we remove the need for it by disabling unwinding.

### Disabling Unwinding

The `eh_personality` language item is used for implementing [stack unwinding]. By default, Rust uses unwinding to run the destructors of all live stack variables in case of panic. This ensures that all used memory is freed and allows the parent thread to catch the panic and continue execution. Unwinding, however, is a complicated process and requires some OS specific libraries (e.g. [libunwind] on Linux or [structured exception handling] on Windows), so we don't want to use it for our operating system.

[stack unwinding]: http://www.bogotobogo.com/cplusplus/stackunwinding.php
[libunwind]: http://www.nongnu.org/libunwind/
[structured exception handling]: https://msdn.microsoft.com/en-us/library/windows/desktop/ms680657(v=vs.85).aspx

There are other use cases as well for which unwinding is undesireable, so Rust provides an option to [abort on panic] instead. This disables the generation of unwinding symbol and thus considerably reduces binary size. There are multiple places where we can disable unwinding. The easiest way is to add the following lines to our `Cargo.toml`:

```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

This sets the panic strategy to `abort` for both the `dev` profile (used for `cargo build`) and the `release` profile (used for `cargo build --release`). Now the `eh_personality` language item should no longer be required.

[abort on panic]: https://github.com/rust-lang/rust/pull/32900

However, if we try to compile it now, another language item is required:

```
> cargo build
error: requires `start` lang_item
```

### The `start` attribute
The `start` language item defines the real entry point for the application. This function is then responsible for calling the `main` function.
