---
title: "OpenID Connect Bridge Profile for SAML 2.0 Service Providers"
abbrev: "OIDC-SAML SP Bridge"
category: info
docname: draft-connect-saml-bridge-profile-latest
submissiontype: independent
number:
date:
v: 3
# area: Security
workgroup: OpenID Connect Working Group

author:
 -
    fullname: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC2119:
  RFC4648:
  RFC6749:
  RFC7591:
  RFC7636:
  RFC8174:
  RFC8176:
  RFC8414:
  RFC8693:
  RFC9700:
  OIDC-CORE:
    title: "OpenID Connect Core 1.0 incorporating errata set 2"
    target: "https://openid.net/specs/openid-connect-core-1_0-18.html"
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M.B. Jones
      - ins: B. de Medeiros
      - ins: C. Mortimore
    date: false
  OIDC-DISCOVERY:
    title: "OpenID Connect Discovery 1.0 incorporating errata set 2"
    target: "https://openid.net/specs/openid-connect-discovery-1_0.html"
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M.B. Jones
      - ins: E. Jay
    date: false
  OIDC-REGISTRATION:
    title: "OpenID Connect Dynamic Client Registration 1.0 incorporating errata set 2"
    target: "https://openid.net/specs/openid-connect-registration-1_0.html"
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M.B. Jones
    date: false
  OIDC-ENTERPRISE-EXTENSIONS:
    title: "OpenID Connect Enterprise Extensions 1.0"
    target: "https://openid.net/specs/openid-connect-enterprise-extensions-1_0.html"
    author:
      - ins: D. Hardt
      - ins: K. McGuinness
    date: 2025-09
  SAML2-CORE:
    title: "Assertions and Protocols for the OASIS Security Assertion Markup Language (SAML) V2.0"
    target: "https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf"
    author:
      - org: OASIS
    date: 2005
  SAML2-METADATA:
    title: "Metadata for the OASIS Security Assertion Markup Language (SAML) V2.0"
    target: "https://docs.oasis-open.org/security/saml/v2.0/saml-metadata-2.0-os.pdf"
    author:
      - org: OASIS
    date: 2005
  SAML2-BINDINGS:
    title: "Bindings for the OASIS Security Assertion Markup Language (SAML) V2.0"
    target: "https://docs.oasis-open.org/security/saml/v2.0/saml-bindings-2.0-os.pdf"
    author:
      - org: OASIS
    date: 2005
  SAML2-PROFILES:
    title: "Profiles for the OASIS Security Assertion Markup Language (SAML) V2.0"
    target: "https://docs.oasis-open.org/security/saml/v2.0/saml-profiles-2.0-os.pdf"
    author:
      - org: OASIS
    date: 2005
  SAML2-SUBJ-ID:
    title: "SAML V2.0 Subject Identifier Attributes Profile Version 1.0"
    target: "https://docs.oasis-open.org/security/saml-subject-id-attr/v1.0/saml-subject-id-attr-v1.0.html"
    author:
      - org: OASIS
    date: 2019
  OIDC-FC-LOGOUT:
    title: "OpenID Connect Front-Channel Logout 1.0"
    target: "https://openid.net/specs/openid-connect-frontchannel-1_0.html"
    author:
      - ins: M.B. Jones
    date: false
  OIDC-BC-LOGOUT:
    title: "OpenID Connect Back-Channel Logout 1.0"
    target: "https://openid.net/specs/openid-connect-backchannel-1_0.html"
    author:
      - ins: M.B. Jones
      - ins: J. Bradley
    date: false
  SAML-OIDC-MIGRATION:
    title: "OpenID Connect Migration Profile for SAML 2.0 Service Providers"
    target: "https://mcguinness.github.io/connect-saml-profiles/#go.draft-connect-saml-migration-profile.html"
    author:
      - ins: K. McGuinness
    date: false

informative:
  RFC7009:
  RFC7522:
  RFC7662:
  I-D.ietf-oauth-identity-assertion-authz-grant:

...

--- abstract

This document defines an interoperability profile for deployments
that present a SAML 2.0 Identity Provider interface to existing
Service Providers (SPs) while using an OpenID Provider (OP) as the
underlying authentication authority. A Bridge component acts as an
OIDC Relying Party toward the OP and as a SAML IdP facade toward
the SP, translating OIDC authentication into a signed SAML
assertion.

Two operational patterns are defined. The browser-based flow lets a
SAML SP initiate authentication through the Bridge using SAML Web
SSO; the Bridge authenticates the end-user via the OP and delivers
a SAML assertion to the SP's ACS endpoint. A Token Exchange variant
lets a confidential client exchange an OIDC ID Token for a SAML
assertion without a browser-based SAML exchange.

This document is the inverse companion to {{SAML-OIDC-MIGRATION}}.

--- middle

# Introduction {#introduction}

Many deployments that have adopted OpenID Connect as their primary
authentication protocol still need to serve SAML 2.0 Service Providers that
cannot be immediately migrated. Operating a separate SAML IdP alongside the OP
duplicates authentication authority, splits the session model across two
protocol stacks, and leaves the OP's authentication decisions disconnected from
the SAML assertions the SPs receive.

This profile addresses cross-protocol interoperability and identity continuity
across the OIDC-to-SAML boundary. The scenarios below illustrate where
preserving the existing SAML trust relationship matters:

* **Continuing to serve legacy SAML SPs.** An organization has consolidated
  onto an OP but still operates SAML SPs that cannot be migrated. The Bridge
  exposes a SAML IdP endpoint that the SPs can continue to use without changes
  to their metadata, trust configuration, or ACS logic.

* **Single authentication authority across protocols.** Each SP-specific NameID
  is derived deterministically from the OP subject, so the user appears to the
  SP under the same identifier they had previously, with the OP performing the
  underlying authentication and reflecting that decision in the issued SAML
  assertion.

* **Programmatic SAML assertion issuance.** A back-end client holding an OIDC
  ID Token can present that token to the Bridge's token endpoint and receive a
  SAML assertion for a specific SP, without a browser-based SAML exchange. This
  allows scripted migration tooling, integration tests, and service-to-service
  flows to construct valid assertions from validated OIDC authentication
  events.

* **Non-disruptive consolidation.** SPs continue to receive assertions through
  their existing SAML federation interface while the deployment moves
  authentication to a single OP, preserving subject identifiers, attribute
  release policy, and assurance signaling.

This profile:

* defines a deployment model for a Bridge component that acts as an OIDC RP
  toward the OP and as a SAML IdP facade toward the SP;
* defines per-SP-Binding configuration that scopes subject computation and
  attribute release to each SP relationship;
* defines rules for translating OIDC subject identifiers into SAML NameIDs
  that preserve SP-specific subject continuity;
* defines rules for translating OIDC claims into SAML attributes and
  authentication context; and
* defines a Token Exchange variant in which an ID Token is exchanged for a
  SAML assertion without a browser-based flow.

This profile is the inverse companion to {{SAML-OIDC-MIGRATION}},
which defines how a SAML assertion can be exchanged for an OAuth 2.0
or OpenID Connect token. The two profiles share a common Bridge
entity and are designed to be deployed together when an organization
needs to maintain SAML SPs on different migration timelines.

## Overview {#overview}

This profile involves four actors:

* **SAML SP** -- the existing SAML 2.0 Service Provider, unmodified.
* **Bridge** -- the component this profile defines. The Bridge
  acts as a SAML IdP toward the SP and as an OIDC Relying Party
  toward the OP.
* **OP** -- the backing OpenID Provider that authenticates
  end-users.
* **Client** (Token Exchange mode only) -- a confidential OAuth
  client that has obtained an OIDC ID Token from the OP through
  its own registration and exchanges it for a SAML assertion.

The Bridge serves SAML SPs in two modes.

### Browser-Based Mode {#overview-browser}

~~~
   +--------+                              +----------+
   |        |---(A)-- SAML AuthnRequest -->|          |
   |        |                              |          |
   |  SAML  |   (Bridge runs an OIDC       |          |
   |   SP   |    Authorization Code flow   |  Bridge  |
   |        |    with the OP; end-user     |          |
   |        |    authenticates at the OP)  |          |
   |        |<--(B)-- SAML Response -------|          |
   +--------+         (signed assertion)   +----------+
~~~

(A) The SAML SP redirects the end-user's browser to the Bridge
    with a SAML `AuthnRequest`. The Bridge starts an OIDC
    Authorization Code flow at the backing OP and receives an ID
    Token after the end-user authenticates.

(B) The Bridge derives an SP-specific NameID
    ({{nameid-construction}}), maps OIDC claims to SAML attributes
    per the SP-Binding's policy ({{claim-attribute-mapping}}), and
    posts a signed SAML `Response` to the SP's ACS endpoint
    ({{browser-based-flow}}).

### Token Exchange Mode {#overview-token-exchange}

~~~
   +---------+                              +----------+
   |         |---(A)-- OIDC ID Token ------>|          |
   |         |         (token-exchange)     |          |
   | Client  |                              |  Bridge  |
   |         |<--(B)-- SAML Assertion ------|          |
   |         |         (signed)             |          |
   +---------+                              +----------+
~~~

(A) A confidential client at the Bridge's token endpoint, having
    obtained an ID Token from the OP through its own OIDC client
    registration, presents the ID Token as the `subject_token` of
    an {{RFC8693}} Token Exchange request.

(B) The Bridge validates the ID Token, derives an SP-specific
    NameID for the SP-Binding bound to the authenticated client,
    constructs a signed SAML assertion, and returns it
    ({{token-exchange}}). No browser interaction is involved.

In both modes, the SP sees the Bridge as a standard SAML IdP and
needs no change to its SAML metadata, ACS configuration, or trust
relationship.

## Non-Goals {#non-goals}

The profile is scoped to OIDC-to-SAML interoperability and identity continuity.
It does NOT define:

* a generalized SAML 2.0 IdP, including SAML-native authentication, generic
  SAML SLO orchestration, AttributeQuery, AuthzDecisionStatement, or
  SP-specific SAML extensions;
* federation establishment between otherwise independent OIDC and SAML parties;
* API authorization architectures, scope design, or resource-server policy;
* provisioning, SCIM integration, or account-lifecycle workflows;
* governance models (approval, attestation, audit, segregation of duties);
* migration methodology (rollout patterns, cutover sequencing); or
* SAML EncryptedAssertion or EncryptedID issuance.

# Conventions and Terminology

## Requirements Notation

{::boilerplate bcp14-tagged}

## Terminology

This document uses the terms "authorization server", "client", "client
identifier", "issuer", and "scope" from OAuth 2.0, OAuth 2.0 Authorization
Server Metadata, Token Exchange, and OpenID Connect. It uses the terms
"Identity Provider", "Service Provider", "entityID", "Assertion", "Attribute",
"AttributeStatement", "AuthnStatement", and "AuthnContext" from SAML 2.0.

For the purposes of this document:

OpenID Provider (OP):
: The authorization server that issues OIDC ID Tokens and is the ultimate
  authentication authority for end-users in this profile.

