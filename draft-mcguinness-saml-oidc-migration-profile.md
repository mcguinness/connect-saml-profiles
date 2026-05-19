---
title: "SAML 2.0 to OpenID Connect and OAuth 2.0 Migration Profile 1.0"
abbrev: "SAML-OIDC Migration"
category: info

docname: draft-mcguinness-saml-oidc-migration-profile-latest
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
  RFC7662:
  RFC8174:
  RFC8176:
  RFC8414:
  RFC8693:
  RFC8707:
  RFC9493:
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

informative:
  RFC7009:
  RFC7522:
  I-D.ietf-oauth-identity-assertion-authz-grant:
  SAML2-IOP:
    title: "SAML V2.0 Metadata Interoperability Profile Version 1.0"
    target: "https://docs.oasis-open.org/security/saml/Post2.0/sstc-metadata-iop.html"
    author:
      - org: OASIS
    date: 2009
  INCOMMON-ATTRS:
    title: "InCommon Federation Attributes"
    target: "https://incommon.org/federation/attributes/"
    author:
      - org: InCommon
    date: false
  INCOMMON-OID:
    title: "InCommon Object Identifiers"
    target: "https://incommon.org/community/mace-registries/incommon-object-identifiers/"
    author:
      - org: InCommon
    date: false
  EDUPERSON:
    title: "eduPerson Object Class Specification"
    target: "https://refeds.org/specifications/eduperson"
    author:
      - org: REFEDS
    date: false
  REFEDS-RS:
    title: "REFEDS Research and Scholarship Entity Category"
    target: "https://refeds.org/category/research-and-scholarship"
    author:
      - org: REFEDS
    date: false

...

--- abstract

This document defines an interoperability profile for SAML 2.0 Service
Provider deployments that adopt OpenID Connect and OAuth 2.0 while
preserving the existing SAML federation trust relationship. The profile
extends that relationship rather than replacing it: client metadata binds
an OAuth client to an existing SAML SP entity identifier, authorization
server metadata binds an OAuth issuer to an existing SAML IdP entity
identifier, and mapping rules translate SAML subject identifiers,
authentication context, and attributes into OpenID Connect claims so
that the same end-user is represented by the same subject after the
transition.

Two operational patterns are defined. Token Exchange allows a client to
present a SAML assertion and receive an access token, refresh token, or
ID Token. A SAML assertion introspection extension allows a client to
validate a SAML assertion and receive normalized JSON claims without
issuing any OAuth token. Both patterns accept either a signed SAML
assertion or a signed SAML `Response` wrapper conveying exactly one
bearer assertion.

The intent is to let relying parties adopt OAuth 2.0 and OpenID Connect
without changing the underlying SSO trust model, without breaking
subject continuity, and without relinking existing user accounts. The
profile is scoped to federation interoperability and identity
continuity; it does not define authorization architectures,
provisioning, governance models, or generalized SAML feature parity.

--- middle

# Introduction {#introduction}

Many enterprise and SaaS deployments have a stable SAML 2.0 trust
relationship between an Identity Provider (IdP) and a Service Provider
(SP). When a relying party moves to OpenID Connect or OAuth 2.0, it
typically must create a new client registration, a new issuer
relationship, and a new subject identifier model. That break in
continuity disrupts the subject identifiers, claim release policy,
assurance signaling, and operational trust the deployment already
relies on.

This profile addresses federation interoperability and identity
continuity across the SAML-to-OIDC transition. The four scenarios
below illustrate where preserving the existing trust relationship
matters:

* **API access from a SAML-authenticated session.** An SP holds a SAML
  assertion and needs an OAuth access token to call APIs protected by an
  authorization server operated by the same authority as the SAML IdP. Token
  Exchange under this profile converts the existing assertion into an access
  token, refresh token, or ID Token without a separate interactive login.

* **Delegated SAML processing.** An application wants identity claims as
  normalized JSON without implementing SAML XML parsing or XML signature
  validation. The introspection pattern lets the authorization server absorb
  that complexity and return structured claims over a plain HTTPS endpoint,
  leaving the application with no SAML dependency.

* **Post-quantum readiness for relying parties.** SAML XML signature
  libraries are not well positioned for post-quantum cryptography
  migration. By submitting assertions to the authorization server and
  receiving JOSE-based tokens in return, a relying party can retire
  its XML processing stack and adopt PQ-ready JWT libraries without
  waiting for PQ support to land in SAML tooling. The authorization
  server and SAML IdP retain their SAML XML processing and migrate it
  on a coordinated timeline.

* **Non-disruptive migration.** An SP wants to adopt OAuth 2.0 and OpenID
  Connect without forcing end users to reauthenticate or requiring
  administrators to reconfigure existing identity bindings. This profile
  preserves the subject identifiers and claim release policy the SAML SP
  already relies on, enabling a gradual migration.

This profile:

* binds an OAuth client to an existing SAML SP Entity ID;
* binds an OAuth issuer to an existing SAML IdP Entity ID;
* defines how a SAML assertion can be exchanged for an OAuth 2.0 or OpenID
  Connect token;
* defines how a SAML assertion can be introspected when token issuance is not
  needed; and
* preserves subject and claim continuity across the migration.

What this profile does not define is enumerated in {{non-goals}}.

This profile extracts the migration-specific aspects of the approach defined in
the Identity Assertion JWT Authorization Grant
{{I-D.ietf-oauth-identity-assertion-authz-grant}} and defines them as a
standalone, reusable OAuth and OpenID Connect profile.

## Non-Goals {#non-goals}

The profile is scoped to federation interoperability and identity
continuity. It does NOT define:

* API authorization architectures, scope design, or resource-server
  authorization policy;
* entitlement systems or portable role/group authorization;
* provisioning, SCIM integration, or account-lifecycle workflows;
* runtime authorization policy expression or evaluation;
* governance models (approval, attestation, audit, segregation of
  duties);
* continuous access evaluation protocols (CAEP and similar);
* migration methodology (rollout patterns, cutover sequencing); or
* generalized SAML feature parity: SAML SLO, AuthzDecisionStatement,
  AttributeQuery, SP-specific extensions, and Holder-of-Key
  SubjectConfirmation are out of scope.

# Conventions and Terminology

## Requirements Notation

{::boilerplate bcp14-tagged}

## Terminology {#terminology}

This document uses the terms "authorization server", "client", "client
identifier", "issuer", and "scope" from OAuth 2.0, OAuth 2.0 Authorization
Server Metadata, Token Exchange, and OpenID Connect. It uses the terms
"Identity Provider", "Service Provider", "entityID", "Assertion", "Attribute",
"AttributeStatement", "AuthnStatement", and "AuthnContext" from SAML 2.0.

For the purposes of this document:

SAML IdP Entity ID:
: The SAML entityID of the asserting party that issues SAML assertions for the
  deployment.

SAML SP Entity ID:
: The SAML entityID of the relying party deployment that receives SAML
  assertions. In common SAML deployments, this is also the audience value used
  for that relying party.

Migration Client:
: An OAuth 2.0 or OpenID Connect client that has a `saml_sp_entity_id`
  registered and represents an existing SAML SP relationship. This document
  uses "client" as a shorthand for Migration Client when the context is clear.

Subject Continuity:
: The property that the same end-user is represented to the migrated relying
  party by a stable identifier that does not require account relinking.

Local Account:
: The authorization server's internal account record for the end-user to which
  the SAML assertion is linked for token issuance and claim release.

Stable Local Subject Key:
: A non-reassignable, unique local identifier for the resolved Local Account,
  used as input when this profile requires a derived subject value. It MUST be
  stable for the lifetime of the account, MUST be unique within the
  authorization server's namespace, and MUST NOT be a user-facing mutable
  attribute such as an email address, username, or display name. Suitable
  examples include an opaque internal account UUID or an LDAP `entryUUID`.

Authorization Basis:
: An explicit policy statement, established prior to processing of a SAML
  assertion, that authorizes a specific Migration Client to receive a specific
  output (token type, scope, target service, or claim set) for a resolved Local
  Account under a bound `saml_sp_entity_id`. An Authorization Basis MAY come
  from static configuration, administrative consent, a previously recorded
  authorization grant at the same authorization server, or another policy
  mechanism that is established before the SAML input is processed.

# Applicability and Deployment Model {#applicability-and-deployment-model}

This profile assumes that an administrative authority operates, or tightly
governs, both:

* a SAML 2.0 IdP identified by a SAML IdP Entity ID; and
* an OAuth 2.0 Authorization Server or OpenID Provider identified by an OAuth
  issuer.

The profile establishes three bindings:

* issuer binding: the OAuth issuer is bound to the SAML IdP Entity ID;
* client binding: the OAuth client is bound to the SAML SP Entity ID;
* subject and claim binding: OpenID Connect subject and claim values are bound
  to the identifiers and statements that were already used for the SAML SP.

When these bindings are present, the authorization server can preserve subject
format, claim release policy, and assurance semantics from the prior SAML
deployment instead of treating the migrated client as a wholly new relying
party.

This profile is intended for migrations in which the SAML IdP relationship
and the OAuth authorization server relationship are known to correspond to
the same relying-party deployment. It is not intended to establish trust
between otherwise unrelated SAML and OAuth parties.

The profile is applicable only when the authorization server can resolve
each assertion deterministically to exactly one active Local Account before
issuing tokens or returning an active introspection response. Resolution
MUST use either a previously persisted linkage for the trusted SAML
deployment or at least one stable, policy-accepted SAML subject input
present in the assertion.

The authorization server consumes either:

* a signed SAML `Assertion`; or
* a signed SAML `Response` containing exactly one bearer `Assertion`.

When a signed SAML `Response` wrapper is used, this profile requires the
authorization server to validate the `Status` element as defined in
{{saml-input-validation-and-binding}}. Other deployment-required response-level checks
(for example, `Destination` or `InResponseTo`) remain outside this profile
unless the deployment has separately established how they are performed.

This profile is defined only for confidential clients. Public clients MUST
NOT use the Token Exchange or direct SAML assertion introspection patterns
defined here.

This profile binds an OAuth issuer to a single SAML 2.0 IdP deployment.
That deployment is identified by `saml_idp_entity_id`, which may be
published at the authorization server metadata level as the default and
MAY be overridden per client registration when the SAML IdP issues
per-SP entityIDs (see {{client-registration-saml-idp-entity-id}}).
Deployments that need to bridge multiple SAML IdP deployments require
separate OAuth issuer instances.

# Migration Binding Rationale {#migration-binding-rationale}

This profile uses the SAML SP Entity ID as the primary migration key because
SAML pairwise subject identifiers, attribute release policy, and audience
restrictions are already typically bound to that value.

This profile uses Token Exchange {{RFC8693}} rather than the SAML 2.0 Bearer
Assertion Profile for OAuth 2.0 {{RFC7522}}. The SAML assertions exchanged
here are identity assertions, not authorization grants: they carry no OAuth
scopes, and their audience is the SAML SP Entity ID rather than the
authorization server. RFC 7522 requires the assertion to be addressed to the
authorization server with an OAuth authorization scope, which would break the
existing SAML trust relationship this profile is designed to preserve.

The authorization server and the SAML IdP are separate components, but they
operate within the same administrative trust boundary: the same organization
controls both. Token Exchange accepts third-party tokens as subject
credentials and allows the client to specify the desired output type via
`requested_token_type`, without requiring the SAML assertion to be reissued
for a different audience.

The primary guarantee against assertion forwarding is a three-way binding
among:

* the authenticated client identity;
* the client's registered authorization for a specific `saml_sp_entity_id`;
  and
* the assertion audience matching that identifier.

Together these ensure that only the Migration Client legitimately holding the
SAML SP relationship can exchange an assertion for that SP. The Migration
Client, acting as the SAML SP, is also responsible for validating SAML bearer
confirmation fields such as `Recipient` and `InResponseTo` as part of its own
front-channel SAML processing before submitting the assertion to the
authorization server.

An authorization server MAY support both this profile and {{RFC7522}} at
the same token endpoint. The `grant_type` parameter disambiguates them
without ambiguity: this profile uses
`urn:ietf:params:oauth:grant-type:token-exchange` with the SAML assertion
in `subject_token`, while RFC 7522 uses
`urn:ietf:params:oauth:grant-type:saml2-bearer` with the SAML assertion
in `assertion`. The two grant types have incompatible audience semantics
(this profile binds the assertion audience to the `saml_sp_entity_id`;
RFC 7522 binds it to the authorization server), so the routing decision
is determined entirely by `grant_type` and not by inspection of the
assertion. A single client registration MAY enable both grant types.

# Authorization and Consent Model {#authorization-and-consent-model}

The authorization server MUST NOT treat a valid SAML assertion as end-user
consent to issue OAuth tokens, release OpenID Connect claims, or return an
active introspection response. The assertion proves the authentication event and
supplies identity inputs; authorization remains an authorization server policy
decision.

Before issuing a token or returning an active introspection response, the
authorization server MUST have an explicit authorization basis. That basis:

* MUST be bound to the authenticated client, the `saml_sp_entity_id`, and the
  resolved Local Account;
* for Token Exchange, MUST be bound to the requested token type, requested
  scope, and, for access tokens, the resolved target service set; and
* for introspection, MUST authorize the authenticated client to validate the
  presented SAML input and receive any returned claims for the bound
  relying-party relationship.

The authorization basis MAY come from static configuration, administrative
consent, a previously recorded authorization grant at the same authorization
server, or another policy mechanism that is established before the SAML input is
processed.

The authorization server MUST NOT create, expand, or infer an authorization
grant solely from the presence, validity, or contents of the SAML assertion.
SAML attributes MAY be used as inputs to an already-established authorization
policy rule, but the policy rule itself MUST exist independently of the
assertion presented in the current request.

For requests involving refresh tokens, ID Tokens, OpenID Connect scopes, target
services, or claim release, the authorization server MUST apply the following
additional rules:

* if the requested scope includes `offline_access`, policy MUST authorize
  refresh token issuance to the authenticated client for the resolved Local
  Account and bound `saml_sp_entity_id`;
* a prior SAML SSO event by itself MUST NOT authorize refresh token issuance;
* if the requested token type is
  `urn:ietf:params:oauth:token-type:id_token`, or if an access token request
  includes `openid`, policy MUST authorize release of the resulting OpenID
  Connect subject and claims to the authenticated client under the bound
  `saml_sp_entity_id`;
* the authorization server MUST NOT release claims solely because corresponding
  SAML attributes are present in the assertion; and
* if the requested OAuth scope, OpenID Connect scope, target service, or claim
  set exceeds what the prior SAML relying-party relationship and current
  authorization server policy permit, the authorization server MUST either
  narrow the granted scope and returned claims to the permitted set or reject the
  request with `invalid_scope`, `invalid_target`, or `unauthorized_client`, as
  appropriate.

If end-user consent is required by local policy and no valid consent or
equivalent policy exists, the authorization server MUST reject the request with
`invalid_scope` or `unauthorized_client` according to local policy.

# Client Registration {#client-registration}

## `saml_sp_entity_id` {#client-registration-saml-sp-entity-id}

This document defines the `saml_sp_entity_id` client metadata parameter.

The `saml_sp_entity_id` value is a string that identifies the SAML 2.0 SP
Entity ID to which the OAuth client is bound.

When `saml_sp_entity_id` is present:

* its value MUST exactly match the SAML SP Entity ID used in the existing SAML
  deployment;
* the authorization server MUST bind the client registration to that SAML SP
  relationship for purposes of subject continuity and claim release policy;
* the authorization server MAY allow multiple OAuth clients to share the same
  `saml_sp_entity_id` when local policy determines that those clients represent
  the same historical relying-party relationship;
* the authorization server MUST reject registration if the value is unknown,
  disallowed, or local policy does not authorize the client to use that
  binding; and
* the authorization server MUST ensure that the client authenticated at runtime
  is one of the clients authorized to use that `saml_sp_entity_id`.

An authorization server MAY support `saml_sp_entity_id` for dynamic client
registration, static client registration, or both. For statically registered
clients, the `saml_sp_entity_id` binding MAY be established through
administrative configuration rather than a client registration request.

Because this profile is defined only for confidential clients, an authorization
server MUST NOT enable this profile for a public client registration.

### Authorizing the `saml_sp_entity_id` Binding {#client-registration-saml-sp-entity-id-authorization}

The `saml_sp_entity_id` binding is security-critical. It determines which
SAML assertions the client may exchange and which subject continuity and
claim release relationships the client inherits from the existing SAML
deployment. Establishing this binding through an unauthenticated or
self-asserted client registration request is incompatible with this profile.

For statically registered clients, the binding MUST be established through
administrative configuration controlled by the authority that operates or
governs the SAML deployment.

For clients registered through dynamic client registration {{RFC7591}}, the
authorization server MUST require at least one of the following before
accepting a registration that sets `saml_sp_entity_id`:

* a software statement {{RFC7591}} signed by a trust anchor administratively
  associated with the bound `saml_sp_entity_id`, where the software
  statement asserts the requesting party's entitlement to that value;
* a registration credential (such as an initial access token, mTLS client
  certificate, or equivalent) administratively bound to the permitted
  `saml_sp_entity_id` value or to a set of values that includes the
  requested one; or
* an equivalent administrative authorization established before the
  registration request.

The authorization server MUST NOT accept a dynamic registration request
that sets `saml_sp_entity_id` solely on the basis of the request's own
contents. Anonymous or open dynamic client registration MUST NOT be enabled
for clients that bind a `saml_sp_entity_id` under this profile.

The same authorization rules apply to subsequent client configuration
updates: a request that adds, removes, or changes `saml_sp_entity_id` MUST
satisfy the same authorization requirements as the original registration.

When multiple OAuth clients share the same `saml_sp_entity_id`, the
authorization server MUST treat them as the same relying-party context for
claim release and subject continuity under this profile.

If multiple OAuth clients sharing the same `saml_sp_entity_id` present
conflicting `subject_type` values, the authorization server MUST reject the
registration or runtime request that would create inconsistent subject
semantics.

Clients SHOULD register `subject_type` explicitly when using this profile.
When `subject_type` is not registered but `saml_sp_entity_id` is present, the
authorization server MUST apply `public` as the effective subject type. This
default aligns with OpenID Connect Core behavior when `subject_type` is not
registered; deployments that previously relied on SAML SP-specific (pairwise)
identifiers SHOULD register `subject_type=pairwise` explicitly to preserve that
behavior. When `subject_type` is `pairwise` and `saml_sp_entity_id` is present,
the authorization server MUST use the `saml_sp_entity_id` as the relying-party
input for pairwise subject continuity.

