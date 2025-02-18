# Debugging the compiler
[debugging]: #debugging

This chapter contains a few tips to debug the compiler. These tips aim to be
useful no matter what you are working on.  Some of the other chapters have
advice about specific parts of the compiler (e.g. the [Queries Debugging and
Testing chapter](./incrcomp-debugging.html) or the [LLVM Debugging
chapter](./backend/debugging.md)).

## Configuring the compiler

By default, rustc is built without most debug information. To enable debug info, 
set `debug = true` in your config.toml. 

Setting `debug = true` turns on many different debug options (e.g., `debug-assertions`, 
`debug-logging`, etc.) which can be individually tweaked if you want to, but many people 
simply set `debug = true`. Check out the comments in config.toml.example for more info.

You will need to rebuild the compiler once you've changed any configuration options. 

## `-Z` flags

The compiler has a bunch of `-Z` flags. These are unstable flags that are only
enabled on nightly. Many of them are useful for debugging. To get a full listing
of `-Z` flags, use `-Z help`.

One useful flag is `-Z verbose`, which generally enables printing more info that
could be useful for debugging.

## Getting a backtrace
[getting-a-backtrace]: #getting-a-backtrace

When you have an ICE (panic in the compiler), you can set
`RUST_BACKTRACE=1` to get the stack trace of the `panic!` like in
normal Rust programs.  IIRC backtraces **don't work** on MinGW,
sorry. If you have trouble or the backtraces are full of `unknown`,
you might want to find some way to use Linux, Mac, or MSVC on Windows.

In the default configuration (without `debug` set to `true`), you don't have line numbers 
enabled, so the backtrace looks like this:

```text
stack backtrace:
   0: std::sys::imp::backtrace::tracing::imp::unwind_backtrace
   1: std::sys_common::backtrace::_print
   2: std::panicking::default_hook::{{closure}}
   3: std::panicking::default_hook
   4: std::panicking::rust_panic_with_hook
   5: std::panicking::begin_panic
   (~~~~ LINES REMOVED BY ME FOR BREVITY ~~~~)
  32: rustc_typeck::check_crate
  33: <std::thread::local::LocalKey<T>>::with
  34: <std::thread::local::LocalKey<T>>::with
  35: rustc::ty::context::TyCtxt::create_and_enter
  36: rustc_driver::driver::compile_input
  37: rustc_driver::run_compiler
```

If you set `debug = true`, you will get line numbers for the stack trace. 
Then the backtrace will look like this:

```text
stack backtrace:
   (~~~~ LINES REMOVED BY ME FOR BREVITY ~~~~)
             at /home/user/rust/compiler/rustc_typeck/src/check/cast.rs:110
   7: rustc_typeck::check::cast::CastCheck::check
             at /home/user/rust/compiler/rustc_typeck/src/check/cast.rs:572
             at /home/user/rust/compiler/rustc_typeck/src/check/cast.rs:460
             at /home/user/rust/compiler/rustc_typeck/src/check/cast.rs:370
   (~~~~ LINES REMOVED BY ME FOR BREVITY ~~~~)
  33: rustc_driver::driver::compile_input
             at /home/user/rust/compiler/rustc_driver/src/driver.rs:1010
             at /home/user/rust/compiler/rustc_driver/src/driver.rs:212
  34: rustc_driver::run_compiler
             at /home/user/rust/compiler/rustc_driver/src/lib.rs:253
```

## Getting a backtrace for errors
[getting-a-backtrace-for-errors]: #getting-a-backtrace-for-errors

If you want to get a backtrace to the point where the compiler emits an
error message, you can pass the `-Z treat-err-as-bug=n`, which will make
the compiler panic on the `nth` error on `delay_span_bug.` If you leave
off `=n`, the compiler will assume `1` for `n` and thus panic on the
first error it encounters.

This can also help when debugging `delay_span_bug` calls - it will make
the first `delay_span_bug` call panic, which will give you a useful backtrace.

For example:

```bash
$ cat error.rs
```

```rust
fn main() {
    1 + ();
}
```

