---
title: "Rustifying Bash"
subtitle: "Writing dynamic builtins in rust"
date: 2022-06-08T13:08:38-06:00
tags: ["bash", "pkgcraft", "rust"]
---

In the previous post on extending bash, using builtins was mentioned as a way
to improve extensibility. Rather than writing native bash functions or spawning
processes to run external tools, pkgcraft implements all its bash command
support using builtins.

For example, the `inherit` command used to load eclasses, `die`, and all
install-related functionality (e.g. `doins`) are all implemented as
builtins[^1]. This allows for a more seamless experience compared to pkgcore
which implements all of this natively in bash using a simple daemon that sends
messages via shared fds to communicate between the python and bash sides.

Note that these builtins are not readily available for use in regular bash
since most are highly Gentoo specific, often rely on underlying build state,
and aren't built in a fashion that can be externally exposed. However, the
design work done to support bash in rust also allows creating builtins
compatible with standard bash.

For those interested in bash and rust, the following walkthrough explains how
dynamic builtins work, describes some of the rust support required for
interoperability, and discusses why they're useful.

## Dynamic builtin basics

For background, bash includes builtins used daily by many, e.g. `cd`, `echo`,
and `source` are all builtins. In addition to these, external builtins can be
loaded dynamically from shared object files via `enable`:

```bash
$ enable -f path/to/shared/object.so builtin_name
```

which uses `dlopen` to open the shared object and `dlsym` to load the symbol
for the related builtin, if it exists. The builtin is then registered
internally as dynamically loaded and can be used until it is either
unloaded or the shell exits.

To remove a previously loaded builtin use:

```bash
$ enable -d builtin_name
```

Running a builtin is the same as running most other commands:

```bash
$ builtin_name args to pass
```

In terms of default execution precedence, similarly named functions come first,
then builtins, and finally external binaries. This means if an in scope
function and loaded builtin have the same name, running that name in the shell
will run the function and not the builtin.

The version of bash installed by most distros should support dynamic builtins
inherently because bash itself doesn't provide a disable mechanism; however,
Gentoo manually hacks the configure script to disable support by default. In
order to enable it, make sure to build bash with the `plugins` USE flag
enabled.

Builtins have access to nearly all bash's underlying API; however, they are
mainly limited to running in command form using simple string arguments. In
other words, scoped builtins that form more complex expressions, e.g. bash's
conditional expression `[[ ]]`, generally require parser and/or grammar level
changes that aren't possible to achieve in a basic builtin.

## Creating builtins in rust

One of the tricky parts supporting dynamic builtins in rust is that it has no
support of life before main or lib init similar to C. Therefore, we must
determine some way to provide external symbols for builtin structs that can't
be initialized globally before init. To do this rust relies on linker support
for runtime initialization via `DT_INIT_ARRAY` for ELF objects (and similar on
other platforms). This allows running a specified function during the
library loading process that replaces Option wrapped, globally defined,
static mutables with their actual builtin structs required by bash[^2].

Beyond building the shared objects, pkgcraft provides support for interacting
with bash's C API in rust via [scallop](https://github.com/pkgcraft/scallop).
This enables performing most anything that can be done natively in bash. For
example, bash variables can be bound, unbound, and marked as readonly. However,
it should be noted that scallop is a young project so it only supports what
pkgcraft has needed thus far, has many rough edges, and doesn't come close to
wrapping all of bash's exported API.

In addition to scallop, pkgcraft also provides pkgcraft-bash which is mainly an
example project to create dynamic builtins. For our purposes, we'll be
exploring scallop and pkgcraft-bash while using them to demonstrate how
rust-based builtins work.

#### Development environment

First, the required tools for building the code should be installed. This
includes a recent rust compiler, C compiler, and a recent version of bash that
supports loading dynamic builtins from shared objects. I leave it up to the
reader to leverage `rustup` and/or their distro's package manager to install
the required build tools (and others such as `git` that are implied
requirements).