When `saml_sp_entity_id` is present, a registered `sector_identifier_uri`
{{OIDC-REGISTRATION}} MUST NOT alter pairwise subject calculation under this
profile. If a client registration includes both `saml_sp_entity_id` and
`sector_identifier_uri`, the authorization server MUST either ensure that the
`sector_identifier_uri` represents the same historical relying-party context
as the bound `saml_sp_entity_id` or reject the registration or request as
inconsistent.

## `saml_idp_entity_id` {#client-registration-saml-idp-entity-id}

This document also defines `saml_idp_entity_id` as an OPTIONAL client
metadata parameter. It carries the same syntax and semantics as the
authorization server metadata parameter of the same name
({{metadata-saml-idp-entity-id}}), but applies to a single client
registration.

Per-client `saml_idp_entity_id` exists for SAML deployments in which the
IdP issues per-SP entityIDs rather than a single global entityID. Common
examples include multi-tenant SaaS IdPs that emit per-tenant or
per-relationship entityIDs, federation hubs that present different
entityIDs to different SPs, and enterprise IdPs configured for per-SP
audit or compliance isolation. In such deployments, the assertion
`Issuer` for the relying-party relationship represented by
`saml_sp_entity_id` is different from the authorization server's
default `saml_idp_entity_id` and is specific to that historical SP-IdP
pair.

When `saml_idp_entity_id` is present in a client registration, it
overrides the authorization server metadata value for assertions
submitted by the authenticated client. The authorization server MUST
validate the SAML Assertion Issuer (and, when a `Response` wrapper is
submitted, the SAML Response Issuer) against the effective
`saml_idp_entity_id` for the authenticated client: the client-level
value if present, otherwise the authorization server metadata value.

The per-client `saml_idp_entity_id` binding is security-critical for the
same reasons as `saml_sp_entity_id`: it determines which SAML Issuer the
client may rely on for assertion validation. The authorization rules in
{{client-registration-saml-sp-entity-id-authorization}} apply equally
to the per-client `saml_idp_entity_id` parameter. Anonymous or open
dynamic client registration MUST NOT be permitted to set or change this
value.

Migration Clients learn their effective `saml_idp_entity_id` through the
registration mechanism by which they were established: statically
registered clients receive the value through administrative
configuration; dynamically registered clients receive it in the
registration response defined by {{RFC7591}}; and clients with access
to a Dynamic Client Registration Management endpoint can refresh their
view through RFC 7592. At runtime, a Migration Client also observes the
SAML IdP's effective entityID directly from the `Issuer` element of
the SAML Response or Assertion it receives, which the authorization
server then validates against the registered value. A mismatch produces
an `invalid_request` rejection (see {{token-exchange-error-response}}),
which clients can use to detect drift between their registration and
the IdP's current configuration. This profile does not define any other
discovery mechanism, and in particular does not expose per-client
`saml_idp_entity_id` values through authorization server metadata.

## `saml_metadata_uri` {#client-registration-saml-metadata-uri}

This document also defines `saml_metadata_uri` as an OPTIONAL client
metadata parameter. It carries the same syntax and semantics as the
authorization server metadata parameter of the same name
({{metadata-saml-metadata-uri}}), but applies to a single client
registration.

Per-client `saml_metadata_uri` is intended for deployments that use
per-client `saml_idp_entity_id`
({{client-registration-saml-idp-entity-id}}) and exchange SAML metadata
on a per-relationship basis. When registered, the URI MUST resolve to a
SAML metadata document whose `EntityDescriptor` `entityID` exactly
matches the client's effective `saml_idp_entity_id`. The validation
rules in {{metadata-saml-metadata-uri}} (HTTPS scheme requirement,
EntityDescriptor verification, signature handling) apply unchanged.

When `saml_metadata_uri` is present in a client registration, it
overrides the authorization server metadata value for purposes of SAML
key material and other IdP metadata used to validate assertions
submitted by this client.

# Authorization Server Metadata {#authorization-server-metadata}

The metadata parameters defined in this section are registered in the
IANA "OAuth Authorization Server Metadata" registry established by
{{RFC8414}}. Per {{OIDC-DISCOVERY}} Section 3, where OAuth 2.0
Authorization Server Metadata and OpenID Provider configuration
overlap, that same IANA registry is authoritative for both. The
metadata parameters defined here are therefore equally valid in:

* OAuth 2.0 Authorization Server Metadata documents published at the
  `.well-known/oauth-authorization-server` endpoint defined by
  {{RFC8414}}; and
* OpenID Provider configuration documents published at the
  `.well-known/openid-configuration` endpoint defined by
  {{OIDC-DISCOVERY}}.

An authorization server that is also an OpenID Provider SHOULD publish
the applicable parameters in whichever discovery document(s) its
clients consume. Their semantics and processing are identical in
either context.

## `saml_idp_entity_id` {#metadata-saml-idp-entity-id}

This document defines the `saml_idp_entity_id` authorization server metadata
parameter.

The `saml_idp_entity_id` value is a string that identifies the SAML 2.0 IdP
Entity ID bound to the OAuth issuer.

When `saml_idp_entity_id` is present:

* it MUST equal a SAML IdP Entity ID used by the corresponding SAML IdP;
* it MUST identify a SAML IdP that is operated by or under the administrative
  control of the same entity that operates the OAuth authorization server
  identified by this OAuth issuer;
* it is the default value against which incoming SAML assertion issuers are
  validated when no per-client `saml_idp_entity_id`
  ({{client-registration-saml-idp-entity-id}}) is registered for the
  authenticated client.

In deployments where every Migration Client registers a per-client
`saml_idp_entity_id` override, the authorization server metadata value
MAY be omitted. The authorization server MUST nevertheless ensure, for
every authenticated client, that an effective `saml_idp_entity_id` is
configured before consuming SAML input from that client; absence of both
the AS-level and client-level values for an authenticated client MUST
cause the SAML input to be rejected.

The OpenID Connect `iss` claim and OAuth authorization server `issuer` metadata
value remain OAuth issuer identifiers. They are not required to equal any
SAML IdP Entity ID, and they commonly will not, because OAuth issuer values
are HTTPS URLs with OAuth-specific discovery semantics.

## `saml_metadata_uri` {#metadata-saml-metadata-uri}

This document defines the `saml_metadata_uri` authorization server metadata
parameter.

The `saml_metadata_uri` value is an HTTPS URI that resolves to the SAML
metadata document describing the default SAML IdP entity bound to the OAuth
issuer. When per-client `saml_idp_entity_id` overrides are in use, this
authorization server metadata value is the default; clients MAY register a
per-client `saml_metadata_uri` override
({{client-registration-saml-metadata-uri}}) for SAML metadata describing
their specific SP-IdP relationship.

When `saml_metadata_uri` is present:

* the referenced SAML metadata MUST describe the effective
  `saml_idp_entity_id` for the same context (the authorization server
  metadata `saml_idp_entity_id` for the AS-level URI, or the client-level
  `saml_idp_entity_id` for a per-client URI);
* any implementation that fetches `saml_metadata_uri` MUST verify that the
  retrieved document is a well-formed SAML metadata document containing an
  `EntityDescriptor` whose `entityID` attribute exactly matches the
  effective `saml_idp_entity_id` for the same context; if this validation
  fails, the retrieved document MUST NOT be used;
* if the retrieved SAML metadata document includes an XML signature,
  implementations SHOULD validate that signature against a pre-configured
  trust anchor before using the document;
* the authorization server SHOULD make the metadata available in a form suitable
  for standard SAML metadata processing;
* clients and migration tooling MAY use the metadata to correlate the OAuth
  issuer with the pre-existing SAML trust configuration.

The SAML metadata document referenced by `saml_metadata_uri` SHOULD include
an `AdditionalMetadataLocation` element pointing at the authorization
server metadata document for the bound OAuth issuer, with the `namespace`
attribute set to a URI identifying the relevant metadata format (for
example, `https://datatracker.ietf.org/doc/html/rfc8414` for OAuth
authorization server metadata, or
`https://openid.net/specs/openid-connect-discovery-1_0.html` for OpenID
Provider configuration). This forms a bidirectional binding between the
SAML IdP entity and the OAuth issuer, both of which can be verified
through their respective protocol metadata.

## `token_exchange_requested_token_types_supported` {#metadata-token-exchange-requested-token-types-supported}

This document defines the `token_exchange_requested_token_types_supported`
authorization server metadata parameter.

The `token_exchange_requested_token_types_supported` value is a JSON array of
strings containing the token type URIs that the authorization server accepts as
`requested_token_type` values for SAML token exchange under this profile.

If the authorization server publishes metadata and supports Token Exchange
under this profile, it SHOULD include this metadata value.

When `token_exchange_requested_token_types_supported` is present:

* each value MUST be one of
  `urn:ietf:params:oauth:token-type:refresh_token`,
  `urn:ietf:params:oauth:token-type:id_token`, or
  `urn:ietf:params:oauth:token-type:access_token`;
* if `urn:ietf:params:oauth:token-type:id_token` is included, the
  authorization server MUST be an OpenID Provider; and
* if `urn:ietf:params:oauth:token-type:access_token` is included, the
  authorization server MUST support the target-selection processing defined by
  the Token Exchange access token rules for access token requests.

This profile intentionally limits `requested_token_type` to these three values.
Other token type URIs defined by {{RFC8693}}, such as
`urn:ietf:params:oauth:token-type:jwt`, are outside the scope of this profile.

## `introspection_token_types_supported` {#metadata-introspection-token-types-supported}

This document defines the `introspection_token_types_supported`
authorization server metadata parameter.

The `introspection_token_types_supported` value is a JSON array of strings.
Each string is a token type URI identifying a token format that the
authorization server accepts for direct introspection at its introspection
endpoint according to {{saml-assertion-introspection}}.

When `introspection_token_types_supported` includes
`urn:ietf:params:oauth:token-type:saml2`:

* the authorization server's `introspection_endpoint` metadata MUST be present;
* the introspection endpoint MUST accept the request pattern defined in
  {{saml-assertion-introspection}};
* the authorization server SHOULD accept
  `token_type_hint=urn:ietf:params:oauth:token-type:saml2`.

## Capability and Metadata Matrix {#capability-and-metadata-matrix}

The following table summarizes the authorization server metadata used to signal
capabilities defined by this profile.

| Capability | Required metadata | Conditional metadata | Notes |
| --- | --- | --- | --- |
| SAML issuer binding (AS-level default) | `saml_idp_entity_id` when no client uses a per-client override | `saml_metadata_uri` | `saml_idp_entity_id` identifies the default SAML IdP bound to the OAuth issuer. `saml_metadata_uri` is optional but, when present, MUST describe the effective `saml_idp_entity_id` for the same context. |
| SAML issuer binding (per-client override) | Client-level `saml_idp_entity_id` for deployments where the IdP emits per-SP entityIDs | Client-level `saml_metadata_uri` | Per-client overrides allow each Migration Client to bind a distinct SAML IdP entityID and metadata document; see {{client-registration-saml-idp-entity-id}} and {{client-registration-saml-metadata-uri}}. |
| Token Exchange with SAML input | `grant_types_supported` containing `urn:ietf:params:oauth:grant-type:token-exchange` when `grant_types_supported` is published | `token_exchange_requested_token_types_supported` | `token_exchange_requested_token_types_supported` SHOULD be published when Token Exchange is supported under this profile. |
| Token Exchange issuing ID Tokens | `token_exchange_requested_token_types_supported` containing `urn:ietf:params:oauth:token-type:id_token` | OpenID Connect Discovery metadata required by {{OIDC-DISCOVERY}} | The authorization server MUST be an OpenID Provider. |
| Token Exchange issuing access tokens | `token_exchange_requested_token_types_supported` containing `urn:ietf:params:oauth:token-type:access_token` | `resource` and `audience` support according to local policy | The authorization server MUST support target-selection processing for access token requests. |
| Direct SAML assertion introspection | `introspection_endpoint`; `introspection_token_types_supported` containing `urn:ietf:params:oauth:token-type:saml2` | None | The introspection endpoint MUST accept the request pattern defined by {{saml-assertion-introspection}}. |
| SAML Response wrapper input | No separate metadata parameter | Capability is implied only for protocol patterns where the authorization server accepts such input | Support for a signed SAML `Response` wrapper is optional. If supported, the rules in {{response-wrapper-processing}} apply. |

Omission of `token_exchange_requested_token_types_supported` or
`introspection_token_types_supported` does not imply support for the
corresponding capability. A client or migration tool MUST NOT assume support for
Token Exchange requested token types or direct SAML assertion introspection
unless the relevant metadata or an equivalent administrative configuration
indicates that support.

## SAML Key Material {#saml-key-material}

This document does not define an authorization server metadata parameter for
SAML signing keys or encryption keys. That omission is intentional.

SAML keys are published and processed through SAML metadata and `KeyDescriptor`
elements, while OAuth and OpenID Connect signing keys are published and
processed through `jwks_uri` or `jwks`. Even when the same underlying key
material is reused, it MUST be published and validated using the metadata and
encoding rules of each protocol separately.

## Federation Metadata Coexistence {#federation-metadata-coexistence}

A deployment migrating from SAML 2.0 to this profile typically operates
both protocols in parallel for some period. The two protocols publish
metadata through independent mechanisms that this profile does not
unify: SAML 2.0 metadata as `EntityDescriptor` documents
({{SAML2-METADATA}}), OAuth/OIDC metadata at the
`.well-known/oauth-authorization-server` and
`.well-known/openid-configuration` endpoints ({{RFC8414}},
{{OIDC-DISCOVERY}}). Both can be published from the same authority
simultaneously.

Deployments SHOULD observe the following during parallel operation:

* The SAML IdP `entityID` and the OAuth `issuer` identify the same
  administrative authority but remain distinct values used in their
  own protocol contexts; see {{saml-and-oauth-issuer-boundary}} for
  the validation rule, including incidental string equality.
* SAML signing keys (published via `KeyDescriptor`) and OAuth/OIDC
  signing keys (published via `jwks_uri`) rotate on independent
  schedules. A SAML-side key rollover does not affect ID Token
  signature validation, and a JWS-side rollover does not affect SAML
  assertion signature validation. Deployments SHOULD coordinate
  rollover windows to avoid cross-protocol confusion in operational
  tooling.
* The bidirectional metadata pointer described in
  {{metadata-saml-metadata-uri}} (SAML metadata
  `AdditionalMetadataLocation` referencing the OAuth/OIDC metadata
  document, and AS metadata `saml_metadata_uri` referencing the SAML
  metadata document) is OPTIONAL but RECOMMENDED for discovery tooling
  during the migration period. It allows administrators and federation
  registries to traverse from either side to the other.
* Discovery clients that consume SAML metadata to locate the IdP, and
  separately consume OAuth/OIDC metadata to locate the authorization
  server, are unchanged. This profile does not alter the discovery
  flow for either protocol.

This profile does not define a mechanism for retiring the SAML side of
a deployment, nor for archiving SAML metadata after a migration
completes. Those operational decisions are outside the profile.

# SAML Input Validation and Binding {#saml-input-validation-and-binding}

This section applies whenever the authorization server directly consumes SAML
input under this profile.

## Accepted SAML Inputs {#accepted-saml-inputs}

This profile uses an assertion-centric processing model: the effective security
token is always one SAML `Assertion`. The submitted SAML input MUST be either:

* a signed SAML `Assertion`; or
* a signed SAML `Response` containing exactly one bearer `Assertion`. When a
  `Response` wrapper is submitted, the authorization server extracts the
  enclosed assertion and treats it as the effective assertion.

The SAML input MUST contain exactly one unencrypted SAML 2.0 bearer assertion.
Encrypted assertions, SAML protocol messages other than `Response`, and
assertions using subject confirmation methods other than bearer are out of
scope.

The token type identifier `urn:ietf:params:oauth:token-type:saml2` identifies
both input forms. The `Response` wrapper is a transport convenience that the
Migration Client may forward directly from the SAML SP exchange; it carries no
independent authorization semantics under this profile.

## Response Wrapper Processing {#response-wrapper-processing}

If a signed SAML `Response` is submitted, the authorization server:

* MAY use the response signature to establish integrity and SAML Response Issuer
  authenticity for the enclosed `Assertion` when that `Assertion` is not
  independently signed;
* MUST NOT treat successful validation of a SAML `Response` signature as proof
  that response-level fields such as `Destination`, `Consent`, or
  `Response/@InResponseTo` have been validated;
* MUST NOT allow response-level fields by themselves to determine subject
  continuity, claim release, or target binding under this profile;
* MUST verify that the top-level `StatusCode` value is
  `urn:oasis:names:tc:SAML:2.0:status:Success`;
* MUST reject the response if a subordinate `StatusCode` element is present; and
* MAY ignore `StatusMessage` and `StatusDetail` except for logging or
  diagnostics.

This profile does not define full response-level protocol processing.
Deployments that require additional response-level checks MUST perform them
outside this profile before or during extraction of the effective `Assertion`.
Such additional checks SHOULD include, at minimum, verification that
`Response/@Destination` matches the intended endpoint, validation that
`Response/@InResponseTo`, if present, corresponds to a previously issued
`<AuthnRequest>` ID, and confirmation that the `Response/@ID` value has not been
replayed.

## Encrypted Content {#encrypted-content}

This profile does not define processing of encrypted SAML content. The
authorization server MUST reject any submitted SAML input that is itself an
`EncryptedAssertion`, and MUST reject any effective assertion containing
`EncryptedID` or `EncryptedAttribute` elements.

In SAML deployments that require encryption between the IdP and SP, the
Migration Client, acting as the SAML SP, is responsible for decrypting the
SAML input using the SP's decryption key before submitting it under this
profile. The plaintext SAML input the Migration Client submits MUST satisfy
the signature requirements in {{saml-input-validation-and-binding}}. In practice, this
means the IdP MUST sign the inner `Assertion` independently, since a
signature on the enclosing `Response` element does not survive decryption
of an `EncryptedAssertion` it contains.

The introspection pattern does not eliminate the SAML SP-side decryption
key in encryption-required deployments, because that key is held by the
SAML SP and not by the authorization server.

## SAML and OAuth Issuer Boundary {#saml-and-oauth-issuer-boundary}

This profile uses distinct issuer identifiers from different protocol contexts:

* SAML Response Issuer: the value of the SAML `Response/Issuer` element, when a
  SAML `Response` wrapper is submitted;
