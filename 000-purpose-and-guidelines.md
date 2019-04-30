+++
up = 0
title = "Purpose and Guidelines"
created = 2019-4-25
authors = [
  "~hidrel-fabtel Morgan Sutherland <morgan@tlon.io>"
]
editor = "~nidsut-tomdun Jared Tobin <jared@tlon.io>"
type = process 
status = draft
implementation = https://github.com/urbit/proposals
+++

# Abstract

This document describes a lightweight editorial process designed to formalize technical decisions, document processes, and provide a method for anyone to participate in the Urbit project by submitting a structured proposal. 

"UP" stands for Urbit Proposal. A UP is a design document proposing a feature, architectural decision, protocol, interface, or technical process.

These guidelines are based on [COSS](https://rfc.unprotocols.org/spec:2/COSS/), [EIP](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1.md), [BIP](https://github.com/bitcoin/bips/blob/master/bip-0001.mediawiki), and [PEP](https://github.com/python/peps/blob/master/pep-0001.txt).

# Motivation 

UPs are intended to be the primary mechanisms for proposing new functionality, documenting design decisions, and collecting community technical input. The UP format is designed to be specific enough to clearly define and track proposals through their lifecycle while remaining flexible enough to accommodate varying levels of rigor and different types of contributions. Because UPs are maintained as text files in a versioned repository, their revision histories serve as the historical record of the project.

For implementors, UPs are a convenient way to organize their ideas, get community buy-in, and track the progress of their implementations.

For managers and external parties, UPs are a way to get a comprehensive picture of the state of the project as a whole. 

# Specification

## Types

- **Feature** UPs describe the design of a new feature or component
- **Architecture** UPs describe a design or re-design of a major system component
- **Protocol** UPs describe the formal specification of a protocol or interface that needs to be rigorously specified, including cryptographic protocols 
- **Informational** UPs describe a design issue or provide general guidelines or information to the Urbit community, but does not propose or describe a feature, system component, protocol, or interface
- **Process** UPs describe or proposes a change to a process used in the Urbit project

## Format

UPs are markdown files named `xxx-short-title.md` including proper metadata and are encouraged to follow the structure described below and outlined in the [proposal template](proposal-template.md).

### Metadata

- A UP must begin with `title`, `authors`, `created`, and `status` metadata. Try to keep the title succinct. Multiple authors are welcome in an array, and authors may use their name, ship-name, or email.
- If your UP requires (builds on top of) or replaces (supersedes) another UP, reference it by number.
- If there's a reference implementation (code, working proof-of-concept) included with the UP, please include a link.
- After submission, your UP will be given a `type`, `up` number, `status`, and a link to where it can be discussed.

```
+++
up = <number> 
type = <type> 
status = <status> 
title = "<title>"
created = <y-m-d>
modified = <y-m-d>
authors = [
  "~ship-name First Last <example@example.com>"
  "~ship-name First Last <example@example.com>"
]
editor = "<editor-assigned-by-editors>"
requires = [
  "UP <number>"
  "UP <number>"
] 
superseded-by = "UP <number>"
supersedes = "UP <number>" 
implementation = <link-to-implementation>
comments = <link-to-comments>
+++
```

### Sections

UPs are encouraged to include an abstract, motivation, and specification section, however the following sections are recommendations.

#### Abstract

The abstract is a short (~200 word) informal description of the issue being addressed.

#### Motivation

The motivation is a possibly long, informal description of the issue being addressed. The motivation is critical. It should clearly explain the problem that the UP solves. 

#### Rationale

The rationale contextualizes the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.

#### Specification

The specification is the core of the proposal and should describe the what's being proposed in as much detail as needed for implementation. 

#### Implementation

The implementation describes how the proposal will be implemented.

## Workflow

UPs move through a series of stages from idea to implementation. The author, with the help of the editors, is responsible for stewarding it through these stages.

### RFC

The UP process begins with a new idea for Urbit, called an RFC ("request for comment"). Each RFC must have a champion ("author") - someone who writes it, following the [UP format](#format), shepherds the discussions in the appropriate forums, and attempts to build community consensus around the idea. The editor should first attempt to ascertain whether the idea is suitable for a UP. Posting to the urbit-dev mailing list <dev@urbit.org)> is the best way to go about this, however a well-developed proposal may also be submitted directly as a pull-request to the [proposals repository](https://github.com/urbit/proposals). RFCs may take the form of an email explaining the idea or an attached markdown file following the [proposal format](#format) as layed out in the [proposal template](proposal-template.md).

An RFC should contain a single key proposal or new idea. The more focused, the better. If in doubt, start with informal discussion. Bug fixes and small patches don't need UPs; they should be filed as issues or pull-requests directly in their respective project repositories. 

### Draft

When the author is certain that their RFC is good candidate for a UP, they are encouraged to submit a pull-request to the [proposals repository](https://github.com/urbit/proposals), adding a markdown file with appropriate TOML front-matter, named "XXX-short-title.md".  

Once added, an editor will:

1. Review the draft
2. Request changes if necessary
3. Add an `editor` 
4. Add a `up` number
5. Update the `status` to "Draft"
6. Update the `comments` URI.

The author may encourage ongoing discussion in the PR comments or at the `comments` URI. UPs should remain as drafts until implemented, deferred, or rejected.

### Stable 

When a UP has been implemented, a PR may be submitted to change the `status` to "stable." This indicates that the UP describes functionality or a process that is currently in-use.

### Deprecated

A UP that describes functionality or a process that has been removed, fallen out of use, or superseded by another UP may be marked "deprecated." If superseded, the new UP should be referenced in the metadata.

### Deferred

A draft UP that has been proposed, but delayed, may be marked "deferred." 

### Rejected

A draft UP that isn't deemed acceptable may be marked "rejected." 

### Withdrawn

A draft UP that is withdrawn by the author may be marked "withdrawn."

# Transferring Ownership

If it becomes necessary to assign a UP a new champion, a new author can be added to the `authors` metadata. If you are interested in assuming ownership of a UP, send a message asking to take it over, addressed to both the original author and the UP editor. If the original author doesn't respond to email in a timely manner, the UP editor will make a unilateral decision.

# Editor Responsibilities

For each new UP, the editors will do the following:

- Read the UP to check if it is ready: sound and complete
- Check that the `title` accurately describes the content and is appropriately terse
- Edit for language (spelling, grammar, sentence structure), markup, and code style
- If revisions are required, either propose changes or make specific requests 

Once the UP is in good shape, the editors will:

- Officially designate an editor and add their name to the metadata
- Assign a UP number (almost always just the next available number)
- Update the `status` to "Draft"
- List the UP in README.md
- Merge the pull request when the author is ready (allowing some time for further peer review)
- Advise the author on next steps, if necessary
