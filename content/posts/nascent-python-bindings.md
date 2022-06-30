---
title: "Binding the World"
subtitle: "Developing python language bindings for pkgcraft"
date: 2022-06-29T11:29:42-07:00
tags: ["python", "pkgcraft", "Gentoo"]
---

One of Gentoo's major weaknesses is the lack of a shared implementation that
natively supports bindings to other languages for core, specification-level
features such as dependency format parsing. Due to this deficiency, over the
years I've seen the same algorithms implemented in Python, C, Bash, Go, and
more at varying levels of success.

Now, I must note that I don't mean to disparage these efforts especially when
done for fun or to learn a new language; however, it often seems they end up
being used in tools or services used by the wider community. Then as the
specification slowly evolves and original authors move on, developers are stuck
maintaining multiple implementations if they want to keep the related tools or
services relevant.

In an ideal world, the canonical implementation for a core feature set is
written in a language that can be easily bound by other languages offering
developers the choice to reuse this support without having to write their own.
To exhibit this possibility, one of pkgcraft's goals is to act as a core
library supporting language bindings.

### Design

Interfacing rust code with another language often requires a C wrapper library
in order to perform efficiently and/or sidestep rust's lifetime model that
clashes with more ownership-based languages. Bindings then build on top of this
C layer, allowing ignorance of the rust underneath.