* SAML Assertion Issuer: the value of the effective SAML `Assertion/Issuer`
  element;
* Trusted SAML IdP Entity ID: the effective `saml_idp_entity_id` for the
  authenticated client, which is the client-level `saml_idp_entity_id`
  ({{client-registration-saml-idp-entity-id}}) when registered, otherwise
  the authorization server metadata value
  ({{metadata-saml-idp-entity-id}}); and
* OAuth issuer identifier: the authorization server issuer value used in OAuth
  authorization server metadata and as the OpenID Connect ID Token `iss` claim.

The SAML Assertion Issuer MUST match the Trusted SAML IdP Entity ID. When a SAML
`Response` wrapper is submitted, the SAML Response Issuer MUST also match the
Trusted SAML IdP Entity ID.

The OAuth issuer identifier and the Trusted SAML IdP Entity ID identify related
but distinct protocol issuers. The `iss` claim in an ID Token issued under this
profile MUST be the OAuth issuer identifier, not the SAML Assertion Issuer or
SAML Response Issuer.

The authorization server MUST NOT copy a SAML issuer value into the ID Token
`iss` claim or use a SAML issuer value as a substitute for OAuth issuer
metadata.

The OAuth issuer identifier and the Trusted SAML IdP Entity ID MAY have
identical string values when a deployment co-locates the SAML IdP and the
OAuth authorization server. Such equality is incidental: each value is
still used in its own protocol context, and validation still uses each
value through its own metadata and signature path.

## Signature, SAML Issuer, and Audience Binding {#signature-saml-issuer-and-audience-binding}

The authorization server MUST:

1. determine the effective `Assertion` to be processed:
   * if the submitted SAML input is an `Assertion`, that `Assertion` is the
     effective assertion and it MUST be signed;
   * if the submitted SAML input is a `Response`, that `Response` MUST be
     signed, it MUST contain exactly one enclosed `Assertion`, and that
     enclosed `Assertion` becomes the effective assertion for the remaining
     rules in this section;
2. validate the signed SAML element or elements using SAML metadata and SAML
   key material, not JOSE metadata;
3. if the submitted SAML input is a `Response`, verify that:
   * the SAML `Response/Issuer` value matches the Trusted SAML IdP Entity ID
     defined in {{saml-and-oauth-issuer-boundary}};
   * the top-level `Response/Status/StatusCode/@Value` is
     `urn:oasis:names:tc:SAML:2.0:status:Success`; and
   * no nested `Response/Status/StatusCode/StatusCode` element is present;
4. validate the effective `Assertion` according to the assertion
   processing rules in {{SAML2-CORE}} Section 2, including
   `Assertion/Issuer` ({{SAML2-CORE}} Section 2.3.3), subject and
   subject confirmation ({{SAML2-CORE}} Sections 2.4-2.4.1.3), and
   conditions ({{SAML2-CORE}} Section 2.5), enforcing both
   `Conditions/@NotBefore` and `Conditions/@NotOnOrAfter`;
5. verify that the effective SAML `Assertion/Issuer` value matches the Trusted
   SAML IdP Entity ID defined in {{saml-and-oauth-issuer-boundary}};
6. verify that the effective `Assertion` contains at least one
   `AudienceRestriction` element;
7. verify that the bound `saml_sp_entity_id` appears as an `Audience`
   value in every `AudienceRestriction` element in the effective
   assertion, since multiple `AudienceRestriction` elements are
   conjunctive under {{SAML2-CORE}} Section 2.5.1.4 (each restriction
   must be satisfied independently). Additional `Audience` values within
   each `AudienceRestriction` are permitted and MUST NOT by themselves
   invalidate the assertion; and
8. reject the SAML input if the authenticated client, the
   `saml_sp_entity_id` registered to that client, and the validated
   `Audience` matching that `saml_sp_entity_id` do not all align, as
   required by the three-way binding in {{migration-binding-rationale}}.

An authorization server consuming SAML input under this profile MUST reject any
encrypted assertion, any submitted SAML input that is neither an `Assertion` nor
a `Response` as defined by this profile, or any assertion using a subject
confirmation method outside this profile. Endpoint-specific error behavior is
defined by {{token-exchange-error-response}} and {{introspection-error-response}}.

## Bearer Subject Confirmation Data Validation {#bearer-subject-confirmation-data-validation}

The authorization server MUST identify at least one usable `SubjectConfirmation`
element with the bearer method URI (`urn:oasis:names:tc:SAML:2.0:cm:bearer`).
If multiple bearer `SubjectConfirmation` elements are present, any one valid
bearer confirmation is sufficient. For a usable bearer `SubjectConfirmationData`:

* `NotOnOrAfter`, if present, MUST be in the future at the time of processing;
* `Recipient`, if present, MUST have been validated by the Migration Client as
  part of its SAML SP processing before the assertion is submitted. The
  authorization server SHOULD additionally verify that `Recipient` is
  consistent with the ACS URLs registered for the bound `saml_sp_entity_id` as
  a defense-in-depth measure, and MUST reject the assertion if `Recipient`
  identifies the authorization server's own token endpoint or introspection
  endpoint;
* `InResponseTo`, if present, MUST be validated either by the component that
  handled the front-channel SAML exchange or by equivalent state known to the
  authorization server;
* if `InResponseTo` is absent, the assertion MAY still be accepted, except
  that deployments requiring SP-initiated request correlation for the bound
  SAML relationship MUST reject such assertions;
* `Address`, if present, MUST be validated either by the component that
  handled the front-channel SAML exchange or by deployment-specific means
  available to the authorization server; and
* if any present value cannot be validated under the preceding rules, the
  authorization server MUST reject the assertion as unusable under this
  profile.

The primary forwarding-prevention guarantee in this profile is the three-way
binding defined in {{migration-binding-rationale}}, not `Recipient`
validation.

## Replay and Freshness {#replay-and-freshness}

The authorization server MUST enforce any applicable SAML one-time-use,
replay-prevention, assertion freshness, and proxying restrictions associated
with the trusted SAML deployment.

At minimum, the authorization server MUST detect reuse of the same
`Assertion/@ID` value from the same SAML Assertion Issuer until the
assertion's validity window or local freshness window has expired. Whether
reuse within that window is accepted, rejected, or further constrained by the
authenticated client and bound `saml_sp_entity_id` is a deployment policy
decision; a stricter rule, where stated below, takes precedence.

The replay-detection state MUST be shared across all endpoints that consume
SAML input under this profile; an `Assertion/@ID` consumed at the Token
Exchange endpoint MUST be visible to replay detection at the introspection
endpoint, and vice versa.

If the SAML assertion's `Conditions` element includes a `OneTimeUse`
condition, the authorization server MUST treat the assertion as exhausted
immediately after its first successful use and MUST reject any subsequent
submission of the same assertion, even within its validity window.

An active introspection response or successful Token Exchange response does
not, by itself, require the assertion to be treated as exhausted unless local
replay policy or SAML `OneTimeUse` semantics require that result.

A client retry of a transient-failure response (network error, timeout,
authorization server overload) using the same SAML assertion is not a
replay attack. Authorization servers SHOULD distinguish idempotent
retries from replay attempts -- for example, by recognizing that the
same authenticated client is submitting the same `Assertion/@ID`
within a short window after a non-success response, and returning the
same outcome rather than rejecting as replay. Deployments enforcing
SAML `OneTimeUse` cannot offer this accommodation, since `OneTimeUse`
consumes the assertion on first successful use regardless of client
intent.

Authorization servers SHOULD permit a limited clock skew tolerance when
evaluating time-based conditions such as `Conditions/@NotBefore`,
`Conditions/@NotOnOrAfter`, and `SubjectConfirmationData/@NotOnOrAfter`. Any
clock skew allowance MUST be consistent with local security policy and SHOULD
NOT exceed five minutes, in accordance with {{SAML2-CORE}} Section 2.5.1.2.

When an `AuthnStatement` is present, the authorization server SHOULD verify
that the time elapsed since `AuthnStatement/@AuthnInstant` does not exceed a
deployment-configured freshness window. In the absence of a stricter local
policy, a maximum freshness window of 8 hours is RECOMMENDED, consistent with
typical enterprise SSO session durations. This assertion-age check is
independent of `Conditions/@NotOnOrAfter` and `SessionNotOnOrAfter`.

Client authentication does not replace SAML assertion validation. A valid
client cannot cause an invalid or misbound SAML assertion to become
acceptable.

# Token Exchange Using a SAML Assertion {#token-exchange-using-a-saml-assertion}

This section defines how a client can use OAuth 2.0 Token Exchange to obtain
OAuth 2.0 or OpenID Connect tokens from a SAML 2.0 assertion associated with an
existing SAML SP relationship.

An authorization server supporting this pattern MUST support Token Exchange as
defined by {{RFC8693}}. If the authorization server publishes metadata, its
`grant_types_supported` metadata SHOULD include
`urn:ietf:params:oauth:grant-type:token-exchange`.

An authorization server that supports
`requested_token_type=urn:ietf:params:oauth:token-type:id_token` under this
profile MUST also be an OpenID Provider and MUST comply with {{OIDC-CORE}} for
the ID Tokens that it issues.

Under this pattern, the Migration Client authenticates the user through the
existing SAML 2.0 deployment, receives a SAML assertion for the SAML SP
identified by `saml_sp_entity_id`, and presents that assertion, or a signed
SAML `Response` that conveys it, to the authorization server token endpoint
using Token Exchange. The issued token is determined by the
`requested_token_type` parameter and authorization server policy. This profile
defines issuance of refresh tokens, access tokens, and ID Tokens.

## Request {#token-exchange-request}

The client makes a Token Exchange request to the token endpoint using the
parameters described below.

The following non-normative example requests an ID Token from a SAML assertion:

~~~ http
POST /token HTTP/1.1
Host: as.example.edu
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Asaml2
&subject_token=PHNhbWxwOlJlc3BvbnNlLi4u
&requested_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aid_token
&scope=openid%20profile%20email
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer
&client_assertion=eyJhbGciOiJSUzI1NiIs...
~~~

The request parameters are summarized below:

| Parameter | Requirement | Applies To | Summary |
|---|---:|---|---|
| `grant_type` | REQUIRED | all requests | Identifies the request as OAuth 2.0 Token Exchange. |
| `subject_token` | REQUIRED | all requests | Carries the base64url-encoded SAML `Assertion` or SAML `Response` wrapper. |
| `subject_token_type` | REQUIRED | all requests | Identifies the effective subject token as SAML 2.0. |
| `requested_token_type` | REQUIRED | all requests | Selects refresh token, ID Token, or access token issuance. |
| `scope` | Conditional | all requests | Required for refresh token and ID Token requests; optional for access token requests. |
| `resource` | OPTIONAL | access token requests | Identifies resource URI targets using {{RFC8707}}. |
| `audience` | OPTIONAL | access token requests | Identifies logical target services using {{RFC8693}}. |

`grant_type`:

REQUIRED. The value `urn:ietf:params:oauth:grant-type:token-exchange`.

`subject_token`:

REQUIRED. The base64url-encoded octet sequence, as defined in Section 5 of
{{RFC4648}}, of the XML serialization of either:

* a signed SAML 2.0 `<Assertion>` element obtained for the existing SAML SP
  relationship; or
* a signed SAML 2.0 `<Response>` containing exactly one bearer `<Assertion>`
  for that relationship.

The encoded value MUST NOT be line wrapped and SHOULD NOT include padding
characters (`=`). This encoding convention follows the operational form used by
{{RFC7522}}. Many base64 libraries emit padding by default; clients using
such libraries SHOULD strip the trailing `=` characters before sending.
Authorization servers SHOULD accept padded values to maximize interoperability
with such libraries. When a SAML `<Response>` is supplied, it is used only as
a signed wrapper from which this profile extracts the effective assertion,
according to {{saml-input-validation-and-binding}}.

`subject_token_type`:

REQUIRED. The value `urn:ietf:params:oauth:token-type:saml2`.
When a SAML `<Response>` wrapper is supplied, this token type identifies the
effective enclosed SAML 2.0 assertion rather than the wrapper element itself.

`requested_token_type`:

REQUIRED. This profile permits only the values shown in the following table.

{{RFC8693}} marks `requested_token_type` as OPTIONAL at the Token Exchange
protocol level. This profile narrows that rule: an omitted
`requested_token_type` MUST be rejected with `invalid_request`. The three
output types defined here have incompatible processing paths -- refresh
tokens establish persistent sessions, ID Tokens are short-lived identity
assertions, and access tokens require target service resolution -- so no
default value is safe. Explicit client intent is required to avoid this
ambiguity.

| requested_token_type value | Requested token | scope requirement | Target-selection requirement |
|---|---|---|---|
| `urn:ietf:params:oauth:token-type:refresh_token` | Refresh token | `scope` MUST be present and MUST include `offline_access`. An absent `scope` parameter MUST be treated as a missing `offline_access` value and rejected with `invalid_request`. | This profile defines no `resource` or `audience` semantics. Clients SHOULD NOT send them. |
| `urn:ietf:params:oauth:token-type:id_token` | ID Token | `scope` MUST be present and MUST include `openid`. An absent `scope` parameter MUST be rejected with `invalid_request`. | This profile defines no `resource` or `audience` semantics. Clients SHOULD NOT send them. |
| `urn:ietf:params:oauth:token-type:access_token` | Access token | `scope` is OPTIONAL. The request MAY omit `scope` entirely or contain only OAuth scope values without `openid`. | The client MAY send `resource`, `audience`, or both. When both are present, they MUST identify a consistent target service set as understood by the authorization server. |

Target-selection processing for access token requests is defined in
{{token-exchange-common-processing}}.

`scope`:

Conditional. The `scope` requirement depends on `requested_token_type`, as
defined above. This profile permits ordinary OAuth scope values in addition to
OpenID Connect scopes.

This profile does not define a token-endpoint parameter for fine-grained claim
selection. Claims included in a directly issued ID Token, and claims later made
available through UserInfo, are determined by the granted scope, the rules in
{{claim-mapping}}, and the claim release policy for the relying-party relationship
represented by the bound `saml_sp_entity_id`.

`resource`:

OPTIONAL. For access token requests, one or more resource indicators as defined
by {{RFC8707}}. Clients requesting an access token under this profile SHOULD
send a single `resource` value when a resource URI is known.

`audience`:

OPTIONAL. For access token requests, one or more logical target service
identifiers as defined by {{RFC8693}}.

The target-selection rules for `resource` and `audience` depend on
`requested_token_type`, as defined above.

This profile does not define the use of `actor_token`, `actor_token_type`,
or `authorization_details`. The authorization server MUST reject any Token
Exchange request that includes any of these parameters with `invalid_request`.
Actor-based delegation is outside the migration use case and would establish
a delegation chain that the SAML assertion does not authorize.
`authorization_details` processing is not defined by this profile.

Only confidential clients can use this profile. Client authentication at the
token endpoint uses the client's registered OAuth 2.0 client authentication
method. The authenticated client MUST be a confidential client authorized
for the `saml_sp_entity_id` that matches the SAML assertion audience.

## Authorization Server Processing {#token-exchange-authorization-server-processing}

### Common Processing Rules {#token-exchange-common-processing}

Upon receiving the request, the authorization server MUST:

1. authenticate the client in accordance with its registered client
   authentication method;
2. validate the SAML input as described in {{saml-input-validation-and-binding}};
3. verify that the authenticated client is authorized for the
   `saml_sp_entity_id` that matches the SAML assertion audience;
4. verify that the requested token type is defined by this profile and is
   permitted for the authenticated client;
5. resolve the SAML assertion to exactly one active Local Account as described
   in {{local-subject-resolution}};
6. for access token requests, resolve the requested target service set from
   the `resource` and `audience` parameters, the implicit UserInfo target
   when `openid` is requested, or a preconfigured default target service set
   for the authenticated client when neither `resource` nor `audience` is
   provided and local policy permits;
7. evaluate the requested scope against the policy applicable to the same
   relying-party relationship represented by the existing SAML SP and the
   resolved target service set, if any;
8. derive the end-user subject according to {{subject-identifier-mapping}} and
   any claims needed for the issued token according to {{claim-mapping}}; and
9. issue the requested token type, bound to the authenticated client and the
   end-user where applicable, if the request is approved.

Steps 1 through 3 are mutually dependent: client authentication determines which
`saml_sp_entity_id` bindings are applicable, and assertion audience validation
requires knowing those bindings. Implementations SHOULD evaluate these steps as
an integrated unit.

The authorization server MUST reject the request if the SAML assertion was
issued for a different SAML SP than the one bound to the authenticated client.

The authorization server MUST reject the request if subject resolution yields no
Local Account, more than one candidate Local Account, or a Local Account that
is disabled, suspended, deprovisioned, or otherwise not permitted to
authenticate.

The authorization server MUST enforce the token-type-specific scope and target
selection requirements defined in {{token-exchange-request}}. If the
authorization server cannot determine an acceptable target service set for an
access token request, or if the request asks for multiple target services that
policy does not allow to share a single token, it MUST reject the request with
`invalid_target`.

When `requested_token_type` is
`urn:ietf:params:oauth:token-type:refresh_token` or
`urn:ietf:params:oauth:token-type:id_token`, the authorization server SHOULD
ignore `resource` and `audience` if present.

When `requested_token_type` is
`urn:ietf:params:oauth:token-type:access_token` and the requested scope includes
`openid`, the authorization server:

* MUST be an OpenID Provider;
* MUST publish a `userinfo_endpoint` in accordance with OpenID Connect Discovery
  {{OIDC-DISCOVERY}};
* MUST resolve the UserInfo endpoint as part of the target service set when both
  `resource` and `audience` are omitted;
* MAY include the UserInfo endpoint in the target service set when explicit
  `resource` and/or `audience` values are present and policy allows a single
  token to cover both the UserInfo endpoint and those targets; and
* MUST either omit `openid` from the granted scope or reject the request with
  `invalid_target` or `invalid_scope` if the resolved target service set does
  not include the UserInfo endpoint.

When `requested_token_type` is
`urn:ietf:params:oauth:token-type:access_token`, the requested scope does not
include `openid`, and both `resource` and `audience` are omitted, the
authorization server MAY use a preconfigured default target service set for the
authenticated client and bound `saml_sp_entity_id`, or it MAY reject the request
with `invalid_target`.

