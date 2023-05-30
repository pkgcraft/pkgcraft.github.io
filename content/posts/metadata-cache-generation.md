---
title: "Metadata cache generation"
subtitle: "Sailing through the doldrums of bash"
date: 2023-05-27T09:21:36-07:00
tags: ["bash"]
---

Bash is slow. Supporting a nested inheritance structure on top of bash makes it
even slower. Without metadata caches, processing ebuilds would be
extraordinarily more painful than it already is. Imagine the extra time it
would take to source all the ebuilds required to run a command such as `emerge
-e world` before dependency resolution can begin. Clearly the importance of
using a cache when dealing with a large corpus of highly nested bash code
cannot be understated.

For ebuild repos, the metadata cache is currently located at metadata/md5-cache
from the repo root and is populated by sourcing each ebuild with all its eclass
dependencies, tracking state over nested inherits for incrementally handled
variables, and then writing the relevant metadata key/value mapping to the
cache. A cache is updated by iterating over its entries and either comparing
file timestamps or using embedded checksums to see if an entry is outdated.

While loading package information for dependency resolution or any other
tree-wide activity, the cache is first accessed to determine if it has the
relevant information before falling back to generating the values. This allows
package managers and related tools to avoid sourcing ebuilds when using
using globally-scoped package metadata speeding up the process immensely.

# Package manager support

Those doing ebuild and/or eclass development should either be familiar with
portage's `egencache` or pkgcore's `pmaint regen` commands. Both provide
metadata cache generation functionality for their respective ecosystem and are
generally required when running on ebuild repos from git lacking bundled,
pre-generated metadata.

Different implementations do varying amounts of verification and processing
during metadata generation with both portage and pkgcore mainly treating the
data as raw strings performing little verification, instead farming that task
out to tools such as `pkgcheck`. Verification requires parsing the metadata
strings into their related data structures which can extend generation time too
much when using parsing routines written in slower languages. In general, this
is resolved by developers running tooling like pkgcheck on their local system
that verifies the data before commits are pushed to the public repo.

On the other side, pkgcraft currently tries to verify as much as possible
during metadata generation. In general this means any verification requiring
only localized knowledge will be performed, but anything requiring repo-wide
info such as dependency visibility is not. For example, all dependency fields
are parsed into their related data structures, catching any invalid formatting.

# Pkgcraft tooling