For pkgcraft, this C library is provided via
[pkgcraft-c](https://github.com/pkgcraft/pkgcraft-c), currently wrapping
pkgcraft's core depspec functionality (package atoms) in addition to providing
the initial interface for config, repo, and package interactions.

For some languages it's also possible to develop bindings or support directly
in rust. There are a decent number of currently evolving, language-specific
projects that allow non-rust language development including pyo3 for python,
rutie for ruby, neon for Node.js, and others. These projects generally wrap the
unsafe C layer internally, allowing for simpler development. Generally
speaking, I recommend going this route if performance levels and project goals
can be met.

Originally, pkgcraft used [pyo3](https://github.com/PyO3) for its python
bindings. If one is familiar with rust and python, the development experience
is relatively pleasant and allows simpler builds using maturin rather then the
pile of technical debt that distutils, setuptools, and its extensions provide
when trying to do anything outside the ordinary.

However, pyo3 has a couple, currently unresolved issues that lead me to abandon
it. First, the speed of its class instantiation is barely equal or slower than
a native python implementation, even for simple classes. It should be noted
this is only important if your design is such that you must support creating
thousands of native object instances at a python level. It can be possible to
lower this overhead by only exposing functionality to interact with large
groups of rust objects. Not to mention, for most developers coming from native
python the performance hit won't be overly noticeable. In any case, class
instantiation overhead will probably decrease as the project matures and more
work is done on optimization.

More importantly, pyo3 does not support exposing any object that contains
fields using explicit lifetimes. This means any struct that contains borrowed
fields can't be directly exported due to the clashes between the memory models
and ownership designs of rust and python. It's quite possible to work around
this, but that often means copying data around in order for the python side to
obtain ownership. Whether this is acceptable will largely depend on how much
you feel the performance hit.

For my part, having experience writing native extensions using the CPython API
as well as cython, the workarounds necessary to avoid exposing borrowed objects
weren't worth the effort, especially because pkgcraft requires a C API anyway
to support C itself and languages lacking compatibility layer projects. Thus I
rewrote pkgcraft's python bindings using cython instead which immediately
raised performance near to levels I was initially expecting; however, the
downside is quite apparent since the bindings have to manually handle all the
type conversions and resource deallocation while calling through the C wrapper.
It's a decent amount more work, but I think the performance benefits are worth
it.

### Development

First, the tools for building the code should be installed. This includes a
recent rust compiler and C compiler. I leave it up to the reader to leverage
`rustup` and/or their distro's package manager to install the required build
tools (and others such as `git` that are implied).

Next, the pkgcraft code must be pulled down. The easiest way to do this is to
recursively clone pkgcraft-workspace which should include semi-recent
submodules for all the pkgcraft projects:

```bash
git clone --recursive-submodules https://github.com/pkgcraft/pkgcraft-workspace.git
cd pkgcraft-workspace
```

From this workspace, the pkgcraft-c project can be built and various shell
variables set in order to build python bindings via the following command:

```bash
$ source ./build pkgcraft-c
```

This builds pkgcraft into a shared library that is exposed to the python build
via setting `$LD_LIBRARY_PATH` and `$PKG_CONFIG_PATH`. Once that completes the
python bindings can be built and tested via tox:

```bash
$ cd pkgcraft-python
$ tox -e python
```

When doing development on bindings leveraging a C library it's also valuable to
run the same testsuite under valgrind looking for the seemingly inevitable
memory leaks, seemingly exacerbated by rust requiring all allocations to be
returned in order to be freed safely since it historically didn't use the
system allocator. For pkgcraft, this is provided via another tox target:

```bash
$ tox -e valgrind
```

If you're familiar with valgrind, we mainly care about the definitely and
indirectly lost categories of memory leaks, the other types relate to global
objects or caches that aren't explicitly deallocated on exit. The valgrind
target for tox should error out if any memory leaks are detected so if it
completes successfully no leaks were detected.

### Benchmarking vs pkgcore and portage

Stepping away from regular development towards more interesting data, pkgcraft
provides rough processing and memory benchmark suites in order to compare its
nascent python bindings with pkgcore and portage. Currently these only focus on
atom object instantiation, but may be extended to include other functionality
if the API access isn't too painful for pkgcore and/or portage.

To run the processing time benchmarks that leverage
[pytest-benchmark](https://pypi.org/project/pytest-benchmark) use:

```bash
$ tox -e bench
```

For a summary of benchmark results only including the mean and standard
deviation:

{{< rawhtml >}}
<style type="text/css">
.ansi2html-content { display: inline; white-space: pre-wrap; word-wrap: break-word; }
.ansi1 { font-weight: bold; }
.ansi31 { color: #aa0000; }
.ansi32 { color: #00aa00; }
.ansi33 { color: #aa5500; }
</style>
<pre class="ansi2html-content">
<span class="ansi33">----------------- benchmark 'test_bench_atom_random': 4 tests ------------------</span>
Name (time in us)                              Mean             StdDev
<span class="ansi33">--------------------------------------------------------------------------------</span>
test_bench_atom_random[pkgcraft-Atom]     <span class="ansi32"></span><span class="ansi1 ansi32">   4.5395 (1.0)    </span><span class="ansi32"></span><span class="ansi1 ansi32">   0.3722 (1.0)    </span>
test_bench_atom_random[pkgcraft-cached]   <span class="ansi1">   6.2360 (1.37)   </span><span class="ansi1">   1.3386 (3.60)   </span>
test_bench_atom_random[pkgcore-atom]      <span class="ansi1">  30.9767 (6.82)   </span><span class="ansi1">   1.1428 (3.07)   </span>
test_bench_atom_random[portage-Atom]      <span class="ansi31"></span><span class="ansi1 ansi31">  50.2636 (11.07)  </span><span class="ansi31"></span><span class="ansi1 ansi31">  19.7562 (53.07)  </span>
<span class="ansi33">--------------------------------------------------------------------------------</span>

<span class="ansi33">--------------------- benchmark 'test_bench_atom_static': 4 tests ----------------------</span>
Name (time in ns)                                  Mean                 StdDev
<span class="ansi33">----------------------------------------------------------------------------------------</span>
test_bench_atom_static[pkgcraft-cached]   <span class="ansi32"></span><span class="ansi1 ansi32">     217.2820 (1.0)    </span><span class="ansi32"></span><span class="ansi1 ansi32">       5.9821 (1.0)    </span>
test_bench_atom_static[pkgcraft-Atom]     <span class="ansi1">     725.2229 (3.34)   </span><span class="ansi1">      41.6775 (6.97)   </span>
test_bench_atom_static[pkgcore-atom]      <span class="ansi1">  28,331.4369 (130.39) </span><span class="ansi1">     942.0003 (157.47) </span>
test_bench_atom_static[portage-Atom]      <span class="ansi31"></span><span class="ansi1 ansi31">  33,794.6625 (155.53) </span><span class="ansi31"></span><span class="ansi1 ansi31">  14,358.8390 (&gt;1000.0)</span>
<span class="ansi33">----------------------------------------------------------------------------------------</span>

<span class="ansi33">----------------- benchmark 'test_bench_atom_sorting_best_case': 2 tests ----------------</span>
Name (time in us)                                        Mean            StdDev
<span class="ansi33">-----------------------------------------------------------------------------------------</span>
test_bench_atom_sorting_best_case[pkgcraft-Atom]   <span class="ansi32"></span><span class="ansi1 ansi32">    6.1195 (1.0)    </span><span class="ansi32"></span><span class="ansi1 ansi32">  0.2011 (1.0)    </span>
test_bench_atom_sorting_best_case[pkgcore-atom]    <span class="ansi31"></span><span class="ansi1 ansi31">  936.9403 (153.11) </span><span class="ansi31"></span><span class="ansi1 ansi31">  5.5534 (27.61)  </span>
<span class="ansi33">-----------------------------------------------------------------------------------------</span>

<span class="ansi33">---------------- benchmark 'test_bench_atom_sorting_worst_case': 2 tests -----------------</span>
Name (time in us)                                         Mean            StdDev
<span class="ansi33">------------------------------------------------------------------------------------------</span>
test_bench_atom_sorting_worst_case[pkgcraft-Atom]   <span class="ansi32"></span><span class="ansi1 ansi32">    6.2702 (1.0)    </span><span class="ansi32"></span><span class="ansi1 ansi32">  0.3301 (1.0)    </span>
test_bench_atom_sorting_worst_case[pkgcore-atom]    <span class="ansi31"></span><span class="ansi1 ansi31">  924.1410 (147.39) </span><span class="ansi31"></span><span class="ansi1 ansi31">  6.9942 (21.19)  </span>
<span class="ansi33">------------------------------------------------------------------------------------------</span>
</pre>
{{< /rawhtml >}}

From these results the pkgcraft python bindings instantiate atom objects about
5-6x faster than pkgcore and about 10x faster than portage. For static atoms
when using the cached pkgcraft implementation (for object reuse) this increases
to about 150x faster than portage meaning that portage should probably look
into using an LRU cache for directly created atom objects. With respect to
pkgcore's static result, it also appears to not use caching; however, it does
support atom instance caching internally so the benchmark is avoiding that
somehow.

When comparing sorting, pkgcraft is well over two orders of magnitude ahead of
pkgcore and I imagine portage would fare even worse, but it doesn't natively
support atom object comparisons so isn't included here.

To run the memory benchmarks use:

```bash
$ tox -e membench
```

Which currently produces output similar to:

```
Static atoms (1000000)
----------------------------------------------
implementation       memory     (elapsed)
----------------------------------------------
pkgcraft             474.2 MB   (0.94s)
pkgcraft-cached      8.7 MB     (0.27s)
pkgcore              8.4 MB     (1.12s)
portage              795.5 MB   (10.62s)

Dynamic atoms (1000000)
----------------------------------------------
implementation       memory     (elapsed)
----------------------------------------------
pkgcraft             955.2 MB   (2.93s)
pkgcraft-cached      957.9 MB   (3.56s)
pkgcore              1.3 GB     (31.01s)
portage              4.0 GB     (56.22s)

Random atoms (1000000)
----------------------------------------------
implementation       memory     (elapsed)
----------------------------------------------
pkgcraft             945.4 MB   (3.75s)
pkgcraft-cached      21.3 MB    (1.30s)
pkgcore              20.9 MB    (2.67s)
portage              3.6 GB     (46.77s)
```

This benchmark creates a list of a million objects for three different atom
types while timing how long and how much memory (using resident set size) each
implementation uses.

For static atoms, note that pkgcraft-cached and pkgcore's memory usage is quite
close with pkgcore slightly edging ahead probably due to the extra fields
pkgcraft's rust implementation stores internally to speed up comparisons.
Another point of interest is that pkgcraft's uncached implementation still
beats pkgcore in processing time, meaning pkgcraft is able to parse and
instantiate the objects in rust, called via a C wrapper library, and then wrap
them into native python objects faster than pkgcore can do what basically
amounts to attribute requests and dictionary lookups. Portage is last by a
large margin as it doesn't directly cache atom objects.

With dynamic atoms, no implementation has a memory usage edge since every atom
is different making caching irrelevant. From this the uncached pkgcraft
implementation has the edge since it doesn't perform cache lookups or store a
cache. Pkgcore's memory usage is comparatively respectable, but uses about an
order of magnitude more processing time. Portage is again last by a increased
margin and appears to perform inefficiently when storing more complex atoms.

Finally, random atoms try to model closer to what is found across the tree in
terms of cache hits. With this result, it appears that using pkgcraft's cached
implementation probably is a good idea for large sets of atoms with occasional
overlap.

### Looking to the future

From the rough benchmarks above, it seems apparent both pkgcore and portage
could decrease their overall processing time and memory usage by moving to
using package atom support from pkgcraft python bindings. While I'm unsure how
much of a factor overall such a change would cause, it should at least be
noticeably worthwhile when processing large amounts of data, e.g. scanning the
entire tree with pkgcheck or sorting atoms during large dependency resolutions.

In terms of feasibility, it's probably easier to inject the current pkgcraft
bindings into portage since its atom support subclasses string objects while
pkgcore's subclasses an internal restriction class, but both should be possible
with some rework. Realistically speaking, neither is likely to occur because
both projects lack maintainers with the required combination of skill, time,
and interest to do it and in doing so would generally restrict the project to
supporting fewer targets due to rust's current lack of support for older
architectures which may be somewhat resolved if a viable GCC rust
implementation is ever released.