For any request that includes `scope`, the authorization server MUST apply the
authorization and consent rules in {{authorization-and-consent-model}}. If no
applicable grant policy exists for a requested scope, the authorization server
MUST reject the request with `invalid_scope`.

### Successful Response {#token-exchange-successful-response}

For any successful request, the authorization server MUST return a Token Exchange
response as defined by {{RFC8693}}. The response:

* MUST include `issued_token_type` set to the issued token type and equal to one
  of `urn:ietf:params:oauth:token-type:refresh_token`,
  `urn:ietf:params:oauth:token-type:id_token`, or
  `urn:ietf:params:oauth:token-type:access_token`;
* MUST include `access_token` containing the issued token;
* MUST include `token_type` set to `N_A` when the issued token type is
  `urn:ietf:params:oauth:token-type:refresh_token` or
  `urn:ietf:params:oauth:token-type:id_token`, and otherwise set to the
  applicable access token type such as `Bearer`;
* MAY include `expires_in` as defined by {{RFC8693}}; and
* MUST include `scope` when the granted scope differs from the requested scope,
  as required by {{RFC6749}} and {{RFC8693}}.

{{RFC8693}} uses the `access_token` response member to carry all issued token
types. Clients MUST interpret the `access_token` value according to
`issued_token_type`, as specified in {{RFC8693}} Section 2.2.

### Refresh Token {#token-exchange-refresh-token}

When issuing a refresh token, the authorization server MUST ensure that the
issued refresh token preserves the subject continuity established by the SAML
assertion. In particular, later use of the refresh token for the same client
MUST result in the same `sub` value that would have been derived directly from
the SAML assertion under {{local-subject-resolution}},
{{subject-identifier-mapping}}, and {{claim-mapping}}.

If Token Exchange under this profile issues a refresh token, the client can use
the standard OAuth 2.0 `refresh_token` grant at the same authorization server.

When the authorization server issues a replacement refresh token as part of
refresh token rotation, the replacement token MUST be bound to the same Local
Account as the original refresh token, MUST produce the same `sub` value
under {{subject-identifier-mapping}} when used by the same migrated client,
and MUST NOT extend the SAML session bound established by the originating
assertion (see {{authentication-event-claims}}).

When the granted scope includes `openid`, the authorization server SHOULD
return an ID Token in the first successful refresh token response for that
refresh token. Returning such an ID Token gives the client the migrated subject
and authentication claims in a signed, verifiable format. Any returned ID Token:

* MUST use the `iss` value of the authorization server;
* MUST use the `sub` value defined by {{subject-identifier-mapping}};
* MUST apply the claim mapping rules in {{claim-mapping}};
* MUST preserve the original authentication time from the SAML
  `AuthnStatement`, if known, in the `auth_time` claim; and
* MAY include the `sub_id` claim defined in
  {{saml-subject-identifier-claim}} when the granted scope includes
  `saml_subject` and policy permits release; if included, its members
  MUST be derived from the SAML NameID context that established the
  refresh token, which the authorization server therefore persists
  alongside the refresh token state.

Subsequent refresh token responses are governed by {{OIDC-CORE}}. If an
ID Token is returned in a later refresh response, it MUST continue to preserve
the same `sub` and authentication continuity unless a later authentication
event changes those values under {{OIDC-CORE}}. The same continuity rule
applies to `sub_id` when it was included in earlier ID Tokens for the
same migrated authorization context.

The refresh token issued under this profile MAY also be used as an input to
other specifications that accept
`urn:ietf:params:oauth:token-type:refresh_token` as a Token Exchange
`subject_token_type`, but those subsequent exchanges are defined by the
respective specifications, not by this document.

A refresh token issued under this profile is a regular OAuth 2.0 refresh
token once issued. {{RFC6749}} Section 6 requires that the scope of a
subsequent refresh token request not exceed the scope originally granted
to the refresh token. To make the migrated-session model interoperate
cleanly with that rule, the authorization server MUST set the refresh
token's granted scope, at issuance time, to the full envelope of scopes
authorized by the authorization basis
({{authorization-and-consent-model}}) for the authenticated client and
the resolved Local Account, even when the initial Token Exchange
request asked for a narrower scope. The Token Exchange response MUST
include this granted scope in its `scope` member when it differs from
the requested scope, per {{RFC6749}} Section 3.3 and {{RFC8693}}
Section 2.2.

Subsequent refresh token requests (per {{RFC6749}} Section 6) MAY
therefore request any scope, `resource`, or `audience` value within the
granted envelope, including values not present in the initial Token
Exchange request. The authorization server MUST apply
{{authorization-and-consent-model}} to each such request and MUST
reject a request whose `scope`, `resource`, or `audience` falls outside
the granted envelope with `invalid_scope` or `invalid_target` as
appropriate. Resource indicator semantics in refresh token requests
follow {{RFC8707}} and {{RFC8693}} and are not modified by this profile.

The SAML SP relationship determines whether the Migration Client may
bootstrap a session under this profile; it does not by itself impose a
ceiling on later use of the resulting refresh token.

Subsequent refresh token grants are not re-validated against the original
SAML assertion. The assertion may have expired, may have been issued for a
different purpose, or may no longer satisfy this profile's input-validation
rules; none of that affects the validity of refresh token use, which is
governed solely by authorization server policy and the `SessionNotOnOrAfter`
cap (when one was established).

When a refresh token issued under this profile expires or is revoked, the client
MAY obtain a new SAML assertion through the SAML 2.0 deployment and perform a
new Token Exchange request. The client MAY also use any other OAuth 2.0 or
OpenID Connect flow that the authorization server supports.

Session termination and revocation for refresh tokens issued under this
profile are defined in {{session-termination-and-revocation}}.

### ID Token {#token-exchange-id-token}

When issuing an ID Token directly from Token Exchange, that ID Token:

* MUST use the `iss` value of the authorization server;
* MUST contain an `aud` claim whose value is exactly the authenticated
  client's client identifier, as required by the ID Token validation rules of
  {{OIDC-CORE}}. Any `audience` parameter present in the Token Exchange
  request MUST be ignored for purposes of ID Token audience construction;
* MUST use the `sub` value defined by {{subject-identifier-mapping}};
* MUST apply the claim mapping rules in {{claim-mapping}};
* MUST preserve the original authentication time from the SAML
  `AuthnStatement`, if known, in the `auth_time` claim;
* MUST set `iat` to the time the Token Exchange response is issued, not to
  the SAML `AuthnInstant`;
* MUST set `exp` to a time appropriate for a directly issued ID Token under
  local policy. `exp` MUST NOT be taken directly from the SAML assertion's
  `Conditions/@NotOnOrAfter`. When the ID Token represents the same
  SAML-authenticated session and `SessionNotOnOrAfter` is present, `exp` MUST
  NOT be later than `SessionNotOnOrAfter`; this cap is a session bound, not a
  translation of the SAML assertion lifetime;
* MUST NOT include `nonce`, `at_hash`, or `c_hash`, because no preceding
  OpenID Connect authentication request or authorization code exists from
  which these values could be derived;
* SHOULD NOT include `azp`. If `azp` is included, it MUST equal the
  authenticated client identifier; and
* MAY include the `sub_id` claim defined in {{saml-subject-identifier-claim}}
  when the granted scope includes `saml_subject` and policy permits
  release.

An ID Token issued under this profile MAY be encrypted as a Nested JWT
per {{OIDC-CORE}} when the client has registered an
`id_token_encrypted_response_alg` and the authorization server supports
that encryption. Encryption is independent of the rules above and does
not change which claims the ID Token carries.

### Access Token {#token-exchange-access-token}

This document does not define the syntax of an access token issued by this
exchange. However, when issuing an access token, the authorization server:

* MUST bind the token to the authenticated client, the granted scope, and the
  resolved target service set;
* MUST bind any end-user authorization carried by the token to the resolved
  Local Account and to a subject representation consistent with
  {{subject-identifier-mapping}};
* MUST audience-restrict the token to the resolved target service set; and
* MAY represent that audience restriction in the token value itself or
  associated token metadata.

If the granted scope includes `openid`, the authorization server MUST issue an
access token that is valid at the OpenID Provider's UserInfo endpoint. Under
this profile, a granted `openid` scope makes the UserInfo endpoint part of the
token's resolved target service set. If the original request included
`openid` but the issued access token is not valid at the UserInfo endpoint,
the granted scope in the Token Exchange response MUST omit `openid` or the
request MUST have been rejected.

If an access token issued under this profile is presented to the UserInfo
endpoint, the UserInfo response MUST conform to
{{userinfo-responses}}. If the access token was issued without `openid`, the OpenID
Provider MUST reject its use at the UserInfo endpoint according to
{{OIDC-CORE}}.

Any representation of the end-user subject produced from an access token issued
under this profile by the same authorization server for the same client MUST be
consistent with {{subject-identifier-mapping}}.

This document does not define additional introspection semantics for access
tokens issued under this profile. If the authorization server supports access
token introspection, that behavior remains governed by {{RFC7662}}, the
authorization server's access token profile, and local policy.

## Error Response {#token-exchange-error-response}

If the request is malformed, the authorization server MUST return an error
response as defined by {{RFC6749}} and {{RFC8693}}.

The authorization server MUST use `invalid_request` for malformed or internally
inconsistent Token Exchange parameters, including SAML input that is invalid,
expired, audience-mismatched, encrypted, supplied in an unsupported wrapper
form, wrapped in a SAML `Response` whose `Status` is not acceptable under this
profile, or otherwise unusable under this profile.

The authorization server SHOULD use:

* `unauthorized_client` when the authenticated client is not permitted to use
  the bound `saml_sp_entity_id` relationship or the requested token type;
* `invalid_target` when the `resource` and `audience` parameters are missing,
  malformed, inconsistent, unknown, or unacceptable for the requested access
  token; and
* `invalid_scope` when the requested scope cannot be granted.

The authorization server SHOULD also use `invalid_request` when: the assertion
cannot be resolved to exactly one active Local Account; the assertion is
rejected as replay or otherwise exhausted under {{saml-input-validation-and-binding}}; or a
currently asserted stable subject identifier conflicts with an existing persisted
mapping and no explicitly authorized administrative remapping exists.

# SAML Assertion Introspection {#saml-assertion-introspection}

This section defines an introspection extension for cases in which the client
does not need OAuth tokens and only needs a SAML assertion to be validated and
translated into normalized JSON claims.

This pattern uses the authorization server's introspection endpoint as
defined by {{RFC7662}}. RFC 7662 describes introspection primarily for OAuth
access and refresh tokens, but it explicitly permits servers to support other
token types identified by `token_type_hint`, and its response structure is
extensible.

The introspection endpoint already provides the infrastructure this pattern
needs: mutual client authentication, a single request/response interaction,
and a binary active/inactive signal that maps directly onto whether the SAML
assertion is valid and authorized for the requesting client. An `active:
false` response for an invalid or unauthorized SAML assertion is semantically
consistent with the RFC 7662 definition of an inactive token.

Under this pattern, the Migration Client authenticates the user through the
existing SAML 2.0 deployment, receives a SAML assertion for the SAML SP
identified by `saml_sp_entity_id`, and submits that assertion, or a signed
SAML `Response` that conveys it, to the introspection endpoint. If the
assertion is valid for the client and the client is authorized to receive the
mapped subject and claims, the authorization server returns an active
introspection response containing a JSON claims object. No OAuth access
token, refresh token, or ID Token is issued.

## Request {#introspection-request}

The client makes an introspection request to the introspection endpoint using
the parameters described below.

The following non-normative example asks the authorization server to validate a
SAML assertion and return normalized JSON claims and SAML protocol metadata:

~~~ http
POST /introspect HTTP/1.1
Host: as.example.edu
Content-Type: application/x-www-form-urlencoded

token=PHNhbWxwOlJlc3BvbnNlLi4u
&token_type_hint=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Asaml2
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer
&client_assertion=eyJhbGciOiJSUzI1NiIs...
~~~

The request parameters are summarized below:

| Parameter | Requirement | Summary |
|---|---:|---|
| `token` | REQUIRED | Carries the base64url-encoded SAML `Assertion` or SAML `Response` wrapper. |
| `token_type_hint` | OPTIONAL | Identifies the effective token as a SAML 2.0 assertion. |

`token`:

REQUIRED. The base64url-encoded octet sequence, as defined in Section 5 of
{{RFC4648}}, of the XML serialization of either:

* a signed SAML 2.0 `<Assertion>` element obtained for the existing SAML SP
  relationship; or
* a signed SAML 2.0 `<Response>` containing exactly one bearer `<Assertion>`
  for that relationship.

The encoded value MUST NOT be line wrapped and SHOULD NOT include padding
characters (`=`). This encoding convention follows the operational form used by
{{RFC7522}}. Many base64 libraries emit padding by default; clients using
such libraries SHOULD strip the trailing `=` characters before sending.
Authorization servers SHOULD accept padded values to maximize interoperability
with such libraries. When a SAML `<Response>` is supplied, it is used only as
a signed wrapper from which this profile extracts the effective assertion,
according to {{saml-input-validation-and-binding}}.

`token_type_hint`:

OPTIONAL. If sent, the value MUST be
`urn:ietf:params:oauth:token-type:saml2`.
When a SAML `<Response>` wrapper is supplied, this token type hint still
identifies the effective enclosed SAML 2.0 assertion rather than the wrapper
element itself. If a `token_type_hint` value other than
`urn:ietf:params:oauth:token-type:saml2` is sent, the authorization server
SHOULD return an HTTP 400 response.

This profile does not define a request parameter for selecting a subset of
normalized claims in the introspection response.

Only confidential clients can use this profile. Client authentication for the
introspection endpoint is performed according to {{RFC7662}} and authorization
server policy. The authenticated client MUST be authorized for the
`saml_sp_entity_id` that matches the SAML assertion audience.

## Authorization Server Processing {#introspection-authorization-server-processing}

### Common Processing Rules {#introspection-common-processing}

Upon receiving the request, the authorization server MUST:

1. authenticate the client and authorize it to use the introspection endpoint;
2. validate the SAML input as described in {{saml-input-validation-and-binding}};
3. verify that the authenticated client is authorized for the
   `saml_sp_entity_id` that matches the SAML assertion audience;
4. resolve the SAML assertion to exactly one active Local Account as described
   in {{local-subject-resolution}};
5. derive the end-user subject according to {{subject-identifier-mapping}} and
   any claims needed for the response according to {{claim-mapping}}; and
6. apply the claim release policy for that same relying-party relationship
   before returning any claims.

### Inactive Response {#introspection-inactive-response}

If the SAML assertion is invalid, expired, audience-mismatched, otherwise not
usable for the client, wrapped in a SAML `Response` whose `Status` is not
acceptable under this profile, cannot be resolved to exactly one active Local
Account, or the authenticated client is not entitled to introspect that specific
assertion or bound relying-party context, the authorization server MUST return a
successful introspection response with `"active": false` and SHOULD NOT include
additional members.

### Successful Response {#introspection-successful-response}

For an active SAML assertion, the authorization server MUST return an
introspection response as defined by {{RFC7662}} with:

* `active` set to `true`;
* the normalized identity claims for this client (`sub` and, where
  applicable, `sub_id`, `auth_time`, `acr`, `amr`, `sid`, and
  attribute-derived claims defined by {{claim-mapping}}) returned as
  top-level members, in parallel with how those claims appear in ID
  Tokens and UserInfo responses; and
* `saml`, when returned, set to a JSON object containing normalized SAML
  protocol metadata as defined below.

The top-level identity claims are values that the authorization server
has already processed under {{local-subject-resolution}},
{{subject-identifier-mapping}}, and {{claim-mapping}}, including subject
resolution and claim release policy. They are not a literal mirror of
the SAML assertion content; the protocol mirror is the `saml` object.

The top-level identity claims:

* MUST include the mapped `sub` value defined by {{subject-identifier-mapping}};
* MAY include `auth_time`, `acr`, `amr`, `sid`, and attribute-derived
  claims defined by {{claim-mapping}};
* MAY include the `sub_id` claim defined in {{saml-subject-identifier-claim}}
  when the introspection-endpoint policy permits release; and
* MUST NOT include claims that would not be releasable to the same
  relying party under this profile.

The authorization server SHOULD return only the minimum normalized claims
necessary for the authorized client and applicable relying-party policy.

The `saml` object contains SAML protocol metadata, not end-user identity
claims. The authorization server MUST NOT duplicate end-user identity
claims inside the `saml` object.

When the `saml` object is returned:

* every member value MUST be derived from SAML input that was successfully
  validated under {{saml-input-validation-and-binding}};
* values from a SAML `Response` wrapper MUST be included only when a signed
  SAML `Response` wrapper was submitted and accepted under
  {{response-wrapper-processing}};
* response-level values that this profile does not validate, such as
  `Destination` or `Response/@InResponseTo`, MAY be returned as extracted
  values, but their presence in the response MUST NOT be interpreted as a
  statement that the authorization server validated them unless a separate
  deployment agreement says so; and
* the authorization server MUST omit any member for which it has no validated or
  extracted value.

The `saml` object is intended for clients that need the authorization server to
perform XML parsing and XML signature validation, but still need protocol values
from the validated SAML input to complete local SAML SP processing.

The `saml` object MAY contain these members:

`response`:
: JSON object containing values extracted from the accepted SAML `Response`
  wrapper. The presence of this member signals that the submitted SAML
  input was a `Response` wrapper; the member is absent when the submitted
  SAML input was a bare `Assertion`.

`assertion`:
: JSON object containing values from the effective SAML `Assertion`.

