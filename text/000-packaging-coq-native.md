- Title: Packaging coq with native_compute

- Drivers: Erik Martin-Dorel (@erikmd), Pierre Roux (@proux01)

----

# Motivation

Currently, the use of native_compute in libraries involving many dependencies is [inconvenient](https://github.com/coq/coq/issues/12564#issuecomment-647546401) (so that nobody actually use this feature). Indeed, it is required to recompile all these dependencies with `-native-compiler yes` option passed to `coqc`. And this need to be done in a specific way for each dependency, depending on the build system it uses (see e.g., https://coq.inria.fr/refman/practical-tools/utilities.html#precompiling-for-native-compute for `coq_makefile`), if possible at all.

The overall aim of this proposal is to provide an `opam` meta-package (named e.g. `coq-native`) that users can install to automatically (re)compile everything with native_compute (no need for a manual `opam pin edit`).

# Summary

This CEPS offers a new strategy for setting the default value for the `-native-compiler` option and in its `opam` packaging.

The proposed design tackle the following two use cases:

1. Most users (who don't even want to hear about `native_compute`) preferred behavior would be compiling Coq with `native-compiler no` by default.
2. `native_compute` users who currently need to recompile all their libraries with environment variables (if possible) which aren't even the same across the build system used by libraries, thus making any efficient use of `native_compute` terribly unpractical.

Also, this proposal uniformizes the configuration across the platforms (while `native_compute` [was permanently disabled for macOS](https://github.com/ocaml/opam-repository/pull/16908) to workaround performance issues https://github.com/coq/coq/issues/11178).

# Detailed design

The proposal is three-fold:

1. Update the `coq` packaging in `opam-repository` so that Coq's standard library is compiled with `-native-compiler no` by default, unless an additional meta-package `coq-native` is installed by the user.
2. Change the default `coqc` option to `-native-compiler on` instead of `-native-compiler ondemand`.
3. Optionally: to enhance `native_compute` support with old versions of Coq (the releases of Coq before the item 2. above is implemented), update the opam packacking of the considered libraries.

Regarding item 1:

* Add an opam metapackage `coq-native`

```
opam-version: "2.0"
maintainer: "Erik Martin-Dorel"
authors: "Coq"
homepage: "https://coq.inria.fr/"
bug-reports: "https://github.com/coq/coq/issues"
conflicts: [
  "coq" {< "8.5"}
]
synopsis: "Dummy package enabling coq's native-compiler flag"
description: """
This package acts as a package flag for the ambient switch, taken into
account by coq (and possibly any coq library) to enable native_compute
at configure time, triggering the generation of .coq-native/* files.

REMARKS:
- you might face with issues installing this package flag under macOS,
  cf. <https://github.com/coq/coq/issues/11178>
- for more details, see <the URL of this CEP>.
"""
```

* Add the following lines in the opam packages of `coq`:

```diff
diff --git a/packages/coq/coq.8.11.2/opam b/packages/coq/coq.8.11.2/opam
index fdb19e9da7..3d3dee7570 100644
--- a/packages/coq/coq.8.11.2/opam
+++ b/packages/coq/coq.8.11.2/opam
@@ -23,6 +23,9 @@ depends: [
   "num"
   "conf-findutils" {build}
 ]
+depopts: [
+  "coq-native"
+]
 build: [
   [
     "./configure"
@@ -33,7 +36,9 @@ build: [
     "-libdir" "%{lib}%/coq"
     "-datadir" "%{share}%/coq"
     "-coqide" "no"
-    "-native-compiler" {os = "macos"} "no" {os = "macos"}
+    "-native-compiler" "yes" {coq-native:installed} "no" {!coq-native:installed}
   ]
   [make "-j%{jobs}%"]
   [make "-j%{jobs}%" "byte"]
```

Regarding item 2:

The `coqc` option `-native-compiler ondemand` corresponds to the default behavior since <https://github.com/coq/coq/commit/4ad6855504db2ce15a474bd646e19151aa8142e2> (8.5beta3).

See [the tabular of this comment](https://github.com/coq/coq/issues/12564#issuecomment-647464937) of a summary of all behaviors.

With the current codebase, the change will amount to replacing [this line of `toplevel/coqargs.ml`](https://github.com/coq/coq/blob/473160ebe4a835dde50d6c209ab17c7e1b84979c/toplevel/coqargs.ml#L101) with:

```ocaml
then NativeOn {ondemand=false}
```

Regarding item 3:

Assuming we want to use `native_compute` with coq-mathcomp-ssreflect (or any library built with `coq_makefile`), the `opam` file has to be extended like this (see e.g. [Coq's refman](https://coq.github.io/doc/master/refman/practical-tools/utilities.html?highlight=coq_makefile#precompiling-for-native-compute)): 

```diff
diff -u /home/emartin/.opam/4.05.0\+coq8.11/.opam-switch/overlay/coq-mathcomp-ssreflect/opam_.\~1\~ /home/emartin/.opam/4.05.0\+coq8.11/.opam-switch/overlay/coq-mathcomp-ssreflect/opam_
--- /home/emartin/.opam/4.05.0+coq8.11/.opam-switch/overlay/coq-mathcomp-ssreflect/opam_.~1~	2020-10-28 16:28:09.630696418 +0100
+++ /home/emartin/.opam/4.05.0+coq8.11/.opam-switch/overlay/coq-mathcomp-ssreflect/opam_	2020-10-28 16:33:19.518309746 +0100
@@ -6,7 +6,8 @@
 dev-repo: "git+https://github.com/math-comp/math-comp.git"
 license: "CeCILL-B"
 
-build: [ make "-C" "mathcomp/ssreflect" "-j" "%{jobs}%" ]
+build: [ make "-C" "mathcomp/ssreflect" "-j" "%{jobs}%" "COQEXTRAFLAGS+=-native-compiler yes" {coq-native:installed}]
 install: [ make "-C" "mathcomp/ssreflect" "install" ]
 depends: [ "coq" { (>= "8.7" & < "8.12~") } ]
``` 


Doing this, one can notice that the appropriate `.coq-native` directory has succesfully been installed along the `.vo` of the library: 
`find ~/.opam/4.05.0+coq8.11 -name ".coq-native" | grep ssreflect`

# Drawbacks

To fully benefit from `native_compute` with currently released
versions of Coq, item 3. is necessary, albeit it should only require some packaging update (`opam`) that could be performed directly in the https://github.com/coq/opam-coq-archive repository.

# Alternatives

N/A

# Unresolved questions

* The implementation of items 1. and 3. above rely on `make`, but how to implement the `dune` counterpart? (Cc @ejgallego).

Cc-ing some devs involved in the feature discussed here (sorry if I forget someone), @Blaisorblade @maximedenes @ejgallego @SkySkimmer @silene @gares @Zimmi48
