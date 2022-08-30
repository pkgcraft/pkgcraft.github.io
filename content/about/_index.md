---
date: 2022-01-18T20:24:13-07:00
title: About Pkgcraft
meta: false
---

# FAQ

- ### Why does this project exist? Isn't portage good enough?

Portage's inner workings can be succinctly described as a towering pile of
spaghetti largely composed of technical debt and inefficiency, its main upsides
being user-interface familiarity and years of accumulated bug fixes. While its
design has been criticized over the years, Gentoo has mostly ignored portage's
underlying issues leading to [pkgcore](https://github.com/pkgcore) and
[paludis](https://paludis.exherbo.org/) arising, various fallout such as
repoman getting replaced by [pkgcheck](https://github.com/pkgcore/pkgcheck),
and a general restraint on progress since few developers volunteer to untangle
code in order to implement complex features or markedly improve the situation.

Pkgcraft aims to take a different approach, supporting language bindings on top
of a core library allowing developers to take advantage of optimized
functionality rather than being forced to reimplement it. This hopefully aids
code reuse and decreases development time for third parties while keeping focus
on improving pkgcraft's API and related documentation.

Beyond bindings, it also experiments with ideas including bundling an extended
version of bash, providing greater efficiency and allowing static binaries for
the package manager itself (which is semi-relevant for source-based package
managers).

- ### Is this a portage rewrite or is pkgcraft targeting replacing portage?

No, while both projects implement functionality described in
[PMS](https://wiki.gentoo.org/wiki/Package_Manager_Specification), pkgcraft
does not plan to duplicate portage's interface or feature set. At its most
basic, it intends to be a base to build more efficient tools. This does allow
for a future in which portage or other established tools leverage pkgcraft to
improve their capabilities, but that is entirely up to the whims of Gentoo
and other developers.

In pkgcraft's case, it may develop alternatives encompassing use cases
currently provided by traditional, Gentoo-related tools, but the majority of
that work is a long way off and probably won't aim for compatibility beyond PMS
support.

- ### Why isn't pkgcraft implemented in C, C++, Python, etc? Why choose Rust?

Choosing Rust was mostly a pragmatic decision. I wanted to use a compiled
programming language meeting the following requirements:

1. decent memory safety guarantees
2. active community improving the language and core implementation
3. able to work with and create efficient, C compatible libraries
4. relatively large, native library ecosystem

Narrowing the field with those priorities, the main candidates are currently
Rust and Go with other stragglers failing for various reasons including Nim,
Zig, and probably some subset of C++. From those two options, Go is too
restrictive for how I wanted to develop pkgcraft and doesn't achieve the same
level of C support allowed by Rust.

Rust's main weaknesses in relation to pkgcraft's goals are probably its current
lack of minor architecture support (compared to C and C++) and its steep
learning curve.

With regards to architecture support, I think this will be resolved with more
time if the language maintains or continues growing its popularity via projects
like the GCC front-end for Rust and/or porting LLVM to more architectures.

In terms of a steep learning curve in comparison to something like Python, this
isn't historically relevant for projects related to Gentoo package management.
Or stated another way, nearly all major development for package managers
targeting Gentoo has been done by singular teams regardless of implementation
language. Those "teams" have changed over time, but rarely are there multiple
developers doing large, sustained amounts of work on the same project. In
short, my response to the steep learning curve is that pkgcraft aims to provide
bindings for "easier" languages that may be used to avoid rust-based
development.

# Project goals

## Short-term

- Wrap bash in a rust-based library allowing process reuse and optional static
  binary build support.

- Write all native bash functionality that would otherwise be required using
  rust-based builtins and shell functionality.

## Long-term

- Develop a threaded, semi-modular resolver and constraint framework that
  provides a more flexible approach to dependency resolution.

- Support various frontends for package building. Currently a simple
  command-line tool is planned but an interactive terminal interface and web
  frontend would be great.

- Provide bindings that allow users to hook into pkgcraft-functionality from
  their language of choice. Currently basic C and Python bindings exist
  wrapping a minimal set of package atom features.

- Support building custom targets, e.g. containers, images, tarballs, etc.

- Develop a new sandboxing framework for segregating package builds from the
  system.

## Dreams

- Replace bash with something threadable, extensible, and modern.