```bash
$ rustc +stage1 error.rs
error[E0277]: cannot add `()` to `{integer}`
 --> error.rs:2:7
  |
2 |       1 + ();
  |         ^ no implementation for `{integer} + ()`
  |
  = help: the trait `Add<()>` is not implemented for `{integer}`

error: aborting due to previous error
```

Now, where does the error above come from?

```bash
$ RUST_BACKTRACE=1 rustc +stage1 error.rs -Z treat-err-as-bug
error[E0277]: the trait bound `{integer}: std::ops::Add<()>` is not satisfied
 --> error.rs:2:7
  |
2 |     1 + ();
  |       ^ no implementation for `{integer} + ()`
  |
  = help: the trait `std::ops::Add<()>` is not implemented for `{integer}`

error: internal compiler error: unexpected panic

note: the compiler unexpectedly panicked. this is a bug.

note: we would appreciate a bug report: https://github.com/rust-lang/rust/blob/master/CONTRIBUTING.md#bug-reports

note: rustc 1.24.0-dev running on x86_64-unknown-linux-gnu

note: run with `RUST_BACKTRACE=1` for a backtrace

thread 'rustc' panicked at 'encountered error with `-Z treat_err_as_bug',
/home/user/rust/compiler/rustc_errors/src/lib.rs:411:12
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose
backtrace.
stack backtrace:
  (~~~ IRRELEVANT PART OF BACKTRACE REMOVED BY ME ~~~)
   7: rustc::traits::error_reporting::<impl rustc::infer::InferCtxt<'a, 'tcx>>
             ::report_selection_error
             at /home/user/rust/compiler/rustc_middle/src/traits/error_reporting.rs:823
   8: rustc::traits::error_reporting::<impl rustc::infer::InferCtxt<'a, 'tcx>>
             ::report_fulfillment_errors
             at /home/user/rust/compiler/rustc_middle/src/traits/error_reporting.rs:160
             at /home/user/rust/compiler/rustc_middle/src/traits/error_reporting.rs:112
   9: rustc_typeck::check::FnCtxt::select_obligations_where_possible
             at /home/user/rust/compiler/rustc_typeck/src/check/mod.rs:2192
  (~~~ IRRELEVANT PART OF BACKTRACE REMOVED BY ME ~~~)
  36: rustc_driver::run_compiler
             at /home/user/rust/compiler/rustc_driver/src/lib.rs:253
```

Cool, now I have a backtrace for the error!

## Getting logging output
[getting-logging-output]: #getting-logging-output

The compiler uses the [`tracing`] crate for logging.

[`tracing`]: https://docs.rs/tracing

The compiler has a lot of [`debug!`] calls, which print out logging information
at many points. These are very useful to at least narrow down the location of
a bug if not to find it entirely, or just to orient yourself as to why the
compiler is doing a particular thing.

[`debug!`]: https://docs.rs/tracing/0.1/tracing/macro.debug.html

To see the logs, you need to set the `RUSTC_LOG` environment variable to your
log filter. Your log filter can be just `debug` to get all `debug!` output and
higher (e.g., it will also include `info!`), or `path::to::module` to get *all*
output (which will include `trace!`) from a particular module, or
`path::to::module=debug` to get `debug!` output and higher from a particular
module.

For example, to get the `debug!` output and higher for a specific module, you
can run the compiler with `RUSTC_LOG=path::to::module=debug rustc my-file.rs`.
All `debug!` output will then appear in standard error.

Note that you can use a partial path and the filter will still work. For
example, if you want to see `info!` output from only
`rustdoc::passes::collect_intra_doc_links`, you could use
`RUSTDOC_LOG=rustdoc::passes::collect_intra_doc_links=info` *or* you could use
`RUSTDOC_LOG=rustdoc::passes::collect_intra=info`.

If you are developing rustdoc, use `RUSTDOC_LOG` instead. If you are developing
Miri, use `MIRI_LOG` instead. You get the idea :)

See the [`tracing`] crate's docs, and specifically the docs for [`debug!`] to
see the full syntax you can use. (Note: unlike the compiler, the [`tracing`]
crate and its examples use the `RUST_LOG` environment variable. rustc, rustdoc,
and other tools set custom environment variables.)

**Note that unless you use a very strict filter, the logger will emit a lot of
output, so use the most specific module(s) you can (comma-separated if
multiple)**. It's typically a good idea to pipe standard error to a file and
look at the log output with a text editor.

So, to put it together:

```bash
# This puts the output of all debug calls in `rustc_middle/src/traits` into
# standard error, which might fill your console backscroll.
$ RUSTC_LOG=rustc_middle::traits=debug rustc +stage1 my-file.rs

