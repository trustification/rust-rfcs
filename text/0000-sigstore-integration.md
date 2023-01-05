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

The verification of the above information relies on a third party (crates.io) not being compromised. In the event that crates.io is compromised, crates pulled from crates.io cannot be verified. Adding signatures in crates.io metadata reduces the severance of such a compromise, as individual crates can still be verified.

Signing and verifying crates puts an extra burden on crate maintainers, who would have to manage their own keys and manually verify crate signatures.

This is where a project like Sigstore comes in. It provides a key management service named Fulcio, which integrates with OpenID Connect services to identify the author and generate temporary key-pair derived from a trust root. The Fulcio service can either be run by crates.io or a separate organization, such as the public Fulcio service. Sigstore also provides Rekor, a tamper-free transparency log, which is used to record the signing key. These records can later be audited.

The proposal is to improve the current situation, by supporting these workflows for users of `cargo` and `crates.io`:

* Signing crates when publishing
* Verifying crates and their dependencies when building

This will provide a way to verify authorship of individual crates not relying on crates.io, and lets the crate maintainers and users choose their source of trust. Some users may manage their own Sigstore instance for full control, or rely on other instances to keep things simple but at the same risk level as today.

Another benefit of this work, not specific to Rust: Package management for programming languages such as Java (Maven), JavaScript (NPM) or Python (Pip) are on their way to support Sigstore. Having the same integration for Rust, means that a company can use the same tooling for key management and audit logs across multiple package managers, and don't have to make the same mistakes when implementing this.

<!-- Why are we doing this? What use cases does it support? What is the expected outcome? -->

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

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

As of YYYY/MM/DD, crates.io and cargo supports signing and verifying crates. The process of `signing` involves generating a signature for a crate, using a private key, and attaching that signature when publishing a crate. The process of `verification` involves downloading a crate (using cargo) and using a _trusted_ public key to verify the authenticity (the signer) and integrity (the contents) of a crate.

Cargo and crates.io supports both of these flows. As a crate publisher, by signing crates you provide additional security for consumers of your crate. As a crate consumer, by verifying crates you can reduce the impact of a compromise of crates.io.

To simplify key management for users, cargo supports using [Sigstore](https://sigstore.dev) as a way to generate keys based on OIDC identities (GitHub, Google etc.) and as an immutable log that can be audited and monitored.

## Configuring

To configure cargo for using Sigstore, edit the configuration file in `$HOME/.cargo/sigstore.toml`:

```
fulcio = "https://fulcio.sigstore.dev"
rekor = "https://rekor.sigstore.dev"
issuer = "https://oauth2.sigstore.dev/auth"
connector = "https://github.com/login/oauth"
```

## Signing

To sign and publish your crate, pass the `--signer` flag to `publish`:

```
cargo publish --signer sigstore
```

This will perform the following steps:

1. Authenticate you using the preferred flow (`connector` setting). In this case, it will open your browser window to GitHub OAuth.
1. Pass the access token to Sigstore (which authorizes the user) in order to generate a key and cert for signing the crate.
1. Sign the .crate using the temporary key and cert.
1. Publish the crate to crates.io with the signature attached.
1. Publish the signing cert to the Sigstore transparency log.

The `--sign` flag may support alternative ways to sign, like manually providing the key or using `gnupg`. Cargo could have a default signer set. In the event of not using sigstore, some of the above steps would be skipped.

## Verification

To verify signatures for a crate using the sigstore signer:

```
cargo verify-signature [<crate>] [--check-dependencies] --signer sigstore
```

This will perform the following steps and checks:

1. Connect to crates.io to retrieve the crate and its signature.
1. Verify that the crate is signed by the signature.
1. Check that the signer is trusted.
1. Check that the signature is present in the Sigstore transparency log.

The command will fail if the crate and optionally its dependencies does not have verifiable signatures. In the same way as for publish, alternative verification methods could be supported.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

<!-- Why should we *not* do this? -->

Introducing crate signatures may cause headaches for crate maintainers if enforced. Therefore, the signing and verification should be an optional feature that doesn't impact todays workflow.

The usefulness of signatures are also dependent on package maintainers using them. If only a small fraction of crates end up using signatures, maybe it will have minimal impact. On the other hand, having the capability at least provides a mechanism that can easily be added to existing crates.

Supporting different systems for signing and verification has a drawback that it could be harder to standardize and use the information on crates.io itself vs. having only one provider like Sigstore.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!--
- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
-->

This design takes one step in the direction of signing and verification of crates. Some maintainers may not want to use Sigstore, in which case this would allow alternative key management systems. However, Sigstore is in the process of being supported by [NPM](https://thenewstack.io/npm-to-adopt-sigstore-for-software-supply-chain-security/), [Maven](https://blog.sonatype.com/maven-central-and-sigstore) and [Python](https://www.python.org/download/sigstore/), and it has a lot of maintainers behind it, so it makes sense to support this provider first.

The impact of not doing this leaves crate owners vulnerable to a crates.io compromise. Organizations that need to have way to verify crate integrity will be left to build their own solution and private package registries.

# Prior art
[prior-art]: #prior-art

There have been discussions on this topic in the past:

* https://github.com/rust-lang/crates.io/issues/75 - this issue raises the initial concern
* https://github.com/sigstore/community/issues/25 - contains an attempt at sigstore support, but focusing on rustup because cargo maintainers being overloaded
* https://github.com/rust-lang/cargo/issues/4768 - G

[NPM](https://thenewstack.io/npm-to-adopt-sigstore-for-software-supply-chain-security/), [Maven](https://blog.sonatype.com/maven-central-and-sigstore) and Python are adopting. For organizations, integrating with Sigstore means that they can reuse the same infrastructure for verifying and auditing their software across programming languages.

## Articles and papers

The [Sigstore Blog](https://blog.sigstore.dev/) contains a lot of articles on the usage of Sigstore. The GitHub tema has published a few [blog posts](https://github.blog/2022-10-25-why-were-excited-about-the-sigstore-general-availability/) on Sigstore. 

A [paper](https://dl.acm.org/doi/abs/10.1145/3548606.3560596) about Sigstore on ACM.

<!--
- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features. 
-->

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should crates.io use the signing information to enforce that the signature of crate-to-be-published matches the publisher?


Currently considered out of scope:

* 
<!--
- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
-->

# Future possibilities
[future-possibilities]: #future-possibilities

One possible evolution of this work is for the crates.io team to use the The Update Framework (TUF) to manage trust roots for a Fulcio instance, which would make it independent of the public Sigstore instances. 

<!--
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
-->
