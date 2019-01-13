# Urbit Proposals

### What is a UP?

An Urbit Proposal (UP) is a design document outlining the proposal of a new Urbit feature or general improvement to the Urbit system or our related processes.

UPs are udon files posted to https://github.com/urbit/proposals.

### What belongs in a successful UP?

A good UP should have the following parts:

- Metadata
- Overview
- Specification
- Rationale
- Integration Plan

#### Metadata

UP _Metadata_ is just a markdown header with some structured data about the proposal in a code block.

Metadata for a new UP could look like this:

```
  Title: An Urbit-Centric Blockchain
  Author: ~ravmel-ropdyl Galen Wolfe-Pauly <galen@tlon.io>
  Created: ~2017.12.22
```

A newly submitted UP should include `Title`, `Author` and `Created` metadata. Try to keep your title succinct. Author can just be your urbit name (like `~tonlur-sarret`). You're free to add your full name, pseudonym or email. Multiple authors are welcome.

If your UP Requires (builds on top of) or Replaces (supersedes) another UP, that should be referenced in the metadata by that UP's number.

If there's a reference implementation (code, working proof-of-concept) included with the UP (see the _Integration Plan_ section below), please include the link to that code too.

After submission your UP will be given a `Number` and `Status`.

So, a complete metadata section after edits and review might look like this:


````
## Metadata

```
  UP: 42
  Title: Bootstrapping Urbit from Ethereum
  Authors: ~ravmel-ropdyl Galen Wolfe-Pauly <galen@tlon.io>
           ~palfun-foslup Fang <fang@fang.io>
  Status: Active
  Created: ~2017.9.20
  Last Modified: ~2017.10.18
  Code: https://github.com/urbit/constitution
```

````

#### Overview

A UP should have a short overview (~200 words) summarizing the specification of the improvement proposal, its motivation/rationale at a high level, and a short mention of the integration plan.

This doesn't have to be particularly detailed. Details should be left for the following three sections.

#### Specification

This section should lay out the detailed technical specification for the improvement proposal to Urbit. Be as detailed as possible. A fully-developed specification is one that can easily be used as a guide for writing code.

#### Rationale

You may be tempted to assume that the rationale for your UP is self-evident. Pretend it isn’t and provide a plain-English description of why you think the change you’re proposing is needed.

In many cases you may be proposing a feature that doesn’t exist. Why do we need it? What new affordance does it provide? How does it fit the spirit of the system?

#### Integration Plan

Lastly, a UP should include an Integration Plan section that proposes a plan of action to implement the proposal should it pass review.

Ideally, you can point to an already existing Git branch that can be cleanly merged. If you’re proposing breaking changes, please detail how you intend to integrate them. If your proposed change would require a continuity breach, please note that as well with a proposed plan of action.

If your UP is simply a proposal for future work, outline how you think other contributors can productively integrate this proposal into the system.

### UP Workflow

We want the UP submission to be easy with a low barrier to entry for new contributors. Really, the UP workflow is pretty simple:

#### Start with an Idea for Urbit

The UP process begins with a new idea for Urbit. A single UP should contain a single key proposal or new idea. The more focused the UP, the better. If in doubt, start with informal discussion. Fora posts and Talk conversations are great places to get constructive feedback on an idea.

Small bug fixes and patches don't really need UPs; they should just be filed as issues and/or pull-requests on Urbit's Github.

One simple way to think about it could be: if you sent a PR for the change you were thinking of, do you think it would get merged without discussion? If so, just send it and don’t bother with a UP.

In cases where you think your PR would be merged but would require a lot of development work, this could be a good opportunity to write a UP. UPs are intended to be a productive way to engage with other potential contributors and coordinate effort on larger undertakings.

#### Submit a UP

TBD

#### UP review and progress

TBD

## Rationale

We often find ourselves organizing infrastructure projects among ourselves, but don’t have a process for communicating them to the outside world. In a way, UPs are just a post format for the core Urbit team to publicly communicate about what we’re building.

There are also a fair number of large-scale projects that we have thought through, but don’t have the resources to build. So, it seemed like a good idea to formalize the process for writing up an urbit proposal. This way, we can organize a record of what needs to be worked on and focus our efforts.

## Integration Plan

By posting this UP, UPs have been integrated. There are a number of features that could be built to support them, but we’ll leave that for future UPs.
