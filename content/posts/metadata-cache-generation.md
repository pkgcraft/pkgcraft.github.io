---
title: "Metadata cache generation"
subtitle: "Sailing through the doldrums of bash"
date: 2023-05-27T09:21:36-07:00
tags: ["bash"]
---

Bash is slow. Supporting a nested inheritance structure on top of bash makes it
even slower. Without metadata caches, processing ebuilds would be
extraordinarily more painful than it already is. Imagine the extra time it
would take to source all relevant ebuilds while resolving dependencies. Clearly
the importance of using a cache when dealing with a large corpus of highly
nested bash code cannot be understated.

For ebuild repos, the metadata cache is currently located at metadata/md5-cache
from the repo root and is created by sourcing each ebuild with all its eclass
dependencies, tracking state over nested inherits for incrementally handled
variables, and then writing the relevant metadata key/value mapping to the
cache. An existing cache is updated by iterating over its entries, comparing
checksums to determine validity, and performing entry creation on failure.

While loading package information for dependency resolution or any other
tree-wide activity, the cache is first accessed to determine if it has the
relevant information before falling back to generating the values. This allows
package managers and related tools to avoid sourcing ebuilds when using
using globally-scoped package metadata speeding up the process immensely.

# Package manager support

Those doing ebuild development should be familiar with either portage's
`egencache` (`emerge --regen`) or pkgcore's `pmaint regen` commands. Both
provide metadata cache generation functionality for their respective ecosystem
and are generally required when running on ebuild repos lacking bundled,
pre-generated metadata.

Metadata generation implementations handle verification differently with both
portage and pkgcore treating the data as raw strings, performing few validity
checks. Proper verification requires parsing metadata strings into relevant
data structures significantly extending generation time when written in slower
languages. This is currently resolved using other development tools (such as
pkgcheck) that perform the metadata parsing, flagging errors as they arise.

Conversely, pkgcraft aims to perform as much verification as much as possible
during generation. Most verification using localized knowledge is performed,
but more performance intensive checks requiring repo-wide info such as
dependency visibility are not. For example, all dependency fields are parsed
into their related data structures, catching any invalid formatting.

# Pkgcraft tooling

Pkgcraft provides various command-line tools under the `pk` command from the
pkgcraft-tools crate with metadata generation support via `pk repo metadata regen`.

### Install

Current release: `cargo install pkgcraft-tools`

From git: `cargo install pkgcraft-tools --git https://github.com/pkgcraft/pkgcraft.git`

