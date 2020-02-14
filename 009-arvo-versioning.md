+++
up = 9
type = "protocol"
title = "Arvo Versioning"
created = 2019-12-10
updated = 2020-02-14
authors = [
  "~nidsut-tomdun Jared Tobin <jared@tlon.io>"
]
status = draft
+++

## Abstract

This document describes the versioning scheme used by the Arvo kernel.  It
consists of two parts:

* the versioning scheme used to describe the individual components that make up
  the Arvo kernel (i.e. of Hoon, the vanes, and so on)
* the versioning scheme used to describe the Arvo kernel as a whole (for
  release and distribution)

## Kelvin Versioning (Overview)

Kelvin versioning was presented in the blog post ["Toward a Frozen Operating
System"][froz].  It is designed to be a versioning system for components that
should eventually freeze (i.e., such that it at some point becomes impossible
to upgrade them any further).

Its informal definition, excerpted from that post, is as follows:

> In Kelvin versioning, a version is an integer in degrees Kelvin. Absolute
> zero is frozen — no further updates are possible. If your Kelvin versions
> don't track your actual progress, you run out of integers.
>
> If tool B sits on platform A, either both A and B must be at absolute zero,
> or B must be warmer than A.
>
> Whenever the temperature of A (the platform) declines, the temperature of B
> (the tool) must also decline.
>
> Of course, if B itself is a platform on which some higher-level tool C
> depends, it must follow the same constraints recursively.

To be clear, "warmer" means "must have version number higher."  If some
platform A is at version 5K and some component B is at version 10K, then B is
warmer than A.  "Colder" follows analogously.

The recursive part of the scheme is often referred to as "telescopic kelvins"
or "telescoping kelvins," but "kelvin versioning" should be interpreted as
subsuming this particular characteristic.

## Kelvin Versioning (Specification)

(The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119][rfcm].)

To restate the above in slightly more formal terms: for any component A
following a kelvin versioning scheme,

* A's version **SHALL** be a nonnegative integer.

* At version 0, new versions of A **MUST NOT** be released.

* New releases of A **MUST** be assigned a new version, and this
  version **MUST** be strictly less than the previous one.

If A supports another component B that also follows a kelvin versioning scheme,
then:

* Either both A and B **MUST** be at version 0, or B's version **MUST** be
  strictly greater than A's version.

* If a new version of A is released and that version supports B, then a new
  version of B **MUST** be released.

These rules apply recursively for any kelvin-versioned component C that is
supported by B, and so on.

## Component Versioning in the Arvo Kernel

