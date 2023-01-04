- Feature Name: cargo_crates_sigstore_integration
- Start Date: 2023-01-03
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This feature enables cargo users to sign crates, and verify signatures of their dependencies, while crates.io will gain the ability to store signatures for published crates.

# Motivation
[motivation]: #motivation

Software supply chain security is an increasingly important piece for any company building and maintaining open source software. A lot of applications depend on third party dependencies without knowing who wrote the software, which can open the door to supply chain attacks. 

At present, crates.io provides the following information about published crates:

* checksum of crate contents
* crate author/ownership

The verification of the above information relies on a third party (crates.io) not being compromised, which unfortunately happened for RubyGems and NPM. 

Signing and verifying crates puts an extra burden on crate maintainers, who would have to manage their own keys and manually verify crate signatures.

This is where a project like Sigstore comes in. It provides a key management service named Fulcio, which integrates with OpenID Connect services to identify the author and generate 
temporary key-pair derived from a trust root. The Fulcio service can either be run by crates.io or a separate organization, such as the public Fulcio service. Sigstore also provides Rekor, a 
tamper-free transparency log, which is used to record the signing key. These records can later be audited.

The proposal is to improve the current situation, by supporting these workflows for users of `cargo` and `crates.io` using Sigstore:

* Signing crates when publishing
* Verifying crates and their dependencies when building


This will provide a way to verify authorship independently of crates.io, but lets the crate maintainers and users to pick their poison: manage their own Sigstore instance for full control, or rely on other instances to simplify their workflow but at a greater risk.

Another benefit of this work, not specific to Rust: Package management for programming languages such as Java (Maven), JavaScript (NPM) or Python (Pip) are on their way to support Sigstore. Having the same integration for Rust, means that a company can use the same tooling for signature verification and auditing, and don't have to make the same mistakes when implementing this.

<!-- Why are we doing this? What use cases does it support? What is the expected outcome? -->

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Signing

To sign a package when publishing, first configure your signing settings:

```
[signing]
fulcio = "https://fulcio.sigstore.dev"
rekor = "https://rekor.sigstore.dev"
issuer = "https://oauth2.sigstore.dev/auth"
ConnectorID = "https://github.com/login/oauth"
```

Next, publish your crate using the `sign` flag:

```
cargo publish --sign
```

This will perform the following steps:

* Open your browser window to GitHub OAuth to verify your identity and retrieve a token
* Pass the token to Fulcio to generate a key/cert for signing
* Sign the .crate and publish to crates.io
* Publish signing cert to Rekor


## Verification

To verify a crate during install:

```
cargo install <package> --verify
```

When building:

```
cargo build --verify
```

These will verify the signature of all crates that contain signatures on crates.io. 

TODO:
- Describing the data flow

<!--
Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.
- Discuss how this impacts the ability to read, understand, and maintain Rust code. Code is read and modified far more often than written; will the proposed feature make code easier to maintain?

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms. -->
-->

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- If this is a language proposal, could this be done in a library or macro instead? Does the proposed change make Rust code easier or harder to read, understand, and maintain?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

One possible evolution of this work is for the crates.io team to use the The Update Framework (TUF) to manage trust roots for a Fulcio instance, which would make it independent of the public Sigstore instances. 

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