Bridge:
: The component that implements three roles simultaneously: an OIDC Relying
  Party at the OP, a SAML IdP facade toward one or more SAML SPs, and (when
  the Token Exchange variant is offered) an OAuth 2.0 Authorization Server at
  its own token endpoint. The Bridge is the entity that constructs and signs
  SAML assertions.

This profile involves OIDC and OAuth client registrations in three distinct
places. Where ambiguity is possible, this document uses the qualifying
prepositional phrase rather than a separate term:

* the Bridge's OIDC client at the OP. Used in the browser-based
  flow. Its `client_id` is the expected `aud` value of ID Tokens
  consumed in that flow.
* a Token Exchange client's OAuth client at the Bridge's token
  endpoint. Used to authenticate the client and to look up its bound
  `saml_sp_entity_id`.
* a Token Exchange client's OIDC client at the OP. Separate from the
  Bridge's own client. Used to obtain the ID Token the client later
  submits to the Bridge. Each SP-Binding configures the set of these
  OP client identifiers it accepts as ID Token `aud`.

SAML SP Entity ID:
: The SAML entityID of a Service Provider that receives SAML assertions from
  the Bridge. This value is used as the audience of constructed assertions and
  as the relying-party key for subject and attribute policy.

SP-Binding:
: The per-SP administrative configuration at the Bridge that records the
  relationship between a SAML SP and the OIDC-based authentication context,
  including subject mapping policy, NameID format, attribute release policy,
  and ACS endpoints.

Local Account:
: The Bridge's or OP's internal account record for the end-user. Subject
  continuity at the Bridge requires that a given (`iss`, `sub`) pair can be
  resolved to one stable SP-specific NameID per SP-Binding.

# Applicability and Deployment Model {#applicability}

This profile assumes that the OP and the Bridge are operated by, or under
the administrative control of, the same authority. It is not intended to
establish federation trust between otherwise independent OIDC and SAML
parties.

The profile establishes three bindings:

* **Issuer binding:** the Bridge's SAML IdP Entity ID is bound to one OP
  issuer. A Bridge serving multiple OPs requires separate Bridge instances.
* **SP-Binding:** the Bridge maintains per-SP-Binding configuration scoping
  subject computation and attribute release (see {{sp-binding-configuration}}).
* **Subject and NameID binding:** the Bridge derives a stable, SP-specific
  NameID from the OP subject (see {{nameid-construction}}).

When these bindings are present, the Bridge can issue SAML assertions that
preserve the subject identifiers and claim release policy each SAML SP
previously relied upon, without requiring SP-side configuration changes.

The profile is applicable only when the Bridge can resolve each
OP-authenticated end-user deterministically to exactly one active Local
Account before constructing a SAML assertion (see
{{local-subject-resolution}}).

# Binding and Issuance Model {#binding-and-issuance-model}

This profile uses the SAML SP Entity ID as the primary translation key
because SAML pairwise subject identifiers, attribute release policy, and
audience restrictions are already bound to that value. The Bridge derives
the SP-specific NameID and applies the SP-specific release policy by
indexing into the SP-Binding configuration rather than by inspecting the
ID Token alone.

The primary guarantee against cross-SP assertion injection in the Token
Exchange variant is a three-way binding among:

* the authenticated client identity at the Bridge's token endpoint;
* the client's registered authorization for a specific `saml_sp_entity_id`;
  and
* the SP-Binding's authorized OP `client_id` set, against which the
  presented ID Token's `aud` is validated.

Together these ensure that only a client legitimately holding the
SP-Binding relationship can drive issuance of an assertion for that SP, and
only from an ID Token that was minted for an authorized OP client. The
Bridge, acting as the SAML Issuer, is responsible for constructing the
assertion's `Recipient`, `AudienceRestriction`, and `InResponseTo`
correctly so that the SP's bearer-confirmation validation succeeds.

This profile uses Token Exchange {{RFC8693}} rather than a direct OIDC flow
at the Bridge's token endpoint because the input is an already-issued ID
Token that proves an end-user authentication event; the Bridge is
converting that proof into a SAML assertion for a known SP, not performing
a new authentication. The `subject_token_type` and `issued_token_type`
values disambiguate this use from other Token Exchange profiles. A Bridge
MAY support this profile alongside other Token Exchange profiles at the
same token endpoint; routing is determined by `subject_token_type` and
`requested_token_type`.

## Issuance Authorization {#authorization-assertion-model}

The Bridge MUST NOT treat a valid ID Token as authorization to issue a SAML
assertion. The ID Token proves the authentication event; the decision to
issue an assertion to a given SP remains a Bridge policy decision, grounded
in the SP-Binding configuration established before any assertion request is
processed.

Before constructing a SAML assertion, the Bridge MUST have an
issuance basis. The basis MUST:

* be bound to the resolved Local Account and the target SP-Binding;
* authorize the specific NameID format and attributes to be released;
  and
* for the Token Exchange variant, authorize the authenticated client
  to request assertions for that SP-Binding.

The issuance basis MAY come from static SP-Binding configuration,
administrative policy, or any equivalent mechanism established
before the assertion is issued.

The Bridge MUST NOT create, expand, or infer an issuance basis solely from
the presence, validity, or contents of the ID Token. OIDC claims MAY inform
an already-established release policy rule, but the policy itself MUST
exist independently of the ID Token presented in the current request.

The Bridge MUST NOT release all OIDC claims to all SPs by default.
Deployments MUST configure explicit attribute release allowlists per
SP-Binding.

# SP-Binding Configuration {#sp-binding-configuration}

Each SP the Bridge serves has an SP-Binding maintained administratively at
the Bridge. An SP-Binding MUST record at minimum:

* the SAML SP Entity ID;
* one or more registered Assertion Consumer Service (ACS) URLs for that SP;
* the NameID format to use for that SP (see {{nameid-format-policy}});
* the attribute release allowlist for that SP;
* the `AuthnContextClassRef` mapping policy for that SP;
* whether SP-initiated flows, IdP-initiated flows, or both are permitted; and
* for the Token Exchange variant, the set of OP `client_id` values whose ID
  Tokens are authorized to trigger assertion issuance for this SP-Binding.

The Bridge MUST validate that any `AuthnRequest` received for an SP
corresponds to a configured SP-Binding before initiating an OIDC flow.

IdP-initiated flows SHOULD be disabled by default. When an SP-Binding
explicitly permits IdP-initiated flows, the Bridge MUST select the target SP
and ACS URL from the SP-Binding's pre-configured values only and MUST employ
an anti-CSRF mechanism (such as a signed, time-limited, Bridge-generated
launch token) to ensure the flow was initiated by an authorized party.

## `saml_sp_entity_id` Client Metadata {#sp-binding-saml-sp-entity-id}

For the Token Exchange variant, a client at the Bridge's token endpoint is
linked to an SP-Binding through the `saml_sp_entity_id` client metadata
parameter defined by {{SAML-OIDC-MIGRATION}}.

When `saml_sp_entity_id` is present in a client registration at the Bridge's
token endpoint:

* its value MUST exactly match the SAML SP Entity ID of an existing
  SP-Binding; the Bridge MUST reject registration otherwise;
* the Bridge MUST bind the client registration to that SP-Binding for
  purposes of NameID derivation and attribute release;
* the Bridge MAY allow multiple clients to share the same
  `saml_sp_entity_id` when local policy determines that those clients
  represent the same SP relationship, in which case the Bridge MUST apply the
  same NameID derivation and attribute release policy to all of them; and
* the Bridge MUST ensure that the client authenticated at runtime is one of
  the clients authorized to use that `saml_sp_entity_id`.

The Token Exchange variant is defined only for confidential clients; the
Bridge MUST NOT enable this profile for a public client registration.

### Authorizing the `saml_sp_entity_id` Binding {#sp-binding-saml-sp-entity-id-authorization}

The `saml_sp_entity_id` binding is security-critical: it determines which
SP-Binding (and therefore which NameID derivation, attribute release policy,
and ACS endpoints) apply to assertions the Bridge constructs for that
client. Establishing this binding through an unauthenticated or self-asserted
client registration request is incompatible with this profile.

For statically registered clients, the binding MUST be established through
administrative configuration controlled by the authority that operates the
Bridge.

For clients registered through dynamic client registration {{RFC7591}}, the
Bridge MUST require at least one of the following before accepting a
registration that sets `saml_sp_entity_id`:

* a software statement {{RFC7591}} signed by a trust anchor administratively
  associated with the bound `saml_sp_entity_id`;
* a registration credential (initial access token, mTLS client certificate,
  or equivalent) administratively bound to the permitted value(s); or
* an equivalent administrative authorization established before the
  registration request.

Anonymous or open dynamic client registration MUST NOT be enabled for
clients that bind a `saml_sp_entity_id` under this profile. The same
authorization rules apply to any subsequent update that adds, removes, or
changes `saml_sp_entity_id`.

## NameID Format Policy {#nameid-format-policy}

Each SP-Binding MUST specify the NameID format for issued assertions.
Supported formats:

* `...persistent`: stable, non-reassignable. Derived per
  {{pairwise-nameid-construction}} or {{public-nameid-construction}}.
* `...emailAddress`: derived from the OIDC `email` claim. Permitted
  only when the SP-Binding allows it and `email` is releasable.
* `...transient`: session-scoped, freshly generated per assertion.
  MUST NOT be persisted or reused. MUST NOT be used for SPs
  requiring cross-session continuity.
* `...unspecified`: SHOULD NOT be used for new SP-Bindings.
  Permitted only for legacy SPs and only when the SP-Binding defines
  the identifier semantics.

When an `AuthnRequest` includes `NameIDPolicy`, the Bridge MUST
verify format compatibility with the SP-Binding and MUST return a
SAML error on incompatibility.

`NameIDPolicy/@AllowCreate` applies only to persistent NameID formats
per {{SAML2-CORE}}. When the requested format is persistent and
`@AllowCreate="false"`:

* if no NameID mapping is persisted for the Local Account and
  SP-Binding, the Bridge MUST return SAML status
  `urn:oasis:names:tc:SAML:2.0:status:InvalidNameIDPolicy` and MUST
  NOT create a mapping;
* if a mapping is already persisted, the Bridge MUST use it.

For non-persistent formats (transient, emailAddress), the Bridge MUST
ignore `@AllowCreate`.

# Bridge Metadata {#bridge-metadata}

## SAML IdP Metadata {#bridge-saml-metadata}

The Bridge MUST expose a SAML `EntityDescriptor` as defined by {{SAML2-METADATA}}
with the Bridge's SAML IdP Entity ID as its `entityID`. This metadata:

* MUST include an `IDPSSODescriptor` element;
* MUST set `IDPSSODescriptor/@WantAuthnRequestsSigned="true"` when
  every configured SP-Binding requires signed `AuthnRequest` messages.
  When at least one SP-Binding permits unsigned requests, the Bridge
  MAY publish `"false"`. In either case, the per-SP-Binding rules in
  {{authn-request-processing}} remain authoritative;
* MUST include at least one `SingleSignOnService` endpoint using the HTTP-POST
  or HTTP-Redirect binding;
* MUST include the key material the Bridge uses to sign SAML assertions in
  `KeyDescriptor` elements with `use="signing"`;
* MUST be accessible at an HTTPS URI the Bridge can publish and maintain; and
* SHOULD be updated whenever the Bridge's signing key material changes.

