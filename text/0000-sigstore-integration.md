- Feature Name: cargo_crates_sigstore_integration
- Start Date: 2023-01-03
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This feature enables cargo users to sign crates, and verify signatures of their dependencies, while crates.io will gain the ability to store signatures for published crates.

TODO:
* Expand on flows showing figures how cargo and crates.io interacts with Sigstore
* Expand on future usage of the data
* Explore a few alternatives


# Motivation
[motivation]: #motivation

Software supply chain security is an increasingly important piece for any company building and maintaining open source software. A lot of applications depend on third party dependencies without knowing who wrote the software, which can open the door to supply chain attacks. 

At present, crates.io provides the following information about published crates:

* checksum of crate contents
* crate author/ownership via GitHub

The verification of the above information relies on a third party (crates.io) not being compromised. In the event that crates.io is compromised, crates pulled from crates.io cannot be verified. Integrating crates.io and cargo with Sigstore reduces the severance of such a compromise, as individual crates can still be verified, and the presence of an immutable log helps auditing and evaluating the impact of an attack.

However, signing and verifying crates have traditionally been an extra burden on crate maintainers, who would have to manage their own keys and manually verify crate signatures.

Sigstore is an open source project run by the Open Source Security Foundation (OpenSSF), and provides a solution to the above problems. It provides a key management service, Fulcio, which integrates with OpenID Connect services to identify the author and generate a key used for signing. Sigstore also provides a tamper-free transparency log, Rekor, which is used to record the signing certificate along with other information extracted from the OpenID token. These records can later be audited.