# This puts the output of all debug calls in `rustc_middle/src/traits` in
# `traits-log`, so you can then see it with a text editor.
$ RUSTC_LOG=rustc_middle::traits=debug rustc +stage1 my-file.rs 2>traits-log

# Not recommended! This will show the output of all `debug!` calls
# in the Rust compiler, and there are a *lot* of them, so it will be
# hard to find anything.
$ RUSTC_LOG=debug rustc +stage1 my-file.rs 2>all-log

# This will show the output of all `info!` calls in `rustc_codegen_ssa`.
#
# There's an `info!` statement in `codegen_instance` that outputs
# every function that is codegen'd. This is useful to find out
# which function triggers an LLVM assertion, and this is an `info!`
# log rather than a `debug!` log so it will work on the official
# compilers.
$ RUSTC_LOG=rustc_codegen_ssa=info rustc +stage1 my-file.rs

# This will show the output of all `info!` calls made by rustdoc
# or any rustc library it calls.
$ RUSTDOC_LOG=info rustdoc +stage1 my-file.rs

# This will only show `debug!` calls made by rustdoc directly,
# not any `rustc*` crate.
$ RUSTDOC_LOG=rustdoc=debug rustdoc +stage1 my-file.rs
```

### Log colors

By default, rustc (and other tools, like rustdoc and Miri) will be smart about
when to use ANSI colors in the log output. If they are outputting to a terminal,
they will use colors, and if they are outputting to a file or being piped
somewhere else, they will not. However, it's hard to read log output in your
terminal unless you have a very strict filter, so you may want to pipe the
output to a pager like `less`. But then there won't be any colors, which makes
it hard to pick out what you're looking for!

You can override whether to have colors in log output with the `RUSTC_LOG_COLOR`
environment variable (or `RUSTDOC_LOG_COLOR` for rustdoc, or `MIRI_LOG_COLOR`
for Miri, etc.). There are three options: `auto` (the default), `always`, and
`never`. So, if you want to enable colors when piping to `less`, use something
similar to this command:

```bash
# The `-R` switch tells less to print ANSI colors without escaping them.
$ RUSTC_LOG=debug RUSTC_LOG_COLOR=always rustc +stage1 ... | less -R
```

Note that `MIRI_LOG_COLOR` will only color logs that come from Miri, not logs
from rustc functions that Miri calls. Use `RUSTC_LOG_COLOR` to color logs from
rustc.

### How to keep or remove `debug!` and `trace!` calls from the resulting binary

While calls to `error!`, `warn!` and `info!` are included in every build of the compiler,
calls to `debug!` and `trace!` are only included in the program if
`debug-logging=true` is turned on in config.toml (it is
turned off by default), so if you don't see `DEBUG` logs, especially
if you run the compiler with `RUSTC_LOG=rustc rustc some.rs` and only see
`INFO` logs, make sure that `debug-logging=true` is turned on in your
config.toml.

### Logging etiquette and conventions

Because calls to `debug!` are removed by default, in most cases, don't worry
about adding "unnecessary" calls to `debug!` and leaving them in code you
commit - they won't slow down the performance of what we ship, and if they
helped you pinning down a bug, they will probably help someone else with a
different one.

A loosely followed convention is to use `debug!("foo(...)")` at the _start_ of
a function `foo` and `debug!("foo: ...")` _within_ the function. Another
loosely followed convention is to use the `{:?}` format specifier for debug
logs.

One thing to be **careful** of is **expensive** operations in logs.

If in the module `rustc::foo` you have a statement

```Rust
debug!("{:?}", random_operation(tcx));
```

Then if someone runs a debug `rustc` with `RUSTC_LOG=rustc::bar`, then
`random_operation()` will run.

This means that you should not put anything too expensive or likely to crash
there - that would annoy anyone who wants to use logging for their own module.
No-one will know it until someone tries to use logging to find *another* bug.

## Formatting Graphviz output (.dot files)
[formatting-graphviz-output]: #formatting-graphviz-output

Some compiler options for debugging specific features yield graphviz graphs -
e.g. the `#[rustc_mir(borrowck_graphviz_postflow="suffix.dot")]` attribute
dumps various borrow-checker dataflow graphs.