SAML SPs that federate with the Bridge use this metadata to validate the Bridge's
assertion signatures. The Bridge's signing keys published in SAML metadata MUST
NOT be confused with the OP's JOSE signing keys published at `jwks_uri`.

## OAuth Authorization Server Metadata {#bridge-as-metadata}

When the Bridge supports the Token Exchange variant, it operates as an
OAuth 2.0 Authorization Server and MUST publish authorization server
metadata at the `.well-known/oauth-authorization-server` location defined
by {{RFC8414}} relative to its issuer identifier. A Bridge that is also an
OpenID Provider MAY additionally publish at
`.well-known/openid-configuration` as defined by {{OIDC-DISCOVERY}}; that
configuration is not required by this profile.

The Bridge's SAML IdP Entity ID, the `oidc_op_issuer` value defined below,
and the `issuer` value in the Bridge's own AS metadata identify three
distinct protocol entities. Clients using the Token Exchange variant MUST
use the Bridge's token endpoint `issuer`, not `oidc_op_issuer`, as the
OAuth issuer for that endpoint.

### `oidc_op_issuer` {#metadata-oidc-op-issuer}

The `oidc_op_issuer` AS metadata parameter is a string identifying the OIDC
issuer of the OP that backs the Bridge. When present:

* it MUST equal the `iss` claim value in ID Tokens issued by the backing
  OP;
* it MUST identify the OP that is operated by or under the administrative
  control of the same entity that operates the Bridge; and
* it MUST be the value against which the Bridge validates the `iss` claim
  of incoming ID Tokens under this profile.

## Capability and Metadata Matrix {#bridge-capability-matrix}

| Capability | Required metadata | Notes |
| --- | --- | --- |
| SAML IdP facade | SAML `EntityDescriptor` with `IDPSSODescriptor` | The Bridge's SAML entity ID and signing keys. |
| Backing OP binding | `oidc_op_issuer` | Identifies the OP whose ID Tokens the Bridge accepts. |
| Token Exchange: ID Token to SAML assertion | `grant_types_supported` containing `urn:ietf:params:oauth:grant-type:token-exchange` | This profile defines only `urn:ietf:params:oauth:token-type:id_token` as the accepted `subject_token_type`. |

A client MUST NOT assume support for the Token Exchange variant unless
`grant_types_supported` indicates that support.

## Federation Metadata Coexistence {#federation-metadata-coexistence}

A Bridge operates two protocol stacks in parallel. SAML 2.0 toward
SPs uses `EntityDescriptor` metadata ({{SAML2-METADATA}}). OpenID
Connect and OAuth 2.0 toward the backing OP, and toward Token
Exchange clients when offered, use
`.well-known/oauth-authorization-server` or
`.well-known/openid-configuration` ({{RFC8414}}, {{OIDC-DISCOVERY}}).
This profile does not unify the two. Deployments SHOULD observe the
following.

* The Bridge's SAML IdP `entityID`, the backing OP's `issuer`, and the
  Bridge's own token endpoint `issuer` are distinct values used in their own
  protocol contexts; incidental string equality does not imply identity.
* SAML signing keys (`KeyDescriptor`) and JOSE signing keys (`jwks_uri`) are
  discovered through their respective protocol metadata. Cross-protocol key
  reuse is permitted, but each set MUST be published in the metadata of the
  protocol that uses it.
* When the Bridge publishes `saml_idp_entity_id` per {{SAML-OIDC-MIGRATION}}
  in its OAuth/OIDC metadata, its value MUST equal the Bridge's SAML IdP
  Entity ID, allowing clients to correlate the token endpoint with the SAML
  IdP facade.

# ID Token Validation {#id-token-validation}

## Accepted ID Token Inputs {#accepted-id-token-inputs}

The Bridge MUST validate ID Tokens per {{OIDC-CORE}} Section 3.1.3.7.
This profile adds the following constraints on top of {{OIDC-CORE}}:

* the `iss` claim MUST match `oidc_op_issuer`. ID Tokens from any
  other issuer MUST be rejected;
* `aud` validation is flow-specific, see below;
* the `iat` freshness window is enforced per {{id-token-freshness}};
* for ID Tokens received via Token Exchange, the Bridge MUST NOT
  validate `nonce`. The Bridge did not issue the authorization
  request and has no expected `nonce` value.

The `aud` claim and (when `aud` is a JSON array) the `azp` claim
are validated against a set of authorized OP `client_id` values
that depends on the flow:

* browser-based flow: the singleton {Bridge's own `client_id` at
  the OP};
* Token Exchange variant: the SP-Binding's administratively
  configured authorized OP `client_id` set.

The Bridge MUST reject an ID Token whose `aud` (and `azp` when
required) is not in the applicable authorized set. The Token
Exchange authorized set MUST NOT be inferred from the presented
token.

## Freshness and Authentication Time {#id-token-freshness}

The Bridge MUST enforce a freshness window before issuing a SAML
assertion. When `auth_time` is present, the Bridge MUST verify that
the time elapsed since `auth_time` does not exceed the configured
window. Recommended defaults:

* 8 hours for the browser-based flow;
* 5 minutes for the Token Exchange variant (the ID Token is a
  credential at rest).

For the Token Exchange variant, when `auth_time` is absent the
Bridge MUST apply the same 5-minute window against `iat`. The Bridge
MUST always verify `iat` freshness.

When deriving `AuthnInstant` from `auth_time`, the freshness window
MUST be evaluated against `auth_time`, not `iat`.

## ID Token Claims Required for This Profile {#required-id-token-claims}

The Bridge MUST NOT issue a SAML assertion unless the ID Token contains a `sub`
claim. The `sub` claim, together with the `iss` claim, forms the stable input
to NameID derivation under {{nameid-construction}}.

The Bridge SHOULD require the ID Token to contain `auth_time` when the
SP-Binding requires `AuthnInstant` to reflect the original end-user
authentication event rather than the Bridge's assertion issuance time. If
`auth_time` is absent, the Bridge MAY use the current time as `AuthnInstant`
and SHOULD document this substitution in local policy.

This profile does not define processing of encrypted ID Tokens. The Bridge MUST
reject any ID Token that cannot be validated as a signed JWT under the rules
above.

# Browser-Based Protocol Flow {#browser-based-flow}

This section defines the browser-based flow in which a SAML SP initiates
authentication, the Bridge authenticates the end-user via the OP, and the
Bridge constructs and delivers a SAML assertion to the SP's ACS. The flow
conforms to the SAML 2.0 Web Browser SSO Profile defined in {{SAML2-PROFILES}}
Section 4.1; this section narrows that profile to the OIDC-backed Bridge
case.

## Flow Overview {#flow-overview}

The browser-based flow proceeds as follows:

1. The end-user accesses the SP. The SP generates a SAML `AuthnRequest` and
   redirects the user's browser to the Bridge's `SingleSignOnService` endpoint.

2. The Bridge receives the `AuthnRequest`. It MUST validate the request per
   {{authn-request-processing}} before proceeding.

3. The Bridge initiates an OIDC Authorization Code flow by redirecting the
   browser to the OP's authorization endpoint per
   {{oidc-authorization-request}}.

4. The OP authenticates the end-user, as required by the Bridge's authorization
   request. The OP redirects the browser back to the Bridge's OIDC redirect URI
   with an authorization code.

5. The Bridge exchanges the authorization code for an ID Token and, optionally,
   an access token at the OP's token endpoint.

6. The Bridge validates the ID Token per {{id-token-validation}}.

7. The Bridge resolves the end-user to a Local Account per
   {{local-subject-resolution}}.

8. The Bridge constructs a SAML assertion per {{assertion-construction}} and
   wraps it in a SAML `Response`.

9. The Bridge delivers the SAML `Response` to the SP's ACS URL via the HTTP-POST
   binding. The Bridge MUST populate `Response/@InResponseTo` with the
   `AuthnRequest/@ID` when the flow is SP-initiated.

## AuthnRequest Processing {#authn-request-processing}

Upon receiving a SAML `AuthnRequest`, the Bridge MUST:

1. validate the `AuthnRequest` signature against the SP-Binding's registered
   signing key. The Bridge MUST validate `AuthnRequest` signatures in a manner
   resistant to XML Signature Wrapping; see {{xsw-incoming}}. An unsigned
   `AuthnRequest` MUST be rejected unless the SP-Binding explicitly permits
   unsigned requests for a documented reason and the deployment threat model
   has accepted the associated risks;
2. verify that `AuthnRequest/Issuer` corresponds to a configured SP-Binding
   and that the SP-Binding permits the binding (HTTP-Redirect or HTTP-POST)
   over which the request was received. The Bridge MUST require
   `AuthnRequest/Issuer/@Format` to be absent or
   `urn:oasis:names:tc:SAML:2.0:nameid-format:entity`; other formats MUST be
   rejected. The Bridge MUST NOT honor `AuthnRequest/Scoping`. If
   `Scoping/IDPList` is non-empty and excludes the Bridge's
   configured OP, the Bridge SHOULD reject with
   `urn:oasis:names:tc:SAML:2.0:status:NoSupportedIDP`.
   `AuthnRequest/NameIDPolicy/@SPNameQualifier`,
   when present, MUST equal the SAML SP Entity ID of the issuing SP-Binding;
   other values MUST be rejected;
3. verify that `AuthnRequest/@Destination`, when present, exactly matches the
   Bridge's `SingleSignOnService` endpoint URL for the binding used;
4. verify that `AuthnRequest/@IssueInstant` is within a deployment-configured
   freshness window (in the absence of stricter local policy, 5 minutes is
   RECOMMENDED). Requests outside the window MUST be rejected;
5. track `AuthnRequest/@ID` values and reject any request whose ID has been
   processed within the freshness window;
6. if `NameIDPolicy` is present, verify that the requested format is
   compatible with the SP-Binding's configured NameID format;
7. if `RequestedAuthnContext` is present, record the values and
   `Comparison` for use in constructing the issued assertion's
   `AuthnContext`. This profile supports `Comparison` of `exact`
   (the {{SAML2-CORE}} default) and `minimum` only; `maximum` and
   `better` MUST be rejected with
   `urn:oasis:names:tc:SAML:2.0:status:NoAuthnContext`. The Bridge
   MUST NOT issue an assertion that does not satisfy the recorded
   `Comparison` semantics, and MUST return a SAML error if it
   cannot satisfy them;
8. if `@ForceAuthn` is `true`, cause a fresh end-user
   authentication by including `prompt=login` in the OIDC
   authorization request. The Bridge MAY also include `max_age=0`;
9. if `@IsPassive` is `true`, include `prompt=none` and return SAML
   status `...NoPassive` when the OP indicates interaction would
   be required;
10. record `AuthnRequest/@ID` for inclusion in `InResponseTo`; and
11. resolve the destination ACS. If `AssertionConsumerServiceURL`
    is present it MUST match a registered ACS URL; if absent, use
    the SP-Binding default; if `AssertionConsumerServiceIndex` is
    present, resolve it against the SP's SAML metadata.
    `AssertionConsumerServiceURL` and
    `AssertionConsumerServiceIndex` MUST NOT both be present.

