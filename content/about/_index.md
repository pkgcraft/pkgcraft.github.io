---
date: 2022-01-18T20:24:13-07:00
title: About Pkgcraft
meta: false
---

# FAQ

## Why does this project exist? Isn't portage good enough?

Portage's inner workings can be bluntly described as a towering pile of
spaghetti largely composed of technical debt and inefficiency, its main upsides
being user-interface familiarity and years of accumulated bug fixes. While its
design has been criticized over the years, Gentoo has mostly ignored portage's
underlying issues leading to [pkgcore](https://github.com/pkgcore) and
[paludis](https://paludis.exherbo.org/) arising, repoman getting replaced by
[pkgcheck](https://github.com/pkgcore/pkgcheck), and a general restraint on
progress since few developers volunteer to untangle code in order to implement
complex features or markedly improve the situation.

Pkgcraft takes a different approach, supporting language bindings on top of a
core library allowing developers to take advantage of optimized functionality
rather than being forced to reimplement it. This hopefully aids code reuse and
decreases development time for third parties while keeping focus on improving
pkgcraft's API and related documentation.

Beyond bindings, it also experiments with ideas including bundling an extended
version of bash, enabling greater efficiency and allowing the possibility of
static binaries for any tools based on it.

## Is this a portage rewrite or is pkgcraft targeting replacing portage?

This is not an official Gentoo project and has no plans to become one. As such,
there is little interest in reimplementing portage-specific functionality for
interoperability. While both projects implement Gentoo's [package manager
specification](https://wiki.gentoo.org/wiki/Package_Manager_Specification),
pkgcraft does not plan to duplicate portage's interface or feature set.
Instead, it intends to be a base for building efficient tools. This does allow
for a future in which portage leverages pkgcraft to improve its capabilities,
but that is entirely up to the whims of portage developers as this project
isn't planning to contribute to portage in any way.

That being said, alternatives may be developed that encompass use cases
currently provided by established tools, but the majority of that work is a
long way off and won't aim for compatibility beyond supporting most of the
official specification. For example, the planned design for the pkgcraft's
package manager will be that of a build daemon supporting various front-ends,
encompassing a much broader set of use cases than can be performed by portage
itself. In theory, it could act both as a replacement for
[catalyst](https://wiki.gentoo.org/wiki/Catalyst) and as a tinderbox while also
enabling more exotic features such as allowing the dep tree for a running build
to be mangled and recalculated on the fly.

## Why can't this project be merged with pkgcore (or paludis)?

As explained in the previous answers, the overarching design doesn't mesh well
with existing projects. Pkgcore began as a direct offshoot of portage and so
copied many of portage's decisions, albeit in a more optimized fashion. It has
successfully shown what a cleaner portage design could be like, but it's also
locked into the Python ecosystem and all that entails. On the other hand,
paludis comes closer to pkgcraft's vision as it includes several language
bindings; however, it has lagged behind newer EAPIs and doesn't allow for
nearly as much experimentation as pkgcraft.

While it's hard to directly merge development efforts, specifications and
defined formats should generally be implemented aiding cooperation where
possible allowing other tools to build on top of pkgcraft if they wish.
Unfortunately, most capabilities beyond direct ebuild interaction are entirely
unspecified so pkgcraft won't be interchangeable with its alternatives when
package manager tooling arrives.

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
  and guidelines for a project.

- Go --- It mostly feels targeted towards services and tooling, leaving C
  compatibility and low level support lacking.

- D/Nim/Zig --- These may be interesting as C or C++ replacements, but they
  currently have minimal community depth with relatively small third party
  ecosystems and lack the same level of memory safety or force the use of a
  garbage collector.

- Python/Ruby/etc --- Any dynamically typed, scripting languages fail to
  support efficient bindings for other languages and aren't performant enough
  in some cases, requiring extensions implemented in compiled languages.

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
targeting Gentoo has been done by individuals regardless of implementation
language. Those individuals have changed over time, but rarely are there
multiple developers doing large amounts of sustained, new development on the
same project. In short, pkgcraft's response to the steep learning curve is that
it aims to provide bindings for many languages letting developers use what
works best for them.

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
