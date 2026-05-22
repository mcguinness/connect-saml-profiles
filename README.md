<!-- regenerate: on (set to off if you edit this file) -->

# OpenID Connect / SAML 2.0 Interoperability Profiles

This repository is the working area for three complementary individual
Internet-Drafts that define how OpenID Connect deployments coexist with
SAML 2.0 federation. The profiles cover three migration surfaces: moving
SAML SPs onto OpenID Connect, letting an OpenID Provider continue serving
SAML SPs that cannot be moved, and standing up an OpenID Provider while
a SAML IdP remains the upstream authentication authority.

Deployments can use any profile alone, or combine them so each SAML SP
can be migrated on its own timeline while authentication authority and
subject continuity remain consistent across the protocol stacks.

## OpenID Connect Migration Profile for SAML 2.0 Service Providers

**Direction:** SAML 2.0 SP → OpenID Connect OP

This profile lets an existing SAML 2.0 Service Provider adopt OpenID
Connect and OAuth 2.0 without breaking its existing SAML federation
trust relationship. An OpenID Provider accepts a SAML assertion via
OAuth 2.0 Token Exchange or via a SAML assertion introspection
extension and returns OpenID Connect ID Tokens, OAuth 2.0 access and
refresh tokens, or normalized JSON claims.

The profile preserves:

* the subject identifier the SAML SP was already using (pairwise or
  public, per the existing deployment);
* the claim release policy the SP relies on;
* the authentication context and assurance signaling carried by the
  SAML assertion; and
* the operational trust between the SAML IdP and SP.

It addresses scenarios such as: an SP holding a SAML assertion needing
an OAuth access token to call APIs; an application wanting normalized
identity claims without implementing SAML XML parsing; and a gradual,
non-disruptive migration of an SP onto OpenID Connect without forcing
end users to reauthenticate or administrators to relink existing
accounts.

* [Editor's Copy](https://mcguinness.github.io/draft-connect-saml-migration/#go.draft-connect-saml-migration-profile.html)

## OpenID Connect Bridge Profile for SAML 2.0 Service Providers

**Direction:** OpenID Connect OP → SAML 2.0 SP

This profile lets an OpenID Provider continue to serve existing SAML
2.0 Service Providers that cannot be migrated, by introducing a Bridge
component that acts as an OpenID Connect Relying Party toward the OP
and as a SAML IdP facade toward the SP. The Bridge authenticates the
end-user via the OP and constructs a signed SAML assertion for delivery
to the SP, preserving the SP's existing trust configuration.

The profile preserves:

* SP-specific subject identifiers (deterministically derived from the
  OIDC subject scoped to the SAML SP entity);
* per-SP attribute release policies, scoped at the Bridge;
* SAML NameID formats and qualifier semantics the SP expects; and
* the SP's existing SAML metadata and ACS endpoints, unchanged.

It addresses scenarios such as: consolidating onto a single OpenID
Provider while continuing to serve unmodified SAML SPs; supporting
SP-initiated SAML Web SSO where the Bridge serves as the SAML IdP
endpoint; and a Token Exchange variant in which an OIDC ID Token is
exchanged for a SAML assertion without a browser-based flow.

* [Editor's Copy](https://mcguinness.github.io/draft-connect-saml-migration/#go.draft-connect-saml-bridge-profile.html)

## OpenID Connect Provider Profile for SAML 2.0 Identity Providers

**Direction:** SAML 2.0 IdP → OpenID Connect OP

This profile defines an OpenID Provider that delegates end-user
authentication to an upstream SAML 2.0 Identity Provider. The OP exposes
a standard OpenID Connect interface to relying parties while acting as a
SAML Service Provider toward the upstream IdP, translating SAML
authentication results into ID Tokens, access tokens, and normalized
claims.

The profile preserves:

* SAML subject continuity when deriving OIDC `sub` values;
* SAML authentication context and session bounds as OIDC claims;
* SAML attribute release policy when producing OIDC claims; and
* the distinction between OP metadata, SAML SP metadata, and upstream
  SAML IdP metadata.

It addresses scenarios such as: standing up an OpenID Provider before a
SAML IdP is retired; onboarding OIDC relying parties while SAML remains
authoritative; and operating a phased SAML deprecation where the OP can
later be paired with the Bridge and Migration profiles.

* [Editor's Copy](https://mcguinness.github.io/draft-connect-saml-migration/#go.draft-connect-saml-idp-profile.html)

## How the profiles fit together

The profiles are complementary, designed to be deployable together when
an organization needs to maintain SAML and OpenID Connect deployments on
different migration timelines:

```text
                 +-----------------+
                 | SAML 2.0 IdP    |
                 +-----------------+
                          |
                          | OpenID Connect Provider Profile
                          | for SAML 2.0 Identity Providers
                          v
                 +-----------------+
                 | OpenID Provider |
                 +-----------------+
                   |             |
                   |             | OpenID Connect Bridge Profile
                   |             | for SAML 2.0 Service Providers
                   |             v
                   |      +-----------------+
                   |      | SAML 2.0 SP     |
                   |      +-----------------+
                   |              |
                   |              | OpenID Connect Migration Profile
                   |              | for SAML 2.0 Service Providers
                   |              v
                   |      +-----------------+
                   +----> | OIDC RP         |
                          +-----------------+
```

| Profile | Direction | Audience |
| --- | --- | --- |
| Migration | SAML 2.0 SP → OpenID Connect RP | SPs that adopt OpenID Connect, preserving SAML federation trust |
| Bridge | OpenID Provider → SAML 2.0 SP | SPs that cannot migrate; OP serves them via a SAML IdP facade |
| IdP-backed OP | SAML 2.0 IdP → OpenID Connect OP | OIDC RPs served by an OP whose upstream authentication authority remains SAML |

The IdP-backed OP profile is useful early in a migration, while SAML
remains authoritative. The Migration profile moves individual SAML SPs
onto OpenID Connect. The Bridge profile lets the OP continue serving SPs
that remain on SAML after the OP becomes the primary authentication
authority.

## Contributing

See the
[guidelines for contributions](https://github.com/mcguinness/draft-connect-saml-migration/blob/main/CONTRIBUTING.md).

The contributing file also has tips on how to make contributions, if you
don't already know how to do that.

## Command Line Usage

Formatted text and HTML versions of the drafts can be built using `make`.

```sh
$ make
```

Command line usage requires that you have the necessary software installed.
See [the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).