The Bridge MUST reject `AuthnRequest` messages for unregistered SP
entities.

The Bridge MAY preserve `RelayState` through the OIDC flow within
the {{SAML2-BINDINGS}} length limit. The Bridge MUST NOT treat
`RelayState` as carrying authorization or identity semantics.

## OIDC Authorization Request {#oidc-authorization-request}

When initiating the OIDC Authorization Code flow, the Bridge:

* MUST use `response_type=code`;
* MUST include `scope=openid`;
* MUST include a fresh, unpredictable `nonce` value bound to the SAML
  `AuthnRequest` context (for example, by computing it as an HMAC over the
  request ID under a Bridge-held key);
* MUST include a `state` value that is unpredictable to third
  parties and integrity-protected (for example, an HMAC over the
  SAML `AuthnRequest/@ID` and a fresh nonce under a Bridge-held key,
  or an authenticated-encryption payload), so the returned response
  can be matched to the original SAML request and cannot be spliced;
* MUST use PKCE {{RFC7636}} with the `S256` code challenge method, in
  accordance with current OAuth security best practice {{RFC9700}};
* SHOULD request the `auth_time` claim via the `claims` parameter.
  Per {{OIDC-CORE}}, `max_age=0` forces re-authentication; it does
  not merely request `auth_time`;
* MAY translate `RequestedAuthnContext` values from the `AuthnRequest` into
  `acr_values` in the OIDC authorization request, when local policy defines
  this translation;
* SHOULD request, via the `scope` and `claims` parameters, the claims needed
  to satisfy the SP-Binding's attribute release policy and NameID derivation;
  and
* MUST translate `AuthnRequest/@ForceAuthn` and `@IsPassive` into the
  corresponding OIDC parameters as described in {{authn-request-processing}}.

The Bridge MUST NOT pass the SAML SP Entity ID to the OP in a way that allows
the OP to alter its authentication behavior based on which SP initiated the
request, unless local policy explicitly authorizes such conditioning. See
{{privacy-considerations}} for the related correlation risk.

## Error Handling {#browser-flow-error-handling}

If the OP returns an authorization error, the Bridge MUST return a SAML error
`Response` to the SP's ACS URL. The Bridge MUST NOT expose the OP's internal
error codes or descriptions to the SP. Common OIDC error codes SHOULD be mapped
to SAML status codes as follows:

* `login_required` or `interaction_required` →
  `urn:oasis:names:tc:SAML:2.0:status:NoPassive` (when the SP requested
  passive authentication) or `urn:oasis:names:tc:SAML:2.0:status:AuthnFailed`
  otherwise;
* `consent_required` or `access_denied` →
  `urn:oasis:names:tc:SAML:2.0:status:RequestDenied`;
* `temporarily_unavailable` or `server_error` →
  `urn:oasis:names:tc:SAML:2.0:status:Responder`; and
* any other OP error → `urn:oasis:names:tc:SAML:2.0:status:Responder`.

If ID Token validation fails, the Bridge MUST return a SAML error `Response`
and MUST NOT issue an assertion.

# Token Exchange: ID Token to SAML Assertion {#token-exchange}

This section defines how a client can use OAuth 2.0 Token Exchange
to obtain a SAML 2.0 assertion from an OIDC ID Token.

A Bridge supporting this pattern MUST support Token Exchange as defined by
{{RFC8693}}. If the Bridge publishes metadata, its `grant_types_supported`
metadata SHOULD include `urn:ietf:params:oauth:grant-type:token-exchange`.

Under this pattern, the client obtains an ID Token from the OP by any
OIDC flow permitted by local policy, then presents that ID Token to the Bridge's
token endpoint. The Bridge validates the ID Token, resolves the end-user,
derives the SP-specific NameID, maps claims to attributes, constructs a SAML
assertion for the SP bound to the client's `saml_sp_entity_id`, and returns
the signed SAML assertion in the Token Exchange response.

## Request {#token-exchange-request}

The client makes a Token Exchange request to the Bridge's token endpoint using
the parameters described below.

The following non-normative example requests a SAML assertion from an OIDC
ID Token using `private_key_jwt` client authentication:

~~~ http
POST /token HTTP/1.1
Host: bridge.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aid_token
&subject_token=eyJhbGciOiJSUzI1NiIsImtpZCI6IjEifQ...
&requested_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Asaml2
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer
&client_assertion=eyJhbGciOiJSUzI1NiJ9...
~~~

The request parameters are summarized below:

| Parameter | Requirement | Summary |
|---|---:|---|
| `grant_type` | REQUIRED | Identifies the request as OAuth 2.0 Token Exchange. |
| `subject_token` | REQUIRED | Carries the OIDC ID Token issued by the backing OP. |
| `subject_token_type` | REQUIRED | Identifies the subject token as an OIDC ID Token. |
| `requested_token_type` | REQUIRED | Requests a SAML 2.0 assertion. |

`grant_type`:

REQUIRED. The value `urn:ietf:params:oauth:grant-type:token-exchange`.

`subject_token`:

REQUIRED. The serialized OIDC ID Token issued by the OP identified by
`oidc_op_issuer`. The ID Token MUST be submitted in its compact serialization
form as defined by {{OIDC-CORE}}. The Bridge MUST validate this token per
{{id-token-validation}}.

`subject_token_type`:

REQUIRED. The value `urn:ietf:params:oauth:token-type:id_token`.

`requested_token_type`:

REQUIRED. The value `urn:ietf:params:oauth:token-type:saml2`. {{RFC8693}} marks
`requested_token_type` as OPTIONAL at the Token Exchange protocol level. This
profile narrows that rule: an absent or unsupported `requested_token_type` MUST
be rejected with `invalid_request`.

The Bridge derives the target SP from the authenticated client's registered
`saml_sp_entity_id`. This profile does not define use of the `audience`,
`resource`, `scope`, `actor_token`, or `actor_token_type` Token Exchange
parameters; clients MUST NOT send them. If any are present, the Bridge
MUST reject the request with `invalid_request`. The Bridge MUST NOT
allow such parameters to alter the target SP, audience, release policy,
or delegation chain of the issued assertion.

Per {{sp-binding-saml-sp-entity-id}}, the Token Exchange variant is defined
only for confidential clients. The authenticated client MUST be a
confidential client registered with a `saml_sp_entity_id` corresponding to a
configured SP-Binding. Asymmetric-key client authentication methods such as
`private_key_jwt` and `tls_client_auth` SHOULD be preferred over
`client_secret_basic` and `client_secret_post`, consistent with current
OAuth security guidance {{RFC9700}}.

## Bridge Processing {#token-exchange-bridge-processing}

Upon receiving the request, the Bridge MUST:

1. authenticate the client in accordance with its registered client
   authentication method;
2. verify that the authenticated client has a registered `saml_sp_entity_id`
   corresponding to a configured SP-Binding;
3. validate the ID Token as described in {{id-token-validation}};
4. verify that the `requested_token_type` is
   `urn:ietf:params:oauth:token-type:saml2`;
5. resolve the end-user to a Local Account as described in
   {{local-subject-resolution}};
6. derive the SP-specific NameID according to {{nameid-construction}};
7. map claims to SAML attributes according to {{attribute-mapping}} and apply
   the SP-Binding's attribute release policy; and
8. construct and sign the SAML assertion per {{assertion-construction}} and
   return it in the Token Exchange response.

The Bridge MUST reject the request if the ID Token `iss` claim does not match
`oidc_op_issuer`.

The Bridge MUST reject the request if the ID Token `aud` claim does not match
an authorized OP `client_id` configured in the SP-Binding per
{{accepted-id-token-inputs}}.

The Bridge MUST reject the request if the ID Token does not contain a `jti`
claim, or if the (`iss`, `jti`) pair has already been accepted within the
token's validity window.

The Bridge MUST reject the request if subject resolution yields no Local
Account, more than one candidate, or a Local Account that is disabled,
suspended, deprovisioned, or otherwise not eligible.

## Successful Response {#token-exchange-successful-response}

For a successful request, the Bridge MUST return a Token Exchange response as
defined by {{RFC8693}} with:

* `issued_token_type` set to `urn:ietf:params:oauth:token-type:saml2`;
* `access_token` containing the base64url-encoded SAML assertion, as defined
  in Section 5 of {{RFC4648}}, without line wrapping and without padding
  characters (`=`); and
* `token_type` set to `N_A`.

The Bridge SHOULD set `expires_in` to the time, in seconds, until
`SubjectConfirmationData/@NotOnOrAfter` (the delivery window).
`expires_in` MUST NOT exceed the time until
`Conditions/@NotOnOrAfter`. The Bridge SHOULD make
`SubjectConfirmationData/@NotOnOrAfter` no later than
`Conditions/@NotOnOrAfter` ({{subject-confirmation-data}}).

The Bridge MUST NOT include a `refresh_token` in the Token Exchange response.
SAML assertions issued under this profile are short-lived bearer artifacts
that do not have refresh semantics; clients needing a new assertion MUST
present a fresh ID Token in a new Token Exchange request.

The following non-normative example shows a successful response:

~~~ json
{
  "issued_token_type": "urn:ietf:params:oauth:token-type:saml2",
  "access_token": "PHNhbWwyOkFzc2VydGlvbi4uLjwvc2FtbDI6QXNzZXJ0aW9uPg",
  "token_type": "N_A",
  "expires_in": 300
}
~~~

{{RFC8693}} uses the `access_token` response member to carry all issued token
types. Clients MUST interpret the `access_token` value as a base64url-encoded
SAML 2.0 assertion when `issued_token_type` is
`urn:ietf:params:oauth:token-type:saml2`.

## Error Response {#token-exchange-error-response}

Error responses follow {{RFC6749}} Section 5.2 and {{RFC8693}}
Section 2.4. Error code selection follows {{RFC7522}} Section 2.1.1:
defects in the `subject_token` (the OIDC ID Token) are reported as
`invalid_grant`, defects in request structure as `invalid_request`.

The Bridge MUST use:

* `invalid_request` for malformed or inconsistent parameters.
  Examples: missing `subject_token_type`, unsupported
  `requested_token_type`, or presence of any prohibited parameter
  per {{token-exchange-request}};
* `invalid_grant` when the ID Token is parseable but unacceptable.
  This covers an invalid, expired, or wrong-issuer ID Token; an
  `aud` outside the SP-Binding's authorized set; missing or replayed
  `jti`; missing or unresolvable `sub`;
* `unauthorized_client` when the client is not permitted to use the
  bound SP-Binding or its SP Entity ID;
* `server_error` for failures not attributable to the request or the
  ID Token (for example, transient infrastructure issues).

# Local Subject Resolution {#local-subject-resolution}

Before deriving a NameID or constructing a SAML assertion under this profile,
the Bridge MUST resolve the ID Token's `sub` and `iss` claims to exactly one
active Local Account.

The Bridge resolves the Local Account using the pair (`iss`, `sub`) from the
ID Token. This pair is the stable input for subject resolution; the Bridge MUST
NOT treat any other ID Token claim as a primary resolution key. The Bridge MAY
use other claims such as `email` as supplemental hints to locate an existing
Local Account only when local policy has already bound that supplemental value
for the trusted OP to exactly one Local Account.