These all produce `.dot` files. To view these files, install graphviz (e.g.
`apt-get install graphviz`) and then run the following commands:

```bash
$ dot -T pdf maybe_init_suffix.dot > maybe_init_suffix.pdf
$ firefox maybe_init_suffix.pdf # Or your favorite pdf viewer
```

## Viewing Spanview output (.html files)
[viewing-spanview-output]: #viewing-spanview-output

In addition to [graphviz output](#formatting-graphviz-output-dot-files), MIR debugging
flags include an option to generate a MIR representation called `Spanview` that
uses HTML to highlight code regions in the original source code and display
compiler metadata associated with each region.
[`-Z dump-mir-spanview`](./mir/debugging.md), for example, highlights spans
associated with each MIR `Statement`, `Terminator`, and/or `BasicBlock`.

These `.html` files use CSS features to dynamically expand spans obscured by
overlapping spans, and native tooltips (based on the HTML `title` attribute) to
reveal the actual MIR elements, as text.

To view these files, simply use a modern browser, or a CSS-capable HTML preview
feature in a modern IDE. (The default HTML preview pane in *VS Code* is known to
work, for instance.)

## Narrowing (Bisecting) Regressions

The [cargo-bisect-rustc][bisect] tool can be used as a quick and easy way to
find exactly which PR caused a change in `rustc` behavior. It automatically
downloads `rustc` PR artifacts and tests them against a project you provide
until it finds the regression. You can then look at the PR to get more context
on *why* it was changed.  See [this tutorial][bisect-tutorial] on how to use
it.

[bisect]: https://github.com/rust-lang/cargo-bisect-rustc
[bisect-tutorial]: https://github.com/rust-lang/cargo-bisect-rustc/blob/master/TUTORIAL.md

## Downloading Artifacts from Rust's CI

The [rustup-toolchain-install-master][rtim] tool by kennytm can be used to
download the artifacts produced by Rust's CI for a specific SHA1 -- this
basically corresponds to the successful landing of some PR -- and then sets
them up for your local use. This also works for artifacts produced by `@bors
try`. This is helpful when you want to examine the resulting build of a PR
without doing the build yourself.

[rtim]: https://github.com/kennytm/rustup-toolchain-install-master

## Debugging type layouts

The (permanently) unstable `#[rustc_layout]` attribute can be used to dump
the [`Layout`] of the type it is attached to. For example:

```rust
#![feature(rustc_attrs)]

#[rustc_layout(debug)]
type T<'a> = &'a u32;
```

Will emit the following:

```text
error: layout_of(&'a u32) = Layout {
    fields: Primitive,
    variants: Single {
        index: 0,
    },
    abi: Scalar(
        Scalar {
            value: Pointer,
            valid_range: 1..=18446744073709551615,
        },
    ),
    largest_niche: Some(
        Niche {
            offset: Size {
                raw: 0,
            },
            scalar: Scalar {
                value: Pointer,
                valid_range: 1..=18446744073709551615,
            },
        },
    ),
    align: AbiAndPrefAlign {
        abi: Align {
            pow2: 3,
        },
        pref: Align {
            pow2: 3,
        },
    },
    size: Size {
        raw: 8,
    },
}
 --> src/lib.rs:4:1
  |
4 | type T<'a> = &'a u32;
  | ^^^^^^^^^^^^^^^^^^^^^

error: aborting due to previous error
```

[`Layout`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_target/abi/struct.Layout.html