Next, the required pkgcraft subprojects must be pulled down. The easiest way to
do this is to recursively clone pkgcraft-workspace which should include
semi-recent submodule checkouts for all the subprojects:

```bash
git clone --recurse-submodules https://github.com/pkgcraft/pkgcraft-workspace.git
cd pkgcraft-workspace
```

From this workspace, the pkgcraft-bash project can be built via:

```bash
$ cargo build -p pkgcraft-bash --features pkgcraft
```

This should create the shared pkgcraft-bash library
`target/debug/libpkgcraft_bash.so` from which dynamic builtins can be loaded.

### Profiling

In order to aid in bash development with rust, scallop provides a rudimentary
profiling builtin. To load and use it, see the following example:

```bash
$ enable -f target/debug/libpkgcraft_bash.so profile
$ profile sleep 1
profiling: sleep 1
elapsed 3.005011736s, loops: 3, per loop: 1.001670578s
```

In short, it profiles a user-specified command over a period of time while
counting loops completed. This could be extended to run cache warmups and
perform more accurate statistical analysis, but its current form works for
simple benchmarking.

It's quite fair to say that if you start benchmarking bash code then you
probably shouldn't be using bash; however, most Gentoo package managers include
a relatively large amount of bash that should be optimized in cases where it
runs often or in tight loops.

Pkgcraft leverages scallop to sidestep this entirely, allowing all native bash
code required to support operating with ebuilds to be replaced with rust.
Alongside that, this profile builtin helps highlight certain types of runtime
regressions in pkgcraft's builtin support.

### Atom version comparisons

Now that you have some experience with the `profile` builtin, let's compare the
performance of an actual rust-based builtin to similar functionality written
natively in bash for atom version comparisons.

First, download a copy of `eapi7-ver.eclass` that contains the bash
implementation of the version comparison algorithm used in Gentoo for the
`ver_test` command in portage and pkgcore.

```bash
$ wget https://raw.githubusercontent.com/gentoo/gentoo/master/eclass/eapi7-ver.eclass
```

Next, check its performance using the `profile` builtin. Note that if you
started a new bash shell, the `profile` builtin will have to be reloaded.

```bash
$ source eapi7-ver.eclass
$ enable -f target/debug/libpkgcraft_bash.so profile
$ profile ver_test 1.2.3_alpha1-r1 -gt 1.2.3_alpha1-r2
profiling: ver_test 1.2.3_alpha1-r1 -gt 1.2.3_alpha1-r2
elapsed 3.000030955s, loops: 10648, per loop: 281.745µs
```

With that baseline established for the native bash implementation, let's create
a new builtin that wraps pkgcraft support to provide the same functionality.
It's probably easiest to copy pkgcraft's `ver_test` builtin into pkgcraft-bash
with minor alterations in order to make it dynamically loadable.

Use the following diff that currently applies against the pkgcraft-bash repo to
include `ver_test` support (or use it as a guide if it has fallen out of date).