End users can use additional projects like [in-toto](https://in-toto.io/) to build more advanced verification and attestation of software.

The proposal is to improve the current situation, by supporting these workflows for users of `cargo` and `crates.io`:

* Automatic signing of crates when publishing by crate publishers
* Automatic verification of crates and their dependencies by crate consumers

The Sigstore project provides publicly available instance of both Fulcio and Rekor, which avoid overloading crates.io maintainers with setting up additional infrastructure. 

Since crates.io already uses GitHub authenticating users, adding automatic signing based on a GitHub ID is not a significant change for users.

The expected outcome of this work is for cargo to automatically sign crates when published, unless opted out.

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

To simplify key management for users, cargo supports using [Sigstore](https://sigstore.dev) as a way to generate keys based on identities (GitHub) and as an immutable log that can be monitored and audited in the event of a compromise.

## Configuring

By default, cargo will use the publicly available Sigstore instances together with GitHub as the identity provider. To override (if using a different registry than crates.io), the `$HOME/.cargo/sigstore.toml` file can be configured with the location of Sigstore and identity services:

```
fulcio = "https://fulcio.sigstore.dev"
rekor = "https://rekor.sigstore.dev"
issuer = "https://oauth2.sigstore.dev/auth"
connector = "https://github.com/login/oauth"
```

## Signing

Signing a new crate is automatically done on publish:

```
cargo publish
```

Cargo will perform the necessary steps to sign your crate before publishing, and attach the relevant information to the crates.io request.

Upon receiving the new crate metadata, crates.io will verify that the signature to belong to the crate owner. This prevents a compromised cargo version to publish invalid signatures.

Signing an existing crate can be done using `cargo sign`:

```
cargo sign [<crate which you own>]
```

This will retrieve the crate from crates.io, and perform the same steps as for publish to generate the signature, but will attach the signature and certificate to the existing crate.

## Verification

Crate signatures are automatically verified for any crate that is retrieved from `crates.io` that has signatures attached. 

To explicitly verify signatures for a crate, the `verify-signature` subcommand can be used:

```
cargo verify-signature [<crate>] [--skip-dependencies]
```

The commands will fail if the crate (and dependencies) do not have verifiable signatures. 

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The crates.io and cargo types for new crates, will need to be modified to include the following metadata:

* Signature - created based on the generated .crate file on publish
* Email - retrieved as part of identity/GitHub lookup
* Certificate - the public portion of the key that was used to sign the crate

In the event that the crate is not being signed, these fields may be optional/null.


## Cargo publish flow

The cargo publish flow is summarized below:

![publish flow](https://raw.githubusercontent.com/lulf/rfc-resources/main/sigstore/cargo_sigstore-publish.drawio.png)

The following changes must be made to cargo when publishing a crate:

1. Authenticate the publisher using Sigstore OIDC to retrieve a token
1. Generate the temporary key and certificate request
1. Generate a certificate signing request and pass with token to Sigstore Fulcio. 
1. Sign the generated .crate file using the private key
1. Attach signature, e-mail and certificate to crates.io publish request


### crates.io flow

The crates.io sub-flow on publish is described below:

![crates.io flow](https://raw.githubusercontent.com/lulf/rfc-resources/main/sigstore/cargo_sigstore-crates.io.drawio.png)

The following components need changing for crates.io:

1. Additional columns must be added to the PostgreSQL database
1. HTTP API must handle a new version of NewCratePublish request with the signature data
1. The HTTP handler verifies that the signer is allowed to sign the crate

## Cargo verify flow

The cargo flow during verification is shown below:

![verify flow](https://raw.githubusercontent.com/lulf/rfc-resources/main/sigstore/cargo_sigstore-verify.drawio.png)

The following changes must be made to cargo when buildling/verifying a crate:

1. Retrieve the crate contents, signature and certificate.
1. Verify that the crate contents is signed by the signature.
1. Verify that the certificate and signature is present in the transparency log.

## Cargo dependencies

The interaction with Sigstore services is implemented in the `sigstore-rs` crate, which has the following sub-dependencies:

* tough: used to fetch fulcio public key and rekor certificates from Sigstore's TUF repository. 
* openid-connect: this is used to generate keyless signatures via Rust
* oauth2: this is a transitive dependency of openid-connect

These crate use `reqwest` and some async Rust. To handle this, we can either:

* Add the necessary dependencies to cargo with the goal of replacing the curl with reqwest to avoid multiple HTTP clients in use
* Use the lower level APIs in sigstore-rs and implement the HTTP interaction using curl which is used by cargo today. 

<!--
This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.
 -->

# Drawbacks
[drawbacks]: #drawbacks

<!-- Why should we *not* do this? -->

Cargo may be slower if signing and verification is enabled by default. A possible solution could be to make it opt-in, but the drawback is that fewer crates will be signed.

Additional metadata will be stored for each crate, which will increase storage requirements for the index.

There are other approaches to key management out there, such as GPG. However, these typically have a higher effort, and supporting different systems for signing and verification could make it use the information on a compromise.

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
* https://github.com/rust-lang/cargo/issues/4768 - an attempt to provide a signed crates.io index
* https://github.com/withoutboats/rfcs/pull/7 - a modification of the above issue using TUF to signing index commits

[NPM](https://thenewstack.io/npm-to-adopt-sigstore-for-software-supply-chain-security/), [Maven](https://blog.sonatype.com/maven-central-and-sigstore) and PyPI are all in the process of adopting Sigstore. It's not uncommon for organizations to use multiple programming languages, so integrating with Sigstore means that they can reuse the same infrastructure for verifying and auditing their software.

## Articles and papers

The [Sigstore Blog](https://blog.sigstore.dev/) contains a lot of articles related Sigstore. In particular, [this post](https://blog.sigstore.dev/signatus-ergo-securus-who-can-sign-what-with-tuf-and-sigstore-ea4d3d84b8b6) covers the problems faced by similar package systems like PyPI.

The GitHub tema has published a few [blog posts](https://github.blog/2022-10-25-why-were-excited-about-the-sigstore-general-availability/) on Sigstore. 

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

* Should crates.io use the signing information to enforce that the signature of crate-to-be-published matches the publisher?
* Opt-in or opt-out of signing and/or verification

<!--
- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
-->

# Future possibilities
[future-possibilities]: #future-possibilities

One possible evolution of this work is for the crates.io team to use the The Update Framework (TUF) to manage trust roots for their own Fulcio instance, which would make it independent of the public Sigstore instances. 

Using sigstore in combination with TUF to provide a signed package index.

Extending other cargo commands to make use of the information stored in the transparency log when listing dependencies and other places where it makes sense.

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