Pkgcraft supports interactive tooling under the `pk` command using subcommand
nesting so its related repo metadata tooling can be found under `pk repo
metadata`. To build `pk` from pkgcraft's git repo, use `cargo build --release
--features tools` or similar. Note that when not developing in rust, it's
better to build in release mode as the default debug mode for builds is often
quite a lot slower.

To use pkgcraft's implementation, run commands such as the following:

- configured repos: `pk repo -r gentoo metadata`
- external repos: `pk repo -r path/to/repo metadata`

Note that the repo argument must be specified and either be the name of a
configured ebuild repo on the system or an external repo pointed to via a path.
Specifying the level of parallelism is supported using the `-j/--jobs` option
which will default to using all available CPU cores when unspecified.

# Implementation details

In a technical sense, pkgcraft tries to avoid bash as much as possible. As
written about in a previous post[^post], all commands and functionality specified
by PMS are implemented as builtins natively in rust that are then given C
compatible wrappers and injected into a bundled version of bash. This allows
pkgcraft to handle tracking metadata using its own internal build state,
avoiding mangling bash variables over nested inherits where possible.

Parallelism is handled in a simplistic fashion in that the entire workflow --
validity checks, ebuild sourcing, metadata structure creation, and file
serialization -- is done in a forked process pool iterator. How it currently
works is raw, unsourced packages are iterated over, forking a new process for
each in a pool limited to a specific size using a bounded semaphore stored in
shared memory. Inside the forked process, the metadata workflow occurs from
with the result is encoded and sent back to the main process using IPC channels
that also work on top of shared memory[^ipc-channel]. The main process unwraps
the result, outputs the error if one occurred to stderr, and tracks the overall
regeneration status during the process.

For security purposes (and also because sandboxing isn't supported yet), ebuild
sourcing is run within a restricted shell environment. This rejects a lot of
functionality usually possible in a regular bash shell including not allowing
external commands to be run. To see the entire list (with minor
differences[^rbash]), run `man rbash`. In addition, sourcing functionality is
configured to act similar to `set -e` being enabled meaning any command failing
in global scope will cause a package to fail sourcing. Overall, the stricter
environment has highlighted a lot of the more questionable bash usage in the
tree such as using `return 0` or declaring local variables in global scope.

# Benchmarks and performance

Rough benchmarks comparing the metadata generation implementations of portage,
pkgcore, and pkgcraft on a modest laptop with 8 cores/16 threads (AMD Ryzen 7
5700U) running against a semi-current gentoo tree with no metadata on top of an
SSD are as follows:

- portage: `egencache -j16` -- approximately 5 minutes
- pkgcore: `pmaint regen -t 16` -- approximately 1 minute, 45 seconds
- pkgcraft: `pk repo metadata -j16` -- approximately 1 minute

From these results, it's clear that portage's main weakness of entirely
respawning bash causes it to lag far behind the other two. The process spawning
overhead is so dominant that running egencache using 16 jobs is roughly
equivalent to running `pmaint regen` using 2-3 jobs or `pk repo metadata` using
1-2 jobs. Due to this, my blunt advice is to avoid using egencache if full repo
metadata generation performance is important to you, especially when running on
older or slower hardware.

On the other hand, pkgcore performs relatively well due to leveraging a bash
daemon while using subshells (forked processes) to generate metadata thus
avoiding most of the unnecessary process overhead done by portage.

Pkgcraft is the fastest by a significant margin while still doing the most
verification work of the three; however, it has the advantage that none of the
underlying bash support is natively written in bash and also currently limits
IPC overhead to the relatively minimal encoding of results, relying on the
operating system's copy-on-write support for forked process pages.

# Package manager compatibility

While all three implementations work from the same specification, none of them
generate output that entirely matches the others. In pkgcraft's case, since it
parses metadata fields internally, it reorders or drop duplicates where
possible since many of the underlying data structures are ordered sets. It also
appears to using slightly different eclass inheritance ordering for the related
field in cases where there are circular inherits.

Beyond file serialization, pkgcraft is also stricter in what it allows in
global scope whether in ebuilds or eclasses due to its bash configuration as
mentioned previously. This currently causes a decent amount of ebuilds to fail
when generating metadata due to command failures in eclass global scope. It is
unlikely portage or pkgcore will ever be able to catch the same level of issues
since they run their metadata generation natively in function scope which hides
many of the issues that can only be seen when running in global scope. One way
to catch them would be to leverage an external bash parser to point them out
during `pkgcheck` runs.

Even with these differences, it should be noted that all package managers
should be able to use the others generated cache output if they adhere to the
specified format. This means it's possible to generate metadata more quickly
using pkgcore and pkgcraft that is then used with portage.

# This is fast... but can it go faster?

There are potentially a few optimizations that might shave more time off a full
tree regen run. On the bash side, every time a builtin may be called a binary
search is run across the registered builtins to determine if one exists.
Replacing those lookups with hash table queries probably could speed up the
execution pipeline for bash code using large amounts of commands.

In addition, initial work has been done to allow bash processes to reset their
internal state in order to allow reuse rather than leveraging subshells
internally or forked processes externally. The idea being that instead of
forking a new process per ebuild, a process pool could reuse its processes by
resetting their state thus saving time by avoiding the underlying OS's fork()
setup time. However, this capability might not come to fruition as its
difficult to make bash properly reset its state[^globals] after errors or signals
occur that cause nonlocal gotos via longjmp().

As mentioned previously, pkgcraft currently parallelizes this process using a
simplistic forked process pool that runs the entire workflow inside it.
Technically, only the sourcing and structure creation should required to run in
a separate process so it could potentially be faster to use parallelized thread
pools for validity and file serialization. However, I imagine the dominant
runtime factor is bash sourcing so this shouldn't be significant for full regen
runs, but could speed up fully up-to-date repo verifications.

# Future metadata-related work

### Language bindings

Beyond using `pk repo metadata`, language bindings should be able to natively
trigger metadata regen for relevant repos. This mainly entails adding support
into pkgcraft-c and determining what level of runtime feedback it provides,
e.g. it could return an iterator of errors, accept a callback function for
error handling, or just use a simple error status. Once that work is complete,
then language bindings can wrap it allowing native metadata regeneration.

### Tooling extensions

As with all programming languages, it is easy to write poorly performant code
in bash. It's probably fair to say shell coding exacerbates that because most
people's interactions with it are either when using an interactive shell or
writing one-off scripts, performance rarely being a priority in either case.
This often leads to inadvertently adding poorly performing code including
extensive subshell or external command use, processing large amounts of
strings, or poorly written loop structures encompassing the previous two
variants.

To help developers debug potential performance issues more subcommands for `pk`
are being looked at as well. Currently the idea is that `pk pkg env` could be
added to provide source timings for given restrictions, allowing debugging
against a single package, an entire category, or the set of packages using a
specific eclass. Furthermore, environment mangling and dumping would be
supported so, for example, all the packages using a specific eclass could be
sourced and dump a specific variable from the sourced environment. This kind of
tooling should give developers more insight into the metadata generation
process and how different types of coding structures affect it.

[^post]: https://pkgcraft.github.io/posts/rustifying-bash-builtins/
[^ipc-channel]: https://crates.io/crates/ipc-channel
[^rbash]: Pkgcraft's bundled version of bash allows redirections to /dev/null in
    restricted mode while bash does not.
[^globals]: Bash intertwines global state everywhere throughout its parser and
    execution pipeline.
