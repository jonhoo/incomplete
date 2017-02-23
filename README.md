# incomplete

[![Crates.io](https://img.shields.io/crates/v/incomplete.svg)](https://crates.io/crates/incomplete)
[![Documentation](https://docs.rs/incomplete/badge.svg)](https://docs.rs/incomplete/)

This crate provides `incomplete!()`, a compile-time checked version of `unimplemented!()`.

Motivation for and discussion around this macro can be found in the
[rust-internals](https://internals.rust-lang.org/t/compile-time-checked-version-of-unimplemented/4837)
thread, as well as the [RFC issue](https://github.com/rust-lang/rfcs/issues/1911). Some of it
is repeated below.

We all know and love the `unimplemented!()` macro for keeping track of things we haven't yet
gotten around to. However, it is only checked at runtime if a certain code path is encountered.
This is fine in many cases, such as for corner-cases you don't expect to have to deal with yet.
But when writing new code from scratch (especially when porting), refactoring large amounts of
code, or building comprehensive new features, you often have segments of code that you haven't
implemented yet, **but you know you'll have to**.

The basic motivation for this macro is to introduce a construct to indicate that a piece of
code *needs* to be filled in (i.e., the program should not compile while it is present), but
that the developer wants to postpone coding up until later.

As an example, consider a refactor that requires replacing all `Foo`s and `Bar`s with `Baz`s.
Most of the conversions are straightforward, but some require deeper changes (like `Baz`
requiring some additional values that aren't readily available in a particular segment of the
code). You may want to first finish translating all the `Foo`s and `Bar`s, and only *after*
that deal with the corner cases.

Normally in this case, the developer might add a `// TODO` comment, or an `unimplemented!()`
statement. However, this approach has some drawbacks. In particular, these both still let the
code compile when present. Thus, after the "first pass", the developer must *remember* to also
grep for all the `unimplemented()`s and `// TODO`s (*and* filter out any that aren't relevant
to the refactor in question).

The macro implemented by this crate provides a way to tell the compiler "don't let the code
compile while this is unimplemented", while still running type checks and the borrow checker so
you can fully complete other segments of code in the meantime.

Ideally, this macro would have its own compiler lint so that the resulting error directly said
something along the lines of "required code segment not completed". However, until RFC issue
1911 sees some progress, this crate hacks around the issue by "abusing" another lint and making
it a fatal warning. The result works mostly as desired: it will only error out after all
compiler passes complete successfully, and can be evaluated either as a statement or as an
expression.

## Example

```rust,ignore
#[macro_use] extern crate incomplete;
fn foo() -> bool {
    incomplete!()
}
fn main() {
    foo();
}
```

produces

```text
 error: value assigned to `incomplete` is never read
 --> src/main.rs:5:5
  |
5 |     incomplete!()
  |     ^^^^^^^^^^^^^
  |
```

## Notes

As of the time of writing, the stable compiler will only produce a *warning*, not an *error*,
when `incomplete!()` is used. This is because it doesn't correctly interpret the use of
compiler directives on statements. On nightly this has been fixed.

Also, this macro abuses an existing Rust compiler lint behind the scenes. Since the compiler
insists on telling you *where* every lint warning-turned-error originates from, you will
unfortunately also get a message along the lines of

```text
note: lint level defined here
 --> src/main.rs:5:13
  |
5 |     let x = incomplete!(bool);
  |             ^^^^^^^^^^^^^^^^^
  = note: this error originates in a macro outside of the current crate
```
