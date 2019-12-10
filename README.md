# Urbit Proposals

Urbit Proposals (UPs) are design documents proposing features, formalizing architectural decisions, protocols, and interfaces, and documenting processes for the Urbit project.

# Process

We suggest first having your UP reviewed as an RFC ("request for comment") on the urbit-dev mailing list <dev@urbit.org>.

1. Review [UP 0](000-purpose-and-guidelines.md) to understand the process.
2. Draft a proposal, loosely following [the template](proposal-template.md).
3. Join the [urbit-dev mailing list](https://groups.google.com/a/urbit.org/forum/#!forum/dev) and send an email to <dev@urbit.org> including your proposal in the body of the email or attached as a markdown file. Provide context and call out specific issues you'd like reviewed.
4. After discussion, fork this repository, add your proposal, and make a pull request.
5. Discussion can continue in the PR. Upon acceptance, as determined by a maintainer, your proposal will be assigned a `up` number, marked as a `draft`, and merged.

You are also welcome to skip discussion on the mailing list if the proposal is mature or has been discussed out of band.

If you have any questions, email <support@urbit.org>.

## Editors

- `~nidsut-tomdun` Jared Tobin <jared@tlon.io>
- `~hidrel-fabtel` Morgan Sutherland <morgan@tlon.io>

## Proposals

UP                                                | Title                                    | Type          | Status     | Lead Author
--------------------------------------------------|------------------------------------------|---------------|------------|-------------------
[0](000-purpose-and-guidelines.md)                | Purpose and Guidelines                   | Process       | RFC        | Morgan Sutherland
[1](001-ford-caching-redux.md)                    | Ford Caching Redux                       | Architecture  | Deprecated | Ted Blackman
[2](002-integrating-urbit-proposals-github.md)    | Integrating Urbit Proposals with Github  | Process       | Deprecated | Matthew Levan
[3](003-modular-front-end-facilities-eyre.md)     | Modular Front-end Facilities for Eyre    | Feature       | Deprecated | `~ponmep-litesem`
[4](004-ford-turboencabulator.md)                 | Ford Turboencabulator                    | Architecture  | Draft      | Ted Blackman
[5](005-asynchronous-multiprocessing.md)          | Asynchronous Multiprocessing             | Architecture  | RFC        | Ted Blackman
[8](008-urbit-hd-wallet.md)                       | Urbit HD Wallet                          | Protocol      | Stable     | Will Kim
[9](009-arvo-versioning.md)                       | Arvo Versioning                          | Protocol      | Draft      | `~nidsut-tomdun`
