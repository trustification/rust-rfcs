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
* crate author/ownership via GitHub

The verification of the above information relies on a third party (crates.io) not being compromised. In the event that crates.io is compromised, crates pulled from crates.io cannot be verified. Integrating crates.io and cargo with Sigstore reduces the severance of such a compromise, as individual crates can still be verified, and the presence of an immutable log helps auditing and evaluating the impact of an attack.

However, signing and verifying crates have traditionally been an extra burden on crate maintainers, who would have to manage their own keys and manually verify crate signatures.

Sigstore is an open source project run by the Open Source Security Foundation (OpenSSF), and provides a solution to the above problems. It provides a key management service, Fulcio, which integrates with OpenID Connect services to identify the author and generate a key used for signing. Sigstore also provides a tamper-free transparency log, Rekor, which is used to record the signing certificate along with other information extracted from the OpenID token. These records can later be audited.

End users can use additional projects like [in-toto](https://in-toto.io/) to build more advanced verification and attestation of software.

The proposal is to improve on the current situation, by supporting these workflows for users of `cargo` and `crates.io`:

* Automatic signing of crates when publishing by crate publishers
* Automatic verification of crates and their dependencies by crate consumers

The Sigstore project provides publicly available instance of both Fulcio and Rekor, which avoid overloading crates.io maintainers with setting up additional infrastructure. 

Since crates.io already uses GitHub authenticating users, adding automatic signing based on a GitHub ID is not a significant change for users.

The expected outcome of this work is for cargo to automatically sign crates when published, unless opted out.

## What about TUF (The Update Framework)? 

 TUF is a set of defined attacks and threat models specific to software distribution systems, and a cleverly designed set of protocols to protect against them. You can use TUF to manage the root keys of a Sigstore instance, just like the [public Sigstore instance](https://blog.sigstore.dev/a-new-kind-of-trust-root-f11eeeed92ef), but also use Sigstore to manage additional TUF roots.
 
 [This article](https://dlorenc.medium.com/using-the-update-framework-in-sigstore-dc393cfe6b52) does a good job of showing the options.
 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Crate signatures and verifications is a way to tie authenticity and integrity of a crate based on public key cryptography. The impliciation of this feature is that, as an end user, you can be sure that the crates that you download from crates.io have not been tampered with, and that they are signed by the correct owner of the package. 

The process of `signing` involves generating a signature for a crate, using a private key, and attaching that signature when publishing a crate. The process of `verification` involves downloading a crate (using cargo) and using a _trusted_ public key to verify the authenticity (the signer) and integrity (the contents) of a crate.

To simplify key management for users, cargo supports using [Sigstore](https://sigstore.dev) as a way to generate keys based on your identity (GitHub) and as an immutable log that can be monitored and audited in the event of a compromise. By default, a publicly available instance ensures that you do not need to manage this infrastructure yourself. If you are running your own crate registry, Sigstore instance and identify provider, you can configure cargo to use that.

## Configuring

By default, cargo will use the publicly available Sigstore instances together with GitHub as the identity provider. To override (if using a different registry than crates.io), the `$HOME/.cargo/sigstore.toml` file can be configured with the location of Sigstore and identity services:

```
fulcio = "https://fulcio.sigstore.dev"
rekor = "https://rekor.sigstore.dev"
issuer = "https://oauth2.sigstore.dev/auth"
connector = "https://github.com/login/oauth"
```

## Signing

Signing a new crate happens automatically when publishing a crate:

```
cargo publish
```

Cargo will perform the necessary steps to sign your crate before publishing, and attach the relevant information to the crates.io request.

Upon receiving the new crate metadata, crates.io will verify that the signature belongs to the crate owner.

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
* Email of Owner - retrieved as part of identity/GitHub lookup with email scope
* Certificate - the public certificate that can be used to verify the crate

In the event that the crate is not being signed, these fields may be optional/null.

## Cargo publish flow

The cargo publish flow is summarized below:

![publish flow](https://raw.githubusercontent.com/lulf/rfc-resources/main/sigstore/cargo_sigstore-publish.drawio.png)

The following changes must be made to cargo when publishing a crate:

1. Authenticate the publisher using OIDC to retrieve a token
1. Generate the signing key and certificate request
1. Generate a certificate signing request and pass with token to Sigstore Fulcio. 
1. Sign the generated .crate file using the private key
1. Attach signature, e-mail and certificate to crates.io publish request

Most of the above is already implemented in [sigstore-rs](https://github.com/sigstore/sigstore-rs).

### crates.io flow

The crates.io sub-flow on publish is described below:

![crates.io flow](https://raw.githubusercontent.com/lulf/rfc-resources/main/sigstore/cargo_sigstore-crates.io.drawio.png)

The following components need changing for crates.io:

1. Additional columns must be added to the PostgreSQL database
1. HTTP API must handle a new version of NewCratePublish request with the signature data
1. The HTTP handler talks to Rekor and verifies that the signature is valid and that owner matches certificate

## Cargo verify flow

The cargo flow during verification is shown below:

![verify flow](https://raw.githubusercontent.com/lulf/rfc-resources/main/sigstore/cargo_sigstore-verify.drawio.png)

The following changes must be made to cargo when building/verifying a crate:

1. Retrieve the crate contents, signature and certificate.
1. Verify that the crate contents is signed by the signature.
1. Verify that the certificate and signature is present in the transparency log.

## Note on cargo dependencies

The interaction with Sigstore services is implemented in the `sigstore-rs` crate, which has the following sub-dependencies:

* tough: used to fetch fulcio public key and rekor certificates from Sigstore's TUF repository. 
* openid-connect: this is used to generate keyless signatures via Rust
* oauth2: this is a transitive dependency of openid-connect

These crate use `reqwest` and some async Rust. To handle this, we can either:

* Add the necessary dependencies to cargo with the goal of replacing the curl with reqwest to avoid multiple HTTP clients in use
* Use the lower level APIs in sigstore-rs and implement the HTTP interaction using curl which is used by cargo today. 

# Drawbacks
[drawbacks]: #drawbacks

Cargo may be slower if signing and verification is enabled by default. A possible solution could be to make it opt-in, but the drawback is that fewer crates will be signed.

Additional metadata will be stored for each crate, which will increase storage requirements for the index.

There are other approaches to key management out there, such as GPG. However, these typically have a higher effort requiring keys to be managed by a process like TUF (The Update Framework). Sigstore is meant to capture some of these common patterns of managing keys and storing information for auditing, which lowers the barrier for signing crates.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Sigstore is in the process of being supported by [NPM](https://thenewstack.io/npm-to-adopt-sigstore-for-software-supply-chain-security/), [Maven](https://blog.sonatype.com/maven-central-and-sigstore) and [Python](https://www.python.org/download/sigstore/), and it has a large community.

The impact of not doing this leaves crate owners vulnerable to a crates.io compromise. Organizations that need to have way to verify crate integrity will be left to build their own solution and private package registries.

One specific alternative is to adopt [TUF](https://theupdateframework.io/) without Sigstore, but managing a full TUF root can be a lot for an already busy crates.io team. RFCs such as [this](https://github.com/withoutboats/rfcs/pull/7) describes using TUF for signing the crates.io index, and later signing crates themselves. Integration with Sigstore does not necessarily rule out the use of TUF for managing a signed crates.io index later, and there is ongoing work on making TUF and Sigstore play nicely together.

# Prior art
[prior-art]: #prior-art

The topic of signing crates have been brought up several times in the past:

* https://github.com/rust-lang/crates.io/issues/75 - this issue raises the initial concern
* https://github.com/sigstore/community/issues/25 - contains an attempt at sigstore support, but focusing on rustup because cargo maintainers being overloaded
* https://github.com/rust-lang/cargo/issues/4768 - an attempt to provide a signed crates.io index
* https://github.com/withoutboats/rfcs/pull/7 - a modification of the above issue using TUF to signing index commits

[NPM](https://thenewstack.io/npm-to-adopt-sigstore-for-software-supply-chain-security/), [Maven](https://blog.sonatype.com/maven-central-and-sigstore) and PyPI are all in the process of adopting Sigstore. It's not uncommon for organizations to use multiple programming languages, so integrating with Sigstore means that they can reuse the same infrastructure for verifying and auditing their software.

## Articles and papers

The [Sigstore Blog](https://blog.sigstore.dev/) contains a lot of articles related Sigstore. In particular, [this post](https://blog.sigstore.dev/signatus-ergo-securus-who-can-sign-what-with-tuf-and-sigstore-ea4d3d84b8b6) covers the problems faced by similar package systems like PyPI.

The GitHub tema has published a few [blog posts](https://github.blog/2022-10-25-why-were-excited-about-the-sigstore-general-availability/) on Sigstore. 

A [paper](https://dl.acm.org/doi/abs/10.1145/3548606.3560596) about Sigstore on ACM.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* Should crates.io use the signing information to enforce that the signature of crate-to-be-published matches the publisher?
* Opt-in or opt-out of signing and/or verification? Making it opt-in could increase the percentage of signed crates.
* How does it relate to later work TUF for signing crates.io index?

# Future possibilities
[future-possibilities]: #future-possibilities

One possible evolution of this work is for crates.io to either use the The Update Framework (TUF) to manage trust roots for their own Sigstore instance, which would make it independent of the public Sigstore instances. Alternatively, use the public Sigstore instance to manage root keys for TUF.

Extending other cargo commands to make use of the information stored in the transparency log when listing dependencies and other places where it makes sense.

Integrating with [in-toto.io](https://in-toto.io) to provide more advanced attestation of artifacts.