The Arvo kernel consists of a number of components arranged in layers.  These,
as of arvo-v309.9 (see [Versioning the Entire
Kernel](#versioning-the-entire-kernel)), are:

* Nock
* Hoon, which is supported by Nock
* Arvo, which is supported by Hoon
* Zuse, which is supported by Arvo
* the vanes (kernel modules e.g. %ames, %behn, etc.), which are supported by
  Zuse

The kernel is thus characterised by five layers, with components in all layers
following a kelvin versioning scheme.

Each lower layer supports the ones above it.  I.e.,

* either Nock and Hoon must both be at 0K, or Hoon is warmer than Nock
* either Hoon and Arvo must both be at 0K, or Arvo is warmer than Hoon
* either Arvo and Zuse must both be at 0K, or Zuse is warmer than Arvo
* either Zuse and any given vane must be at 0K, or the vane is warmer
  than Zuse

Similarly, we have that:

* if Nock cools, then everything else must cool
* if Hoon cools, then Arvo, Zuse, and the vanes must cool
* if Arvo cools, then Zuse and the vanes must cool
* if Zuse cools, then the vanes must each cool

Concretely, the kernel as of arvo-v309.9 consists of the following components
having the associated kelvin versions:

```
* Nock 4K
  * Hoon 141K
    * Arvo 225K
      * Zuse 309K
        * Ames 400K
        * Behn 310K
        * Clay 550K
        * Dill 375K
        * Eyre 900K
        * Ford 800K
        * Gall 1200K
        * Iris 600K
        * Jael 700K
```

Following are several hypothetical examples based on this kernel that
illustrate how the component-level versioning scheme works.

### Example 1

Let's imagine we have accumulated some changes to Ames that we now want to
release.

Since Ames's present version is 400K, we release the changes in a new version,
399K.  The new kernel state is:

```
* Nock 4K
  * Hoon 141K
    * Arvo 225K
      * Zuse 309K
        * Ames 399K   <- changes from 400K to 399K
        * Behn 310K
        * Clay 550K
        * Dill 375K
        * Eyre 900K
        * Ford 800K
        * Gall 1200K
        * Iris 600K
        * Jael 700K
```

### Example 2

Now let's say we have discovered and fixed a bug in Hoon and now want to
release the fix.  Hoon's present version is 141K, so we release the fix in a
new version, 140K.

Since Hoon 140K supports Arvo, Zuse, and each of the vanes, we are obligated to
release new versions of each of these components.  The new kernel state is:

```
* Nock 4K
  * Hoon 140K         <- changes from 141K to 140K
    * Arvo 224K       <- changes from 225K to 224K
      * Zuse 308K     <- changes from 309K to 308K
        * Ames 398K   <- changes from 399K to 398K
        * Behn 309K   <- ..
        * Clay 549K   <- ..
        * Dill 374K   <- ..
        * Eyre 899K   <- ..
        * Ford 799K   <- ..
        * Gall 1199K  <- ..
        * Iris 599K   <- ..
        * Jael 699K   <- changes from 700K to 699K
```

### Example 3

Now let's say someone has made a patch to Behn, and we want to release a new
version that includes it.  However: we are *prohibited* from releasing a new
version of Behn in the present kernel state.

Recall that Zuse, which supports Behn, must be strictly cooler than Behn, or
both Zuse and Behn must be at version 0.  Since Zuse is at 308K, one degree
less than Behn's current version of 309K, any new Behn release would produce an
illegal state.

### Example 4

Now let's say we've made a number of performance improvements to Zuse that we
want to push out.

We do so, releasing a new version of Zuse at 307K, and are thus obligated to
release new versions of each of the vanes that Zuse 307K supports.  The
resulting kernel state is:

```
* Nock 4K
  * Hoon 140K
    * Arvo 224K
      * Zuse 307K     <- changes from 308K to 307K
        * Ames 397K   <- ..
        * Behn 308K   <- changes from 309K to 308K
        * Clay 548K   <- ..
        * Dill 373K   <- ..
        * Eyre 898K   <- ..
        * Ford 798K   <- ..
        * Gall 1198K  <- ..
        * Iris 598K   <- ..
        * Jael 698K   <- changes from 699K to 698K
```

Note that we're obligated to release a new version of Behn, in particular, so
presumably we'd include the patch from Example 3 in our Behn 308K release.

### Example 5

Finally, imagine that we've undertaken a major refactoring of the kernel, such
that the vanes Clay and Gall have been replaced by a single new vane called
Hume.  The vane API definitions live in Zuse, so we define the +task and +gift
interface for Hume there and delete the existing ones for Clay and Gall.

We push out a new version of Zuse, 306K, that contains the changes, and the
release of a new Zuse version obligates us to release new versions of each of
the vanes it supports.  The resulting kernel state is as follows:

```
* Nock 4K
  * Hoon 140K
    * Arvo 224K
      * Zuse 306K     <- changes from 307K to 306K
        * Ames 396K   <- changes from 397K to 396K
        * Behn 307K   <- ..
        * Dill 372K   <- ..
        * Eyre 897K   <- ..
        * Ford 797K   <- ..
        * Hume 1000K  <- initial release at 1000K
        * Iris 597K   <- ..
        * Jael 697K   <- changes from 699K to 698K
```

Note that Clay and Gall have been removed and Hume has been added.  Since Zuse
306K doesn't support Clay or Gall, we're not obligated to release new versions
of those vanes.

The initial version for Hume was chosen arbitrarily -- the only restriction is
that it be strictly warmer than Zuse, which supports it.

## Release Candidates

Note that it is perfectly legal to ship any number of *release candidates* for
any component of the Arvo kernel.  These are simply denoted by a conventional
upward-counting suffix of .rc1, .rc2, and so on.

There are no other particular restrictions on release candidates.

## Versioning the Entire Kernel

In addition to the individual components, we also distinguish between different
versions of the kernel as a whole for distribution, deployment to the network,
etc.

To do this while rigorously tying the kernel's version to the state of its
components, we use a minor variation of the kelvin versioning scheme used for
any given component.

The whole kernel's version is described by the format:

```
arvo-v<zuse version>.<fractional temperature>
```

where *<zuse version>* is the version of Zuse included in this kernel release,
and *<fractional temperature>* is a downward-decrementing counter that changes
whenever any component supported by Zuse changes.

Note that the only mandate on the fractional temperature is that it decrease
whenever components supported by Zuse decrease.  We don't need to decrease the
fractional temperature from .9 to .7 because it contains *two* new vane
versions, for example.

As a convention, the fractional temperature should usually start at .9 (for
a new Zuse version) and decrease as follows:

```
.9, .8, ..., .1, .01, .001, .0001, .00001, ...
```

Whenever the Zuse version changes, the fractional temperature should be reset
to .9.

### Example 6

Consider the initial kernel state of the previous series of examples:

```
* Nock 4K
  * Hoon 141K
    * Arvo 225K
      * Zuse 309K
        * Ames 400K
        * Behn 310K
        * Clay 550K
        * Dill 375K
        * Eyre 900K
        * Ford 800K
        * Gall 1200K
        * Iris 600K
        * Jael 700K
```

This kernel could be released as `arvo-v309.9`, since it includes Zuse at 309K.

### Example 7

Now imagine we release new versions of Ames and Jael.  The new kernel state is:

```
* Nock 4K
  * Hoon 141K
    * Arvo 225K
      * Zuse 309K
        * Ames 399K   <- changes from 400K to 399K
        * Behn 310K
        * Clay 550K
        * Dill 375K
        * Eyre 900K
        * Ford 800K
        * Gall 1200K
        * Iris 600K
        * Jael 699K   <- changes from 700K to 699K
```

This kernel could be released as `arvo-v309.8`, with the fractional temperature
having decreased due to the inclusion of a couple of colder vane versions.

## Versioning the Entire Kernel (Specification)

For an Arvo kernel K:

* K's version **SHALL** be characterised by its Zuse version and a number in
  the interval \[0, 1\) (the "fractional temperature"), typically rendered as:

  ```
  arvo-v<zuse version>.<fractional temperature>
  ```

  the fractional temperature **MAY** be 0 only if Zuse's version is 0.

* At Zuse version 0 and fractional temperature 0, new versions of K **MUST
  NOT** be released.

* New releases of K **MUST** be assigned a new version, and this version
  **MUST** be strictly less than the previous one.

* When a new release of K includes new versions of any component supported be
  Zuse, but not a new version of Zuse proper, its fractional temperature
  **MUST** be less than the previous version.

  Given constant Zuse versions, fractional temperatures corresponding to new
  releases **SHOULD** decrease according to the following schedule:

  ```
  9, 8, 7, .., 1, 01, 001, 0001, ..
  ```

* When a new release of K includes a new version of Zuse, the fractional
  temperature of **SHOULD** be reset to 9.

In addition, the following stipulation is reserved for future use:

* New versions of K **MAY** be indexed by kernel components other than Zuse.
  However, this component **MUST** either be colder than Zuse, or be at version
  0 conjointly with Zuse.

## Whole-Kernel Release Candidates

The presence of any release candidate component makes the entire resulting
kernel a release candidate.  From the kernel state in [Example 7](#example-7),
consider shipping a release candidate of Jael at 698K.rc1, for example; the new
kernel state is:

```
* Nock 4K
  * Hoon 141K
    * Arvo 225K
      * Zuse 309K
        * Ames 399K
        * Behn 310K
        * Clay 550K
        * Dill 375K
        * Eyre 900K
        * Ford 800K
        * Gall 1200K
        * Iris 600K
        * Jael 698K.rc1   <- 698K release candidate
```

Since the previous kernel release was `arvo-v309.8`, this kernel could be
released as `arvo-v309.7.rc1`.

[froz]: https://urbit.org/blog/toward-a-frozen-operating-system/
[rfcm]: https://www.ietf.org/rfc/rfc2119.txt