If the `email` claim is present in the ID Token, the Bridge MUST treat it as
a mutable account attribute, not as a stable subject identifier. It MUST NOT by
itself establish a new Local Account linkage or a new NameID continuity mapping.

This profile does not define just-in-time provisioning. Any deployment-
specific account creation or activation triggered by a validated ID Token
MUST complete before NameID derivation and assertion construction.

If resolution yields no Local Account, more than one candidate Local Account,
or a Local Account that is disabled, suspended, deprovisioned, or otherwise not
eligible to authenticate, the Bridge MUST fail the profile operation and MUST
NOT issue a SAML assertion.

Once a NameID continuity mapping has been established for a resolved Local
Account and SP-Binding pair, the Bridge MUST continue to resolve the same
(`iss`, `sub`) input to the same NameID for that SP-Binding unless an authorized
administrative remapping occurs.

# NameID Construction {#nameid-construction}

The SAML `NameID` in the `Subject` element of every assertion issued under this
profile is the primary subject continuity point for SAML SPs. The Bridge MUST
issue a `NameID` value that is stable for the SP-Binding and that is consistent
with the SP's prior SAML deployment expectations.

## Pairwise NameID Construction {#pairwise-nameid-construction}

When the SP-Binding requires a pairwise `NameID`
(`urn:oasis:names:tc:SAML:2.0:nameid-format:persistent` with SP-specific
semantics), the Bridge MUST determine the `NameID` value using the first
applicable rule. Deployments establishing new SP relationships SHOULD
consider the SAML V2.0 Subject Identifier Attributes Profile
{{SAML2-SUBJ-ID}} as a more modern alternative for new SPs; this profile
addresses the existing `persistent` NameID continuity expectations of SPs
that have not adopted that profile.

1. If a pairwise NameID mapping is already persisted for the same
   Local Account and SP-Binding, the Bridge MUST reuse it.
2. Otherwise the Bridge MUST derive a deterministic, non-reassignable
   pairwise identifier using:

~~~
DST    = "SAML-OIDC-BRIDGE-PAIRWISE-V1"
input  = b64url(DST) || "." || b64url(issuer) || "." || b64url(sub)
       || "." || b64url(sp_entity_id) || "." || b64url(salt)
NameID = base64url( SHA-256( utf8(input) ) )
~~~

   where `b64url(s)` is the unpadded base64url encoding ({{RFC4648}}
   Section 5) of the UTF-8 octets of `s`. The inputs `issuer` and
   `sub` come from the ID Token. The `sp_entity_id` comes from the
   SP-Binding. The `salt` is a deployment-held secret. The Bridge
   MUST persist the resulting mapping.

The `.` separator cannot appear inside a base64url-encoded value, so
the segment join is injective. The leading DST reserves headroom for
a future revision of the derivation.

The `salt` MUST be at least 128 bits of entropy and MUST be kept
confidential. Salt compromise allows derivation of NameID values
from known `(iss, sub, sp_entity_id)` inputs and enables cross-SP
correlation.

The Bridge MUST NOT expose a pairwise NameID value to any SP other than the SP
for whose SP-Binding it was derived.

## Public NameID Construction {#public-nameid-construction}

When the SP-Binding uses a public or issuer-scoped persistent `NameID`
(i.e., the same NameID value is used for the same user regardless of which
SP receives the assertion), the Bridge MUST reuse any persisted public
NameID mapping for the resolved Local Account.

If no persisted mapping exists, the Bridge MUST mint a new public
NameID value using one of:

* a randomly generated opaque identifier with at least 128 bits of
  entropy (RECOMMENDED); or
* the segment encoding from {{pairwise-nameid-construction}}, with
  the `sp_entity_id` segment omitted and a distinct DST:

~~~
DST    = "SAML-OIDC-BRIDGE-PUBLIC-V1"
input  = b64url(DST) || "." || b64url(issuer)
       || "." || b64url(sub) || "." || b64url(salt)
NameID = base64url( SHA-256( utf8(input) ) )
~~~

The Bridge MUST persist the resulting mapping. A purely deterministic
derivation without a Bridge-held secret MUST NOT be used. Any party
knowing the OP `iss` and `sub` could otherwise reproduce the NameID.

The `salt` used here MAY be the same value as the pairwise salt
({{pairwise-nameid-construction}}). It MUST be kept confidential and is
subject to the same compromise considerations.

## EmailAddress NameID {#emailaddress-nameid}

When the SP-Binding specifies `urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress`,
the Bridge MUST use the `email` claim from the ID Token as the `NameID` value,
subject to the following conditions:

* the SP-Binding MUST explicitly permit use of the email address format;
* the `email` claim MUST be present and permitted for release to this SP;
* the `email_verified` claim MUST be `true`, unless the SP-Binding explicitly
  permits unverified email addresses for a documented reason;
* the Bridge MUST include `Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress"`
  in the `NameID` element; and
* the Bridge MUST NOT use this value as the key for persistent subject
  mapping, because email addresses are mutable.

## Transient NameID {#transient-nameid}

When the SP-Binding specifies
`urn:oasis:names:tc:SAML:2.0:nameid-format:transient`, the Bridge
MUST generate a fresh value per assertion from a cryptographically
secure random source. The value MUST contain at least 128 bits of
entropy. Guessable transient NameIDs enable SubjectConfirmation
forgery and session-fixation against SPs that key on them.

The Bridge MUST NOT persist or reuse transient NameID values. The
Bridge MUST NOT use the transient format for SPs that require subject
continuity across sessions.

## Common Rules {#nameid-common-rules}

For all NameID formats:

* before issuing any chosen or derived value as `NameID`, the Bridge MUST ensure
  it conforms to the constraints of the selected NameID format as defined by
  {{SAML2-CORE}};
* the `Format` attribute MUST be present and MUST match the format configured
  in the SP-Binding;
* when using the persistent format with SP-specific semantics, the Bridge MUST
  set `SPNameQualifier` to the SAML SP Entity ID;
* when using the persistent format with issuer-scoped semantics, the Bridge
  MUST set `NameQualifier` to its own SAML IdP Entity ID; and
* the Bridge MUST NOT use a transient NameID for any SP that requires subject
  continuity across sessions.

# Attribute and Authentication Context Mapping {#attribute-mapping}

The mappings in this section govern how OIDC claims are translated into SAML
attributes and authentication context for the issued assertion.

## Authentication Context Mapping {#authn-context-mapping}

Every issued assertion MUST contain an `AuthnStatement`.

`AuthnInstant` is set from the ID Token `auth_time` (converted from
NumericDate to XML `dateTime`). When `auth_time` is absent and the
SP-Binding permits it, the Bridge MAY use its assertion issuance
time. The Bridge MUST NOT use the OIDC token issuance time as
`AuthnInstant` when it does not represent the actual authentication.

`SessionIndex` is OPTIONAL. When emitted, it carries the OIDC `sid`
when that value is unique within the Bridge's SAML issuer;
otherwise the Bridge derives and persists a Bridge-scoped session
identifier bound to the OIDC session.

`AuthnContext` MUST include `AuthnContextClassRef`, determined as
follows. If the SP-Binding defines an `acr`-to-`AuthnContextClassRef`
mapping, the Bridge MUST apply it. Otherwise, when the SP-Binding
permits passthrough, the Bridge MAY emit an ID Token `acr` value
only when it appears in the SP-Binding's allowlist. Otherwise the
Bridge emits `urn:oasis:names:tc:SAML:2.0:ac:classes:unspecified`,
or another default named by the SP-Binding. The Bridge MUST NOT
emit an `AuthnContextClassRef` asserting higher assurance than the
OP's authentication event warrants.

The OIDC `amr` claim {{RFC8176}} carries authentication-method
references, not context classes. The Bridge MUST NOT copy `amr`
into `AuthnContextClassRef`. The Bridge MAY reflect `amr` in an
SP-Binding-defined `AuthnContext` extension element; {{SAML2-CORE}}
defines no such generic extension, so the SP-Binding MUST specify
its namespace, schema, and processing rules.

## Attribute Claim Mapping {#claim-attribute-mapping}

When the issued assertion includes an `AttributeStatement`, the Bridge MUST
apply the SP-Binding's attribute release allowlist before populating any
attribute. The Bridge MUST NOT include attributes that are not on the allowlist
for the target SP-Binding, regardless of which claims are present in the ID
Token.

Common mappings from OIDC claims to SAML attributes:

* `given_name` → `givenName` (attribute Name `urn:oid:2.5.4.42`)
* `family_name` → `sn` (attribute Name `urn:oid:2.5.4.4`)
* `name` → `displayName` (attribute Name `urn:oid:2.16.840.1.113730.3.1.241`)
* `email` → `mail` (attribute Name `urn:oid:0.9.2342.19200300.100.1.3`)
* `preferred_username` → `uid` (attribute Name `urn:oid:0.9.2342.19200300.100.1.1`)
* `phone_number` → `telephoneNumber` (attribute Name `urn:oid:2.5.4.20`)

These mappings are common; deployments SHOULD configure a site-specific mapping
table in each SP-Binding that maps OIDC claim names to the attribute Name and
NameFormat expected by the target SP.

If the SP-Binding specifies a NameFormat for attributes, the Bridge MUST use
that NameFormat when populating `Attribute` elements. If no NameFormat is
configured, the Bridge SHOULD use `urn:oasis:names:tc:SAML:2.0:attrname-format:uri`
when the attribute Name is a URI (such as an OID-based name), and
`urn:oasis:names:tc:SAML:2.0:attrname-format:basic` otherwise.

Attribute processing MUST follow these rules:

* a single OIDC claim value maps to a single `AttributeValue` element;
* a JSON array claim value MAY map to multiple `AttributeValue` elements in the
  same `Attribute` when the SP-Binding permits multi-valued attributes;
* boolean OIDC claims SHOULD map to string `AttributeValue` elements with the
  value `"true"` or `"false"` unless the SP-Binding specifies a different
  encoding;
* the Bridge MUST NOT expose the `sub` claim as an `Attribute` element; the
  OIDC subject is represented exclusively through the NameID; and
* the Bridge MUST NOT include `AttributeStatement` at all if no attributes
  are permitted for release to the target SP.

If the `email_verified` claim is `false` or absent, the Bridge MUST
NOT emit any SAML attribute whose semantics signal verified-email
status. Examples include `eduPersonPrincipalName`
(`urn:oid:1.3.6.1.4.1.5923.1.1.1.6`), a `mail` attribute
(`urn:oid:0.9.2342.19200300.100.1.3`) released with an attribute-level
verification flag, or a deployment-specific verified-email indicator.

The SP-Binding MAY override this rule by explicitly identifying the
specific attribute or attributes whose verified-status semantics are
accepted.

The Bridge MUST NOT infer verification status from the presence of a
`phone_number` claim.

## Assertion and Session Lifetimes {#session-validity-bound}

The SAML assertion lifetime (governed by `Conditions/@NotOnOrAfter` and
`SubjectConfirmationData/@NotOnOrAfter`) is a short-lived artifact independent
of the underlying OIDC session. It MUST NOT be derived from the ID Token's
`exp` claim, which reflects only token validity rather than session lifetime.

