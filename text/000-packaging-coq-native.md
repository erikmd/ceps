- Title: Packaging coq with native_compute

- Drivers: Erik Martin-Dorel (@erikmd), Pierre Roux (@proux01)

----

# Motivation

Currently, the use of native_compute in libraries involving many dependencies is inconvenient (so that nobody actually use this feature). Indeed, it is required to recompile all these dependencies with `-native-compiler yes` option passed to `coqc`. And this need to be done in a specific way for each dependency, depending on the build system it uses (see e.g., https://coq.inria.fr/refman/practical-tools/utilities.html#precompiling-for-native-compute for `coq_makefile`), if possible at all.

The overall aim of this proposal is to provide an `opam` meta-package (named e.g. `coq-native`) that users can install to automatically (re)compile everything with native_compute.

# Summary

This CEPS offers a new strategy for setting the default value for the `-native-compiler` option and in its `opam` packaging.

The proposed design tackle the following two use cases:

1. Most users (who don't even want to hear about `native_compute`) preferred behavior would be compiling Coq with `native-compiler no` by default.
2. `native_compute` users who currently need to recompile all their libraries with environment variables (if possible) which aren't even the same across the build system used by libraries, thus making any efficient use of `native_compute` terribly unpractical.

Also, this proposal uniformizes the configuration across the platforms (while `native_compute` [was permanently disabled for macOS](https://github.com/ocaml/opam-repository/pull/16908) to workaround performance issues https://github.com/coq/coq/issues/11178).

# Detailed design

The proposal is two-fold:

1. Update the `coq` packaging in `opam-repository` so that Coq's standard library is compiled with `-native-compiler no` by default, unless an additional meta-package `coq-native` is installed by the user.
2. 

do as offered above for the standard library (i.e

by default and a dummy OPAM package to recompile Coq with `-native-compiler yes`) ;
* modify Coq's default (so this would only take effect from 8.13 unfortunately) for `native-compiler` to `yes` (instead of `ondemand`) whenever Coq itself is compiled with `-native-compiler yes`.

This way, IINM (at least starting with 8.13, this would imply keeping the current mess for < 8.13), we would have:
1. Coq without any `.coq-native` for most users, who don't really want them
1. `native_compute` users would just have to install the dummy package to get all their libraries recompiled with `native-compiler yes`
1. all this without any work required from library maintainers, no `.opam` nor `configure` to modify (I'm particularly worried by the fact that some library maintainers, whose library doesn't have any computational goal but is used (inside proofs) by `native_compute` intended libraries, would not like to be bothered with `native_compute` things, which is perfectly understandable)










Also, this proposal 

https://github.com/ocaml/opam-repository/pull/16887/files


Also, compiling Coq with flambda options always enabled would cause issues with OCaml < 4.07 − https://coq.discourse.group/t/install-notes-on-coq-and-ocaml-versions-configuration/713
* Compiling Coq with `-native-compiler yes` led to performance issues in macOS − https://github.com/coq/coq/issues/11178 − and with flambda − https://github.com/coq/coq/issues/11107 
* So `native-compiler` and `flambda` options are disabled by default in the packaged Coq releases
* As a result, if we'd like to use these options anyway, it is not convenient at all to setup them (for flambda, some `opam pin edit coq` may be required, or even some fork of the opam repository | for `native-compiler` also, any package relying on `native_compute` *requires* that all its dependencies have been compiled with `-native-compiler yes`, so that all `.coq-native/` files have been properly built − see e.g. [Coq's refman](https://coq.github.io/doc/master/refman/practical-tools/utilities.html?highlight=coq_makefile#precompiling-for-native-compute) or https://github.com/coq/coq/issues/12564#issuecomment-647546401)



Regarding the first use case, not also that there is a performance issue 

Explain how the problem is solved by the CEP, provide a mock up, examples.

# Drawbacks

Is the proposed change affecting any other component of the system? How?

# Alternatives

Yes, do the related works.  What do other systems do?

# Unresolved questions

Questions about the design that could not find an answer.

================ PIERRE TEXT
Sorry for the late answer. Trying to sum up, we have:

It seems to me that the solution offered here perfectly solves 1. but requires a lot of tuning of `.opam` files (or even `configure` scripts ?) to solve 2., which only transfers the `native_compute` hurdle from users to library maintainers (although this is already an improvement).

What about the following offer:
* do as offered above for the standard library (i.e. `-native-compiler no` by default and a dummy OPAM package to recompile Coq with `-native-compiler yes`) ;
* modify Coq's default (so this would only take effect from 8.13 unfortunately) for `native-compiler` to `yes` (instead of `ondemand`) whenever Coq itself is compiled with `-native-compiler yes`.

This way, IINM (at least starting with 8.13, this would imply keeping the current mess for < 8.13), we would have:
1. Coq without any `.coq-native` for most users, who don't really want them
1. `native_compute` users would just have to install the dummy package to get all their libraries recompiled with `native-compiler yes`
1. all this without any work required from library maintainers, no `.opam` nor `configure` to modify (I'm particularly worried by the fact that some library maintainers, whose library doesn't have any computational goal but is used (inside proofs) by `native_compute` intended libraries, would not like to be bothered with `native_compute` things, which is perfectly understandable)

@maximedenes would that seem acceptable to you ? AFAIC, I consider this `native-compiler yes` thing to be the main obstacle to a widespread and easy use of `native_compute`.

@ejgallego how would that work with the dune build ? (sorry, I must admit I still don't fully understand what plays the role of configure and configure options in the dune build process)

================================================================
#### Description of the problem

This meta-issue intends to document the new choices for settings the options passed to Coq's configure script in opam packages (especially those related to native_compute and flambda), then track the progress of this on-going change. 

(It is a follow-up of [this Zulip thread](https://coq.zulipchat.com/#narrow/stream/237663-coq-community-devs.20.26.20users/topic/flambda-using.20Docker-Coq.20images) − login required)

#### Coq Version

Coq ≥ 8.5 (for native_compute) and Coq ≥ 8.7 (for flambda options, since https://github.com/coq/coq/pull/540)

#### Wrap-up of the issues

* Compiling Coq with flambda options always enabled would cause issues with OCaml < 4.07 − https://coq.discourse.group/t/install-notes-on-coq-and-ocaml-versions-configuration/713
* Compiling Coq with `-native-compiler yes` led to performance issues in macOS − https://github.com/coq/coq/issues/11178 − and with flambda − https://github.com/coq/coq/issues/11107 
* So `native-compiler` and `flambda` options are disabled by default in the packaged Coq releases
* As a result, if we'd like to use these options anyway, it is not convenient at all to setup them (for flambda, some `opam pin edit coq` may be required, or even some fork of the opam repository | for `native-compiler` also, any package relying on `native_compute` *requires* that all its dependencies have been compiled with `-native-compiler yes`, so that all `.coq-native/` files have been properly built − see e.g. [Coq's refman](https://coq.github.io/doc/master/refman/practical-tools/utilities.html?highlight=coq_makefile#precompiling-for-native-compute) or https://github.com/coq/coq/issues/12564#issuecomment-647546401)

#### Other related PRs/issues

(beyond those mentioned above)

- https://github.com/coq/coq/issues/11476 `[native_compute] Pre-compilation of coq-bignums`
- PR https://github.com/coq/coq/pull/11814 `Document coq_makefile behavior wrt -native-compiler yes`
- PR https://github.com/coq/coq/pull/12550 `Fix configure usage in .opam`

#### Proposed changes (as suggested in the Zulip thread)

1. Submit a dummy `coq-native` package in opam-repository, that would act as a package flag (installable upon request on a per-switch basis) to enable `-native-compiler` for all Coq ≥ 8.5 then all relevant Coq packages (and trigger their rebuild if one install `coq-native` afterwards!)
2. Append the `-flambda-opts '-O3 -unbox-closures'` flags suggested in [this Discourse post](https://coq.discourse.group/t/install-notes-on-coq-and-ocaml-versions-configuration/713), provided `{ocaml:version >= "4.07"}` (passing the options would be safe (no-op) anyway if no flambda compiler is detected) 

I'll first look after item 1. above, but feel free to react/comment in this issue if need be, or edit my post to add other related PRs/issues, and possibly do more tests later on! 

Cc-ing the devs involved in the features discussed here (sorry if I forget someone), @Blaisorblade @maximedenes @ejgallego @SkySkimmer @silene @proux01 @gares @Zimmi48 FYI

================================================================
Finally, this proposal suggests to update another Coq configure option (`-flambda-optsin`) to leverage the `opam` packaging related to flambda compiler.
