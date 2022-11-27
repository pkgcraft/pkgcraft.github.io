---
title: "Extending bash"
subtitle: "Mangling the shell for better integration"
date: 2022-01-25T01:29:42-07:00
tags: ["bash"]
---

Basing a package manager and related specification on bash is a mistake.
Familiarity and hackability are great in the short-term, but as the novelty
wears off it becomes clear that maintainability, efficiency, and other
attributes hard to wrest from bash's rigid design all suffer. As long as
compatibility remains important for pkgcraft that decision can't be altered;
however, that doesn't mean nothing can be done to improve the situation.

One option is to go pkgcore's route, using IPC with a bash daemon. This enables
sharing bash processes between separate tasks rather than relying on the
simplistic exec-per-use scheme exhibited by portage. Among other effects, this
makes pkgcore's metadata generation approximately five times faster than its
competitor. While providing many marginal improvements, this daemonized
approach still doesn't escape the restrictive boundaries of regular shell
usage. Among other downsides, pkgcore requires subshells (meaning additional
processes) to avoid environment leaks during metadata generation and uses hacky
RPC signaling across pipes since it's hard to work with anything else natively
in bash.

Pkgcraft aims to move beyond daemon functionality and achieve better lower
level integration. The dream of replacing bash with something threadable and
modern is enticing, but it's fairly impossible in the short-term and thus
disregarded. Instead, pkgcraft dives directly into the pit of insanity; it
forks bash[^1] in an effort to achieve its goals.

### Parallelism problems

Since pkgcraft goes to the extent of forking bash, it also takes on the various
deficiencies that hinder its use as a library. For a start, bash is not
thread-safe or reentrant at all. The current design uses an extensive amount of
global mutables to track both its parser and shell state. Bash uses bison which
supports generating reentrant parsers, but the shell itself requires extensive
rework for that to be feasible on a global scale.

With that in mind, in order to support parallel usage a process pool or similar
design must be used. Currently this isn't something that can easily be dropped
into place like python's multiprocessing pool support. Parallelism in rust
centers around threading since its memory safety through enforced lifetimes
highlights threaded execution that's guaranteed data-race free. This means that
most data parallelism support similar to rayon focuses on threaded operation,
disregarding multi-process support entirely. At some point, pkgcraft will have
to address this and probably create its own pool or parallelized iterator
support that reuses processes.

### Error handling

Beyond parallelism issues, bash leverages longjmp() and frame unwinding for its
error handling. This is understandable due to its age, chosen language, and
minimal dependencies, but doesn't lend itself well to interoperability with
rust. For pkgcraft, bash is wrapped where its C code is called from the rust
library and then it can call back into rust support exported to C. The issue
with that is unwinding across rust-based frames from C is undefined behavior.
Hopefully at some point the "C-unwind" ABI[^2] and other related FFI work in
rust is stabilized potentially helping solve the issue in a better fashion.

With respect to longjmp() usage, in order to avoid null jump buffers the forked
version of bash tries to establish jump targets on the main entry points used
by the rust library. This isn't necessary for standard bash because its top
level jump points are all in main() which isn't used when built as a library.
Without this, most error handling segfaults when using `set -e` because bash
tries to jump to an undefined location due to the empty jump buffer.

To cap off its unfriendly error handling, bash generally dumps all its error
messages to stderr which clearly isn't wanted when used as a library. To avoid
this, the rust library passes in callbacks for error and warning handling that
bash calls, passing the raw messages back to rust which are then converted into
native rust errors or logged.

The remaining issue is that unless something like `set -e` is being used on the
bash side, most errors do not cause immediate returns, exits, or any trigger
that could be used on the rust side to return a corresponding error result. To
alleviate this, the most recent bash error is stored on the rust side in a
thread-local variable that can be accessed after relevant calls in order to
determine their error status. This allows functionality such as returning error
results on failed `source` calls while retaining the bash error message without
having to use `set -e`, subshells, and redirection in order to achieve a
similar effect in native bash.

### Leveraging builtins

In terms of extensibility, bash provides support for writing builtins that can
be called like any other command. For example, `set`, `local`, `echo`, and many
more commands provided by bash are builtins. Pkgcraft intends to use builtins
for all the commands that would either be exposed as functions or other public
callables. All other internal functionality will be implemented as methods
on the shell instance wrapping the bash library.

The difficulty comes with sharing state across the FFI border since the
builtins are written in rust, but are called from C in bash. Therefore it's not
easy to write them in a fashion that allows reuse and inter-builtin calls while
also passing some form of mutable context parameter. Once again, pkgcraft uses
a mutable, thread-local instance that builtins are able to import and use
within closures to access and modify build data as required. While somewhat
ugly, this does allow avoiding the even uglier bash variable hacks used by
pkgcore.

[^1]: Pkgcraft's bash fork is available at https://github.com/pkgcraft/bash.
[^2]: https://rust-lang.github.io/rfcs/2945-c-unwind-abi.html