The SAML `SessionNotOnOrAfter` attribute in `AuthnStatement`, when emitted,
represents the end of the authenticated session. The Bridge SHOULD set
`SessionNotOnOrAfter` to the earliest of: the `session_expiry` claim defined
by OpenID Connect Enterprise Extensions {{OIDC-ENTERPRISE-EXTENSIONS}} when
present, the OIDC session expiry as determined from the refresh token
lifetime, the OP's session management state, or the SP-Binding's configured
maximum session duration. If none of these can be determined, the Bridge
SHOULD omit `SessionNotOnOrAfter` rather than guess.

# SAML Assertion Construction {#assertion-construction}

The Bridge MUST construct a SAML 2.0 `Assertion` element conformant with
{{SAML2-CORE}}. This section specifies the required and conditional elements.

## Required Elements {#required-assertion-elements}

The Bridge MUST include the following in every issued assertion:

`Issuer`:
: The Bridge's SAML IdP Entity ID. This value MUST match the `entityID` in the
  Bridge's SAML metadata.

`Subject`:
: A `Subject` element containing:
  * a `NameID` element constructed per {{nameid-construction}}; and
  * a `SubjectConfirmation` element with
    `Method="urn:oasis:names:tc:SAML:2.0:cm:bearer"` and a
    `SubjectConfirmationData` element as specified in
    {{subject-confirmation-data}}.

`Conditions`:
: A `Conditions` element containing:
  * `NotBefore`: set to the Bridge's assertion issuance time, minus a clock
    skew tolerance. In the absence of stricter local policy, a tolerance of
    up to 30 seconds is RECOMMENDED;
  * `NotOnOrAfter`: set to a short interval after issuance. The Bridge SHOULD
    set this to no more than 5 minutes from the assertion's `IssueInstant`;
  * at least one `AudienceRestriction` containing the SAML SP Entity
    ID as one of its `Audience` values. The Bridge MAY include
    additional configured `Audience` values per {{SAML2-CORE}}; and
  * a `OneTimeUse` condition SHOULD be included to instruct the SP to enforce
    one-time consumption of the assertion per {{SAML2-CORE}} Section 2.5.1.5.

`AuthnStatement`:
: An `AuthnStatement` element constructed per {{authn-context-mapping}}.

`AttributeStatement`:
: An `AttributeStatement` element, when at least one attribute is permitted for
  release to the target SP-Binding.

`Signature`:
: A `ds:Signature` element over the `Assertion` element using the Bridge's
  SAML signing key. The Bridge MUST sign the `Assertion` element, not only the
  enclosing `Response` element. The signature MUST use RSA-SHA-256 or a stronger
  algorithm. Signature algorithms using SHA-1 as the message digest MUST NOT be
  used, including RSA with SHA-1
  (`http://www.w3.org/2000/09/xmldsig#rsa-sha1`), DSA with SHA-1
  (`http://www.w3.org/2000/09/xmldsig#dsa-sha1`), and ECDSA with SHA-1
  (`http://www.w3.org/2001/04/xmldsig-more#ecdsa-sha1`).

## SubjectConfirmationData {#subject-confirmation-data}

The `SubjectConfirmationData` element MUST include:

* `Recipient`: the ACS URL of the target SP to which the assertion will be
  delivered. For the browser-based flow, this is the ACS URL from the
  `AuthnRequest` (if valid per the SP-Binding) or the default ACS URL from the
  SP-Binding. For the Token Exchange variant, this is the ACS URL configured
  in the SP-Binding for the authenticated client's `saml_sp_entity_id`,
  regardless of how the client subsequently delivers the assertion;
  the `Recipient` value is what the SP's bearer-confirmation validation
  expects;
* `NotOnOrAfter`: a short expiration time for the subject confirmation, MUST
  NOT exceed the `Conditions/@NotOnOrAfter` value and SHOULD be set to the same
  value; and
* `InResponseTo`: for SP-initiated browser-based flows, MUST be set
  to the `AuthnRequest/@ID`. For IdP-initiated flows and the Token
  Exchange variant, this attribute MUST be omitted. These flows
  produce unsolicited assertions, and {{SAML2-PROFILES}} requires
  `InResponseTo` to be present only when responding to a specific
  `<AuthnRequest>`.

## SAML Response Wrapper {#response-wrapper}

The Bridge MUST wrap the signed `Assertion` in a SAML `Response` element for
delivery via the HTTP-POST binding in the browser-based flow. The `Response`
MUST include:

* `ID`: a fresh, unique identifier generated by the Bridge;
* `IssueInstant`: the current time;
* `Issuer`: the Bridge's SAML IdP Entity ID, as a child element of `Response`;
* `Destination`: the ACS URL of the target SP;
* `InResponseTo`: the `AuthnRequest/@ID`, for SP-initiated flows; MUST
  be omitted for IdP-initiated flows (and the Token Exchange variant
  does not produce a `Response` wrapper at all);
* `Status/StatusCode/@Value`:
  `urn:oasis:names:tc:SAML:2.0:status:Success`; and
* the signed `Assertion` as a direct child element.

The Bridge SHOULD also sign the `Response` element in addition to signing the
enclosed `Assertion`. When both are signed, the `Response` signature provides
an additional integrity layer for the SP's ACS processing.

For the Token Exchange variant, no `Response` wrapper is used; the Bridge
returns only the signed `Assertion` in base64url encoding.

## Assertion ID and Replay Prevention {#assertion-id}

The Bridge MUST generate a unique, unpredictable `Assertion/@ID` for every
issued assertion.

For the Token Exchange variant, the Bridge MUST require the submitted ID
Token to contain a `jti` claim and MUST reject any Token Exchange request
whose (`iss`, `jti`) pair has been previously accepted, until the ID Token's
`exp` has passed.

# Session Termination and Revocation {#session-termination-and-revocation}

An assertion issued under this profile represents a moment in the underlying
OIDC-authenticated session. The OIDC session may end at the OP before the
assertion's `SessionNotOnOrAfter` value would otherwise expire, and the SP's
own SAML session is independent of either. This section defines how the
Bridge connects OIDC-side session signals to the SAML sessions it has
authorized.

SAML Single Logout (SLO) and OpenID Connect logout are not equivalent
mechanisms, and this profile does not guarantee parity between them. SAML SLO
is a multi-party protocol coordinated by the SAML IdP across all participating
SPs; OIDC logout (front-channel, back-channel, and RP-Initiated Logout)
targets OIDC clients individually. The operational semantics, failure modes,
and trust requirements differ. Deployments that need coordinated logout across
both protocols MUST design that coordination outside this profile; this
profile defines only the Bridge's local obligations once it has learned that
the underlying OIDC session has ended.

## Session Binding {#session-binding}

The Bridge MUST maintain a session correlation record binding each issued
SAML session to the OIDC session that authorized it.

When the ID Token contains a `sid` claim, the Bridge MUST use that value as
the OIDC session identifier in the correlation record. When `sid` is absent,
the Bridge MUST establish its own internal session identifier at the time of
the authorization code exchange (browser-based flow) or assertion issuance
(Token Exchange variant) and bind it to the SAML sessions it subsequently
creates.

When `sid` is absent, logout propagation from the OP to the Bridge
depends on the Bridge detecting token revocation or expiry through
the refresh flow rather than a push notification. Deployments
relying on this fallback MUST document the relationship in local
policy.

The Token Exchange variant has a further limitation. The Bridge
holds no refresh token at the OP (the client holds the ID Token
directly), so the Bridge cannot detect OP-side termination through a
refresh failure. Discovery of OP-side termination requires either
(a) a {{OIDC-BC-LOGOUT}} back-channel logout from the OP to the
Bridge carrying a bound `sid`, or (b) an out-of-band revocation
channel.

Deployments using the Token Exchange variant for SPs that require
timely logout propagation SHOULD configure the OP to emit `sid` and
SHOULD register the Bridge as a back-channel logout client at the OP.
Otherwise the SAML session at the SP MUST be treated as valid for
the assertion's `SessionNotOnOrAfter` window.

## Session Termination {#session-termination}

The Bridge MUST treat the earlier of the following as the operative end of
the OIDC-authorized session for a given issued assertion:

* the assertion's `SessionNotOnOrAfter` value, when present;
* any authoritative signal that the underlying OIDC session has ended (a
  back-channel or front-channel logout notification from the OP per
  {{OIDC-BC-LOGOUT}} and {{OIDC-FC-LOGOUT}}, an OP revocation observed through
  refresh, or an equivalent administrative signal); or
* a Bridge policy decision to terminate the session (administrative action,
  risk-based revocation, etc.).

When the OIDC-authorized session has ended, the Bridge:

* MUST treat any further authentication request that relies on that OIDC
  session as requiring reauthentication;
* SHOULD propagate the termination to SPs whose SAML sessions are bound to
  that OIDC session, using SAML SLO when supported (see
  {{saml-single-logout}}); and
* MAY rely on assertion expiry alone when SLO is not supported by the
  affected SP.

This document does not define how the Bridge discovers OP-side session
termination beyond the mechanisms named above.

## SAML Single Logout {#saml-single-logout}

Support for SAML Single Logout (SLO) is OPTIONAL. When the Bridge supports
SLO:

* the Bridge's SAML metadata SHOULD include a `SingleLogoutService` endpoint;
* upon receiving a SAML `LogoutRequest` from an SP, the Bridge MUST:
  * validate the request signature against the SP's key material in the
    SP-Binding configuration, applying the XSW rules in {{xsw-incoming}};
  * verify that `LogoutRequest/Issuer` corresponds to a configured SP-Binding;
  * verify that `LogoutRequest/Subject/NameID` corresponds to a Bridge-managed
    SAML session that was established for the requesting SP-Binding; a
    LogoutRequest whose Subject does not match an active session for that
    SP-Binding MUST be rejected with a SAML error response and MUST NOT cause
    the Bridge to terminate any session;
  * verify that `LogoutRequest/@IssueInstant` is within the same freshness
    window applied to `AuthnRequest` ({{authn-request-processing}}), and that
    its `@ID` has not already been processed; and
  * terminate the SAML session with the requesting SP;
* the Bridge SHOULD propagate the logout to the OP by initiating RP-Initiated
  Logout or revoking the relevant tokens;
* the Bridge SHOULD propagate the logout to other SPs sharing the
  same OIDC session by sending `LogoutRequest` to their SLO
  endpoints. The Bridge MUST maintain a record of SAML sessions
  per OIDC `sid` (or per Bridge-scoped session identifier when `sid`
  is absent) populated at issuance time ({{session-binding}});
* the Bridge MUST return a `LogoutResponse` to the initiating SP even if
  propagation to other parties fails; and
* the Bridge MUST tolerate partial propagation failures gracefully and MUST
  NOT refuse to respond to the initiating SP because a downstream SLO request
  failed.

# Security Considerations {#security-considerations}

## Threat Model {#threat-model}

This profile assumes the following:

* The OP and the Bridge are operated by, or under the administrative control
  of, the same authority (see {{applicability}}). They share a trust
  boundary, but are network-separated components that communicate through
  their respective protocols.
* SAML SPs inherit the trust they already have with the Bridge's SAML IdP
  entity; the Bridge does not establish new federation trust.
