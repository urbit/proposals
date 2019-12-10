+++
up = 9
type = "protocol"
title = "Arvo Versioning"
created = 2019-12-10
authors = [
  "~nidsut-tomdun Jared Tobin <jared@tlon.io>"
]
status = draft
+++

## Abstract

This document describes kelvin versioning, as well as how it is used in the
Arvo kernel.

## Specification

Kelvin versioning was presented in the blog post ["Toward a Frozen Operating
System"][1].  It is designed to be a versioning system for components that
should eventually freeze (i.e., such that it becomes impossible to upgrade them
any further).

Its definition, excerpted from that post, is as follows:

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

The recursive part of the scheme is often referred to as "telescopic kelvins,"
but "kelvin versioning" should be interpreted as subsuming this characteristic.

As an example, consider components A, B, and C such that A hosts B and B hosts
C.  Let A have version 1, B have version 2, and C have version 10.

Let a kelvin-decreasing patch be made to component C.  Then the version
(temperature) of C must decrease (cool), e.g. from 10 to 9.

Note that B can't be cooled further, at present, as the version of B must be
strictly larger than that of the component that hosts it, A.  A must decrement
before B can decrement further.

Now let a kelvin-decreasing patch be made to A.  A's version decreases to zero,
and thus A becomes frozen forever.  As a result, B's version must also
decrement (from 2 to 1) and C's version must decrement in turn (e.g. from 9 to
8).

Finally, let a kelvin-decreasing patch be made to B.  B's version decreases to
zero, freezing it forever, and C's version decreases in turn (from 8 to 7).

## Kelvin Versioning in Arvo

Not all components of Urbit should eventually freeze, so there is a certain
layer above which kelvin versioning is not a desirable versioning scheme.
Consider userspace applications, etc. -- these are the so-called "fronds"
described in ["Towards A Frozen Operating System"][1].

Similarly, Nock runtimes like [Vere][2] and [Jaque][3] are Unix applications,
have Unix dependencies, and are better suited to adopt conventional Unix
version schemes (Vere, for example, employs a major.minor.patch scheme).

The Arvo kernel, on the other hand, is intended to freeze.  It follows a kelvin
versioning scheme consisting of the following components:

  Nock -> Hoon -> Arvo, %zuse

That is: Nock supports Hoon, which supports Arvo proper and %zuse.  Each
component in the stack is cooler -- i.e., has a lower kelvin -- than the one
above it, or they are both at absolute zero.  Note that both Arvo proper and
%zuse are considered to be a single component, versioned according to a single
kelvin (the %zuse kelvin).

%zuse contains the definitions of core abstractions used throughout the kernel
(i.e., in the vanes), in particular -- these are the Arvo moves, $task and
$gift.  It also encompasses the Arvo structural interface used by Nock
runtimes.  As such, the %zuse kelvin effectively serves as a version for the
whole-kernel API.

Components running above the kernel are free at to adopt any versioning scheme
they find appropriate.

