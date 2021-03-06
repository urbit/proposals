+++
up = 4
title = "Ford Turboencabulator"
created = 2018-3-15
authors = [
  "~rovnys-ricfer Ted Blackman <ted@tlon.io>",
  "~littel-ponnys Elliot Glaysher <elliot@tlon.io>",
  "~tonlur-sarret Keaton Dunsford <keaton@tlon.io>"
]
status = draft
supersedes = 1
+++

# Summary 

Ford Turbo is a complete redesign of the ford build system.

# Overview

Ford, Urbit's build system, needs love. Like many of Urbit's components, a beautiful vision has been partly realized. The difference between Ford's Platonic ideal and its current state could be considered the reason behind the slowness of the urbit.org website.

While Ford is a functional reactive build system -- as we'll explore in more detail shortly -- its reactivity is both primitive and implicit. Lacking discernment, Ford often does a few hundred times too much work. This leads to slowness, which leads to sporadic 504 errors, which lead to questions on urbit-meta like "has anyone tried running Urbit on transistors? Right now it's clearly using vacuum tubes." 

In modifying Ford, we hope to nudge it a small step closer to cryogenic perfection. This new design centers around formalizing the idea of a build: 

A build is a function of the Urbit namespace and a date that produces marked, typed data or an error.

The function in the definition of a build is called a "schematic," and it's represented by a Hoon data structure with twenty-five sub-types. A schematic is a (possibly trivial) [DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph) of sub-builds to be performed. The different schematic sub-types transform the results of their sub-builds in different ways.

We call the date in the definition of a build the "formal date" to distinguish it from the time at which the build was performed.

Each build is referentially transparent with respect to its formal date: ask to run that function on the namespace and a particular formal date, and Ford will always produce the same result. So far, most builds in the wild have only used the subset of the namespace exposed by the Clay filesystem on the user's own ship. However, Ford should be able to pull in data from Gall apps, other vanes, and files on other ships.