* The Token Exchange variant is restricted to confidential clients with
  administratively authorized `saml_sp_entity_id` bindings (see
  {{sp-binding-saml-sp-entity-id-authorization}}).

In-scope attacker capabilities and the rules that mitigate them:

* obtaining an ID Token in transit. Mitigated by three-way binding
  ({{binding-and-issuance-model}}), `aud`/`azp` validation
  ({{accepted-id-token-inputs}}), and `jti` replay
  ({{assertion-id}}).
* attempting unauthorized `saml_sp_entity_id` registration. Mitigated
  by {{sp-binding-saml-sp-entity-id-authorization}}.
* forging, replaying, or manipulating a SAML `AuthnRequest` or
  `LogoutRequest`. Mitigated by signature, `Destination`, freshness,
  ID-replay, and XSW rules in {{authn-request-processing}},
  {{saml-single-logout}}, and {{xsw-incoming}}.
* splicing an OIDC authorization response across SAML contexts.
  Mitigated by `state`, `nonce`, and PKCE rules in
  {{oidc-authorization-request}}.
* compromising the Bridge's refresh token at the OP. See
  {{refresh-token-compromise}}.

Out of scope: compromise of the OP or Bridge themselves; infrastructure
threats not specific to this profile; end-user account compromise upstream
of the OP.

## SAML Signature Risks {#saml-signature-risks}

### XML Signature Wrapping on Incoming SAML {#xsw-incoming}

XML Signature Wrapping (XSW) attacks allow an adversary to manipulate a SAML
document by inserting a malicious element while retaining a valid signature
on a benign element elsewhere in the document structure. The Bridge consumes
signed `AuthnRequest` and `LogoutRequest` messages from SPs and MUST validate
those signatures in a manner resistant to XSW. Specifically:

* when validating a signed `AuthnRequest`, the Bridge MUST verify that the
  element referenced by the signature's `Reference/@URI` is the same
  `AuthnRequest` element being processed and is the unique signed element in
  the document being acted upon;
* the same rule applies to signed `LogoutRequest` messages received at the
  Bridge's SLO endpoint.

The Bridge MUST reject any document structure where these bindings cannot be
confirmed unambiguously. Implementations SHOULD use established SAML
processing libraries that have been tested against known XSW attack patterns.

### Outgoing Assertion Signature Algorithms {#outgoing-signature-algorithms}

The Bridge MUST sign every issued assertion with RSA-SHA-256 or stronger.
SHA-1 signature algorithms MUST NOT be used. Deployments SHOULD also reject
MD5-based digests, RSA keys below 2048 bits, and elliptic curves below
128-bit security, and SHOULD track current cryptographic guidance (NIST,
ENISA) for further adjustments.

Compromise of a Bridge signing key allows forgery of assertions for any SP
that trusts the Bridge. The Bridge MUST keep signing keys confidential and
MUST publish them in `KeyDescriptor` elements with `use="signing"`. During
key rotation the Bridge MUST publish both keys long enough for SPs to
refresh their cached metadata; in the absence of stricter local policy, at
least 24 hours (or one complete SP metadata refresh cycle when SPs are
known to cache longer) is RECOMMENDED.

## ID Token Validation Risks {#id-token-validation-risks}

The Bridge MUST perform complete ID Token validation as specified in
{{id-token-validation}}. In particular:

* the Bridge MUST verify the ID Token signature using keys obtained from the
  OP's `jwks_uri`, not from locally cached or manually configured JOSE keys,
  unless a pre-approved key pinning arrangement exists;
* the Bridge MUST validate `aud` according to the rules in
  {{accepted-id-token-inputs}}: the Bridge's own `client_id` for the
  browser-based flow, and an SP-Binding-authorized OP `client_id` for the
  Token Exchange variant; and
* the Bridge MUST NOT rely on the validity of an ID Token as evidence that
  the OP has authorized the issuance of a SAML assertion to any particular
  SP. The issuance basis described in {{authorization-assertion-model}} is
  a separate Bridge policy decision.

The Bridge MUST enforce the freshness window described in
{{id-token-freshness}}. Accepting a stale ID Token would allow assertion
issuance to outlive the underlying OIDC session.

## OIDC Authorization Response Splicing {#authorization-response-splicing}

A browser-based attacker may attempt to splice an authorization
response obtained in one context onto a SAML `AuthnRequest` issued
by the Bridge in a different context. The `state`, `nonce`, and
PKCE requirements in {{oidc-authorization-request}} defeat this
attack: an intercepted authorization code cannot be exchanged by
another party at the OP, and the returned `state` cannot match the
Bridge's expected value for the targeted `AuthnRequest`.

## Metadata Fetching and SSRF {#metadata-ssrf}

The Bridge fetches the OP's discovery document and JOSE key set to validate
ID Tokens. It may also fetch SP SAML metadata to discover ACS URLs and
verification keys. Fetches whose target URI is influenced by registration
data carry a Server-Side Request Forgery (SSRF) risk against internal
services. Implementations MUST restrict fetches to URIs that use the HTTPS
scheme and MUST NOT follow redirects to non-HTTPS URIs. Deployments SHOULD
additionally constrain permitted fetch targets to a pre-approved allowlist of
hosts.

The OP discovery URI and `jwks_uri` are administratively configured at the
Bridge and are not exposed to dynamic registration under this profile.

## Replay and Bearer Risks {#replay-and-bearer-risks}

Issued assertions are bearer artifacts. The short lifetimes and `OneTimeUse`
guidance in {{assertion-construction}} limit, but do not eliminate, the
window during which a captured assertion can be replayed. SPs MUST implement
their own replay detection for the assertions they accept.

For the Token Exchange variant, the `jti` replay rules in {{assertion-id}}
apply. Bridges deployed across multiple nodes MUST share `jti` replay state
across all nodes; per-node replay stores are insufficient.

Holder-of-Key SubjectConfirmation is out of scope. Deployments requiring
sender-constrained SAML credentials MUST establish that constraint outside
this profile.

## Binding Security {#binding-security}

The `saml_sp_entity_id` binding and the SP-Binding configuration are
security-critical. If an unauthorized party can register or manipulate an
`saml_sp_entity_id` value at a client, it inherits the SP-Binding's
NameID derivation, attribute release policy, and ACS endpoints. The Bridge
MUST protect registration and administrative mapping according to
{{sp-binding-saml-sp-entity-id-authorization}}. Anonymous or open dynamic
client registration MUST NOT be permitted to set this value.

When multiple Token Exchange clients share the same `saml_sp_entity_id`,
the Bridge is intentionally treating them as the same SP-Binding context.
Accidental sharing can expose the same pairwise NameID and attribute set
to clients that should have been isolated. Deployments MUST treat such
sharing as an explicit policy decision.

## Token and Claim Security {#token-and-claim-security}

Mutable, transient, or reassignable NameID formats used in place of a
persistent identifier can cause account takeover or misbinding after
identifier reuse; the rules in {{nameid-common-rules}} restrict their use
accordingly.

Incorrect `acr` mapping can overstate the authentication assurance signaled
to the SP; see {{authn-context-mapping}} for the rule that
`AuthnContextClassRef` is taken from the SP-Binding's explicit mapping or
allowlist and is never copied blindly from `acr`. The `amr` claim
{{RFC8176}} MUST NOT be copied directly into `AuthnContextClassRef`.

The Bridge MUST NOT expose the OP `sub` value as a SAML `Attribute` element.
The OIDC subject is translated exclusively into the SAML `NameID`.

Per-SP attribute release rules are normative in
{{claim-attribute-mapping}}; the SP-Binding's explicit allowlist
governs what the Bridge releases, regardless of which claims the ID
Token contains.

## Refresh Token Compromise {#refresh-token-compromise}

When the Bridge holds a refresh token at the OP to maintain its OIDC session,
compromise of that refresh token allows an attacker to obtain fresh ID
Tokens from the OP and drive Bridge assertion issuance for any SP-Binding
the Bridge serves, without further end-user interaction at the OP.

Deployments SHOULD:

* store Bridge refresh tokens with the same protection as Bridge SAML signing
  keys;
* prefer sender-constrained refresh tokens (e.g., DPoP or mTLS) where the OP
  supports them; and
* monitor for unexpected refresh-token use patterns and rotate the Bridge's
  OIDC credentials promptly on suspicion of compromise.

## Protocol Confusion and Transport Security {#protocol-confusion-and-transport}

Mixing SAML trust inputs and OIDC trust inputs creates a risk of protocol
confusion. Implementations MUST validate ID Tokens using OIDC/JOSE key
discovery and MUST validate SAML signatures using SAML metadata.
Implementations MUST NOT assume that a key published in one metadata format
is automatically valid in the other.

The Bridge MUST apply transport security to its token endpoint, SAML
`SingleSignOnService` endpoint, SAML `SingleLogoutService` endpoint, and any
other endpoint used by this profile, in the same manner as the underlying
OAuth, OpenID Connect, and SAML specifications require.

The Bridge's SAML metadata URI and the OP's `jwks_uri` and discovery
endpoint MUST use the HTTPS scheme. The Bridge MUST NOT follow redirects
from HTTPS to non-HTTPS when fetching OP metadata or keys.

# Privacy Considerations {#privacy-considerations}

This profile concentrates identity data flowing from the OP and re-releases
it through SAML assertions. The Bridge sits between two trust contexts that
SPs and end-users would otherwise see as independent, and its design choices
affect the privacy properties of both.

Deployments SHOULD use per-SP pairwise NameID values to preserve subject
isolation across SPs. A pairwise NameID derived per
{{pairwise-nameid-construction}} ensures that an SP cannot correlate its
user population with that of another SP by comparing NameID values; the
derivation includes a Bridge-held salt that the SPs do not see.

The Bridge MUST NOT release more attributes to an SP than would have been
released in a direct SAML federation with an equivalent policy, unless a
separate policy explicitly authorizes broader disclosure. The attribute
release allowlist defined for each SP-Binding is the privacy-relevant
boundary, and the Bridge MUST NOT widen it implicitly because the ID Token
happens to carry additional claims.

The Bridge SHOULD NOT convey the SAML SP Entity ID to the OP (via
`client_id`, `acr_values`, or other authorization-request parameters)
unless local policy has determined that the OP is an appropriate audit
point for SP access. Doing so allows the OP to profile which SPs each user
accesses, which is a privacy violation in many deployment contexts.

When multiple Token Exchange clients share the same `saml_sp_entity_id`,
they will observe the same pairwise NameID and the same attribute release
for the same end-user. Deployments SHOULD grant this shared access only
when those clients are intended to represent the same SP relationship.

The `email`, `phone_number`, and similar OIDC contact claims carry user-
visible identifiers and are likely to be PII under most regulatory regimes.
Where the SP-Binding does not require them, the Bridge SHOULD omit them from
the assertion even if the ID Token includes them. The
`urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress` NameID format
exposes the email address as the subject identifier and carries email-
equivalent sensitivity.

Private claim mappings can introduce new correlation vectors across SPs.
Deployments SHOULD minimize use of private claims and SHOULD avoid emitting
values whose semantics were not already established for the same SP
relationship.