`attributes`:
: Array of JSON objects, each mirroring one logical SAML attribute
  derived from the assertion's `AttributeStatement` elements after the
  combining rules in {{attribute-claims}}. Each object MUST include a
  `name` member (the SAML `Name` value) and a `values` member (a JSON
  array of the attribute's `AttributeValue` contents), and MAY include
  `name_format` (the `NameFormat` URI when present) and `friendly_name`
  (the `FriendlyName` when present). Values inside `values` MUST follow
  the typing rules defined in {{attribute-claims}}; in particular, XML
  schema booleans MAY be emitted as JSON booleans and numeric types as
  JSON numbers when the XML type is unambiguous, with all other values
  emitted as JSON strings. The `attributes` member is intended for
  clients that require the wire-form SAML attribute names (for example,
  `urn:oid:` identifiers) in addition to the normalized top-level
  identity claims.

When present, the `response` object MAY contain these members:

| Member | Type | Source |
| --- | --- | --- |
| `id` | String | `Response/@ID` |
| `issuer` | String | `Response/Issuer` element |
| `issue_instant` | String | `Response/@IssueInstant` |
| `destination` | String | `Response/@Destination` |
| `in_response_to` | String | `Response/@InResponseTo` |
| `status_code` | String | validated value of `Response/Status/StatusCode/@Value` |
| `has_nested_status_code` | Boolean | `true` iff a nested `Response/Status/StatusCode/StatusCode` element was present |

Under this profile, an active response that includes a `response` object
will normally have `has_nested_status_code` set to `false` because
{{response-wrapper-processing}} requires the authorization server to
reject a Response wrapper that contains a nested status code.

When present, the `assertion` object MAY contain these members:

| Member | Type | Source |
| --- | --- | --- |
| `id` | String | `Assertion/@ID` |
| `issuer` | String | `Assertion/Issuer` element |
| `issue_instant` | String | `Assertion/@IssueInstant` |
| `audiences` | Array of strings | `Audience` values from all `AudienceRestriction` conditions |
| `not_before` | String | `Assertion/Conditions/@NotBefore` |
| `not_on_or_after` | String | `Assertion/Conditions/@NotOnOrAfter` |
| `subject_confirmation` | JSON object | fields from the bearer `SubjectConfirmation` treated as usable under {{bearer-subject-confirmation-data-validation}}; see below |

The `subject_confirmation` object MAY contain these members:

| Member | Type | Source |
| --- | --- | --- |
| `method` | String | `Method` value of the usable bearer `SubjectConfirmation` |
| `recipient` | String | `Recipient` value from the `SubjectConfirmationData` |
| `in_response_to` | String | `InResponseTo` value from the `SubjectConfirmationData` |
| `not_on_or_after` | String | `NotOnOrAfter` value from the `SubjectConfirmationData` |

The authorization server MUST omit any member for which it has no
extracted value.

Date and time values in the `saml` object SHOULD be represented as strings
preserving the SAML dateTime semantics. Implementations MAY normalize
equivalent dateTime values to a consistent UTC representation. Clients MUST
compare these values according to SAML dateTime semantics, not by lexical string
comparison unless the representation has first been normalized.

This profile does not require the introspection response to include OAuth token
metadata fields such as `scope`, `token_type`, `exp`, or `iss`, because no
OAuth token is being issued by the introspection operation.

## Error Response {#introspection-error-response}

If the introspection request is malformed, for example because the `token`
parameter is absent or its value cannot be decoded as valid base64url, the
authorization server MUST return an HTTP 400 (Bad Request) error response
as defined by {{RFC7662}}, not an introspection response.

If client authentication fails, or if the client is not authorized to use the
introspection endpoint at all, the authorization server MUST return an HTTP 401
(Unauthorized) or HTTP 403 (Forbidden) response, consistent with {{RFC7662}}
Section 2.2 and authorization server policy.

A syntactically valid request in which the client is authenticated and
authorized, but the presented SAML assertion is invalid, expired, audience-
mismatched, replayed, or otherwise not usable for the client, MUST result in
an introspection response with `"active": false`. The authorization server
SHOULD NOT include additional members in such a response.

# Local Subject Resolution {#local-subject-resolution}

Before deriving `sub` or other end-user claims under this profile, the
authorization server MUST resolve the SAML assertion to exactly one active
Local Account.

This document does not require or define just-in-time provisioning. If a
deployment performs provisioning or account activation based on a validated SAML
assertion, that behavior is outside this profile and MUST complete before the
authorization server applies the subject continuity and claim mapping rules
defined here.

The authorization server MUST perform this resolution using stable SAML subject
identifiers, existing persisted linkages, or equivalent administrative mapping
for the trusted SAML deployment. Releasable attributes such as email address,
display name, telephone number, or generic usernames MUST NOT by themselves be
treated as sufficient evidence to create or select a Local Account under this
profile, although they MAY be used as supplemental hints together with a stable
binding already established by policy.

If the assertion contains a `NameID` whose format is
`urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress`, or whose semantics are
those of an email address under local policy, the authorization server MUST
treat that `NameID` as a mutable account attribute for purposes of this
profile. Such a `NameID` MUST NOT by itself establish a new Local Account
linkage or a new subject continuity mapping. It MAY be used to locate an
existing Local Account only when local policy has already bound that asserted
email identifier for the trusted SAML issuer to exactly one Local Account.

If resolution yields no Local Account, more than one candidate Local Account,
or a Local Account that is disabled, suspended, deprovisioned, or otherwise not
eligible to authenticate, the authorization server MUST fail the profile
operation and MUST NOT issue tokens or return an active introspection response.

Once a continuity mapping has been established for a resolved Local Account,
the authorization server MUST continue to resolve the same subject input to
that same Local Account unless an authorized administrative remapping occurs.

When this document requires a Stable Local Subject Key, the value MUST be the
identifier defined in {{terminology}} for the resolved Local Account and MUST
remain the same across all invocations of this profile at the same
authorization server.

# Subject Identifier Mapping {#subject-identifier-mapping}

Subject continuity is the central guarantee of this profile: the same
end-user MUST be represented to the migrated relying party by a stable
identifier that does not require account relinking. The OpenID Connect
`sub` claim carries that continuity. The authorization server MUST
issue a `sub` value that is stable for the client and that preserves
the semantics of the corresponding SAML deployment.

The rules in this section achieve subject continuity by:

* reusing any persisted `sub` mapping the authorization server already
  holds for the resolved Local Account under this client's effective
  subject type;
* otherwise deriving `sub` from stable SAML subject inputs, in
  preference order: `subject-id` or `pairwise-id` attributes
  ({{SAML2-SUBJ-ID}}), persistent `NameID`, and finally a deterministic
  derivation from the Stable Local Subject Key;
* preserving any pairwise scoping the SAML SP relationship already
  used, via the bound `saml_sp_entity_id`;
* excluding mutable identifiers (email, transient `NameID`, display
  attributes) as `sub` sources; and
* persisting whichever mapping is chosen so the same Local Account
  resolves to the same `sub` on subsequent invocations.

Most of this section concerns the rules for deriving `sub`. The final
subsection ({{saml-subject-identifier-claim}}) defines an OPTIONAL
supplementary claim, `sub_id`, that carries the original SAML NameID
context for clients that need it in addition to `sub`.

## NameID Format Recognition {#nameid-format-recognition}

The following `NameID` formats are recognized for subject mapping purposes
under this profile:

* `urn:oasis:names:tc:SAML:2.0:nameid-format:persistent`: A persistent,
  SP-specific or globally non-reassignable identifier. MAY be used as input
  to the subject mapping rules in this section.
* `urn:oasis:names:tc:SAML:2.0:nameid-format:transient`: A session-scoped,
  non-persistent identifier. MUST NOT be used as input to subject mapping
  under this profile and MUST NOT be used as the OpenID Connect `sub`.
* `urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress`: A mutable account
  attribute. Treated as defined in {{local-subject-resolution}}. MUST NOT be used
  as `sub`.
* `urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified`: Deployment-defined
  semantics. MUST NOT be used as a subject mapping input unless local policy
  has explicitly established the identifier as stable and non-reassignable for
  the trusted SAML issuer.
* `urn:oasis:names:tc:SAML:2.0:nameid-format:entity`: Identifies a system
  entity, not an end-user. MUST NOT be used as a subject mapping input.

Any other `NameID` format MUST NOT be used as a subject mapping input under
this profile unless local policy explicitly establishes its semantics as
persistent and non-reassignable.

## Subject Type Determination {#subject-type-determination}

The authorization server MUST determine whether the migrated client is using a
public subject or a pairwise subject. The registered `subject_type`, if present,
MUST be used for this determination. If no `subject_type` is registered, the
authorization server MUST apply `public` as the effective subject type, in
accordance with {{client-registration-saml-sp-entity-id}}.

If multiple clients sharing the same `saml_sp_entity_id` have registered
conflicting `subject_type` values and no single effective subject type can be
determined per {{client-registration-saml-sp-entity-id}}, the authorization server MUST fail the profile
operation.

When the authorization server uses, or would otherwise use, the
`urn:oasis:names:tc:SAML:attribute:subject-id` or
`urn:oasis:names:tc:SAML:attribute:pairwise-id` attribute as a subject mapping
input, it MUST validate that attribute according to the SAML V2.0 Subject
Identifier Attributes Profile. In particular, the attribute MUST use
`NameFormat` `urn:oasis:names:tc:SAML:2.0:attrname-format:uri` and contain
exactly one non-empty `AttributeValue` that conforms to that profile's syntax
and semantics. If no persisted mapping exists and the attribute corresponding
to the client's effective subject type is present but fails these requirements,
the authorization server MUST reject the profile operation rather than derive a
different `sub` from fallback rules.

If a persisted mapping already exists and the current assertion also contains a
valid subject identifier input matching the client's effective subject type, or
a persistent `NameID` that would otherwise be eligible as a subject mapping
input, the authorization server MUST compare the current asserted identifier
context to the persisted source identifier context. If they differ, the
authorization server MUST NOT silently replace the persisted mapping based only
on the current assertion. The authorization server MAY continue only when an
explicitly authorized administrative remapping has bound the new identifier
context to the same Local Account; otherwise it MUST fail the profile
operation.

When multiple clients share the same `saml_sp_entity_id`, the authorization
server MUST apply the same effective subject mapping rules to all such clients.
In particular, it MUST produce the same `sub` value for the same resolved Local
Account when those clients are operating under the same subject type semantics.

## Pairwise Subject Identifier Mapping {#pairwise-subject-identifier-mapping}

For clients using pairwise subject identifiers, the authorization server MUST
determine `sub` using the first applicable rule:

1. If the authorization server already has a persisted pairwise mapping for
   the same resolved Local Account and the same `saml_sp_entity_id`, it MUST
   reuse that mapping, subject to the mismatch handling rules above.
2. If no persisted mapping exists and the SAML assertion contains a valid
   `urn:oasis:names:tc:SAML:attribute:pairwise-id` attribute (as validated by
   the rules in {{subject-type-determination}}), the authorization server MUST
   use that attribute value as `sub` and MUST persist the resulting mapping.
3. If no persisted mapping exists and the assertion contains a persistent
   `NameID` (`urn:oasis:names:tc:SAML:2.0:nameid-format:persistent`) whose
   semantics are SP-specific, stable, and non-reassignable for that same SAML
   SP relationship, the authorization server MAY use that value as `sub`, and
   it MUST persist the resulting mapping.
4. Otherwise, the authorization server MUST derive a deterministic,
   non-reassignable pairwise identifier by combining the Stable Local Subject
   Key and the bound `saml_sp_entity_id` using a collision-resistant method.
   The pairwise subject computation defined in Section 8 of {{OIDC-CORE}} is a
   suitable derivation method, using `saml_sp_entity_id` as the sector
   identifier input to that computation. The authorization server MUST persist
   the resulting mapping.

The `pairwise-id` attribute value has the format `localpart@scope` as
defined by {{SAML2-SUBJ-ID}}. When a
`pairwise-id` value is used as `sub`, the resulting value carries this scoped
format. Clients MUST NOT treat a `sub` value derived from `pairwise-id` as an
email address or interpret the scope portion as an email domain.

## Public Subject Identifier Mapping {#public-subject-identifier-mapping}

For clients using public subject identifiers, the authorization server MUST
determine `sub` using the first applicable rule:

1. If the authorization server already has a persisted public subject mapping
   for the same resolved Local Account under the same OAuth issuer, it MUST
   reuse that mapping, subject to the mismatch handling rules above.
2. If no persisted mapping exists and the SAML assertion contains a valid
   `urn:oasis:names:tc:SAML:attribute:subject-id` attribute (as validated by
   the rules in {{subject-type-determination}}), the authorization server MUST
   use that attribute value as `sub` and MUST persist the resulting mapping.
3. If no persisted mapping exists and the assertion contains a persistent
   `NameID` (`urn:oasis:names:tc:SAML:2.0:nameid-format:persistent`) whose
   semantics are stable, non-reassignable, and not SP-specific, the
   authorization server MAY use that value as `sub`, and it MUST persist the
   resulting mapping.
4. Otherwise, the authorization server MUST derive a deterministic,
   non-reassignable public identifier from the Stable Local Subject Key alone,
   without client-specific input, using a collision-resistant method. The
   authorization server MUST persist the resulting mapping.

## Common Rules {#subject-identifier-common-rules}

For all subject types:

* before issuing any chosen or derived value as `sub`, the authorization
  server MUST ensure that it satisfies the {{OIDC-CORE}} `sub`
  requirements, including that it be a case-sensitive string no longer than
  255 ASCII characters;
* if the chosen source identifier would exceed that limit or contain non-ASCII
  characters, the authorization server MUST derive a deterministic,
  collision-resistant ASCII `sub` value from the chosen source identifier and
  required context, and it MUST persist the resulting mapping;
* if both `subject-id` and `pairwise-id` are available, the authorization
  server MUST select the one whose semantics match the client's effective
  subject type;
* the authorization server MUST NOT expose a `pairwise-id` value to a client
  other than a client bound to the corresponding `saml_sp_entity_id`;
* the authorization server MUST NOT use a transient identifier as the OpenID
  Connect `sub`;
* the authorization server MUST NOT directly use mutable identifiers such as an
  email address, a `NameID` in the SAML `emailAddress` format, or a
  non-persistent `NameID` format as the OpenID Connect `sub`.

When the only usable SAML subject input is a `NameID` whose semantics are those
of an email address, and {{local-subject-resolution}} succeeds under the rules
above, the authorization server MUST derive `sub` from the resolved Stable
Local Subject Key using the applicable fallback rule in this section rather
than using the asserted email value itself.

When a persistent `NameID` is used as an input to subject mapping, the
authorization server MUST evaluate the identifier together with its qualifier
context. In particular:

* `SPNameQualifier`, if present for pairwise processing, MUST correspond to the
  bound `saml_sp_entity_id` or to an equivalent configured identifier for that
  same relying-party relationship;
* `NameQualifier`, if present, MUST correspond to the trusted SAML IdP
  identity or to an equivalent configured identifier for that same issuer;
* `SPProvidedID`, if present, MUST be treated as part of the source identifier
  context rather than ignored; and
* two `NameID` values that differ in value or qualifier context MUST be treated
  as distinct unless local policy can prove them equivalent.

## SAML Subject Identifier Claim {#saml-subject-identifier-claim}

This document defines the `sub_id` claim, which carries the original
SAML `NameID` context alongside `sub`. It exists for migrated clients
that need that context -- for example, to correlate sessions against
legacy SAML-keyed audit logs or account databases.

`sub_id` is a JSON object following the subject identifier pattern of
{{RFC9493}}. The `format` member discriminates among possible identifier
shapes; future profiles may register additional formats in the IANA
"Security Event Identifier Formats" registry. This document defines one
format, `saml-nameid`, for SAML NameID values. When `sub_id` is present
in an ID Token, UserInfo response, or introspection response under this
profile, it MUST include a `format` member.

When `format` is `saml-nameid`, the `sub_id` object MAY contain the
following members:

`format`:
: String. REQUIRED. The value `saml-nameid`.

`issuer`:
: String. The SAML Assertion Issuer value from the assertion that
  established this identifier. When present, this value MUST match the
  effective `saml_idp_entity_id` for the authenticated client.

`nameid`:
: String. REQUIRED. The textual content of the SAML `NameID` element.

`nameid_format`:
: String. The SAML `NameID/@Format` attribute value when present.

`name_qualifier`:
: String. The SAML `NameID/@NameQualifier` attribute value when
  present.

`sp_name_qualifier`:
: String. The SAML `NameID/@SPNameQualifier` attribute value when
  present.

`sp_provided_id`:
: String. The SAML `NameID/@SPProvidedID` attribute value when
  present.

The authorization server MUST omit any member for which it has no
validated value. Encrypted NameID forms (`EncryptedID`) are rejected
under {{encrypted-content}} and MUST NOT appear in `sub_id`.

### Eligible NameID Formats {#sub-id-eligible-nameid-formats}

A `sub_id` with `saml-nameid` format is a stable subject identifier
reference; consumers persist its members and use them for legacy
correlation. The authorization server MUST therefore restrict the
NameID formats it exposes:

* `urn:oasis:names:tc:SAML:2.0:nameid-format:persistent` -- MAY appear
  in `sub_id`.
* `urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified` -- MAY appear
  in `sub_id` only when local policy has established the NameID as
  stable and non-reassignable for the trusted SAML issuer (the same
  condition imposed in {{nameid-format-recognition}}).
* `urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress` -- SHOULD
  NOT appear in `sub_id`. Email-format NameIDs are mutable account
  attributes, not stable identifiers; exposing them through `sub_id`
  misleads consumers that treat `sub_id` as a durable reference. If an
  emailAddress NameID is nonetheless included, its presence MUST NOT by
  itself cause the authorization server to emit `email_verified=true`
  for the corresponding `email` claim.
* `urn:oasis:names:tc:SAML:2.0:nameid-format:transient` -- MUST NOT
  appear in `sub_id`. Transient NameIDs are session-scoped and lose
  meaning across sessions.
* `urn:oasis:names:tc:SAML:2.0:nameid-format:entity` -- MUST NOT appear
  in `sub_id`. Entity NameIDs identify systems rather than end-users.

If the effective SAML assertion contains no usable `NameID` element
(for example, when the assertion conveys subject information solely
through `subject-id` or `pairwise-id` attributes), the authorization
server MUST NOT emit `sub_id` with the `saml-nameid` format.

### Release Rules {#sub-id-release-rules}

The authorization server MUST release `sub_id` only when authorized by
the claim release policy for the relying-party relationship represented
by the bound `saml_sp_entity_id`, and MUST NOT release it solely because
the SAML assertion contains a `NameID`.

In contexts that carry an OAuth scope -- Token Exchange issuing an ID
Token or refresh token, and UserInfo requests -- release SHOULD be gated
on the `saml_subject` scope. In contexts without OAuth scope, such as
introspection requests, scope gating does not apply; the authorization
server's policy for that endpoint authorizes release in addition to the
claim release policy above.

The `sub_id` claim supplements `sub`; it does not replace it. The `sub`
claim remains the primary continuity point under
{{subject-identifier-mapping}}.

# Claim Mapping {#claim-mapping}

The mappings in this section apply to ID Tokens, UserInfo responses, and
introspection responses under this profile. Values derived from SAML
subject identifiers or authentication statements take precedence over
generic attribute mapping.

The following table summarizes the principal SAML inputs and their
target OpenID Connect claims under this profile. Detailed rules,
including precedence and edge cases, are in the subsections that
follow.

| SAML input | OpenID Connect target | Defined in |
| --- | --- | --- |
| Persistent `NameID`, `subject-id`, `pairwise-id`, or Stable Local Subject Key derivation | `sub` | {{subject-identifier-mapping}} |
| `NameID` element (provenance preserved) | `sub_id` with `saml-nameid` format | {{saml-subject-identifier-claim}} |
| `AuthnStatement/@AuthnInstant` | `auth_time` | {{authentication-event-claims}} |
| `AuthnContext/AuthnContextClassRef` | `acr` | {{authentication-event-claims}} |
| Concrete authentication methods derived from local policy or SAML evidence | `amr` | {{authentication-event-claims}} |
| `AuthnStatement/@SessionIndex` | `sid` | {{authentication-event-claims}} |
| `AttributeStatement` attributes (e.g., `mail`, `givenName`, `sn`, `displayName`) | Registered OIDC claims (`email`, `given_name`, `family_name`, `name`, etc.) | {{attribute-claims}} |
| Other `AttributeStatement` attributes | Private claims by prior agreement | {{attribute-claims}} |

This profile maps identity-bearing values for interoperability. It does
not map SAML data whose purpose is authorization or entitlement
(`AuthzDecisionStatement`, scoped affiliation values intended for
access control, etc.) into OAuth scopes or authorization decisions; see
{{non-goals}}.

## Authentication Event Claims {#authentication-event-claims}

When a SAML assertion contains an `AuthnStatement`, the authorization server
SHOULD map its contents as follows:

* `AuthnStatement/@AuthnInstant`, when mapped, MUST be emitted as the
  OpenID Connect `auth_time` claim as a NumericDate value. Implementations
  MUST convert the XML dateTime value to the number of seconds elapsed since
  1970-01-01T00:00:00Z when producing this NumericDate value.
* `AuthnContext/AuthnContextClassRef` maps to the OpenID Connect `acr`
  claim. Mapping rules are in {{acr-mapping}}.
* `AuthnStatement/@SessionIndex` MAY be used as authoritative input to the
  OpenID Connect `sid` claim, as defined by {{OIDC-FC-LOGOUT}} and
  {{OIDC-BC-LOGOUT}}, when the deployment supports session continuity or logout
  semantics.

  When `sid` is emitted, it MUST identify the OpenID Provider session
  corresponding to the SAML-authenticated session and MUST be unique within
  the OAuth issuer. The authorization server MUST NOT assume that the raw SAML
  `SessionIndex` is already a valid `sid`; it MUST either reuse the raw value
  only when local policy guarantees it is issuer-scoped and collision-resistant
  for this purpose, or derive and persist an OpenID Provider-scoped `sid`
  bound to that SAML-authenticated session. If `SessionIndex` is absent from
  the selected `AuthnStatement`, `sid` MUST NOT be emitted unless the
  authorization server has another authoritative session identifier for that
  SAML-authenticated session.

The OpenID Connect `amr` claim identifies authentication methods, not an
authentication context class. Therefore:

* the authorization server MUST NOT copy the SAML
  `AuthnContextClassRef` directly into `amr`;
* the authorization server MAY populate `amr` only when it can derive concrete
  authentication methods from local policy or explicit SAML authentication
  evidence; and
* when `amr` is populated, the authorization server SHOULD use values from the
  Authentication Method Reference Values registry {{RFC8176}}, and it MUST represent
  `amr` as a JSON array of strings.

If multiple `AuthnStatement` elements are present, the authorization server
MUST select the statement that local policy designates as authoritative for
the OAuth or OpenID Connect transaction. If no local policy designates an
authoritative statement, the authorization server MUST omit derived
authentication claims (`auth_time`, `acr`, `amr`, and `sid`) rather than
emit values from an arbitrary statement. The authorization server MUST NOT
default to selecting by `AuthnInstant` alone, because the most recent
statement may reflect a SAML identity proxying re-issuance rather than the
original end-user authentication. Deployments using proxied assertions
MUST configure which `AuthnStatement` is authoritative.

If the selected `AuthnStatement` contains `SessionNotOnOrAfter`, the
authorization server MUST treat that instant as an upper bound on operations
that preserve the SAML-authenticated session under this profile. In
particular:

* the authorization server MUST NOT issue a refresh token whose validity
  extends beyond `SessionNotOnOrAfter`;
* if it issues an access token or ID Token directly from the assertion and that
  token is intended to represent the same authenticated session, the token
  expiration time MUST NOT extend beyond `SessionNotOnOrAfter`; and
* once `SessionNotOnOrAfter` has passed, the authorization server MUST reject
  further session-preserving use of the assertion or derived refresh token.

Earlier session-end signals and revocation cascade are defined in
{{session-termination-and-revocation}}.

The following SAML data has no direct standard OpenID Connect claim mapping:

* `AuthnContextDecl`
* `AuthnContextDeclRef`
* `AuthnContext/AuthenticatingAuthority`
* `SubjectLocality`

If the `AuthnContext` contains an `AuthnContextDeclRef` element but no
`AuthnContextClassRef`, the authorization server MUST NOT emit an `acr` claim
unless local policy defines a deterministic mapping from that declaration
reference to a specific ACR value. If no such mapping exists, `acr` MUST be
omitted.

Such values MAY be conveyed in private claims by prior agreement between the
authorization server and client. Deployments using such claims SHOULD use
collision-resistant claim names.

### ACR Mapping {#acr-mapping}

The `acr` claim derived from `AuthnContext/AuthnContextClassRef` MUST be
produced as follows:

* if local policy maps SAML context class references to a registered
  Level-of-Assurance scheme, that mapping MUST be applied;
* otherwise, the authorization server SHOULD copy the SAML URI without
  transformation;
* the authorization server MUST NOT advertise an `acr` value in
  `acr_values_supported` that it cannot emit. If `acr_values_supported`
  enumerates only the mapped Level-of-Assurance values, the
  authorization server MAY still emit pass-through SAML
  `AuthnContextClassRef` URIs that are not enumerated, provided the
  asserted URI satisfies the pass-through rule above.

Clients receiving raw SAML authentication context class URIs as `acr`
values (for example,
`urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport`)
MUST be prepared to handle them, as they are not registered OpenID
Connect ACR values.

Non-normative example mappings from common SAML AuthnContextClassRef
URIs to OIDC `acr`/`amr` values appear in
{{authentication-context-mapping-examples}}.

## Attribute Claims {#attribute-claims}

A `NameID` in emailAddress format is not by itself an attribute mapping input.
However, if the assertion contains a `NameID` whose format is
`urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress`, or whose semantics are
those of an email address under local policy, the authorization server MAY use
that value as the OpenID Connect `email` claim when:

* claim release policy for the relying-party relationship permits release of
  `email`;
* the value is bound by local policy to the resolved Local Account; and
* no higher-precedence SAML attribute mapping produces a conflicting `email`
  claim value.

If the emailAddress `NameID` conflicts with an `email` value derived from a
SAML attribute and no deterministic local precedence rule exists, the
authorization server SHOULD omit the `email` claim rather than guess. An
emailAddress `NameID` MUST NOT by itself cause the authorization server to
emit `email_verified=true`.

When a SAML attribute directly corresponds to a registered OpenID
Connect claim, the authorization server SHOULD emit that claim. When
the SAML deployment uses attribute `Name` values based on object
identifiers or URIs, the authorization server MAY apply a
deployment-specific mapping table that resolves those names to the
equivalent OpenID Connect claim names.

Non-normative example mappings for common SAML attributes and for
InCommon and `eduPerson` deployments are provided in
{{incommon-and-eduperson-deployment-guidance}}.

All `AttributeStatement` elements in the assertion MUST be processed as a
single logical attribute set. Attributes having the same `Name` and the same
`NameFormat` are part of the same logical attribute and their values MUST be
combined in document order. Attributes that differ by `Name` or `NameFormat`
MUST be treated as distinct inputs even if their `FriendlyName` values are the
same.

An omitted `NameFormat` MUST be treated as
`urn:oasis:names:tc:SAML:2.0:attrname-format:unspecified` for purposes of
attribute grouping and mapping under this profile.

Mapping decisions MUST be based primarily on the SAML attribute `Name` and
`NameFormat`. `FriendlyName` MAY be used only as a secondary hint when no
deterministic `Name`-based mapping exists. If `Name`, `NameFormat`, and
`FriendlyName` would imply conflicting claim mappings, the authorization server
MUST use the `Name` and `NameFormat` interpretation.

The registered OpenID Connect claims listed above are single-valued string
claims if present. The `auth_time` claim is a single numeric value. The `amr`
claim is an array of strings.

Attribute processing MUST follow these rules:

* a single SAML `AttributeValue` maps to a single JSON value;
* multiple SAML `AttributeValue` elements MAY map to a JSON array only for
  claims whose syntax permits arrays or for private claims defined by prior
  agreement;
* boolean-valued SAML attributes SHOULD map to JSON booleans when the XML type
  is unambiguous;
* if multiple source values are available for a registered single-valued claim,
  the authorization server MUST either apply a deterministic, claim-specific
  precedence rule or omit that claim;
* if multiple distinct SAML attributes could map to the same registered claim,
  the authorization server MUST apply a deterministic precedence rule based on
  SAML attribute identifiers or omit that claim;
* when multiple SAML attributes from the same logical attribute set could
  map to the same registered OpenID Connect claim and they differ only by
  `NameFormat`, the authorization server SHOULD prefer the attribute whose
  `NameFormat` is `urn:oasis:names:tc:SAML:2.0:attrname-format:uri` (the
  canonical federation form) over `:basic` or `:unspecified` forms;
* the authorization server MUST NOT emit a JSON array for a registered
  single-valued claim; and
* generic attribute mapping MUST NOT overwrite `sub`, `auth_time`, `acr`,
  `amr`, or `sid` values that were derived by the rules in this document.

The authorization server MUST NOT infer `email_verified` from the mere
presence of an `email` attribute. The authorization server MUST NOT infer
`phone_number_verified` from the mere presence of a `phone_number` attribute.
These verified status claims MUST be populated only when an explicit
verification assertion is available from the SAML deployment or from a local
policy determination.

Attributes that do not correspond to standard OpenID Connect claims MAY be
returned as private claims when both parties have a prior agreement on syntax
and semantics.

## UserInfo Responses {#userinfo-responses}

If an access token issued under this profile is presented to the OpenID
Provider's UserInfo endpoint, the OpenID Provider MUST process that request in
accordance with {{OIDC-CORE}} and the additional rules in this section.

The UserInfo response:

* MUST include the `sub` claim;
* MUST use a `sub` value consistent with {{subject-identifier-mapping}};
* MUST apply the claim mapping rules in {{authentication-event-claims}} and
  {{attribute-claims}} to any returned claims;
* MAY include the `sub_id` claim defined in {{saml-subject-identifier-claim}}
  when the granted scope includes `saml_subject` and policy permits
  release; and
* MUST NOT include claims that would not be releasable to the same relying
  party under this profile.

The OpenID Provider MUST determine which claims to return based on the granted
scope, any other OpenID Connect claim-selection mechanism it supports, and the
claim release policy applicable to the relying-party relationship represented by
the bound `saml_sp_entity_id`.

If an ID Token and a UserInfo response are both issued under the same
migrated authorization context, the `sub` value in the UserInfo response MUST
exactly match the `sub` value in the corresponding ID Token.

If the UserInfo response includes `auth_time`, `acr`, `amr`, or `sid`, those
values MUST be derived according to {{authentication-event-claims}}. The
same consistency rule applies to `sub_id` when it was emitted in the
corresponding ID Token under {{saml-subject-identifier-claim}}. When an ID
Token has been issued for the same migrated authorization context, the
authorization server MUST:

* cache the values of `auth_time`, `acr`, `amr`, `sid`, and `sub_id` at the
  time of ID Token issuance;
* return those cached values in any UserInfo response for the duration of the
  access token's validity, even if the originating SAML assertion has since
  expired; and
* ensure the UserInfo values for these claims match the corresponding claims
  in the most recently issued ID Token for that migrated authorization
  context.

This profile does not require the UserInfo response to repeat every claim
available in an ID Token, nor does it require claims omitted from the ID
Token to be present in UserInfo. Any claim returned from UserInfo under this
profile MUST reflect the same Local Account resolution, subject continuity,
and claim release policy used for the access token.

Any claim returned from UserInfo MUST be permitted by the claim release
policy in effect at the time of access token issuance and by the claim
release policy in effect at the time of the UserInfo request. The
authorization server MAY retain the issuance-time policy snapshot for
continuity, but MUST NOT use that snapshot to return claims disallowed by
current policy or by the current Local Account or session state.

# Session Termination and Revocation {#session-termination-and-revocation}

A token issued under this profile is anchored to a SAML-authenticated
session that may end before the token itself would otherwise expire.
This section defines how the authorization server connects session
termination signals to tokens it has issued, and how token revocation
under this profile cascades through derived artifacts. It does not
define a protocol for signaling SAML SLO between the SAML IdP and the
authorization server; that is out of scope.

SAML Single Logout (SLO) and OpenID Connect logout are not equivalent
mechanisms, and this profile does not guarantee parity between them.
SAML SLO is a multi-party protocol coordinated by the SAML IdP across
all participating SPs; OIDC logout (front-channel, back-channel, and
RP-Initiated Logout) targets OIDC clients individually. The operational
semantics, failure modes, and trust requirements differ. Deployments
that need coordinated logout across both protocols MUST design that
coordination outside this profile; this profile defines only the
authorization server's local obligations once it has learned that a
SAML-authenticated session has ended (see {{session-termination}}).

## Session Termination {#session-termination}

The authorization server MUST treat the earlier of the following as the
operative end of the migrated session for a given originating SAML
assertion:

* the assertion's `SessionNotOnOrAfter` value, when present in the
  authoritative `AuthnStatement` ({{authentication-event-claims}});
* any authoritative signal that the underlying SAML-authenticated
  session ended earlier (for example, a SAML SLO event observed at the
  IdP that the authorization server learns through deployment-specific
  means);
* an authorization server policy decision to terminate the migrated
  session (administrative action, risk-based revocation, etc.).

When the migrated session has ended, the authorization server:

* MUST reject further use of any refresh token derived from the
  originating assertion for session-preserving operations;
* SHOULD revoke or expire access tokens derived from those refresh
  tokens according to local capabilities; and
* MAY propagate the termination to OpenID Connect clients through
  {{OIDC-BC-LOGOUT}} when the deployment supports back-channel logout
  signaling, using the `sid` claim
  ({{authentication-event-claims}}) to identify the affected session.

This document does not define how the authorization server discovers
SAML SLO events. Deployments that require coordinated SAML and OIDC
logout MUST establish that discovery through deployment-specific means
outside this profile.

## Token Revocation {#token-revocation}

OAuth 2.0 Token Revocation {{RFC7009}} applies to tokens issued under
this profile without modification. Beyond the generic RFC 7009 rules,
this profile imposes the following cascade semantics:

* revocation of a refresh token issued under this profile MUST end the
  migrated session bound to that refresh token. The authorization
  server SHOULD revoke or expire all access tokens derived from the
  revoked refresh token according to local capabilities;
* revocation of an access token issued under this profile MUST end
  that token's usability but does not by itself terminate the migrated
  session or invalidate the underlying refresh token (if any).

The "migrated session bound to that refresh token" refers to the session
represented by the specific refresh token being revoked, not all sessions
for the same Local Account. Revoking one refresh token does not affect
other refresh tokens issued for the same Local Account (for example,
tokens issued for other devices) unless deployment policy explicitly
cascades. To end all sessions for a Local Account, the authorization
server MUST take the account-level action described in
{{session-termination}}.

If the authorization server learns that the resolved Local Account has
been disabled, deprovisioned, or suspended, it MUST end the migrated
session as described in {{session-termination}}.

A subsequent submission of a new SAML assertion through this profile
({{token-exchange-using-a-saml-assertion}}) establishes a new migrated
session that is not affected by the prior revocation.

# Security Considerations {#security-considerations}

## Threat Model {#threat-model}

This profile assumes the following:

* The SAML IdP and the authorization server are operated by, or under the
  administrative control of, the same authority (see
  {{applicability-and-deployment-model}}). They share a trust boundary,
  but are network-separated components that communicate through their
  respective protocols.
* Migration Clients are confidential clients (public clients are
  excluded; see {{client-registration-saml-sp-entity-id}}). Their client
  authentication credentials are not directly disclosed to end-users.
* The Migration Client's `saml_sp_entity_id` (and per-client
  `saml_idp_entity_id` if used) binding is established under the
  authorization rules in
  {{client-registration-saml-sp-entity-id-authorization}}; self-asserted
  dynamic registration of these values is disallowed.