We can now say Ford is a functional build system, since each build is a function. We have not yet explained how it's a functional _reactive_ build system. With new Ford, you can subscribe to results of a build. Ford tracks the result of a "live" build consisting of a static schematic and the ever-changing current date. Whenever this live build's result changes, Ford sends you the new result and the formal date of the build (the date which would cause the same result if you asked Ford to build that schematic again). This is in fact a classic [FRP](https://en.wikipedia.org/wiki/Functional_reactive_programming) paradigm.

These are the formal semantics of new Ford, but it doesn't poll for new results or anything grotesque like that. The implementation is event-driven, like the rest of Urbit. While performing a build, Ford registers each namespace access as a dependency and also notes whether the dependency is "live," meaning the path within the namespace updates with time. For example a live Clay dependency would update the +case within the +beam over time. A dependency that is not live is said to be a "once dependency", and a request to perform a build without subscribing to its future changes is called a "once build."

After finishing a build, Ford subscribes to updates on the build's dependencies. For now, this mostly means it subscribes to Clay for file changes. Whenever any of the files in the subscription have new contents, Clay will notify Ford, which will then rerun any live builds that depend on any of the changed files and send its subscribers the new results.

This matches the semantics of live builds defined above. If someone had asked for a build of the schematic with a formal date `d2` just before the changed Clay files, Ford would respond with the result of the previous build with formal date `d1`, which would still be an accurate representation of the schematic's result at `d2`, since Ford knows none of its dependencies changed between `d1` and `d2`.

Note that Ford can only calculate dependencies after running a build, not before. This is because Ford can be thought of as an interpreter for schematics, rather than a compiler, in the sense that it can't have a dependency-gathering step followed by a build step. The dependencies of some schematics must be calculated based on results, e.g. the `%alts` schematic, which tries a sequence of sub-builds until one succeeds. If the first sub-build succeeds, the build depends only on that first sub-build, but if the first fails and the second succeeds, the build depends on both.

This dynamicity implies we don't know what we depend on until we depend on it. Most build systems have this property, but this part of Ford's job is easier than for most Unix-based build systems: Ford draws all resources from an immutable namespace, and it can track every access of that namespace.

Ford might produce a build's result asynchronously, in a subsequent Arvo event. This happens when accessing the namespace doesn't complete synchronously, such as when grabbing a file from another ship. Ford guarantees it will respond with build results in chronological order using the formal date, not the order in which the builds completed.

Ford does _not_ guarantee it will notify a subscriber of a changed build only once per change. In common usage it will not send duplicate notifications, but it might if its cache was recently wiped.

Ford uses dependency tracking and cached results of previous builds to eliminate excess work. When rerunning a live build, Ford "promotes" cached results to the new time if the build's dependencies haven't changed since the previous build's formal date. Ford also checks whether each new result is the same as the previous result, and if it is, Ford knows not to rerun builds that depend on the newly completed build. Ford also caches results of intermediate builds (sub-builds).

This cache promotion system solves the "sidebar problem," where every Fora post depends on a sidebar, which depends on the title of each post. In old Ford, a change to one Fora post would cause a rebuild of the sidebar, even if that post's title didn't change. That sidebar rebuild would in turn cause every Fora page to rebuild, which was slow enough that the page would fail to load.

In new Ford, since we check the title against the previous title, if it's the same we don't need to rebuild the sidebar or the other pages. If the title does change, we'll be able to promote the build results of the other titles and page contents (sub-builds), so Ford will only do the minimal work necessary to stay up-to-date.

New Ford will generally cache the latest result of each live build. When a new version is run, the old results will be removed from storage to free memory. Because of this, cache size should not grow with each rebuild -- new Ford will have a cache size roughly proportional to the number of live builds. Then we should be able to turn off the timer-based cache wiping on the Fora ship.

However, even though the cache should not grow indefinitely, it might still consume a lot of memory, so it's important for Ford to be able to respond to a request to trim its cache. New Ford includes a basic [LRU](https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU)) cache reclamation scheme to provide this functionality.

# Specification

## Data Structures

```
::
::  sys/ford/hoon
::
|%
::  
::  +axle: overall ford state
::
+=  axle
  $:  ::  date: date at which ford's state was updated to this data structure
      ::
      date=%~2018.3.14
      ::  state-by-ship: storage for all the @p's this ford has been
      ::
      ::    Once the cc-release boot sequence lands, we can remove this
      ::    mapping, since an arvo will not change @p identities. until
      ::    then, we need to support a ship booting as a comet before
      ::    becoming its adult identity.
      ::
      state-by-ship=(map ship ford-state)
  ==
::  +ford-state: all state that ford maintains for a @p ship identity
::
+=  ford-state
  $:  ::  results: all stored build results
      ::
      ::    Ford generally stores the result for all the most recently
      ::    completed live builds, unless it's been asked to wipe its cache.
      ::
      results=(map build cache-line)
      ::  builds-by-schematic: all attempted builds, sorted by time
      ::
      ::    For each schematic we've attempted to build at any time,
      ::    list the formal dates of all build attempts, sorted newest first.
      ::
      builds-by-schematic=(map schematic (list @da))
      ::  builds-by-date: all attempted builds, grouped by time
      ::
      builds-by-date=(map @da (set schematic))
      ::  components: bidirectional linkages between sub-builds and clients
      ::
      ::    The first of the two jugs maps from a build to its sub-builds.
      ::    The second of the two jugs maps from a build to its client builds.
      ::
      components=(bi-jug sub-build=build client=build)
      ::  blocks: map from +dependency to all builds waiting for its retrieval
      ::
      blocks=(jug dependency build)
  ::
  ::  build request tracking
  ::
      ::  listeners: external requests for a build, both live (:live=&) and once
      ::
      listeners=(jug build [listener=duct live=?])
      ::  builds-by-listener: reverse lookup for :listeners; find build by duct
      ::
      builds-by-listener=(map duct build)
  ::
  ::  update tracking
  ::
      ::  live-leaf-builds: which live builds depend on a live +dependency
      ::
      live-leaf-builds=(jug dependency build)
      ::  live-root-builds: which live builds depend on any files in a +disc
      ::
      live-root-builds=(jug disc build)
      ::  dependencies: dependencies of a live build
      ::
      dependencies=(map build (jug disc dependency))
      ::  dependency-updates: all clay updates we need to know about
      ::
      ::    dependency-updates stores all Clay changes at dates that
      ::    Ford needs to track because Ford is tracking attempted builds with
      ::    that formal date. The changed dependencies are grouped first by
      ::    date, then within a single date, they're grouped by +disc.
      ::
      dependency-updates=(map @da (jug disc dependency))
  ==
::  +build: a referentially transparent request for a build.
::
::    Each unique +build will always produce the same +build-result
::    when run (if it completes). A live build consists of a sequence of
::    instances of +build with the same :plan and increasing :date.
::
+=  build
  $:  ::  date: the formal date of this build; unrelated to time of execution
      ::
      date=@da
      ::  plan: the schematic that determines how to run this build
      ::
      plan=schematic
  ==
::  +cache-line: a record of our result of running a +build
::
::    Proof that a build has been run. Might include the result if Ford is
::    caching it. If Ford wiped the result from its cache, the result will
::    be replaced with a tombstone so Ford still knows the build has been
::    run before. Otherwise, contains the last accessed time of the result,
::    for use in cache reclamation.
::
+=  cache-line
  $%  ::  %result: the result of running a +build, and its last access time
      ::
      $:  %result
          ::  last-accessed: the last time this result was accessed
          ::
          ::    Updated every time this result is used in another build or
          ::    requested in a build request.
          last-accessed=@da
          ::  result: the referentially transparent result of a +build
          ::
          result=build-result
      ==
      ::  %tombstone: marker that this build has been run and its result wiped
      ::
      [%tombstone ~]
  ==
::  +bi-jug: bi-directional jug
::
::    A pair of jugs. The first jug maps from :a to :b; the second maps
::    backward from :b to :a. Both jugs must be kept in sync with each other to
::    be a valid bi-jug. If :a and :b are the same, it can be used to represent
::    arbitrary DAGs traversable in approximately log time in either direction.
::
++  bi-jug
  |*  $:  ::  a: key type for forward mapping; value type for backward mapping
          ::
          a=mold
          ::  b: value type for forward mapping: key type for backward mapping
          ::
          b=mold
      ==
  (pair (jug a b) (jug b a)))
--
::
::  sys/zuse/hoon
::
::  |ford: build system vane interface
::
++  ford  ^?
  |%
  ::  |able:ford: ford's public +move interface
  ::
  ++  able  ^?
    |%
    ::  +task:able:ford: requests to ford
    ::
    +=  task
      $%  ::  %make: perform a build, either live or once
          ::
          $:  %make
              ::  plan: the schematic to build
              ::
              plan=schematic
              ::  date: the formal date of the build, or ~ for live
              ::
              date=(unit @da)
          ==
          ::  %kill: stop a build; send on same duct as original %make request
          ::
          [%kill ~]
          ::  %wegh: produce memory usage information
          ::
          [%wegh ~]
          ::  %wipe: clear cache
          ::
          [%wipe ~]
      ==
    ::  +gift:able:ford: responses from ford
    ::
    +=  gift
      $%  ::  %mass: memory usage; response to %wegh +task
          ::
          [%mass p=mass]
          ::  %made: build result; response to %make +task
          ::
          $:  %made
              ::  date: formal date of the build
              ::
              date=@da
              ::  result: result of the build; either complete build, or error
              ::
              $=  result
              $%  ::  %complete: contains the result of the completed build
                  ::
                  [%complete build-result]
                  ::  %incomplete: couldn't finish build; contains error message
                  ::
                  [%incomplete tang]
      ==  ==  ==
    --
  ::  +disc: a desk on a ship; can be used as a beak that varies with time
  ::
  +=  disc  (pair ship desk)
  ::  +view: a possibly time-varying view of a +disc
  ::
  ::    Either it contains a +case, which means it's pinned to a time,
  ::    or the (unit case) is ~, meaning the time can vary.
  ::
  +=  view  (trel ship desk (unit case))
  ::  +rail: a possibly time-varying full path
  ::
  ::    Either its +view contains a +case, which means it's pinned to a time
  ::    (in which case it's equivalent to a +beam), or the (unit case) is ~,
  ::    meaning the time can vary.
  ::
  +=  rail  (pair view spur)
  ::  +dependency: dependency on a value from the urbit namespace
  ::
  +=  dependency
    $%  ::  live dependency on a clay path; varies with time
        ::
        [%clay-live ren=care:clay bel=(pair disc spur)]
        ::  once dependency on a clay path; pinned time
        ::
        [%clay-once ren=care:clay bem=beam]
        ::  live dependency on a gall path; varies with time
        ::
        [%gall-live ren=care:clay bel=(pair disc spur)]
        ::  once dependency on a gall path; pinned time
        ::
        [%gall-once ren=care:clay bem=beam]
    ==
  ::  +build-result: the referentially transparent result of a +build
  ::
  ::    A +build produces either an error or a result. A result is a tagged union
  ::    of the various kinds of datatypes a build can produce. The tag represents
  ::    the sub-type of +schematic that produced the result.
  ::
  +=  build-result
    $%  ::  %error: the build produced an error whose description is :message
        ::
        [%error message=tang]
        ::  %result: result of successful +build, tagged by +schematic sub-type
        ::
        $:  %result
            $^  [head=build-result tail=build-result]
            $%  [%$ cage]
                [%alts build-result]
                [%bake cage]
                [%bunt cage]
                [%call vase]
                [%cast cage]
                [%core vase]
                [%dude build-result]
                [%file cage]
                [%hood hood]
                [%path (pair disc spur)]
                [%plan vase]
                [%reef vase]
                [%ride vase]
                [%slit type]
                [%slim (pair type nock)]
                [%scry cage]
                [%vale cage]
                [%volt cage]
            ::
            ::  clay diff and merge operations
            ::
                [%diff cage]
                [%join cage]
                [%mash cage]
                [%mute cage]
                [%pact cage]
    ==  ==  ==
  ::
  ::  +schematic: plan for building
  ::
  ++  schematic
    ::    If the head of the +schematic is a pair, it's an auto-cons
    ::    schematic. Its result will be the pair of results of its
    ::    sub-schematics.
    ::
    $^  [head=schematic tail=schematic]
    ::
    $%  ::  %$: literal value. Produces its input unchanged. 
        ::
        $:  %$
            ::  literal: the value to be produced by the build
            ::
            literal=cage
        ==
        ::  %alts: alternative build choices
        ::
        ::    Try each choice in :choices, in order; accept the first one that
        ::    succeeds. Note that the result inherits the dependencies of all
        ::    failed schematics, as well as the successful one.
        ::
        $:  %alts
            ::  choices: list of build options to try
            ::
            choices=(list schematic)
        ==
        ::  %bake: run a file through a renderer
        ::
        $:  %bake
            ::  renderer: name of renderer; also its file path in ren/
            ::
            renderer=term
            ::  query-string: the query string of the renderer's http path
            ::
            query-string=coin
            ::  path-to-render: full path of file to render
            ::
            path-to-render=rail
        ==
        ::  %bunt: produce the default value for a mark
        ::
        $:  %bunt
            ::  location: where in clay to load the mark from
            ::
            location=view
            ::  mark: name of mark; also its file path in mar/
            ::
            mark=term
        ==
        ::  %call: call a gate on a sample
        ::
        $:  %call
            ::  gate: schematic whose result is a gate
            ::
            gate=schematic
            ::  sample:  schematic whose result will be the gate's sample
            ::
            sample=schematic
        ==
        ::  %cast: cast the result of a schematic through a mark
        ::
        $:  %cast
            ::  location: where in clay to load the mark from
            ::
            location=view
            ::  mark: name of mark; also its file path in ren/
            ::
            mark=term
            ::  input: schematic whose result will be run through the mark
            ::
            input=schematic
        ==
        ::  %core: build a hoon program from a source file
        ::
        $:  %core
            ::  source-path: clay path from which to load hoon source
            ::
            source-path=rail
        ==
        ::  %dude: wrap a failure's error message with an extra message
        ::
        $:  %dude
            ::  error: a trap producing an error message to wrap the original
            ::
            error=(trap tank)
            ::  attempt: the schematic to try, whose error we wrap, if any
            ::
            attempt=schematic
        ==
        ::  %hood: create a +hood from a hoon source file
        ::
        $:  %hood
            ::  source-path: clay path from which to load hoon source
            ::
            source-path=rail
        ==
        ::  %path: resolve a path with `-`s to a path with `/`s
        ::
        ::    Resolve +file-path to a path containing a file, replacing
        ::    any `-`s in the path with `/`s if no file exists at the
        ::    original path. Produces an error if multiple files match,
        ::    e.g. a/b/c and a/b-c, or a/b/c and a-b/c.
        ::
        ::    TODO verify current implementation
        ::
        $:  %path
            ::  location: the +disc within which to resolve :file-path
            ::
            location=disc
            ::  file-path: the path to resolve
            ::
            file-path=@tas
        ==
        ::  %plan: build a hoon program from a preprocessed source file
        ::
        $:  %plan
            ::  source-path: the clay path of the hoon source file
            ::
            source-path=rail
            ::  query-string: the query string of the http request
            ::
            query-string=coin
            ::  assembly: preprocessed hoon source and imports
            ::
            assembly=hood
        ==
        ::  %reef: produce a hoon+zuse kernel. used internally for caching
        ::
        [%reef ~]
        ::  %ride: eval hoon as formula with result of a schematic as subject
        ::
        $:  %ride
            ::  formula: a hoon to be evaluated against a subject
            ::
            formula=hoon
            ::  subject: a schematic whose result will be used as subject
            ::
            subject=schematic
        ==
        ::  %scry: lookup a value from the urbit namespace
        ::
        $:  %scry
            ::  request: the request to be made against the namespace
            ::
            request=dependency
        ==
        ::  %slim: compile a hoon against a subject type
        ::
        $:  %slim
            ::  compile-time subject type for the :formula
            ::  
            subject-type=type
            ::  formula: a +hoon to be compiled to (pair type nock)
            ::
            formula=hoon
        ==
        ::  %slit: get type of gate product
        ::
        $:  %slit
            ::  gate: a vase containing a gate
            ::
            gate=vase
            ::  sample: a vase containing the :gate's sample
            ::
            sample=vase
        ==
        ::  %vale: coerce a noun to a mark, validated
        ::
        $:  %vale
            ::  location: where in clay to load the mark from
            ::
            location=view
            ::  mark: name of mark to use; also file path in mar/
            ::
            mark=term
            ::  input: the noun to be converted using the mark
            ::
            input=*
        ==
        ::  %volt: coerce a noun to a mark, unsafe
        ::
        $:  %volt
            ::  location: where in clay to load the mark from
            ::
            location=view
            ::  mark: name of mark to use; also file path in mar/
            ::
            mark=term
            ::  input: the noun to be converted using the mark
            ::
            input=*
        ==
    ::
    ::  clay diff and merge operations
    ::
        ::  %diff: produce marked diff from :first to :second
        ::
        $:  %diff
            ::  location: where in clay to load the mark from
            ::
            location=view
            ::  old: schematic producing data to be used as diff starting point
            ::
            start=schematic
            ::  new: schematic producing data to be used as diff ending point
            ::
            end=schematic
        ==
        ::  %join: merge two diffs into one diff; produces `~` if conflicts
        ::
        $:  %join
            ::  location: where in clay to load the mark from
            ::
            location=view
            ::  mark: name of the mark to use for diffs; also file path in mar/
            ::
            mark=term
            ::  first: schematic producing first diff
            ::
            first=schematic
            ::  second: schematic producing second diff
            ::
            second=schematic
        ==
        ::  %mash: force a merge, annotating any conflicts
        ::
        $:  %mash
            ::  location: where in clay to load the mark from
            ::
            location=view
            ::  mark: name of mark used in diffs; also file path in mar/
            ::
            mark=term
            ::  first: schematic producing first diff
            ::
            first=schematic
            ::  second: schematic producing second diff
            ::
            second=schematic
        ==
        ::  %mute: mutate a noun by replacing its wings with new values
        ::
        $:  %mute
            ::  subject: schematic producing the noun to mutate
            ::
            subject=schematic
            ::  mutations: axes and schematics to produce their new contents
            ::
            mutations=(list (pair wing schematic))
        ==
        ::  %pact: patch a marked noun by applying a diff
        ::
        $:  %pact
            ::  location: where in clay to load the mark from
            ::
            location=view
            ::  mark: name of mark to use in diff; also file path in mar/
            ::
            mark=term
            ::  start: schematic producing a noun to be patched
            ::
            start=schematic
            ::  diff: schematic producing the diff to apply to :start
            ::
            diff=schematic
        ==
    ==
  ::
  ::  +scaffold: program construction in progress
  ::
  ::    A source file with all its imports and requirements, which will be
  ::    built and combined into one final product.
  ::
  +=  scaffold
    $:  ::  zuse-version: the kelvin version of the standard library
        ::
        zuse-version=@ud
        ::  structures: files from %/sur which are included
        ::
        structures/(list cable)
        ::  libraries: files from %/lib which are included
        ::
        libraries/(list cable)
        ::  cranes: a list of resources to transform and include
        ::
        cranes/(list crane)
        ::  sources: hoon sources, either parsed or on the filesystem
        ::
        sources/(list brick)
    ==
  ::  +cable: a reference to something on the filesystem
  ::
  +=  cable
    $:  ::  expand-namespace: expose internal faces to subject
        ::
        expand-namespace=?
        ::  file-path: location in clay
        ::
        file-path=term
        ::  remote-location: if not `~`, remote location of file
        ::
        remote-location=(unit (pair case ship))
    ==
  ::  +brick: hoon code, either directly specified or referencing clay
  ::
  +=  brick
    $%  $:  ::  %direct: inline parsed hoon
            ::
            %direct
            source=hoon
        ==
        $:  ::  %indirect: reference to a hoon file in clay
            ::
            %indirect
            location=beam
    ==  ==
  ::  +truss: late-bound path
  ::
  ::    TODO: the +tyke data structure should be rethought, possibly as part
  ::    of this effort since it is actually a `(list (unit hoon))`, when it
  ::    only represents @tas. It should be a structure which explicitly
  ::    represents a path with holes that need to be filled in.
  ::
  +=  truss
    $:  pre/(unit tyke)
        pof/(unit {p/@ud q/tyke})
    ==
  ::  +crane: parsed rune used to include and transform resources
  ::
  ::    Cranes lifting cranes lifting cranes!
  ::
  ::    A recursive tree of Ford directives that specifies instructions for
  ::    including and transforming resources from the Urbit namespace.
  ::
  +=  crane
    $%  $:  ::  %fssg: `/~` hoon literal
            ::
            ::    `/~ <hoon>` produces a crane that evaluates arbitrary hoon.
            ::
            %fssg
            =hoon
        ==
        $:  ::  %fsbc: `/$` process query string
            ::
            ::    `/$` will call a gate with the query string supplied to this
            ::    build. If no query string, this errors.
            ::
            %fsbc
            =hoon
        ==
        $:  ::  %fsbr: `/|` first of many options that succeeds
            ::
            ::    `/|` takes a series of cranes and produces the first one
            ::    (left-to-right) that succeeds. If none succeed, it produces
            ::    stack traces from all of its arguments.
            ::
            %fsbr
            ::  choices: cranes to try
            ::
            choices=(list crane)
        ==
        $:  ::  %fsts: `/=` wrap a face around a crane
            ::
            ::    /= runs a crane (usually produced by another ford rune), takes
            ::    the result of that crane, and wraps a face around it.
            ::
            %fsts
            ::  face: face to apply
            ::
            face=term
            ::  crane: internal build step
            ::
            =crane
        ==
        $:  ::  %fsdt: `/.` null-terminated list
            ::
            ::    Produce a null-terminated list from a sequence of cranes,
            ::    terminated by a `==`.
            ::
            %fsdt
            ::  items: cranes to evaluate
            ::
            items=(list crane)
        ==
        $:  ::  %fscm: `/,` switch by path
            ::
            ::    `/,` is a switch statement, which picks a branch to evaluate
            ::    based on whether the current path matches the path in the
            ::    switch statement. Takes a sequence of pairs of (path, crane)
            ::    terminated by a `==`.
            ::
            %fscm
            ::  cases: produces evaluated crane of first +spur match
            ::
            cases=(list (pair spur crane))
        ==
        $:  ::  %fscn: `/%` propagate extra arguments into renderers
            ::
            ::    `/%` will forward extra arguments (usually from Eyre) on to
            ::    any renderer in :crane. Without this, renderers that use `/$`
            ::    to read the extra arguments will crash.
            ::
            %fscn
            =crane
        ==
        $:  ::  %fspm: `/&` pass through a series of marks
            ::
            ::    `/&` passes a crane through multiple marks, right-to-left.
            ::
            %fspm
            ::  marks: marks to apply to :crane, in reverse order
            ::
            marks=(list mark)
            =crane
        ==
        $:  ::  %fscb: `/_` run a crane on each file in the current directory
            ::
            ::    `/_` takes a crane as an argument. It produces a new crane
            ::    representing the result of mapping the supplied crane over the
            ::    list of files in the current directory. The keys in the
            ::    resulting map are the basenames of the files in the directory,
            ::    and each value is the result of running that crane on the
            ::    contents of the file.
            %fscb
            =crane
        ==
        $:  ::  %fssm: `/;` operate on
            ::
            ::    `/;` takes a hoon and a crane. The hoon should evaluate to a
            ::    gate, which is then called with the result of the crane as its
            ::    sample.
            ::
            %fssm
            =hoon
            =crane
        ==
        $:  ::  %fscl: `/:` evaluate at path
            ::
            ::    `/:` takes a path and a +crane, and evaluates the crane with
            ::    the current path set to the supplied path.
            ::
            %fscl
            ::  path: late bound path to be resolved relative to current beak
            ::
            ::    This becomes current path of :crane
            ::
            path=truss
            =crane
        ==
        $:  ::  %fskt: `/^` cast
            ::
            ::    `/^` takes a +mold and a +crane, and casts the result of the
            ::    crane to the mold.
            ::
            %fskt
            ::  mold: evaluates to a mold to be applied to :crane
            ::
            mold=hoon
            =crane
        ==
        $:  ::  %fszp: `/!mark/` evaluate as hoon, then pass through mark
            ::
            %fszp
            =mark
        ==
        $:  ::  %fszy: `/mark/` passes current path through :mark
            ::
            %fszy
            =mark
    ==  ==
  --
```

Ford's main API is the `%make` task, which produces one or more `%made` responses. If the `%make` contains a date, that becomes the formal date of a once build, which will send a single `%made` response to conclude the contract, unless a `%kill` request is sent on the same duct as the build request before it completes. Otherwise the build is a live build, which will send `%made` responses for each change to the build result until the subscription is canceled by sending a `%kill` request on the same duct as the build request.

If Ford is unable to complete a build because the build tried to access part of the namespace that was unavailable (and Ford knows it's unavailable, which isn't the case for files on other ships), it will send an `%incomplete` `%made` response to indicate its inability to complete the build. This maintains referential transparency for builds Ford can't complete. If asked again to perform the build, Ford will rerun the build and might complete it if that part of the namespace is now available.

If Ford is unable to complete a build in a sequence of live builds, Ford will send an `%incomplete` `%made` response for that build, but will continue producing subsequent builds when dependencies change.

The `%wipe` task causes Ford to clear half its cached results. There is no ack for this task. While we previously thought we were going to perform in-vane cache clearing (see UP 1), we are now targeting a model where vere will send a memory pressure event to arvo, which would send %wipe events to the vanes.

The `%wegh` task is a request from Arvo asking Ford to "weigh" its memory usage. Ford responds with a `%mass` gift.

## Control Flow

Ford's major construction entry points are:

- `++start-build`
- `++rebuild`
- `++unblock`

All of these call `++execute`, the generic build/rebuild gate. The other major tasks Ford can be asked to perform are:

- `++cancel`, which cancels a build and any sub-builds, and
- `++reclaim-cache`, which trims stored build results to free memory.

### `++start-build`

1. Call `++execute` on the sample `+=build`.

### `++rebuild`

- We hear from Clay about changed dependencies.
- Store the dependency updates from Clay in `dependency-updates.ford-state`.
- Look up in `live-leaf-builds.ford-state` to find the potentially affected builds.
- Run `++execute` on each of those potentially affected leaf builds.

### `++unblock`

- We hear from Clay about unblocked dependencies.
- We look up what builds are linked to those dependencies, in `blocked.ford-state`.
- We run `++execute` on those builds.

### `++cancel`

- Look up `duct` in `builds-by-listener.ford-state` to find the build to cancel.
- Remove the duct from the build.
- Run `++cleanup` on the build.

### `++reclaim-cache`

Cache reclamation entails removing the results of builds from storage to free memory. This must be kept separate from dependency tracking, which is the domain of `++cleanup`. Iterate linearly through the `results.ford-state`, sorting by `last-accessed`. When we trim a `+=cached-result`, replace the `+=build-result` with a `+=tombstone` so we still know the build has been completed.

### Internal Logic

The function responsible for most of Ford's internal control flow is `++execute`, which traverses a graph of builds and dependencies, building as much as it can and staging side effects (moves) for later release. When it needs to run a build, it calls `++make`, which either performs the build or blocks. `++execute` handles all traversal from build to build, and `++make` handles the logic within a single `+=build`.

#### `++execute`

`++execute` takes in a `+=build` as its sample, which is a `(pair @da schematic)`.

First, check whether any builds exist in `builds-by-schematic.ford-state` for the sample schematic and date. For the sample `+=schematic`, if the first date elements we find in the `(list @da)` are newer than the sample time (the time passed in for the Ford event), then skip those elements, because future builds of the sample schematic can't affect this current requested build. If we find a match with our sample time _exactly_, then use that exact build. If we find an element with a time _earlier_, but not exactly matching our sample time, then we may be able to use that build, but first we have to check whether any dependencies have changed between that build's time and our current sample time.

If we don't find an earlier build, we need to put a new build into the state (both `builds-by-schematic` and `builds-by-date` with `++create-build`. If we do find an earlier build, check whether any dependencies have changed between that build's `date` and this build's `date`. Find the old build's sub-builds in `components.ford-state`. For each of them, check in `builds-by-schematic` to see if there's a more recent build for that schematic. If there is, check in `results.ford-state` to see if the more recent build has a result. If it doesn't have a result or the result is the same as the result of the older build (ignoring `last-accessed`), then we consider that sub-build unchanged.

If all sub-builds are unchanged, we need to check whether any of the old build's external dependencies intersect with any of the dependency-updates at the current build's `date`. If they do, we cannot promote the old build's result, and need to create a new build. If all sub-builds and external dependencies are unchanged, we can promote the old build's result to the new time and complete this build immediately. After promoting a build, we update the old build's clients to link to the new build following the same procedure as described below for the case in which `++make` completes a build whose result is the same as the result of the previous build of that schematic.

Now that we have a build to work on, check if the build is done by checking if it has a `+=build-result` (not a `+=tombstone`) in `results.ford-state`. If it is done, update the result's `last-accessed` field to `now` and return. Otherwise, the build isn't done, so run `++make` on it.

`++make` takes a `+=build and produces either a `+=build-result` or a `(set block)`, where a `+=block` is either a `+=dependency` or another build.

If `++make` produces a `(set block)`, then for each block, if it's a build, link the sub-build to this current build in `components.ford-state` and call `++execute` on the sub-build. If the block is a dependency, register the blocked dependency for the current build in `blocks.ford-state`.

The diagram below depicts the progression of `++execute` as it performs a fresh build. White circles are unfinished builds, and green circles are completed builds.

![+execute diagram](https://media.urbit.org/fora/proposals/posts/~2018.3.15..04.24.35..a47f~/UP-4-Ford-Turboencabulator-execute.png)

If `++make` produces a `+=build-result`, update `results.ford-state`, with `last-accessed` set to `now`.

When `++make` produces a completed build `b`, check whether its result is new or the same as the previous build. So look up `b` itself in `builds-by-schematic.ford-state`, then iterate once more in the list containing `b` to find out if there's a most recent older build for `b`'s schematic -- let's call it `a`. If `a` doesn't exist, then start rebuilding `b`'s clients (fresh build). If `a` does exist, then check `a`'s result. `b` should only be considered unchanged if `a` exists and its result is the same as `b`'s result. If that's the case, then add a new bidirectional link between that `a` and `b` in `rebuilds`.

The following diagram illustrates the changes in component linkages as some sub-builds rebuild at later formal dates. Green circles are completed builds, dotted black circles are builds we knew we didn't have to rerun, and curved green lines are bidirectional links in `rebuilds`.

![:rebuild diagram](https://media.urbit.org/fora/proposals/posts/~2018.3.15..04.24.35..a47f~/UP-4-Ford-Turboencabulator-rebuilds.png)

Otherwise, the result of the build is different then the old version. Whenever we complete a build and the result is new, we run the following procedure:

First, we find the most recent previous build of the same schematic. If that doesn't exist, we move onto the next step. But if it does exist, then we check whether it's complete, i.e. it has an entry in `results.ford-state`, either a `+=build-result` or a `+=tombstone`. If it's not complete, move onto the next step. If it is complete, check whether there are any live ducts listening on that old build. If there are, that means the newly completed build is a root build, so add the build to the set of live root builds completed in this event, which we'll use later to assemble a Clay subscription. For each of the ducts listening on the old build, attach them to this new build, and take them off of the old build. Once we've removed all of the live ducts from the old build, then run `++cleanup` on the old build.

Now we have the set of all of the ducts for this current build that we just finished. For each of those ducts, enqueue a `%made` response on that duct to notify the listener that this build is complete at the build's date. Afterwards, remove all ducts that are not live from this current build.

If the build that we just ran has clients in `components.ford-state`, recurse into those and collect their results. It's crucial that we perform all of the rebuilds at this date before rebuilding future builds.

If the build doesn't have clients, but the previous build is complete and has clients, then recurse into the old clients' schematics at the time of the new build.

Then, check if there is a future build with the same schematic (the closest one to the current build's date). If there is, then call `++execute` on that (the reason for this is if the build we just completed is actually old, and there's another build that might also finish in this event. The trick here is making sure that when we queue up these `%made` moves, we need to queue them in the correct order, such that if we run `++execute` on the future build like we just ran `++execute` on this old build that just finished, that the `%made` events get sent in chronological order in that list of moves, because that's part of Ford's contract -- the `%made` moves for any given build are to be emitted in chronological order, and it can't miss any).	

#### `++make`

`++make` takes in a `+=build` and produces either a result, or a set of things it blocked on -- either sub-builds or external dependencies. It also modifies the `ford-state` by linking the build and its sub-builds in `components.ford-state`.

If the sub-builds don't already exist in the `ford-state`, `++make` creates and stores them -- but does not run them, which allows `++execute` to handle cross-build control flow. `++make` handles each type of schematic differently.

In general, it can choose which dependencies to register based on results of other sub-builds. For example, to handle an `%alts` schematic, which represents a build with a list of fallback ("alternative") builds, `++make` registers a dependency on the first build, but only if the first build failed will it register a dependency on the second build.

To help show how `++make` works, we've included some pseudocode for the autocons schematic case (`[schematic schematic]`) and the `%alts` case, along with some supporting code.

```
+=  block
  $%  [%build build]
      [%dependency dependency]
  ==
...
++  make
  |=  b=build
  ^-  [(each build-result (set block)) ford-state]
  ?-    -.plan.b
      ^  ::  schematic sub-type: autocons
    ::
    =^  hed  +>.$  (depend-on -.plan.b b)
    =^  tal  +>.$  (depend-on +.plan.b b)
    ::
    =|  blocks=(set block)
    =?  blocks  ?=(~ hed)  (~(put in blocks) [%build -.plan.b])
    =?  blocks  ?=(~ tal)  (~(put in blocks) [%build +.plan.b])
    ::
    ?^  blocks  ::  if anything blocked, we're not done
      ::
      [[%| blocks] +>.$]
    ::
    =/  result  [u.hed u.tal]
    [[%& result] +>.$]
  ::
      %alts  ::  schematic sub-type: fallback options
    ::
    =/  alts=(list schematic)  +.plan.b
    ?~  alts  [[%& `tang`"all options failed"] +>.$]
    ::  
    =^  first-result  +>.$  (depend-on i.alts b)
    ?~  first-result
      =/  blocks  (sy ~[%build date.b i.alts])
      [[%| blocks] +>.$]
    ::
    ?:  ?=(tang u.first-result)
      ::  failure case; try the next build
      ::
      $(+.plan.b t.alts)
    ::  success case
    ::
    =/  result  u.first-result
    [[%& result] +>.$]
  ::
      ...  other schematic types
  ==
::
++  depend-on
  |=  [plan=schematic parent=build]
  ^-  [(unit build-result) ford-state]
  ::
  =/  kid=build  [date.parent plan]
  ::
  =.  +>.$  (find-or-create kid)
  =.  components  (~(put in-bi-jug components) parent kid)
  ::
  [(~(get by results) kid) +>.$]
::
++  find-or-create
  |=  b=build  ^-  ford-state
  ?:  (has-build b)
    +>.$
  (create-build b)  ::  puts build in `builds-by-date` and `builds-by-schematic`
::
++  has-build
  |=  b=build  ^-  ?
  =/  plans  (~(get by builds-by-date) date.b)
  ?~  plans  |
  (~(has by u.plans) plan.b)
```
 
#### `++subscribe-to-clay`

At the end of an event during which we performed builds (fresh build, rebuild, or unblock), we need to update our subscriptions to Clay. This involves examining which builds finished and using that information to determine which Clay subscriptions need to be made and which Clay paths we care about.

For each live root build that completed during this event, grab the keys of its jug from disk to dependency in `dependencies.ford-state`, and unify those into one set of disks representing the set of disks that we'll need to make Clay requests on. For each disk, look up all of the root builds that depend on that disk in `live-root-builds.ford-state`, and unify those into one set of root builds that depend on any of the disks.

Then, create one Clay request for each disk. To do this, for each disk, for each root build, look up the dependencies of the root build that are on this disk by recursively traversing sub-builds in `components.ford-state` and, for each sub-build, collecting its `(jug disk dependency)` if it exists in `dependencies.ford-state`. From those dependencies, we can create a set of `[care path]`s, which forms part of the Clay request. The beak in the Clay request is built from the disk and the time of the Clay file change, which we obtain from the time of any of the root builds that completed during this event (they should all be the same time).

#### `++cleanup`

`++cleanup` takes in a `+=build` sample and produces the modified Ford state. It runs the following procedure:
 
- Check if any client builds (in `components`), old builds (in `rebuilds`), or ducts (in `listeners`) depend on this build
- If somebody cares about the build, then no-op and return
- Otherwise, nobody cares about us. So delete the build in:
  - results
  - builds-by-schematic
  - builds-by-date
    - Remove the schematic for the build by date
    - Check if there are any other schematics mapped to the date
    - If there are, continue
    - If there aren't, remove the whole date key-value pair, and remove that date's entry from dependency-updates
  - components
    - For each of the build's clients, remove the build from its clients' kids
    - For each of the build's kids, remove the build from it's kids' clients
    - Store the kids in a local variable
    - Remove all of the clients of the build
    - Remove all of the kids of the build
  - rebuilds
    - Repeat what we did with `components`
  - dependencies
    - Recursively calculate the dependencies of the build and store it somewhere in this ++gc function
    - Calculate the disks of the build and store it somewhere in this ++gc function
    - Then delete the mapping in dependencies
  - blocks
    - For each dependency this build relied on, delete the build from its set
  - live-leaf-builds
    - For each dependency this build relied on, delete the build from its set
  - live-root-builds
    - For each disk this build relied on, delete the build from its set
  - deletion-queue
- Recurse on this build's kids

# Rationale

This design errs heavily on the side of reducing rebuild times. We designed it to be able to handle a spreadsheet with thousands of rows and do the minimum amount of work to keep it up-to-date.

The `:rebuilds` field in `+ford-state` is an example of this goal: it was added to obviate the need for clients of an unchanged build to all update their component linkages from the old sub-build to the new sub-build. Instead, the `:rebuilds` field stores the necessary information in such a way that only one data structure mutation needs to be performed, and the clients builds can go untouched.

Because it has more cached checkpoints per build than previous Ford, it will generally use more memory per build -- although the added granularity might also decrease duplication somewhat. Because it cleans up after itself properly, we expect its memory footprint to be safe for most use cases.

We erred on the side of simplicity rather than performance with the cache reclamation algorithm, which performs the only full traversal of one of Ford's major data structures. Everything else uses a secondary index to avoid a traversal, but we expect cache reclamation to execute infrequently enough that we triaged designing a priority queue that also supports random access. If this assumption proves incorrect, it should be possible to improve the performance of reclamation without altering the rest of the design.

# Integration plan

We plan to test this into existence, schematic by schematic. Here's the sequence of schematics to test:

- `%$` no-op
- `%dude` adds error message
- `^` auto-cons (pair of schematics)
- `%alts` fallback options
- Namespace Accessors
  - `%scry` simplest namespace schematic: (sync, async) x (Clay, Gall)
  - `%path` convert a Clay path from `/a-b-c` to `/a/b/c`; test async 
- Hoon compilation and related tasks
  - `%reef` simplest compilation task: Hoon+Zuse kernel
  - `%slim` compile a hoon against a subject type
  - `%slit` get type of gate product (using the Hoon compiler)
  - `%ride` run schematic through a hoon
  - `%call` call a Hoon gate
  - `%hood` create a +hood from a Hoon source file
  - `%plan` build pre-processed Hoon source
  - `%core` build a Hoon source file (test without +cranes)
- Mark handling
  - `%bunt` simplest mark schematic: bunt (produce the default value of) a mark
  - `%volt` convert a value to a mark, unsafe
  - `%vale` convert a value to a mark, validated
  - `%cast` run a file through a mark
  - `%bake` run a file through a renderer
  - `%core` build a Hoon source file (test with +cranes, one +crane at a time)
- Clay diffing:
  - `%diff` diff
  - `%join` merge
  - `%pact` patch
  - `%mash` annotate
  - `%mute` mutant

In order to test these schematics, we'll need a test harness for Ford. We'll start new Ford as a library to facilitate testing. To test the first few schematics, we'll need only a very basic test harness. Once we test schematics that involve Clay files, we'll need our test harness to include a mock Clay. This mock Clay might be ad-hoc just for this testing, or it might be a userspace Clay compiled from the real Clay source.

Since we'll be testing a new Ford into existence, what sort of test harness does that imply?

A vane has an interface where it either:

- `++take`s in a vane specific `++sign` and produces a `(list move)` and new
  core state.
- `++call`'s in a vane specific `++task` and produces a `(list move)` and new
  core state.

Conceptually, we'll only have a few operations to test Ford. These apply to all vanes, so we will build the test harness with the expectation that we will generalize it to any vane:

- Set data in the scry namespace for Ford to query with its injected namespace-accessing function (a `+sley`).
- Pass in a `++sign` to `++take`, receive a list of moves.
- Pass in a `++task` to `++call`, receive a list of moves.
- Make assertions about the last list of moves that Ford sent out (either `+gift`s or `+note`s).
- Make assertions about the vane's internal state. This is tricky (or potentially not possible) for a live vane, because vanes are lead cores (`^?`) with opaque contexts. This problem can be sidestepped while testing Ford into its initial existence by making fetal Ford a library in `/lib`.

What we want is a way to evaluate these in order. We could make a list of these actions to run serially. It might be more flexible to provide a `=^`-based API, though.

Here is some pseudocode to test bunting a mark:

```
++  test-bunt
  ::  add a file to the test-vane scry namespace
  ::
  =.  scry-state  (put-in-scry %c /mar/noun/hoon .^(@t %cx /===/mar/noun/hoon))
  ::  ask ford to %bunt a mark; this should access the namespace synchronously
  ::
  ::    This produces a list of moves and a mutated :ford-core that we could use
  ::    later in this test to get Ford to perform a sequence of operations.
  ::
  =^  mow  ford-core  (send-task %f ford-core [%make [%bunt %noun] `now])
  ::  ford should produce one move, immediately
  ::
  ?~  mow  (fail-test "%bunt should be synchronous")
  ?^  t.mow  (fail-test "%bunt should only produce one move") 
  ::  ford's move should contain a successful result: `0`
  ::  
  (expect-gift %f i.mow [%made date=now result=[%complete [%result [%bunt 0]]]])
```

The testing will also entail verifying cache performance and freeing of resources, as well as testing the various edge cases to verify the algorithm:

- A once build that overlaps with a live build
- A leaf build changes, but not its parent
- Two or more live builds overlap
- A live build is cancelled that's a subset of another live build
- A once build is requested at a future date
- Multiple incomplete live rebuilds of the same schematic (test the result order)
- A once build with a Gall scry at a date older than the current time -- should send `%incomplete` response
- A sequence of live builds with some that don't complete -- should continue rebuilding later builds
- Cache reclamation of an incomplete once build

Other changes that need to be made to integrate this work:
- Change the Ames API to relinquish the `%make` task.
- Remove the `/#` Ford rune. Uses of that rune should instead request multiple builds that nest within one another, inside an autocons if synchronization is desired.
- [`/_` should not cause renderers to fail silently](https://github.com/urbit/arvo/issues/655)

Due to the number of changes needed both to Ford and to other parts of Arvo to support this work, we plan to breach in the changes.

Ongoing work can be found in the [ford-turbo](https://github.com/urbit/arvo/tree/ford-turbo) branch.