Bridge-side audit logs combine OIDC and SAML protocol identifiers (OP `sub`,
Bridge session ID, SP Entity ID, NameID) and so are uniquely positioned to
deanonymize pairwise NameIDs. The Bridge SHOULD retain such logs for the
minimum period required by deployment policy and SHOULD apply the access
controls appropriate for that combined data.

# IANA Considerations {#iana-considerations}

## OAuth Authorization Server Metadata {#iana-as-metadata}

This document requests registration of the following value in the OAuth
Authorization Server Metadata registry established by {{RFC8414}}. Per
{{OIDC-DISCOVERY}} Section 3, the same registry is authoritative for OpenID
Provider configuration metadata.

* Metadata Name: `oidc_op_issuer`
* Metadata Description: OIDC issuer identifier of the OpenID Provider that
  backs the Bridge SAML IdP facade
* Change Controller: IETF
* Specification Document(s): This document, {{metadata-oidc-op-issuer}}

The `saml_sp_entity_id` client metadata parameter and the
`saml_idp_entity_id` AS metadata parameter are defined by
{{SAML-OIDC-MIGRATION}} and reused here with consistent semantics; no
additional registration is required. The Token Exchange grant type
`urn:ietf:params:oauth:grant-type:token-exchange` and the token type URIs
`urn:ietf:params:oauth:token-type:id_token` and
`urn:ietf:params:oauth:token-type:saml2` are registered by {{RFC8693}};
this document does not modify those registrations.

--- back

# Examples

This appendix is non-normative. Examples use the `private_key_jwt` client
authentication method as defined by {{OIDC-CORE}}; token values and client
assertion JWTs are shortened placeholders. All examples assume a Bridge
with the following configuration:

* SAML IdP Entity ID: `https://bridge.example.com/saml/idp`
* OP issuer: `https://op.example.com`
* Bridge token endpoint: `https://bridge.example.com/token`

and an SP-Binding for a calendar application:

* SP Entity ID: `https://calendar.example.com/saml/sp`
* ACS URL: `https://calendar.example.com/saml/acs`
* NameID format: `urn:oasis:names:tc:SAML:2.0:nameid-format:persistent` (pairwise)
* Attribute release: `mail`, `displayName`, `givenName`, `sn`
* Authorized OP `client_id` values for the Token Exchange variant:
  `migration-tool`

## Bridge Authorization Server Metadata

A non-normative excerpt of the Bridge's authorization server metadata
published at
`https://bridge.example.com/.well-known/oauth-authorization-server`:

~~~ json
{
  "issuer": "https://bridge.example.com",
  "token_endpoint": "https://bridge.example.com/token",
  "grant_types_supported": [
    "urn:ietf:params:oauth:grant-type:token-exchange"
  ],
  "token_endpoint_auth_methods_supported": [
    "private_key_jwt",
    "tls_client_auth"
  ],
  "oidc_op_issuer": "https://op.example.com",
  "saml_idp_entity_id": "https://bridge.example.com/saml/idp"
}
~~~

## Browser-Based SAML SSO Flow

The calendar application initiates SAML SSO by redirecting Alice's browser to
the Bridge with an `AuthnRequest`. The Bridge has no existing OIDC session for
Alice and initiates an authorization code flow with the OP. After Alice
authenticates at the OP, the OP redirects back to the Bridge with an
authorization code. The Bridge exchanges the code for an ID Token, validates it,
derives Alice's pairwise NameID for the calendar SP, maps her claims to SAML
attributes, and posts a signed SAML `Response` to the calendar ACS.

The Bridge's outbound OIDC authorization request (HTTP-Redirect to the OP):

~~~ http
GET /authorize?response_type=code
  &client_id=bridge-client
  &redirect_uri=https%3A%2F%2Fbridge.example.com%2Fcallback
  &scope=openid%20profile%20email
  &state=abc123
  &nonce=xyz789
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256
  &claims=%7B%22id_token%22%3A%7B%22auth_time%22%3A%7B%22essential%22%3Atrue%7D%7D%7D
  HTTP/1.1
Host: op.example.com
~~~

After the OP authenticates Alice and returns an authorization code, the Bridge
exchanges it for tokens and receives an ID Token whose claims include:

~~~ json
{
  "iss": "https://op.example.com",
  "sub": "248289761001",
  "aud": "bridge-client",
  "exp": 1776808500,
  "iat": 1776804900,
  "auth_time": 1776794400,
  "nonce": "xyz789",
  "sid": "op-sid-abc123",
  "acr": "urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport",
  "email": "alice@example.com",
  "given_name": "Alice",
  "family_name": "Ng",
  "name": "Alice Ng"
}
~~~

The Bridge validates the ID Token, resolves Alice's Local Account, derives a
pairwise NameID for the calendar SP-Binding, and constructs the following SAML
`Assertion` (simplified, namespace declarations omitted):

~~~ xml
<saml:Assertion
    ID="_bridge-assertion-7f3a9c21"
    IssueInstant="2026-04-25T18:00:00Z"
    Version="2.0">
  <saml:Issuer>https://bridge.example.com/saml/idp</saml:Issuer>
  <ds:Signature>...</ds:Signature>
  <saml:Subject>
    <saml:NameID
        Format="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent"
        SPNameQualifier="https://calendar.example.com/saml/sp">
      _pairwise-8a3f21bc9d4e5f6a7b
    </saml:NameID>
    <saml:SubjectConfirmation
        Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
      <saml:SubjectConfirmationData
          Recipient="https://calendar.example.com/saml/acs"
          InResponseTo="_sp-authnrequest-8f3a"
          NotOnOrAfter="2026-04-25T18:05:00Z"/>
    </saml:SubjectConfirmation>
  </saml:Subject>
  <saml:Conditions
      NotBefore="2026-04-25T17:59:55Z"
      NotOnOrAfter="2026-04-25T18:05:00Z">
    <saml:AudienceRestriction>
      <saml:Audience>https://calendar.example.com/saml/sp</saml:Audience>
    </saml:AudienceRestriction>
    <saml:OneTimeUse/>
  </saml:Conditions>
  <saml:AuthnStatement
      AuthnInstant="2026-04-25T15:00:00Z"
      SessionIndex="op-sid-abc123">
    <saml:AuthnContext>
      <saml:AuthnContextClassRef>
        urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
      </saml:AuthnContextClassRef>
    </saml:AuthnContext>
  </saml:AuthnStatement>
  <saml:AttributeStatement>
    <saml:Attribute
        Name="urn:oid:0.9.2342.19200300.100.1.3"
        NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"
        FriendlyName="mail">
      <saml:AttributeValue>alice@example.com</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute
        Name="urn:oid:2.16.840.1.113730.3.1.241"
        NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"
        FriendlyName="displayName">
      <saml:AttributeValue>Alice Ng</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute
        Name="urn:oid:2.5.4.42"
        NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"
        FriendlyName="givenName">
      <saml:AttributeValue>Alice</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute
        Name="urn:oid:2.5.4.4"
        NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"
        FriendlyName="sn">
      <saml:AttributeValue>Ng</saml:AttributeValue>
    </saml:Attribute>
  </saml:AttributeStatement>
</saml:Assertion>
~~~

The Bridge wraps this assertion in a signed SAML `Response` and HTTP-POST binds
it to the calendar ACS at `https://calendar.example.com/saml/acs`.

## Token Exchange: ID Token to SAML Assertion

A migration tool, registered at the OP as `migration-tool` and at the
Bridge with `saml_sp_entity_id=https://calendar.example.com/saml/sp`,
has obtained an ID Token from the OP on behalf of Alice using a
permitted OIDC flow and needs to produce a SAML assertion for the
calendar SP. The decoded ID Token payload is:

~~~ json
{
  "iss": "https://op.example.com",
  "sub": "248289761001",
  "aud": "migration-tool",
  "exp": 1776808500,
  "iat": 1776804900,
  "jti": "op-id-token-7f3a",
  "auth_time": 1776794400,
  "acr": "urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport",
  "email": "alice@example.com",
  "given_name": "Alice",
  "family_name": "Ng",
  "name": "Alice Ng"
}
~~~

The migration tool authenticates and presents the ID Token to the
Bridge's Token Exchange endpoint (`subject_token` shown as a shortened
placeholder):

~~~ http
POST /token HTTP/1.1
Host: bridge.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token_type=urn:ietf:params:oauth:token-type:id_token
&subject_token=eyJhbGciOi...signature
&requested_token_type=urn:ietf:params:oauth:token-type:saml2
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=eyJhbGciOiJSUzI1NiJ9...
~~~

The Bridge authenticates the client, validates the ID Token, derives
Alice's pairwise NameID for the calendar SP-Binding, and returns:

~~~ json
{
  "issued_token_type": "urn:ietf:params:oauth:token-type:saml2",
  "access_token": "PHNhbWwyOkFzc2VydGlvbiBJRD0iX2JyaWRnZS1hc3NlcnRpb24tN2YzYTljMjEiLi4uPi4uLjwvc2FtbDI6QXNzZXJ0aW9uPg",
  "token_type": "N_A",
  "expires_in": 300
}
~~~

The `access_token` value is a base64url-encoded SAML assertion constructed
identically to the browser-based flow example, except that `InResponseTo` is
omitted from `SubjectConfirmationData` because no prior `AuthnRequest` was
issued.

# Paired Profile Summary

This appendix is non-normative.

The following table summarizes the correspondence between this profile and its
companion {{SAML-OIDC-MIGRATION}}.

| Dimension | {{SAML-OIDC-MIGRATION}} | This profile |
|---|---|---|
| Direction | SAML → OIDC/OAuth | OIDC → SAML |
| Input credential | SAML Assertion | OIDC ID Token |
| Output credential | Access token / ID Token / Refresh token | SAML Assertion |
| Authentication authority | SAML IdP | OpenID Provider (OP) |
| Translation component | OAuth AS / OpenID Provider | Bridge (SAML IdP facade) |
| Client binding parameter | `saml_sp_entity_id` (client metadata) | `saml_sp_entity_id` (client metadata at Bridge) |
| AS metadata: bound entity | `saml_idp_entity_id` (SAML IdP entity) | `saml_idp_entity_id` (Bridge's own SAML entity) |
| AS metadata: peer system | — | `oidc_op_issuer` (backing OP) |
| Subject input | SAML NameID → OIDC `sub` | OIDC `sub` → SAML NameID |
| Attribute direction | SAML attributes → OIDC claims | OIDC claims → SAML attributes |
| ACR direction | `AuthnContextClassRef` → `acr` | `acr` → `AuthnContextClassRef` |
| auth_time direction | `AuthnInstant` → `auth_time` (NumericDate) | `auth_time` (NumericDate) → `AuthnInstant` (dateTime) |
| Pairwise subject key | `saml_sp_entity_id` (OIDC Core §8 input) | SP Entity ID + salt (hash-based derivation) |
| Replay prevention | Assertion ID tracking at AS | ID Token `jti` tracking at Bridge; Assertion ID tracking for issued assertions |
| Logout: SP → IdP | SAML SLO → AS session revocation | SAML SLO → Bridge propagates to OP |
| Logout: IdP → SP | OIDC back-channel logout | Bridge propagates SAML SLO to SP |