The following attacker capabilities are in scope:

* An attacker who obtains a SAML assertion in transit (through XSS in
  the SP, intermediary compromise, log leakage, or similar). The
  three-way binding ({{migration-binding-rationale}}) and replay
  detection ({{replay-and-freshness}}) limit what such an assertion
  enables.
* An attacker who attempts to dynamically register a client with a
  `saml_sp_entity_id` or `saml_idp_entity_id` they are not authorized
  to bind. The registration authorization rules
  ({{client-registration-saml-sp-entity-id-authorization}}) require
  out-of-band authorization for these values.
* An attacker who attempts XML Signature Wrapping, replay, or
  algorithm-downgrade attacks against a submitted SAML assertion. The
  signature, replay, and algorithm rules in
  {{saml-input-validation-and-binding}} and the considerations below
  address these.
* An attacker who attempts to induce SSRF through a maliciously
  registered `saml_metadata_uri`. HTTPS-only fetches, redirect
  restrictions, and host allowlisting limit this.

The following are out of scope for this profile and remain the
deployment's responsibility:

* Compromise of the SAML IdP or the authorization server themselves
  (including their signing keys).
* Side channels, traffic analysis, and other infrastructure-level
  threats not specific to this profile.
* SAML SLO discovery: the mechanism by which the authorization server
  learns of a SAML session ending. Propagation from the authorization
  server to OpenID Connect clients via {{OIDC-BC-LOGOUT}} is OPTIONAL
  and addressed in {{session-termination-and-revocation}}.
* End-user account compromise upstream of the SAML IdP.

## SAML Assertion and Signature Risks {#assertion-signature-risks}

