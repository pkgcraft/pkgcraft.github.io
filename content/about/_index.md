---
date: 2022-01-18T20:24:13-07:00
title: About Pkgcraft
meta: false
---

# FAQ

## Why does this project exist? Isn't portage good enough?

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

## Is this a portage rewrite or is pkgcraft targeting replacing portage?

While both projects implement functionality described in
[PMS](https://wiki.gentoo.org/wiki/Package_Manager_Specification), pkgcraft
does not plan to duplicate portage's interface or feature set. At its most
basic, it intends to be a base for building efficient tools. This does allow
for a future in which portage or other projects leverage pkgcraft to improve
their capabilities, but that is entirely up to the whims of Gentoo and
other developers.

In pkgcraft's case, it may develop alternatives encompassing use cases
currently provided by established tools, but the majority of that work is a
long way off and probably won't aim for compatibility beyond PMS support. For
example, the planned design for the pkgcraft's package manager will be that of
a build daemon supporting various front-ends, encompassing a much broader set
of use cases than can be performed by portage itself. In theory, it could act
both as a replacement for [catalyst](https://wiki.gentoo.org/wiki/Catalyst) and
as a tinderbox while also enabling more exotic features such as allowing the
dep tree for a running build to be mangled and recalculated on the fly.

## Why isn't pkgcraft implemented in C, C++, Python, etc? Why choose Rust?

Choosing Rust was a pragmatic decision using the following requirements:

1. compiled, statically-typed language
2. decent memory safety guarantees
3. active community improving the language and core implementation
4. able to work with and create efficient, C compatible libraries
5. relatively large, native library ecosystem

Narrowing the field with those priorities, Rust is the only candidate that
fulfills them all. For more background info, the following list includes
considered alternatives along with some of the reasons why they were rejected:

- C --- Lacks any memory safety guarantees and requires far too much work to do
  high level coding without leveraging all sorts of libraries. However, it is
  decent as a glue layer used to provide language bindings as it's the lingua
  franca of software.

- C++ --- Like C, it also lacks memory safety guarantees without strict limits
  and guidelines for a project. In addition, while its community has done a
  decent job at improving the language over the years (compared to the likes of
  C) the language has a large tail of accumulated baggage.

- Go --- While good for writing services and tools, it doesn't come close to
  matching Rust's level of C compatibility and low level support in general.

- Nim/Zig --- Both of these are interesting as C or C++ replacements, but they
  lack the community depth that Rust currently has and thus have smaller third
  party ecosystems.

- Python/Ruby/etc --- Any dynamically typed, scripting languages fail to easily
  support efficient bindings for other languages and aren't performant enough
  in certain cases, requiring to implement extensions in external, compiled
  languages anyway.

The main detriments of Rust in relation to pkgcraft's aspirations are its
current lack of minor architecture support (compared to C and C++) and its
steep learning curve, neither of which precludes pkgcraft from reaching its
goals.

With regards to architecture support, this may be resolved with more time via
projects like the [GCC front-end for Rust](https://github.com/Rust-GCC/gccrs)
and general LLVM porting work especially if Rust's corporate popularity
continues to grow, providing more opportunities for funding.

In terms of a steep learning curve in comparison to something like Python, this
isn't historically relevant for projects related to Gentoo package management.
Or stated another way, nearly all major development for package managers
targeting Gentoo has been done by singular teams regardless of implementation
language. Those "teams" have changed over time, but rarely are there multiple
developers doing large, sustained amounts of work on the same project. In
short, pkgcraft's response to the steep learning curve is that it aims to
provide bindings for "easier" languages that may be used to avoid rust-based
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
