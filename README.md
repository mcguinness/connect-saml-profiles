<!-- regenerate: on (set to off if you edit this file) -->

# OpenID Connect / SAML 2.0 Interoperability Profiles

This repository is the working area for two complementary individual
Internet-Drafts that define how OpenID Connect deployments coexist with
existing SAML 2.0 Service Providers. The two profiles are inverse
companions: one moves SAML SPs onto OpenID Connect, the other lets an
OpenID Provider continue serving SAML SPs that cannot be moved.

Deployments can use either profile alone, or both together with a
common bridge entity, allowing each SAML SP to be migrated on its own
timeline while a single OpenID Provider remains the authentication
authority.

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

## How the two profiles fit together

The two profiles are inverse companions, designed to be deployable
together when an organization needs to maintain SAML SPs on different
migration timelines:

| Profile | Direction | Audience |
| --- | --- | --- |
| Migration | SAML 2.0 SP → OpenID Connect RP | SPs that adopt OpenID Connect, preserving SAML federation trust |
| Bridge | OpenID Provider → SAML 2.0 SP | SPs that cannot migrate; OP serves them via a SAML IdP facade |

Both profiles share the same OpenID Provider as the authentication
authority. A single deployment can use the Migration profile for SPs
that are moving to OpenID Connect and the Bridge profile for SPs that
remain on SAML, with both sets of SPs anchored to one OP and consistent
subject continuity across the two protocol stacks.

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