```diff
diff --git a/src/builtins.rs b/src/builtins.rs
index 83a6e63..7055281 100644
--- a/src/builtins.rs
+++ b/src/builtins.rs
@@ -1,6 +1,7 @@
 use scallop::builtins::DynBuiltin;

 mod atom;
+mod ver_test;

 #[export_name = "profile_struct"]
 static mut PROFILE_STRUCT: Option<DynBuiltin> = None;
@@ -22,9 +23,10 @@ pub(super) extern "C" fn initialize() {
         // update struct pointers
         unsafe {
             atom::ATOM_STRUCT = Some(atom::BUILTIN.into());
+            ver_test::VER_TEST_STRUCT = Some(ver_test::BUILTIN.into());
         }

         // add builtins to known run() mapping
-        update_run_map([&atom::BUILTIN]);
+        update_run_map([&atom::BUILTIN, &ver_test::BUILTIN]);
     }
 }
diff --git a/src/builtins/ver_test.rs b/src/builtins/ver_test.rs
new file mode 100644
index 0000000..24dfecc
--- /dev/null
+++ b/src/builtins/ver_test.rs
@@ -0,0 +1,46 @@
+#![cfg(feature = "pkgcraft")]
+use std::str::FromStr;
+
+use pkgcraft::atom::Version;
+use scallop::builtins::{Builtin, ExecStatus, DynBuiltin};
+use scallop::variables::string_value;
+use scallop::{Error, Result};
+
+const LONG_DOC: &str = "Perform comparisons on package version strings.";
+
+#[doc = stringify!(LONG_DOC)]
+pub(crate) fn run(args: &[&str]) -> Result<ExecStatus> {
+    let pvr = string_value("PVR").unwrap_or_else(|| String::from(""));
+    let pvr = pvr.as_str();
+    let (v1, op, v2) = match args.len() {
+        2 if pvr.is_empty() => return Err(Error::Builtin("$PVR is undefined".into())),
+        2 => (pvr, args[0], args[1]),
+        3 => (args[0], args[1], args[2]),
+        n => return Err(Error::Builtin(format!("only accepts 2 or 3 args, got {n}"))),
+    };
+
+    let v1 = Version::from_str(v1)?;
+    let v2 = Version::from_str(v2)?;
+
+    let ret = match op {
+        "-eq" => v1 == v2,
+        "-ne" => v1 != v2,
+        "-lt" => v1 < v2,
+        "-gt" => v1 > v2,
+        "-le" => v1 <= v2,
+        "-ge" => v1 >= v2,
+        _ => return Err(Error::Builtin(format!("invalid operator: {op}"))),
+    };
+
+    Ok(ExecStatus::from(ret))
+}
+
+#[export_name = "ver_test_struct"]
+pub(super) static mut VER_TEST_STRUCT: Option<DynBuiltin> = None;
+
+pub(super) static BUILTIN: Builtin = Builtin {
+    name: "ver_test",
+    func: run,
+    help: LONG_DOC,
+    usage: "ver_test 1 -lt 2-r1",
+};
--
2.35.1
```

Once the diff is applied, rebuild pkgcraft-bash with pkgcraft support enabled
from the root of the workspace which will currently build the profile, atom,
and ver_test builtins.

```bash
$ cargo build -p pkgcraft-bash --features pkgcraft
```

Now, profile `ver_test` again making sure to use the builtin implementation.

```bash
$ unset -f ver_test
$ enable -f target/debug/libpkgcraft_bash.so profile ver_test
$ profile ver_test 1.2.3_alpha1-r1 -gt 1.2.3_alpha1-r2
profiling: ver_test 1.2.3_alpha1-r1 -gt 1.2.3_alpha1-r2
elapsed 3.000010097s, loops: 252482, per loop: 11.882µs
```

From the result, note that the rust implementation is over 20x faster than the
native bash version. Through further work this can potentially be improved with
more changes to bash's builtin support. For example, bash currently does a
binary search in its builtins array to find if a matching builtin exists before
executing it. This should be quicker to perform as a simple hash table lookup
instead.

## Rust or bash, your call

Overall, I personally find most programming languages to be more maintainable
than bash in the long-term for any well-written code longer than relatively
simple scripts. Add in rust's ability to be exported via its FFI interface to
any language that has C interoperability and it should become apparent why I
prefer implementing such support in rust rather than bash.

If scallop keeps improving its wrapper API around bash, support for writing
bash functionality in rust should continue to improve as well. Looking forward,
it's feasible something like [bats](https://github.com/bats-core/bats-core)
could be written in rust or scallop's functionality could be exported to
another language, for example allowing python to natively interact with bash.

For the time being, I'll just continue using it for one of the main reasons I
created it: trying to avoid writing extensive code in bash.

[^1]: They can currently be found in the `pkgsh/builtins` subdirectory of the
  [pkgcraft crate](https://github.com/pkgcraft/pkgcraft/tree/main/src/pkgsh/builtins).
[^2]: The [ctor crate](https://crates.io/crates/ctor) can make this easier via procedural macros.
