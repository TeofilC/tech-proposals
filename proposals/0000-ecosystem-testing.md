# Haskell Ecosystem Testing, or rejuvenating head.hackage

## Abstract

In this proposal, we lay out a plan for extending GHC's `head.hackage`
into an ecosystem testing framework that GHC contributors, library
authors and end-users can utilise to ensure that the ecosystem works
well together, and when breakage occurs, can be used to track the
progress of downstream fixes.

This entails modernising and extending our existing infrastructure while
reducing its maintenance burden.

Our hope is that this can greatly increase the community's trust in the
stability of the Haskell ecosystem, and greatly decrease the cost of
upgrading to significant changes to the ecosystem, whether that comes from the
release of a new compiler version, a new version of `base`, the version bump of
a core library or the release of a new version of a widely used package on
Hackage.

## Background

Users of Haskell desire *stability* from their ecosystem, which
encompasses the compiler (GHC, MHS, etc), core libraries (especially
`base`), and the set of libraries more broadly. In
particular, I want to emphasise two senses of *stability*:

- *interface stability*: the interface of a new version of a component
  shouldn't break downstream user's code through breaking changes.

- *bug stability*: a new version of a component shouldn't introduce new
  bugs.
  
The GHC project already has a very strong commitment to stability. See
the GHC Steering Committe's [stability
principles](https://ghc-proposals.readthedocs.io/en/latest/principles.html#id5).
Similarly, the Core Libraries Committee maintains a proposal process for
any change to the interface of `base`, which requires an impact
assessment, and these reports are carefully weighed to
avoid unnecessary breaking changes.

Yet, despite our best efforts, new versions of software will always introduce
a certain level of instability. Both breaking changes and bugs encourage
users to continue using older versions of software. However, the more
users take the plunge and try out the new version, the quicker
downstream software can be patched to deal with breaking changes and to
discover and fix bugs. This can lead to a vicious cycle where low trust
in the stability of new components leads to low adoption, which leads to
slow stabilisation, which leads to worse trust, and so on. This can make
an especially large difference for projects like GHC where it can take a
long time to cut a new release.

The aim of this proposal is to do more of this ecosystem-wide
integration testing earlier ("shift left") to before releases of key
projects like GHC, `base`, etc are made, and to vastly decrease the
labour involved in doing so. Ideally we would like to enable workflows
where [some form of ecosystem
testing](https://gitlab.haskell.org/ghc/ghc/-/work_items/26809) happens
before PRs are merged to GHC/`base`/etc. We also want to enable
end-users, both in industry and library authors, to easily test their
code against GHC nightlies, prereleases, or unreleased versions of other
key components, and to increase the trust of our users in both our
ecosystem testing tooling and in the ecosystem in general.

This belief is borne from my experience as a maintainer of
`head.hackage` and as an industrial user who has often tested GHC prereleases. 
Time and time again, I have found that it is much easier
to fix builds of `head.hackage` closer to the time that the breakage
occurs. This allows one to quickly find the commit which caused the
breakage, and often allows you to discuss a non-breaking variant of the
change with the author. Being able to identify the cause, also makes it
easier to write any patches, and an entry for the migration guide. On
the other hand, when looking at several months of breakages, it's much
harder to separate out individual causes and track down which specific
commit caused them. 
I have also found that whenever I test a GHC prerelease, I am likely to
encounter some as yet unnoticed bug or breaking change due to the relatively
complexity of the industrial codebases on which I work.

Beyond GHC and `base`, any widely used component of the Haskell
ecosystem benefits from having the facility to easily test changes
against their downstream consumers and ensure that no unwanted breakage
occurs.

GHC's API has no stability guarantees at present. This has long been
something the project has sought to improve, but the surface area is too
large at present to do much about this. I believe that extensive testing
of consumers of the GHC API on `head.hackage` can be a helpful step
towards building an understanding of which parts of it are being used,
which changes are painful breaking changes and which are harmless
refactors to functions that consumers don't use. Some GHC plugins are likely
not publicly distributed and these too could be tested with the
`head.hackage` infrastructure running locally in a corporate environment
and tickets could be opened with GHC to discuss the breaking change.

While we want to keep breaking changes to a minimum, there are times
when we intentionally want to release a breaking change in a mindful and
coordinated manner. In this case, significant amount of labour needs to
go into applying fixes to downstream consumers and coordinating this
work. For instance, part of the CLC proposal process requires creating patches for 
all packages broken by a change, but there is currently no place where these patches need to go.
And sometimes they can get lost between the proposals\' acceptance and the release of GHC.
We hope that `head.hackage` can expand to be the place where this
co-ordination can happen, and where we can track the status of
downstream patches.

## Problem Statement

GHC already does some ecosystem testing via `head.hackage`.
`head.hackage` is a set of patches for packages on Hackage and a list of
unpatched packages from Hackage, which are all (in-theory) buildable with the
compiler from the HEAD commit of each active branch of GHC.
`head.hackage`\'s CI builds all of these packages and produces a Hackage
overlay (think of it like a lightweight fork of Hackage) with the
patches applied along with a `cabal.project` file with constraints that
force using compatible packages, which end-users can then apply to their
projects.

`head.hackage` is already well integrated into the GHC developer
experience. One can simply add a label to an MR and a `head.hackage`
pipeline will be triggered against it.

Yet, the benefits of `head.hackage` are limited by several issues:

- It is a project with a great deal of technical debt, eg, the list of
  extra package to build is configured via a (bash script)[https://gitlab.haskell.org/ghc/head.hackage/-/blob/master/ci/config.sh].

- The amount of packages on `head.hackage` are limited (approx 700) and much fewer
  than Stackage (3442 in LTS 24.57).

- `head.hackage` often includes packages which fail to build, meaning
  that it is difficult for GHC contributors to distinguish expected from
  unexpected failures when running it against their MRs.

- Each package has its own build plan, which makes it difficult to
  reason about and reproduce failures locally, as you normally create a
  project which includes a larger subset of the package set.

- `head.hackage` doesn't include any metadata about *why* packages have  been
  included or patched. This makes it difficult to know if packages/patches can
  be removed or should be kept. Some patches are over 6 years old.

- There are no efforts made in the normal workflow to upstream patches, which
  means that patches can live in `head.hackage` indefinitely, and someone else
  needs to duplicate effort to create a patch against the upstream repository at
  some point.

- While it is good practice to also update the GHC migration guide when
  adding a patch to `head.hackage`, there is no guarantee that this
  happens.

- `head.hackage` is not widely used outside of the GHC project.

- `head.hackage`'s policy of patching Hackage uploads rather than
  upstream repositories isn't well suited to quickly moving projects
  that are tightly integrated into the GHC API like `liquid-haskell`.

We aim to solve these issues.

The Core Libraries Committee's proposal process for `base` also requires
testing against the ecosystem. This is done in order to produce an
impact assessment for every breaking change.
[`clc-stackage`](https://github.com/haskell/clc-stackage) is the
recommended way to do this. This project requires users to run a build
of stackage for the latest supported GHC, which is always older than the
HEAD of the GHC repository, which is what the `base` change would be
made against. Using this tool requires cherry-picking one's patch to an
older copy of the GHC repository, building GHC, and then running this
tool against that. I believe it would be easier for contributors to
submit CLC proposals if a similar tool to build all of Stackage was
integrated into the GHC contributor workflow (potentially avoiding
having to interact with the GHC build experience locally), and that the
`head.hackage` tooling could be expanded to accomplish that.
In practice, the overhead of running a tool like `clc-stackage` means that it is
only run on proposals where there is quite likely to be lots of breaking
changes, and most proposals either do a more lightweight impact assessment, eg,
by grepping Hackage, or none at all.

If `base` ever moved out of the GHC repository then it would have its CI
system, but it would still have similar requirements to do ecosystem
testing, and any tool shouldn't assume that GHC is what is being tested.
We might just as easily be testing a core library such as `base` or
`template-haskell` or a different library altogether.

Such an expansion of `head.hackage` must be accompanied by a decrease in
its maintenance burden.

## Prior Art and Related Efforts

- The [GHC.X.Hackage proposal](https://github.com/haskellfoundation/tech-proposals/pull/27)
  also proposed to expand `head.hackage` to improve the GHC upgrade
  experience. Many of the benefits from that proposal would also be
  gained from this proposal. We differ in some key details though (as we
  shall see in the following section), namely that patches shouldn't
  live indefinitely on `head.hackage`, and that we do not emphasise use
  as a Hackage overlay.

- The [A process to document the GHC
  API](https://github.com/haskellfoundation/tech-proposals/pull/66)
  proposal has some overlap with this (or at least the initial version
  of that proposal did), insofar as keeping track of the breakages of
  downstream users of the GHC API can give us a good picture of which
  parts of the GHC API are actively being used and which changes are
  particularly painful.

- There have been past efforts to run `head.hackage` against all of
  Stackage nightly. These were successuflly run a few times but the
  changes were never merged. See
  [head.hackage#72](https://gitlab.haskell.org/ghc/head.hackage/-/work_items/72).

- [head.hackage#53](https://gitlab.haskell.org/ghc/head.hackage/-/work_items/53)
  is an effort to enable `index-state` support for the generated Hackage
  overlay. I take this to be a largely orthogonal effort. This proposal
  aims to enable workflows that do not require going via the Hackage
  overlay.

- [`crater`](https://github.com/rust-lang/crater) is Rust's ecosystem
  testing tool, despite not having support for patching packages (as far
  as I can tell) and being much younger than `head.hackage`, it seems to
  be much more successful in getting adoption from rustc developers and
  the community at large. See this
  [ticket](https://gitlab.haskell.org/ghc/head.hackage/-/work_items/112)
  for ideas we can take from `crater`.

## Technical Content

### `head.hackage` Today

At present, `head.hackage` primarily consists of a set of patches
against bindists of packages uploaded to Hackage, along with a set of
packages which we test unpatched, a index-state pin of Hackage, and
supporting infrastructure.

A CI run of head.hackage is given a branch of the GHC repository as
input. It will then download an appropriate bindist from the nightly CI
of that branch, install it, produce a Hackage overlay from the set of
patched packages, and then build each package we have specified in the
repository.

When building a package, we construct a throwaway cabal project with a
fake package which just depends on the package we want to test. This is
done for two reasons:

1.  cabal-install only produces log files for dependencies (not packages
    local to a project). TODO: link issue

2.  By creating a project for each package, we give the cabal solver
    freedom to pick distinct versions of dependencies.

On the other hand, the workflow for adding new patches or updating
existing patches, is to create one large cabal project which includes
minimal git repositories for each package where the current commit is
the bindist from Hackage, and so any changes can easily be turned into a
patch file.

This difference in workflow means that CI failures are often tricky to
reproduce locally.

There is a script in the repository to drop obsolete patches, those
where the version on Hackage is newer than the version we have patched,
but that means that we tend to accumulate patches that were either never
submitted upstream or where the upstream package is completely
unmaintained. Thus much of the maintenance burden of `head.hackage`
tends to be taken up by updating patches for unmaintained packages,
since these packages tend to be particularly brittle.

There is often no data about *why* a particular package is included in
`head.hackage`, so it is difficult to know whether a package should be
dropped.

As GHC HEAD is always moving, and often introducing new failures, it
becomes very difficult to merge MRs with green CI into `head.hackage`,
as one either needs to keep adding new commits, or accept that one will
have to merge the current MR and create a new one for the new failures.
This can be especially demotivating for first time contributors.

`head.hackage` includes some of GHC's integration tests and some
integration tests from downstream packages included as submodules. Given
that `head.hackage` is often in a failing state, and that these tests
are marked as acceptable failures in Gitlab CI, we do not currently get
much benefit from these test suites. The usage of submodules also
introduces extra maintenance overhead, developer experience issues, and
isn't sustainable as a way to add more test suites to `head.hackage`.

`head.hackage` is also not the only place that GHC's integration tests
live, there are a handful of repositories that hosts these test suites,
each with its own CI config, and other idiosyncrasies.

### Proposal

I am proposing some fundamental changes to the design and workflows of
`head.hackage`. I would like to make `head.hackage` deterministic, make the
configuration use a structured file format, switch to a Stackage-like snapshot
model, and enable wider running of tests suites and benchmarks.

#### Deterministic CI

`head.hackage` needs to be made deterministic. In practice, this means
that each commit of `head.hackage` needs to pin the bindists that it
should be run against. This solves the problem that an MR can gain new
failures from GHC HEAD moving forward. It also makes `head.hackage`
vastly more useful for end-users, since they can download that specific
bindist and be confident that the `head.hackage` Hackage overlay will
actually give them versions of packages that work with their GHC.

We would need to create infrastructure to automate creating MRs to
update GHC version pins as part of GHC's nightly pipeline. That pipeline
already runs `head.hackage`. We would collect any failures and mark them
as expected failures in `head.hackage`'s configuration along with the
nightly commit that introduced the failure. This would be an immense
labour saving for `head.hackage` maintainers, as then we wouldn't need
to hunt through the pipeline history to find when a failure first
appeared. It would also make it trivial to merge these MRs, as they
would always lead to passing CI.

#### Structured configuration

That would entail migrating the configuration format to something that
could easily be machine edited. Currently `head.hackage` is configured
through a bash script. I would like to replace it with a set of JSON or
YAML files. This would also let us start collecting much more metadata
about each patched package. We should also start recording metadata
about each distinct breaking change and link them to the patches.

A folder of markdown fragments with metadata, which describe each
breaking change, and link to patches, can easily be used to generate
migration reports for each version of GHC. I suggest we move these into
`head.hackage` and serve them as a static site. If we integrate this
into the `head.hackage` workflow, we can ensure that any change that
breaks `head.hackage` also gets a migration guide entry.

#### Stackage style snapshots

In order to make best use of this, we need to vastly increase the amount
of packages tested on `head.hackage`. I would like to change it so that
`head.hackage` tests a Stackage-esque package set for each active
version of GHC. These might be distinct package sets, eg, GHC-9.10 might
be tested against Stackage LTS while 9.12 is tested against Stackage
nightly, while GHC HEAD is tested against a pared down version of GHC
nightly with patches applied. We could then also output Stackage
snapshots in addition to `cabal.project` files from `head.hackage` for
end-users to make use of. This would bring us closer to the workflows
employed by Stackage and the nixpkgs Haskell infrastructure, and would
enable future sharing of labour.

While testing more packages would increase the computational cost of
`head.hackage`, I do not think it needs to increase the maintenance
burden. The default behaviour would be to drop failing packages, so in
the worst case, we would simply test fewer packages. Although I believe
more testing, earlier in the process, and our existing commitments to
stability entail that relatively few packages should fail to build. Most
failures would likely be package bound changes from boot libraries, and
often we can just apply patches quite mechanically to solve these, or
use `allow-newer`.

Part of this would be to test all packages in a snapshot together in a
single cabal project. This would align the local testing workflow with
CI and greatly simplify things. This would require either patching
`cabal-install` to allow collecting logs for local packages or use an 
alternative build system like Nix. My preferred option is to use Nix.

#### Submitting patches upstream

Currently `head.hackage` patches are done against the latest version of
the package on Hackage. I am proposing to base patches on PRs against
the upstream repository. We would hold back from creating such PRs, of
course, if the maintainer communicates to us that they only want patches
once the version of GHC is released.

This change would align the `head.hackage` workflow with that of
standard industrial practice when upgrading to the latest Haskell
compiler. It would allow us to share labour with these users, and I hope
it would encourage such users to share their sets of patches via
`head.hackage`. As part of this, we would produce `cabal.project`
fragments that would include `source-repository-override`s rather than
going via the Hackage overlay. This would allow more industrial users to
make use of `head.hackage` as the specific commits and patches would be
much easier to audit than from a Hackage overlay.

We would link to the patch in our metadata and this would allow us to
track the life cycle of the patch. This tracking is especially helpful
if package `A` depends on `B` and both need to be patched. The patch to
`A` would need to be merged first and only then could the patch to `B`
be merged, and so we could let maintainers know when it is possible for
them to make progress. This would also be a large labour saving for
industrial users, if they choose to get their `cabal.project` snippets
from `head.hackage`. As they would no longer need to track the status of
each patch themselves.

For certain packages that are very tightly coupled to GHC such as
`liquid-haskell` and other GHC plugins, we should always build the
latest version from the upstream repository. This is a practice, I have
informally engaged in for the last while and it has significantly
improved the experience of maintaining these patches on `head.hackage`,
although the current design means that the patches look like giant diffs
from the released version in source control.

In certain cases, it will not be viable to patch the upstream unreleased
version, eg, if a widely used library completely redesigns their
interface. Then too many downstream packages would break. In such cases
it would be fine to make an exception and patch the latest Hackage
release.

#### Testing and benchmarking

Test suites are invaluable way of catching both bugs and interface
changes. Currently `head.hackage` includes some of GHC's integration
testing infrastructure and test suites for some downstream packages
included as submodules.

I propose that we consolidate GHC's out-of-tree testing infrastructure
in `head.hackage` including `test-primops`. This would reduce the amount
of CI infrastructure we have to maintain, and mean that we would need to
look in fewer places to find failures.

I propose that we should run the test suite for every package on
`head.hackage` by default. And that we make every test failure count as
a failure of the CI job. It is fine if we record some test suites as
known failures in our metadata and don't count them as failures from
them on as long as we have an explanation and a ticket to investigate
and fix them.

I would also like to run benchmarks for at least all core libraries and
write the results into a database, which we then graph in the GHC
Grafana.

#### Incremental builds

Currently each run of the `head.hackage` CI shares nothing with other runs. Each package has to be re-built from scratch.
This is fine for GHC nightly pipelines where the version of GHC changes each time, but isn't appropriate for MRs to `head.hackage`.
In this case, you are normally iterating on a small set of patches, and it can be frustrating to have to wait multiple hours for CI to run because of a typo.
The developer experience therefore would greatly benefit from incremental builds.

I would like to implement this by using Nix to build packages in CI.

### Outcome

The result of this work should give us a much more robust infrastructure for testing the Haskell ecosystem.
We will be able to easily test entire Stackage snapshots and run test suites.

It will clear up a great deal of technical debt in `head.hackage`, and make it less labour intensive to maintain going forward.

It also opens the way for further collaboration between Stackage, nixpkgs and `head.hackage`. 
It would be good if we could eventually share the same format for defining package sets with overlays.
After this proposal is implemented both nixpkgs and `head.hackage` will take Stackage snapshots and add extra patches and annotations on top.
One day we could agree on a shared format and it would be possible 

## Timeline

N/A.

## Budget

TBC

## Stakeholders

- GHC developers

- `head.hackage` contributors

- Core Libraries Committee

- Haskell library authors

- Industrial users

## Success

I have broken down the tasks into a set of core deliverables. 

### Deliverable 1: Deterministic CI

- The bindists used is pinned in a file in the repository.
- GHC Nightly pipelines create MRs to update the pinned bindists.

### Deliverable 2: Structured configurations

- The configuration of `head.hackage` is encoded in a set of JSON or YAML files.
- This configuration includes the list of non-patched packages to build, and metadata about patches, such as the upstream PR.
- GHC Nightly pipelines are updated to also update the list of known failures in automatically created MRs.

### Deliverable 3: Stackage style snapshots and incremental builds

- Each tested version of GHC is tested against a Stackage snapshot based on the upstream Stackage snapshots.
- CI is updated to use Nix to build packages. The local workflow for editing patches still uses `cabal-install`.
- We export these snapshots in the static site.
- We continue to distribute a Hackage overlay with the patches.

### Deliverable 4: Run test suites and benchmarks

- We run test suites and benchmarks for all packages in our snapshots.

### Deliverable 5: Documentation

- We expand contributor documentation to encourage more people to get involved in `head.hackage`.

### Deliverable 6: Improved static site and patch progress tracking
- ...