### IdP-Initiated SSO and Missing `InResponseTo` {#idp-initiated-sso}

When a SAML assertion is submitted without an `InResponseTo` element in its
`SubjectConfirmationData`, the authorization server cannot confirm that the
assertion was issued in response to a specific `<AuthnRequest>`. This
pattern is common in IdP-initiated SSO flows. Deployments SHOULD require
SP-initiated flows and SHOULD reject assertions that lack `InResponseTo`
unless the deployment threat model has explicitly assessed and accepted
the associated risks, including session fixation and assertion injection.

### XML Signature Wrapping {#xsw}

XML Signature Wrapping (XSW) attacks allow an adversary to manipulate a
SAML document by inserting a malicious element while retaining a valid
signature on a benign element elsewhere in the document structure.
Implementations MUST validate SAML signatures in a manner resistant to
XSW attacks. Specifically:

* when the authorization server relies on an assertion signature, it
  MUST verify that the element referenced by the signature's
  `Reference/@URI` is the same assertion element being processed; and
* when the authorization server relies on a signed SAML `Response`
  wrapper, it MUST verify that the element referenced by the
  signature's `Reference/@URI` is the same `Response` element being
  processed, and that the effective enclosed assertion selected under
  {{saml-input-validation-and-binding}} is the unique assertion bound
  to that validated wrapper.

The authorization server MUST reject any document structure where these
bindings cannot be confirmed unambiguously. Implementations SHOULD use
established SAML processing libraries that have been tested against
known XSW attack patterns.

### Deprecated Cryptographic Algorithms {#deprecated-algorithms}

Implementations MUST NOT accept SAML assertions signed using deprecated
cryptographic algorithms. Any signature algorithm using SHA-1 as the
message digest MUST be rejected, including RSA with SHA-1
(`http://www.w3.org/2000/09/xmldsig#rsa-sha1`), DSA with SHA-1
(`http://www.w3.org/2000/09/xmldsig#dsa-sha1`), and ECDSA with SHA-1
(`http://www.w3.org/2001/04/xmldsig-more#ecdsa-sha1`). Authorization
servers SHOULD require RSA with SHA-256 or stronger and SHOULD document
their minimum acceptable algorithm requirements.

Beyond SHA-1, deployments SHOULD reject other algorithms whose security
margins are no longer adequate by current cryptographic guidance,
including MD5-based digests, RSA signatures with key sizes below 2048
bits, and elliptic curves with effective security below 128 bits.
Authorization servers SHOULD track current guidance from bodies such as
NIST and ENISA and adjust their minimum-algorithm policy accordingly.
SHA-1 is named above because of its prevalence in legacy SAML deployments.

## Metadata Fetching and SSRF {#metadata-ssrf}

When an authorization server or migration tooling fetches a
`saml_metadata_uri` at either the authorization server metadata level or
the client registration level, the fetch target is derived from a
configuration value that may be attacker-influenced in some deployment
models. Per-client `saml_metadata_uri` values registered through dynamic
client registration are particularly exposed, since they originate from
a registering party. Implementations MUST restrict fetches to URIs that
use the HTTPS scheme and MUST NOT follow redirects to non-HTTPS URIs.
Deployments SHOULD additionally constrain permitted fetch targets to a
pre-approved allowlist of hosts, in order to prevent Server-Side Request
Forgery (SSRF) attacks against internal services. The registration
authorization rules in
{{client-registration-saml-sp-entity-id-authorization}} apply to
per-client `saml_metadata_uri` values, so an attacker using dynamic
client registration cannot freely set this value.

## Replay and Bearer Assertion Risks {#replay-and-bearer-risks}

SAML assertions used with Token Exchange or introspection are bearer
artifacts. Authorization servers MUST detect and prevent replay of the
same assertion according to the security properties of the underlying
SAML deployment, any SAML `OneTimeUse` condition, and the replay rules
in {{saml-input-validation-and-binding}}. Client authentication alone
is not sufficient protection against replay of a stolen assertion.

Authorization servers deployed across multiple nodes MUST share
assertion identifier state used for replay detection across all nodes,
or use equivalent distributed coordination. Per-node replay stores are
insufficient.

When this profile accepts a signed SAML `Response` as a wrapper for the
effective assertion, response-level signature validation does not by
itself validate response-level protocol fields such as `Destination` or
`Response/@InResponseTo`. This profile requires the wrapped `Response`
to carry a top-level SAML `StatusCode` of `Success` with no subordinate
status code, but other response-level protocol checks remain outside
this profile. Deployments that depend on those additional checks MUST
ensure they are performed outside this profile before the enclosed
assertion is used to issue tokens or return claims.

## Binding Security {#binding-security}

The bindings defined by `saml_sp_entity_id` and `saml_idp_entity_id`
are security-critical, both at the authorization server metadata level
and at the per-client registration level. If a client can register an
arbitrary `saml_sp_entity_id` or `saml_idp_entity_id`, it may be able
to exchange any SAML assertion captured for that SP-IdP pair into
OAuth tokens or claims under the bound relying-party relationship,
collapsing the three-way binding defined in
{{migration-binding-rationale}}. Authorization servers MUST therefore
protect registration and administrative mapping of both values
according to {{client-registration-saml-sp-entity-id-authorization}}.
In particular, anonymous or open dynamic client registration MUST NOT
be permitted to set either value.

When multiple OAuth clients share the same `saml_sp_entity_id`, the
authorization server is intentionally treating them as the same
relying-party context. Such sharing MUST therefore be an explicit IdP
or authorization server policy decision. Accidental sharing can expose
the same pairwise `sub` values and claims to clients that should have
been isolated from one another.

## Token and Claim Security {#token-and-claim-security}

Identifiers used as the OpenID Connect `sub` claim MUST satisfy the
stability rules in {{subject-identifier-common-rules}}; mutable,
transient, or reassignable `NameID` formats are excluded there
because their use as `sub` can cause account takeover or misbinding
after identifier reuse.

Incorrect `amr` translation can overstate the assurance of an
authentication. Implementations SHOULD pass through the SAML
`AuthnContextClassRef` as the `acr` value rather than attempting to
derive `amr` from it, and SHOULD emit `amr` values only when the
actual authentication methods are known.

The introspection alternative can expose normalized claims without
minting OAuth tokens. Authorization servers therefore MUST strongly
authenticate clients to the introspection endpoint and MUST return
`"active": false` when the client is not entitled to introspect the
presented SAML assertion.

When issuing an ID Token directly from Token Exchange, the
authorization server MUST ensure that the resulting token meets the
same signing, audience, issuer, and claim integrity requirements that
would apply if the token had been issued through an ordinary OpenID
Connect flow.

Access tokens issued under this profile are bearer tokens by default.
They are therefore subject to the usual risks associated with bearer
credentials, including replay by any party that obtains the token
value through interception, log leakage, or compromise of an
intermediary. Deployments protecting sensitive resources SHOULD
consider sender-constrained access tokens, such as DPoP-bound tokens
or mTLS client-certificate-bound tokens, in accordance with the
underlying OAuth sender-constraint specifications. This profile does
not preclude their use: a Migration Client MAY present a DPoP proof or
use mTLS client authentication at the token endpoint, and the
authorization server MAY issue sender-constrained access tokens in
the Token Exchange response when its policy and the client's
registration support that mechanism.

## Protocol Confusion and Transport Security {#protocol-confusion-and-transport}

Mixing SAML trust inputs and OAuth trust inputs creates a risk of
protocol confusion. Implementations MUST validate SAML assertions
using SAML metadata and MUST validate ID Tokens and other JOSE objects
using OAuth or OpenID Connect key discovery. Implementations MUST NOT
assume that a key published in one metadata format is automatically
valid in the other.

Authorization servers MUST apply transport security to the token
endpoint, introspection endpoint, metadata retrieval, and any other
endpoint used by this profile in the same manner as the underlying
OAuth, OpenID Connect, and SAML specifications that they extend.

# Privacy Considerations {#privacy-considerations}

This profile exists in part to preserve privacy properties during migration.
Deployments that previously relied on SAML pairwise subject identifiers SHOULD
continue to use pairwise identifiers after migration rather than silently
switching to public identifiers.

The `saml_sp_entity_id` parameter is important for privacy because it provides a
stable relying-party key for pairwise continuity. If a deployment instead
derives pairwise subjects from unrelated OAuth client metadata, the migrated
application can observe a different subject than it observed as a SAML SP,
forcing account relinking or introducing correlation errors.

When multiple OAuth clients share the same `saml_sp_entity_id`, they will
observe the same relying-party-specific subject continuity and claim release
under this profile. Deployments SHOULD do this only when those clients are
intended to represent the same historical relying-party context.

Authorization servers SHOULD release no more claims to the migrated client than
would have been available to the same relying party in the SAML deployment,
unless separate policy authorizes broader disclosure.

Private claim mappings can introduce new correlation vectors across migrated
clients. Deployments SHOULD therefore minimize use of private claims and SHOULD
avoid emitting values whose semantics were not already established for the same
relying-party relationship.

When the introspection pattern is used, the authorization
server SHOULD return only the minimum claims necessary for the authorized
client, because introspection can reveal identity data without the normal
indirection of token issuance.

Releasing the `sub_id` claim ({{saml-subject-identifier-claim}}) exposes
the original SAML subject identifier context, including the SAML IdP
entityID and any `NameID` qualifiers. This information is not normally
exposed through the `sub` claim, which carries the authorization
server's mapped value and may differ from the original SAML identifier.
Deployments SHOULD release `sub_id` only when the migrated client
genuinely needs the original SAML context (for example, for legacy
account linkage). A SAML `NameID` in `emailAddress` format carries
email-equivalent sensitivity and MUST be handled accordingly.

When multiple OAuth clients share the same `saml_sp_entity_id`, they
will observe the same `sub_id` value for the same Local Account, in
parallel with their already-shared `sub` continuity. This is consistent
with the relying-party context they share and does not introduce a new
correlation vector beyond what `sub` already exposes, but deployments
SHOULD ensure that this sharing is an intentional policy decision.

# IANA Considerations {#iana-considerations}

## OAuth Dynamic Client Registration Metadata {#iana-client-registration-metadata}

This document requests registration of the following values in the OAuth
Dynamic Client Registration Metadata registry established by {{RFC7591}}:

* Client Metadata Name: `saml_sp_entity_id`
* Client Metadata Description: Identifier of the SAML 2.0 Service Provider
  entity bound to the client for migration and subject continuity
* Change Controller: IETF
* Specification Document(s): This document, {{client-registration-saml-sp-entity-id}}

* Client Metadata Name: `saml_idp_entity_id`
* Client Metadata Description: Per-client override for the identifier of the
  SAML 2.0 Identity Provider entity expected to issue assertions for this
  client, when the SAML IdP emits per-SP entityIDs
* Change Controller: IETF
* Specification Document(s): This document, {{client-registration-saml-idp-entity-id}}

* Client Metadata Name: `saml_metadata_uri`
* Client Metadata Description: Per-client override for the HTTPS URI of the
  SAML metadata document describing the SAML 2.0 Identity Provider entity
  expected to issue assertions for this client
* Change Controller: IETF
* Specification Document(s): This document, {{client-registration-saml-metadata-uri}}

## OAuth Authorization Server Metadata {#iana-as-metadata}

This document requests registration of the following values in the OAuth
Authorization Server Metadata registry established by {{RFC8414}}:

* Metadata Name: `saml_idp_entity_id`
* Metadata Description: Default identifier of the SAML 2.0 Identity Provider
  entity bound to the authorization server issuer; may be overridden per
  client registration
* Change Controller: IETF
* Specification Document(s): This document, {{metadata-saml-idp-entity-id}}

* Metadata Name: `saml_metadata_uri`
* Metadata Description: Default HTTPS URI for the SAML metadata describing
  the SAML Identity Provider entity bound to the authorization server
  issuer; may be overridden per client registration
* Change Controller: IETF
* Specification Document(s): This document, {{metadata-saml-metadata-uri}}

* Metadata Name: `token_exchange_requested_token_types_supported`
* Metadata Description: JSON array of token type URIs accepted as
  `requested_token_type` values for SAML token exchange under this profile
* Change Controller: IETF
* Specification Document(s): This document, {{metadata-token-exchange-requested-token-types-supported}}

* Metadata Name: `introspection_token_types_supported`
* Metadata Description: JSON array of token type URIs identifying token types
  accepted for direct introspection at the authorization server's
  introspection endpoint according to this profile
* Change Controller: IETF
* Specification Document(s): This document, {{metadata-introspection-token-types-supported}}

## OAuth Token Type Hints {#iana-token-type-hints}

This document requests registration of the following value in the OAuth Token
Type Hints registry established by {{RFC7009}}. The token type URI below is
defined by {{RFC8693}} as a token type identifier; this registration makes it
available as a `token_type_hint` value at introspection and revocation
endpoints, which use a separate registry.

* Hint Value: `urn:ietf:params:oauth:token-type:saml2`
* Change Controller: IETF
* Specification Document(s): This document, {{introspection-request}}

## OAuth Token Introspection Response Registry {#iana-introspection-response-registry}

This document requests registration of the following values in the OAuth Token
Introspection Response registry established by {{RFC7662}}:

* Response Name: `saml`
* Response Description: JSON object containing normalized SAML protocol metadata
  extracted from the validated SAML input
* Change Controller: IETF
* Specification Document(s): This document, {{introspection-successful-response}}

* Response Name: `sub_id`
* Response Description: Structured subject identifier carrying provenance
  about the identifier, returned at the top level of the introspection
  response in parallel with its placement in ID Tokens and UserInfo
  responses
* Change Controller: IETF
* Specification Document(s): This document, {{introspection-successful-response}}, {{saml-subject-identifier-claim}}

* Response Name: `auth_time`
* Response Description: Time of end-user authentication, expressed as a
  NumericDate value, with the same semantics as the OpenID Connect
  `auth_time` claim; returned at the top level of the introspection
  response under this profile
* Change Controller: IETF
* Specification Document(s): This document, {{introspection-successful-response}}, {{authentication-event-claims}}

* Response Name: `acr`
* Response Description: Authentication Context Class Reference value
  with the same semantics as the OpenID Connect `acr` claim; returned
  at the top level of the introspection response under this profile
* Change Controller: IETF
* Specification Document(s): This document, {{introspection-successful-response}}, {{authentication-event-claims}}

* Response Name: `amr`
* Response Description: JSON array of Authentication Method Reference
  values with the same semantics as the OpenID Connect `amr` claim;
  returned at the top level of the introspection response under this
  profile
* Change Controller: IETF
* Specification Document(s): This document, {{introspection-successful-response}}, {{authentication-event-claims}}

* Response Name: `sid`
* Response Description: Session identifier with the same semantics as
  the OpenID Connect `sid` claim ({{OIDC-FC-LOGOUT}}, {{OIDC-BC-LOGOUT}});
  returned at the top level of the introspection response under this
  profile
* Change Controller: IETF
* Specification Document(s): This document, {{introspection-successful-response}}, {{authentication-event-claims}}

Other OpenID Connect standard claims listed in the IANA "JSON Web Token
Claims" registry (such as `email`, `email_verified`, `given_name`,
`family_name`, `name`, `preferred_username`, `phone_number`, and
`phone_number_verified`) MAY also appear as top-level members of the
introspection response under this profile when the claim mapping rules
in {{claim-mapping}} produce them. Their semantics in the introspection
response are the same as in the corresponding ID Token or UserInfo
response; this document does not redefine those claims and does not
introduce additional introspection-registry entries for them. A
deployment that includes such a claim in its introspection response is
relying on the existing JWT Claims registry definition.

## JSON Web Token Claims {#iana-jwt-claims}

This document requests registration of the following claim in the IANA
"JSON Web Token Claims" registry established by RFC 7519:

* Claim Name: `sub_id`
* Claim Description: Structured subject identifier carrying provenance
  about the identifier (such as the original SAML NameID context)
  alongside the primary `sub` claim
* Change Controller: IETF
* Reference: This document, {{saml-subject-identifier-claim}}

## OAuth Scope Values {#iana-oauth-scope-values}

This document defines the following OAuth scope value:

* Scope Value: `saml_subject`
* Description: Requests release of the `sub_id` claim defined in
  {{saml-subject-identifier-claim}}, carrying the original SAML
  `NameID` context for the migrated relying-party relationship
* Reference: This document, {{saml-subject-identifier-claim}}

## Security Event Identifier Formats {#iana-security-event-identifier-formats}

This document requests registration of the following value in the IANA
"Security Event Identifier Formats" registry established by {{RFC9493}}:

* Format Name: `saml-nameid`
* Description: Subject identifier format representing a SAML 2.0 `NameID`
  with its qualifiers and SAML Issuer
* Change Controller: IETF
* Reference: This document, {{saml-subject-identifier-claim}}

--- back

# Examples

This appendix is non-normative.

Unless otherwise noted, these examples use the `private_key_jwt` client
authentication method as defined by {{OIDC-CORE}}. Encoded SAML values and
client assertion JWTs are shortened placeholders.

The examples focus on the three primary deployment scenarios for this profile:
converting SAML input into an ID Token, delegating SAML validation to the
introspection endpoint, and converting SAML input into OAuth access tokens or
refresh tokens.

All examples assume a confidential client with the following registration:

~~~ json
{
  "client_name": "Calendar Example",
  "redirect_uris": [
    "https://calendar.example.com/callback"
  ],
  "token_endpoint_auth_method": "private_key_jwt",
  "token_endpoint_auth_signing_alg": "RS256",
  "jwks_uri": "https://calendar.example.com/jwks.json",
  "subject_type": "pairwise",
  "saml_sp_entity_id": "https://calendar.example.com/saml/sp"
}
~~~

Each request below carries `client_assertion_type` and `client_assertion`
parameters as the client authentication mechanism. The decoded
`client_assertion` JWT payload is shown once below; subsequent examples
reuse a shortened placeholder for the encoded JWT.

~~~ json
{
  "iss": "s6BhdRkqt3",
  "sub": "s6BhdRkqt3",
  "aud": "https://login.example.com/token",
  "jti": "5f3d8c2f-7e9a-4b1d-9c7e-0a1f2b3c4d5e",
  "exp": 1776804960,
  "iat": 1776804900
}
~~~

and an authorization server publishing the following relevant metadata:

~~~ json
{
  "issuer": "https://login.example.com",
  "authorization_endpoint": "https://login.example.com/authorize",
  "token_endpoint": "https://login.example.com/token",
  "userinfo_endpoint": "https://login.example.com/userinfo",
  "jwks_uri": "https://login.example.com/jwks.json",
  "introspection_endpoint": "https://login.example.com/introspect",
  "registration_endpoint": "https://login.example.com/register",
  "grant_types_supported": [
    "authorization_code",
    "refresh_token",
    "urn:ietf:params:oauth:grant-type:token-exchange"
  ],
  "saml_idp_entity_id": "https://login.example.com/idp",
  "saml_metadata_uri": "https://login.example.com/idp/metadata",
  "token_exchange_requested_token_types_supported": [
    "urn:ietf:params:oauth:token-type:refresh_token",
    "urn:ietf:params:oauth:token-type:id_token",
    "urn:ietf:params:oauth:token-type:access_token"
  ],
  "introspection_token_types_supported": [
    "urn:ietf:params:oauth:token-type:saml2"
  ]
}
~~~

## Convert SAML to an ID Token

The calendar application has completed SAML Web SSO and received a signed
assertion from the IdP for Alice, a user who already has a Local Account at
the authorization server. The application wants a signed OpenID Connect ID
Token it can verify locally and use to establish Alice's session, without
implementing a separate OIDC authorization code flow. It presents the SAML
assertion to the token endpoint via Token Exchange, requesting an ID Token
with `scope=openid profile email saml_subject` so that the ID Token also
carries the original SAML NameID context in `sub_id`. The authorization
server validates the assertion, resolves Alice's Local Account, derives
her pairwise `sub`, maps her authentication claims, and returns an ID
Token directly in the Token Exchange response.

~~~ http
POST /token HTTP/1.1
Host: login.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:id_token
&scope=openid+profile+email+saml_subject
&subject_token=PHNhbWwyOkFzc2VydGlvbiB4bWxuczpzYW1sMj0iLi4uIj4uLi48L3NhbWwyOkFzc2VydGlvbj4
&subject_token_type=urn:ietf:params:oauth:token-type:saml2
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=eyJhbGciOiJSUzI1NiIsImtpZCI6ImNsaWVudC0xIn0...
~~~

~~~ json
{
  "issued_token_type": "urn:ietf:params:oauth:token-type:id_token",
  "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjIyIn0...",
  "token_type": "N_A",
  "scope": "openid profile email saml_subject",
  "expires_in": 3600
}
~~~

The ID Token carried in the `access_token` response member might contain claims
like:

~~~ json
{
  "iss": "https://login.example.com",
  "sub": "p7b4cf5d-9c2f-4f22-a6b9-6e3d8df5a1b0",
  "sub_id": {
    "format": "saml-nameid",
    "issuer": "https://login.example.com/idp",
    "nameid": "alice-pairwise-7c3f",
    "nameid_format": "urn:oasis:names:tc:SAML:2.0:nameid-format:persistent",
    "name_qualifier": "https://login.example.com/idp",
    "sp_name_qualifier": "https://calendar.example.com/saml/sp"
  },
  "aud": "s6BhdRkqt3",
  "exp": 1776805200,
  "iat": 1776804900,
  "auth_time": 1776794400,
  "acr": "urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport",
  "sid": "op-sid-61b7d66f-4a6f-4f04-b0e5-9b8176d92ad0",
  "email": "alice@example.com",
  "given_name": "Alice",
  "family_name": "Ng"
}
~~~

## Delegate SAML Validation with Introspection

The calendar application wants to avoid implementing SAML XML signature
validation and assertion processing entirely. It uses an SP-initiated flow:
before redirecting Alice to the IdP, it generates an `AuthnRequest` and saves
local state including the request ID and ACS URL. When the IdP returns a signed
SAML Response to the ACS, the application submits it directly to the AS's
introspection endpoint rather than parsing it. The AS validates the XML
signatures, checks the assertion against all profile rules, and returns
normalized identity claims at the top level of the response alongside SAML
protocol metadata in a `saml` object. The application then correlates the
returned `saml` values against its stored request state to confirm the
response was intended for this flow, and uses the top-level identity claims
(`sub`, `email`, etc.) to establish Alice's session.

Before redirecting Alice to the IdP, the SP stores local request state such as:

~~~ json
{
  "request_id": "_sp-authnrequest-8f3a",
  "acs_url": "https://calendar.example.com/saml/acs",
  "sp_entity_id": "https://calendar.example.com/saml/sp",
  "idp_entity_id": "https://login.example.com/idp"
}
~~~

After the IdP returns a SAML Response to the ACS, the application submits it
to the introspection endpoint:

~~~ http
POST /introspect HTTP/1.1
Host: login.example.com
Content-Type: application/x-www-form-urlencoded

token=PHNhbWwycDpSZXNwb25zZSB4bWxuczpzYW1sMj0iLi4uIiB4bWxuczpzYW1sMnA9Ii4uLiI-Li4uPC9zYW1sMnA6UmVzcG9uc2U-
&token_type_hint=urn:ietf:params:oauth:token-type:saml2
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=eyJhbGciOiJSUzI1NiIsImtpZCI6ImNsaWVudC0xIn0...
~~~

~~~ json
{
  "active": true,
  "sub": "p7b4cf5d-9c2f-4f22-a6b9-6e3d8df5a1b0",
  "sub_id": {
    "format": "saml-nameid",
    "issuer": "https://login.example.com/idp",
    "nameid": "alice-pairwise-7c3f",
    "nameid_format": "urn:oasis:names:tc:SAML:2.0:nameid-format:persistent",
    "name_qualifier": "https://login.example.com/idp",
    "sp_name_qualifier": "https://calendar.example.com/saml/sp"
  },
  "auth_time": 1776794400,
  "acr": "urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport",
  "sid": "op-sid-61b7d66f-4a6f-4f04-b0e5-9b8176d92ad0",
  "email": "alice@example.com",
  "given_name": "Alice",
  "family_name": "Ng",
  "saml": {
    "response": {
      "id": "_d71b9f4f8b5b4a4b8f2f",
      "issuer": "https://login.example.com/idp",
      "issue_instant": "2026-04-21T18:00:00Z",
      "destination": "https://calendar.example.com/saml/acs",
      "in_response_to": "_sp-authnrequest-8f3a",
      "status_code": "urn:oasis:names:tc:SAML:2.0:status:Success",
      "has_nested_status_code": false
    },
    "assertion": {
      "id": "_a75adf55d9a24d6f8c2b",
      "issuer": "https://login.example.com/idp",
      "issue_instant": "2026-04-21T18:00:00Z",
      "audiences": [
        "https://calendar.example.com/saml/sp"
      ],
      "not_before": "2026-04-21T17:55:00Z",
      "not_on_or_after": "2026-04-21T18:05:00Z",
      "subject_confirmation": {
        "method": "urn:oasis:names:tc:SAML:2.0:cm:bearer",
        "recipient": "https://calendar.example.com/saml/acs",
        "in_response_to": "_sp-authnrequest-8f3a",
        "not_on_or_after": "2026-04-21T18:05:00Z"
      }
    },
    "attributes": [
      {
        "name": "urn:oid:0.9.2342.19200300.100.1.3",
        "name_format": "urn:oasis:names:tc:SAML:2.0:attrname-format:uri",
        "friendly_name": "mail",
        "values": ["alice@example.com"]
      },
      {
        "name": "urn:oid:2.5.4.42",
        "name_format": "urn:oasis:names:tc:SAML:2.0:attrname-format:uri",
        "friendly_name": "givenName",
        "values": ["Alice"]
      },
      {
        "name": "urn:oid:2.5.4.4",
        "name_format": "urn:oasis:names:tc:SAML:2.0:attrname-format:uri",
        "friendly_name": "sn",
        "values": ["Ng"]
      }
    ]
  }
}
~~~

After the authorization server validates XML signatures and SAML assertion
processing rules and returns the active response, the SP performs local
correlation and location checks over the returned JSON values, for example:

~~~ text
saml.response is present
saml.response.issuer == stored.idp_entity_id
saml.assertion.issuer == stored.idp_entity_id
saml.response.destination == stored.acs_url
saml.response.in_response_to == stored.request_id
saml.assertion.subject_confirmation.recipient == stored.acs_url
saml.assertion.subject_confirmation.in_response_to == stored.request_id
stored.sp_entity_id is in saml.assertion.audiences
saml.response.status_code == "urn:oasis:names:tc:SAML:2.0:status:Success"
saml.response.has_nested_status_code == false
current_time < saml.assertion.subject_confirmation.not_on_or_after
current_time < saml.assertion.not_on_or_after
~~~

## Convert SAML to an Access Token or Refresh Token

The calendar application holds a SAML assertion for Alice and needs OAuth
tokens for two different purposes: calling a protected payments API in the same
organization, and establishing a long-lived session that can outlive the SAML
assertion itself. Both are handled via Token Exchange; the `requested_token_type`
parameter selects which artifact the AS issues.

In the first case, the application needs a Bearer access token scoped to the
payments API. It uses resource indicators to identify the target resource and
requests only the scopes required for that service:

~~~ http
POST /token HTTP/1.1
Host: login.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:access_token
&scope=payments.read+payments.write
&resource=https%3A%2F%2Fapi.example.com%2Fpayments
&audience=payments-api
&subject_token=PHNhbWwyOkFzc2VydGlvbiB4bWxuczpzYW1sMj0iLi4uIj4uLi48L3NhbWwyOkFzc2VydGlvbj4
&subject_token_type=urn:ietf:params:oauth:token-type:saml2
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=eyJhbGciOiJSUzI1NiIsImtpZCI6ImNsaWVudC0xIn0...
~~~

~~~ json
{
  "issued_token_type": "urn:ietf:params:oauth:token-type:access_token",
  "access_token": "mF_9.B5f-4.1JqM",
  "token_type": "Bearer",
  "scope": "payments.read payments.write",
  "expires_in": 3600
}
~~~

In the second case, the application wants a refresh token to bootstrap a
long-lived migrated session. It includes `offline_access` in scope alongside
the OpenID Connect scopes it needs. The AS issues a refresh token bound to
Alice's Local Account; subsequent refresh token grants can produce access tokens
and ID Tokens without requiring a new SAML assertion:

~~~ http
POST /token HTTP/1.1
Host: login.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:token-type:refresh_token
&scope=openid+offline_access+profile+email
&subject_token=PHNhbWwycDpSZXNwb25zZSB4bWxuczpzYW1sMj0iLi4uIiB4bWxuczpzYW1sMnA9Ii4uLiI-Li4uPC9zYW1sMnA6UmVzcG9uc2U-
&subject_token_type=urn:ietf:params:oauth:token-type:saml2
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=eyJhbGciOiJSUzI1NiIsImtpZCI6ImNsaWVudC0xIn0...
~~~

~~~ json
{
  "issued_token_type": "urn:ietf:params:oauth:token-type:refresh_token",
  "access_token": "vF9dft4qmTcXkZ26zL8b6u",
  "token_type": "N_A",
  "scope": "openid offline_access profile email",
  "expires_in": 1209600
}
~~~

# Attribute Mapping Examples {#incommon-and-eduperson-deployment-guidance}

This appendix is non-normative.

## Common SAML Attribute Mappings

Many SAML deployments use a small set of attribute names that map
directly to registered OpenID Connect claims. Common examples:

| SAML attribute (friendly name) | OpenID Connect claim |
| --- | --- |
| `givenName` (or `given_name`) | `given_name` |
| `sn`, `surname`, or `family_name` | `family_name` |
| `displayName` (or `name`) | `name` |
| `mail` (or `email`) | `email` |
| `uid` (or `preferred_username`) | `preferred_username` |
| `telephoneNumber` (or `phone_number`) | `phone_number` |

The rules in {{attribute-claims}} (in particular, the NameFormat
preference rule preferring `urn:oasis:names:tc:SAML:2.0:attrname-format:uri`
over `:basic`/`:unspecified`) govern how these are resolved when the
assertion uses `urn:oid:` Name values alongside friendly names.

## InCommon and eduPerson Deployments

Many InCommon deployments use `eduPerson` attributes, including the
REFEDS Research and Scholarship attribute bundle. InCommon deployments
commonly use `urn:oid:` Name values in addition to friendly names; the
mapping rules in {{attribute-claims}} govern how such OID-based names
are resolved to OpenID Connect claims. The core rules in
{{claim-mapping}} still govern claim typing, precedence, and subject
handling. Within that framework, deployments commonly use the following
mappings and constraints:

* `mail` to `email`
* `displayName` to `name`
* `givenName` to `given_name`
* `sn` or `surname` to `family_name`
* `eduPersonNickname` to `nickname`, when locally useful
* `telephoneNumber` to `phone_number`, when locally useful
* `eduPersonPrincipalName` to `preferred_username` only when local policy
  permits release as a user-facing identifier; it should not determine `sub`
* `eduPersonScopedAffiliation` in a collision-resistant private claim that
  preserves its multi-valued syntax
* `eduPersonAffiliation` and `eduPersonPrimaryAffiliation` in collision-
  resistant private claims rather than overloaded standard claims
* `eduPersonEntitlement` in a collision-resistant private claim; it should not
  be translated directly into OAuth `scope`
* `eduPersonAssurance` in a collision-resistant private claim or as input to
  local assurance policy; it should not be copied directly into `amr` and
  should not overwrite `acr` unless local policy explicitly defines that
  translation
* `eduPersonUniqueId` as a stable account-linking or subject-continuity input,
  and only secondarily as a private claim when needed
* `eduPersonTargetedID` as a legacy pairwise subject continuity input, not as a
  general-purpose user-facing claim
* `eduPersonOrcid` in a collision-resistant private claim
* `eduPersonPrincipalNamePrior` only by explicit policy for relying parties that
  require historical identifier information

For InCommon-style bundles used only to preserve legacy application behavior,
the authorization server SHOULD prefer standard OpenID Connect claims when
there is a good semantic match and SHOULD retain the original attribute
semantics in private claims rather than mapping them to unrelated standard
claims such as `groups`, `roles`, or `scope`.

# Authentication Context Mapping Examples {#authentication-context-mapping-examples}

This appendix is non-normative.

The `acr` and `amr` mapping rules in {{authentication-event-claims}}
defer to local policy for the choice of registered Level-of-Assurance
scheme. This appendix provides example mappings from common SAML
`AuthnContextClassRef` URIs to OpenID Connect `acr` values and to `amr`
values from the IANA "Authentication Method Reference Values" registry
{{RFC8176}}. Deployments MAY adopt these mappings, adjust them, or
define their own as local policy dictates.

## Common SAML AuthnContextClassRef URIs

| SAML AuthnContextClassRef | Suggested `amr` | Suggested `acr` (NIST 800-63) | Suggested `acr` (REFEDS) |
| --- | --- | --- | --- |
| `urn:oasis:names:tc:SAML:2.0:ac:classes:Password` | `["pwd"]` | aal1 | `https://refeds.org/assurance/IAP/low` |
| `urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport` | `["pwd"]` | aal1 | `https://refeds.org/assurance/IAP/low` |
| `urn:oasis:names:tc:SAML:2.0:ac:classes:TLSClient` | `["mfa", "swk"]` (when client cert + password) | aal2 | `https://refeds.org/assurance/IAP/medium` |
| `urn:oasis:names:tc:SAML:2.0:ac:classes:X509` | (single factor) `["swk"]` or `["hwk"]` depending on key storage | varies | varies |
| `urn:oasis:names:tc:SAML:2.0:ac:classes:Smartcard` | `["sc"]` | varies | varies |
| `urn:oasis:names:tc:SAML:2.0:ac:classes:SmartcardPKI` | `["sc", "hwk"]` | aal2 or aal3 | `https://refeds.org/assurance/IAP/medium` or higher |
| `urn:oasis:names:tc:SAML:2.0:ac:classes:Kerberos` | `["pwd"]` (typically Kerberos backed by password) | aal1 | `https://refeds.org/assurance/IAP/low` |
| `urn:oasis:names:tc:SAML:2.0:ac:classes:TimeSyncToken` | `["otp"]` | aal2 | `https://refeds.org/assurance/IAP/medium` |
| `urn:oasis:names:tc:SAML:2.0:ac:classes:MultiFactorContract` | `["mfa"]` | aal2 | `https://refeds.org/assurance/IAP/medium` |
| `urn:oasis:names:tc:SAML:2.0:ac:classes:MultiFactorPhysicalContract` | `["mfa", "hwk"]` | aal3 | `https://refeds.org/assurance/IAP/high` |
| `urn:oasis:names:tc:SAML:2.0:ac:classes:PreviousSession` | (none; SAML deployment-specific) | (none) | (none) |
| `urn:oasis:names:tc:SAML:2.0:ac:classes:unspecified` | (none) | (none) | (none) |

## Notes on these mappings

* `amr` is a JSON array; multi-factor SAML classes naturally produce
  multiple values (typically including `mfa`).
* `acr` values shown for NIST 800-63 are strings of the form `"aal1"`,
  `"aal2"`, `"aal3"`. Deployments MAY use URI forms such as those
  defined by REFEDS for federated higher-education contexts.
* The Smartcard / SmartcardPKI / X509 rows depend heavily on key
  storage (software vs. hardware-bound) and whether a second factor
  (PIN, password) is also asserted. Deployments SHOULD configure the
  mapping based on the IdP's actual authentication setup, not the
  SAML class URI alone.
* The Kerberos row reflects that most enterprise Kerberos deployments
  are password-derived; deployments using stronger Kerberos factors
  (smart card initial credentials, etc.) SHOULD map accordingly.
* `PreviousSession` and `unspecified` provide no derivable assurance
  signal; `acr` SHOULD be omitted, and `amr` SHOULD NOT be populated
  from them.
* When the SAML deployment uses non-standard `AuthnContextClassRef`
  URIs (vendor-specific, IdP-specific, or federation-defined), the
  authorization server MAY pass the URI through to `acr` per
  {{authentication-event-claims}}, with `amr` populated only when the
  local mapping explicitly authorizes specific values.

## Pass-through and absence

When local policy does not provide a mapping for an asserted
`AuthnContextClassRef`, the authorization server SHOULD copy the SAML
URI to `acr` without transformation per
{{authentication-event-claims}}. The authorization server MUST NOT
populate `amr` in this case, since deriving `amr` from an unmapped
class URI would overstate the assurance.

# Acknowledgments
{:numbered="false"}

This draft was informed by the SAML interoperability work in the OAuth working
group identity assertion draft and by the deployed semantics of the OASIS SAML
subject identifier profile.