Pre-built binaries are also provided for [releases on supported
platforms](https://github.com/pkgcraft/pkgcraft/releases).

### Usage

Incrementally generate metadata for the configured `gentoo` repo:

- `pk repo metadata regen gentoo`

Force a full regen:

- `pk repo metadata regen -f path/to/repo`

Note that a repo must be specified and either be the name of a configured
ebuild repo on the system or an external repo pointed to via a path. Specifying
the level of parallelism is supported using the `-j/--jobs` option which
defaults to the system's number of logical CPU cores when unset.

# Implementation details

In a technical sense, pkgcraft avoids bash as much as possible. As described in
the post on [rustifying
bash](https://pkgcraft.github.io/posts/rustifying-bash-builtins/), all
bash-related functionality is implemented in rust building on top of a bundled
version of bash. This allows tracking metadata using more flexible and
efficient data structures than bash variables.

Parallelism is handled in a simplistic fashion by running the entire workflow
-- validity checks, ebuild sourcing, metadata structure creation, and file
serialization -- in a forked process pool. Each process runs the metadata
workflow for a single package, encoding the result and sending it via
inter-process communication (IPC) to the main process that unwraps it, handling
error logging while tracking overall status. After all packages are processed
the main process exits, failing if any errors occurred.

For security purposes (and because sandboxing isn't supported yet), ebuild
sourcing is run within a restricted shell environment. This rejects a lot of
functionality possible in regular bash including running external commands. To
see the entire list (with minor differences[^rbash]), run `man rbash`. In
addition, sourcing functionality is configured to act similar to `set -e` being
enabled, but instead triggered by internal bash errors and not command exit
statuses. Overall, the stricter environment has highlighted many instances of
questionable bash usage in the tree such as using `return 0` or declaring local
variables in global scope.

# Benchmarks and performance

Rough benchmarks comparing the metadata generation implementations of portage,
pkgcore, and pkgcraft on a modest laptop with 8 cores/16 threads (AMD Ryzen 7
5700U) running against a semi-current gentoo tree with no metadata on top of an
SSD are as follows:

- portage: `egencache -j16` -- approximately 5m
- pkgcore: `pmaint regen -t 16` -- approximately 1m20s
- pkgcraft: `pk repo metadata regen -j16` -- approximately 30s

For comparative parallel efficiency, pkgcraft achieves the following when using
different amounts of jobs:

- pkgcraft: `pk repo metadata regen -j8` -- approximately 40s
- pkgcraft: `pk repo metadata regen -j4` -- approximately 1m
- pkgcraft: `pk repo metadata regen -j2` -- approximately 2m
- pkgcraft: `pk repo metadata regen -j1` -- approximately 4m

For a valid metadata cache requiring no updates:

- portage: `egencache -j16` -- approximately 7s
- pkgcore: `pmaint regen -t 16` -- approximately 14s
- pkgcraft: `pk repo metadata regen -j16` -- approximately .2s

Note that these results are approximated averages for multiple runs without
flushing memory caches. Initial runs of the same commands will be slower from
additional I/O latency.

From the results above, the effects from one of portage's design flaws can
clearly be seen. Any time the bash side of portage is used, a new bash instance
is started. The process spawning overhead is so dominant that running
`egencache` using 16 jobs is roughly equivalent to running `pmaint regen` using
2-3 jobs and even slower than `pk repo metadata regen` using a single job. Due to
this, it's best to avoid portage's metadata generation support if performance
is a priority, especially on slower hardware.

In constrast, pkgcore performs relatively well due to leveraging a bash daemon
that spawns subshells (forked processes) to generate metadata thus avoiding
most of the unnecessary process overhead. This approach could be copied into
portage to provide the same benefits, but that would require extensive rework
of the bash functionality and IPC interface.

Pkgcraft is the fastest by a significant margin while also doing the most
verification of the three; however, it has the advantage that the underlying
support isn't written in bash and currently limits IPC overhead to the
relatively minimal encoding of results, relying on the operating system's
copy-on-write support for forked memory pages to "transfer" data into each
process.

# Package manager compatibility

While all three projects implement the same specification, none produces
matching output. In pkgcraft's case metadata fields are parsed internally,
reordering and removing duplicates where possible since many of the underlying
data structures are ordered sets. It also uses different ordering for circular
eclass inherits.

Beyond file serialization, pkgcraft is stricter in what it allows in global
scope whether in ebuilds or eclasses due to its bash configuration as mentioned
previously. This currently causes a decent amount of ebuilds to fail when
generating metadata due to command failures in global scope. It is unlikely
portage or pkgcore will ever be able to catch the same level of issues since
they are designed to run their metadata generation natively in function scope
which hides many of the issues. One alternative would be to leverage an
external bash parser flagging issues during `pkgcheck` runs.

Even with these differences, package managers should be able to use caches
created via any implementation if they adhere to the specified format. This
means it's possible to generate metadata using quicker solutions, pkgcore or
pkgcraft, that is then used with portage.

# This is fast... but can it go faster?

There are potentially a few optimizations that might shave more time off a full
tree regen run. On the bash side, every time a builtin may be called a binary
search is run across the registered builtins to determine if one exists.
Replacing those lookups with hash table queries could potentially speed up the
execution pipeline for bash code using large amounts of commands if the hash
function used[^fnv] runs in less time than a binary search. Including
pkgcraft's builtins, a worst case binary search takes approximately seven steps
so this work may not be worth it.

In addition, initial work has been done to allow bash processes to reset their
internal state in order to allow reuse rather than leveraging subshells
internally or forked processes externally. The idea being that instead of
forking a new process per ebuild, a process pool could reuse its processes by
resetting their state thus saving time by avoiding the underlying operating
system's fork() setup time. However, this capability might not come to fruition
as its difficult to make bash properly reset its state[^globals] after errors
or signals occur that cause nonlocal gotos via longjmp().

As mentioned previously, pkgcraft currently parallelizes metadata generation
with a simplistic pool iterator using a forked process per package.
Technically, only sourcing is required to run in a separate process so it may
be faster to use thread pools for validity, structure creation, and file
serialization. However, the dominant runtime factor for full regen runs is bash
sourcing so this shouldn't be significant, but could speed up repo
verification.

# Future metadata-related work

### Language bindings

Beyond using `pk repo metadata regen`, language bindings should be able to natively
trigger metadata regen for relevant repos. This mainly entails adding support
into pkgcraft-c and determining what level of runtime feedback it provides,
e.g. it could return an iterator of errors, accept a callback function for
error handling, or just use a simple error status. Once that work is complete,
then language bindings can wrap it allowing native metadata regeneration.

### Tooling extensions

As with all programming languages, it's easy to write poorly performant code in
bash. It's probably fair to say shell coding exacerbates the situation because
most interactions with it are interactive shell usage or one-off scripts,
performance rarely being a priority in either case. This often leads to
inadvertently adding poorly performing code including extensive subshell or
external command use, processing large amounts of strings, or poorly written
loop structures encompassing the previous two variants.

To help developers debug potential performance issues more subcommands for `pk`
are being looked at as well. Currently the idea is that `pk pkg env` could be
added to provide source timings for given restrictions, allowing debugging
against a single package, an entire category, or the set of packages using a
specific eclass. Furthermore, environment mangling and dumping would be
supported so, for example, all the packages using a specific eclass could be
sourced and dump a specific variable from the sourced environment. This kind of
tooling should give developers more insight into the metadata generation
process and how different types of coding structures affect it.

[^rbash]: Pkgcraft's bundled version allows redirections to /dev/null in
    restricted mode while vanilla bash does not.
[^fnv]: Bash uses
    [FNV-1](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function)
    for hash table support.
[^globals]: Bash intertwines global state everywhere throughout its parser and
    execution pipeline.
