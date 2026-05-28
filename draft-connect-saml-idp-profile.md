---
title: "OpenID Connect Provider Profile for SAML 2.0 Identity Providers"
abbrev: "OIDC Provider for SAML IdPs"
category: info
docname: draft-connect-saml-idp-profile-latest
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
  RFC6749:
  RFC7636:
  RFC8174:
  RFC8176:
  RFC8414:
  RFC9493:
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
  SAML-OIDC-BRIDGE:
    title: "OpenID Connect Bridge Profile for SAML 2.0 Service Providers"
    target: "https://mcguinness.github.io/connect-saml-profiles/#go.draft-connect-saml-bridge-profile.html"
    author:
      - ins: K. McGuinness
    date: false

...

--- abstract

This document defines an interoperability profile for an OpenID Provider
(OP) that delegates end-user authentication to a SAML 2.0 Identity Provider.
The OP exposes a standard OpenID Connect interface to relying parties while
acting as a SAML Service Provider toward an upstream SAML IdP, translating
SAML authentication results into OpenID Connect ID Tokens, OAuth 2.0 access
tokens, and normalized claims.

The intent is to let an organization stand up an OP and onboard OpenID
Connect relying parties while a SAML 2.0 IdP remains the authoritative
authentication source. The OP preserves SAML subject identifiers,
authentication context, and attribute release policy through to the OIDC
layer, so an RP onboarded against this OP sees the same end-user identity
it would see from any other OIDC OP.

This profile is the upstream-facing companion to the OpenID Connect
Migration Profile for SAML 2.0 Service Providers {{SAML-OIDC-MIGRATION}}
and the OpenID Connect Bridge Profile for SAML 2.0 Service Providers
{{SAML-OIDC-BRIDGE}}. Together the three profiles support a phased
deprecation of SAML: this profile enables OP adoption while SAML remains
authoritative; the Bridge profile lets the OP serve remaining SAML SPs
once it becomes authoritative; the Migration profile lets individual SAML
SPs become OIDC clients on their own timeline.

--- middle

# Introduction {#introduction}

Many organizations have a SAML 2.0 IdP as their authoritative
authentication source but want to expose OpenID Connect to relying
parties. Reasons include supporting modern OIDC applications,
consolidating identity onto a single OAuth 2.0 / OpenID Connect
surface, and beginning a phased SAML deprecation. The practical
question in each case is whether an OP can be stood up before the
SAML IdP is retired.

This profile answers that question. It defines an OP that acts as a thin
translation layer: it receives standard OIDC authorization requests from
relying parties, performs SAML 2.0 Web Browser SSO upstream against the
SAML IdP, and returns OIDC tokens. The SAML IdP remains the single
authentication authority; the OP does not introduce new authentication
mechanisms or new trust anchors.

The result is a set of patterns that address recurring needs at the
beginning of a SAML deprecation:

* **Standing up an OP while SAML remains authoritative.** An organization
  can deploy an OP and onboard OIDC relying parties without first retiring
  its SAML IdP. The downstream SAML IdP, federation trust, and existing
  SAML SPs continue to operate unchanged.

* **Preserving subject continuity into OIDC.** OIDC `sub` values delivered
  to relying parties are derived from the SAML NameID for the same user,
  so an RP that has previously seen a user as a SAML SP sees the same
  logical subject through the OP.

* **Pairing with the companion profiles.** When combined with the Bridge
  profile {{SAML-OIDC-BRIDGE}}, the OP can serve both OIDC RPs and
  remaining SAML SPs from a single deployment. When combined with the
  Migration profile {{SAML-OIDC-MIGRATION}}, individual SAML SPs can be
  modernized to OIDC clients on their own timeline.

This profile:

* defines a deployment model for an OP that authenticates by acting as a
  SAML SP toward an upstream SAML IdP;
* defines per-IdP-Binding configuration that records the SAML IdP entity,
  metadata, and authentication context policy;
* defines rules for validating SAML assertions consumed by the OP and
  resolving them to a Local Account;
* defines rules for translating SAML NameIDs into OIDC subject identifiers
  preserving SAML SP-equivalent continuity; and
* defines rules for translating SAML attributes and authentication
  context into OIDC claims and `acr`.

## Overview {#overview}

This profile involves three actors:

* **OIDC RP** -- an OpenID Connect Relying Party that uses the OP
  for authentication.
* **OP** -- the OpenID Provider defined by this profile. The OP
  acts as an OIDC AS toward RPs and as a SAML SP toward the
  upstream SAML IdP.
* **Upstream SAML IdP** -- the SAML 2.0 Identity Provider that
  authenticates end-users. Unmodified by this profile.

~~~
   +-------+               +---------+              +----------+
   |       |--(A)-authzReq>|         |---(B)------->|          |
   |       |               |         | AuthnRequest | Upstream |
   | OIDC  |               |   OP    |              |   SAML   |
   |   RP  |               |         |<--(C)--------|   IdP    |
   |       |<--(D)-tokens--|         |   SAML Resp  |          |
   +-------+               +---------+              +----------+
~~~

(A) An OIDC RP redirects the end-user to the OP's authorization
    endpoint with a standard OIDC authorization request.

(B) The OP has no SAML-authenticated session for the user. It
    selects an IdP-Binding ({{idp-binding-selection}}), constructs
    a signed SAML `AuthnRequest`
    ({{authn-request-construction}}), and redirects the browser
    to the upstream SAML IdP.

(C) The upstream SAML IdP authenticates the end-user and POSTs a
    signed SAML `Response` to the OP's ACS endpoint. The OP
    validates the assertion ({{saml-assertion-validation}}),
    resolves the user to a Local Account
    ({{local-subject-resolution}}), derives a stable OIDC `sub`
    ({{subject-identifier-mapping}}), and maps SAML attributes
    and authentication context to OIDC claims
    ({{claim-mapping}}).

(D) The OP returns an OIDC authorization response to the RP.
    Subsequent token and UserInfo requests follow {{OIDC-CORE}}.

## Non-Goals {#non-goals}

The profile is scoped to OP-side SAML federation and identity continuity.
It does NOT define:

* a generalized SAML 2.0 SP, including SOAP attribute query, ECP, Holder-
  of-Key SubjectConfirmation, or SP-specific SAML extensions;
* federation establishment between otherwise independent OIDC and SAML
  parties; the OP and the SAML IdP are assumed to be under common
  administrative control;
* migration methodology (rollout patterns, cutover sequencing, SAML IdP
  retirement);
* account-linkage bootstrap (how a Local Account is established and bound
  to a SAML NameID before this profile's first run);
* provisioning, SCIM integration, or account-lifecycle workflows;
* API authorization architectures, scope design, or resource-server
  policy;
* governance models (approval, attestation, audit, segregation of duties);
* encrypted SAML element processing beyond decrypting an
  `EncryptedAssertion` addressed to the OP-as-SP role (see
  {{accepted-saml-inputs}}); `EncryptedID` and `EncryptedAttribute` are
  out of scope; or
* a Token Exchange variant. Clients that already hold a SAML
  assertion and want to exchange it for OIDC tokens use
  {{SAML-OIDC-MIGRATION}} instead.

# Conventions and Terminology

## Requirements Notation

{::boilerplate bcp14-tagged}

## Terminology

This document uses the terms "authorization server", "client", "client
identifier", "issuer", and "scope" from OAuth 2.0, OAuth 2.0 Authorization
Server Metadata, and OpenID Connect. It uses the terms "Identity Provider",
"Service Provider", "entityID", "Assertion", "Attribute",
"AttributeStatement", "AuthnStatement", "AuthnContext", "Web Browser SSO",
and "AssertionConsumerService" (the "ACS endpoint") from SAML 2.0.

For the purposes of this document:

OpenID Provider (OP):
: The OAuth 2.0 / OpenID Connect authorization server defined by this
  profile. The OP exposes an OIDC interface to relying parties while
  delegating end-user authentication to an upstream SAML IdP.

SAML IdP:
: The upstream SAML 2.0 Identity Provider that issues SAML assertions
  consumed by the OP. The SAML IdP is the authoritative authentication
  source under this profile.

OP-as-SP:
: The OP's SAML Service Provider role at the upstream SAML IdP. The OP-as-SP
  has its own SAML SP Entity ID, ACS endpoint, and signing key material
  published in SAML SP metadata. References to "the OP's ACS endpoint" or
  "the OP's SAML SP Entity ID" refer to this role.

IdP-Binding:
: The per-SAML-IdP administrative configuration at the OP that records the
  relationship between an upstream SAML IdP and the OP's OIDC layer,
  including SAML IdP entity ID, SAML metadata location, NameID format
  policy, attribute mapping policy, and `AuthnContextClassRef` mapping
  policy.

Local Account:
: The OP's internal account record for the end-user. Subject continuity at
  the OP requires that a given (SAML Issuer, NameID) pair can be resolved
  to one Local Account, and that the resulting OIDC `sub` is stable for
  that Local Account for each relying party context.

# Applicability and Deployment Model {#applicability}

This profile assumes that the OP and the SAML IdP are operated by,
or under the administrative control of, the same authority. It is
not intended to establish federation trust between otherwise
independent OIDC and SAML parties.

The OP can issue OIDC tokens preserving the subject identifiers and
claim release policy already established at the SAML IdP, without
requiring changes to the SAML IdP or its existing SAML SP
relationships. {{binding-and-issuance-model}} defines the binding
mechanism in detail.

The profile is applicable only when the OP can resolve each
SAML-authenticated end-user deterministically to exactly one active
Local Account before issuing OIDC tokens
({{local-subject-resolution}}).

# Binding and Issuance Model {#binding-and-issuance-model}

This profile uses the SAML IdP Entity ID and the OP-as-SP Entity ID as
the binding keys between the SAML and OIDC layers. The OP applies its
IdP-Binding policy by indexing on the SAML Issuer of the received
assertion, not by inspecting the SAML payload alone.

The primary guarantee against cross-IdP assertion injection is a
three-way binding among:

* the OP-as-SP Entity ID, included as the assertion's `AudienceRestriction`;
* the OP's ACS endpoint, included as the assertion's `Recipient`; and
* the SAML AuthnRequest ID the OP issued, included as
  `Response/@InResponseTo` and `SubjectConfirmationData/@InResponseTo`.

Together these ensure that the OP only accepts SAML assertions that were
issued in response to its own AuthnRequest, addressed to its own SP
entity and ACS endpoint, by a SAML IdP it has configured. The OP then
applies the IdP-Binding's subject-mapping and attribute-release policy
before issuing OIDC tokens.

## Authorization Basis and Release Policy {#authorization-assertion-model}

The bindings above are necessary but not sufficient for token
issuance. This subsection separates authentication evidence (what
the SAML assertion proves) from authorization decisions (what the
OP releases).

The OP MUST NOT treat a valid SAML assertion as authorization to
release arbitrary OIDC claims or to grant arbitrary OAuth scopes.
The SAML assertion proves an authentication event and supplies
identity inputs; the authorization basis for any OIDC token
issuance remains an OP policy decision.

Before issuing an OIDC token or releasing OIDC claims, the OP MUST have
an authorization basis that is bound to:

* the authenticated relying-party client at the OP's OIDC layer;
* the requested OIDC scopes and token types; and
* the resolved Local Account.

The OP MUST NOT create, expand, or infer an authorization basis solely
from the presence or contents of the SAML assertion. SAML attributes MAY
inform an already-established release policy rule, but the policy itself
MUST exist independently of the SAML assertion presented in the current
authentication.

The OP MUST NOT release all SAML attributes to all OIDC RPs by default.
Deployments MUST configure explicit attribute release allowlists per OIDC
client (or per RP group), independent of what the SAML IdP releases to
the OP.

# IdP-Binding Configuration {#idp-binding-configuration}

Each upstream SAML IdP that the OP federates with has an IdP-Binding
maintained administratively at the OP. An IdP-Binding MUST record at
minimum:

* the SAML IdP Entity ID;
* a SAML metadata reference (HTTPS URI or administratively configured
  metadata document);
* the OP-as-SP Entity ID used at this IdP (see {{op-saml-metadata}});
* the NameID format policy for assertions from this IdP (see
  {{nameid-policy}});
* the attribute mapping policy that translates SAML attributes from this
  IdP into OIDC claims;
* the `AuthnContextClassRef` mapping policy that translates SAML
  authentication context into OIDC `acr` values; and
* the set of OIDC clients (or client groups) at the OP that are permitted
  to consume identities sourced from this IdP.

The OP MUST validate that the Issuer of any incoming SAML assertion
corresponds to a configured IdP-Binding before processing further. SAML
assertions whose Issuer is not configured MUST be rejected.

Multiple IdP-Bindings MAY coexist at the same OP. When more than one
IdP-Binding is configured, the OP MUST determine which IdP-Binding applies
to a given authentication based on a deterministic selector
(administrative routing, OIDC `acr_values` request parameter,
`login_hint`, or equivalent). The OP MUST NOT allow OIDC clients to
arbitrarily select an IdP-Binding without administrative authorization.

## Declaring the Backing SAML IdP {#saml-idp-entity-id-reuse}

This document reuses the `saml_idp_entity_id` authorization server
metadata parameter defined by {{SAML-OIDC-MIGRATION}}. In this profile,
when the OP publishes `saml_idp_entity_id` in its OAuth/OIDC metadata, its
value identifies the SAML IdP that backs the OP. Deployments with multiple
IdP-Bindings MUST NOT publish a single `saml_idp_entity_id` value; in
those deployments the per-IdP binding is selected at runtime and is not
advertised in the OP's published metadata. See {{multi-idp-deployments}}.

## NameID Format Policy {#nameid-policy}

Each IdP-Binding MUST specify the SAML NameID format(s) it expects from
the IdP and how each is translated into the OP's OIDC subject identifier.
The following formats are recognized under this profile:

* `urn:oasis:names:tc:SAML:2.0:nameid-format:persistent`: A stable,
  non-reassignable identifier. Mapped to OIDC `sub` per
  {{subject-identifier-mapping}}.
* `urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress`: An email
  address. The OP MUST NOT use this value as the persistent `sub` unless
  the IdP-Binding explicitly permits it and a separate stable identifier
  is unavailable; see {{nameid-stability}}.
* `urn:oasis:names:tc:SAML:2.0:nameid-format:transient`: A session-scoped
  random identifier. The OP MUST NOT use a transient NameID as input to
  `sub` derivation. When the SAML IdP issues only a transient NameID, a
  stable identifier MUST be released as a SAML attribute (for example,
  `urn:oasis:names:tc:SAML:attribute:subject-id` from {{SAML2-SUBJ-ID}})
  and the OP MUST use that attribute as the stable subject input; the
  OP MUST reject the assertion if no such attribute is present.
* `urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified`: Deployment-
  defined semantics. SHOULD NOT be used for new IdP-Bindings; supported
  only for compatibility with SAML IdPs that historically emit this
  format, in which case the IdP-Binding MUST define the identifier
  semantics explicitly.

The OP MAY include the SAML V2.0 `subject-id` or `pairwise-id` attribute
from {{SAML2-SUBJ-ID}} as the preferred stable subject identifier when
the SAML IdP supports that profile.

If `NameIDPolicy` is sent in the OP-issued SAML AuthnRequest, its value
MUST be drawn from the IdP-Binding's configured policy and MUST NOT be
controlled by OIDC client request parameters.

# OP Metadata {#op-metadata}

## SAML SP Metadata {#op-saml-metadata}

The OP MUST expose a SAML `EntityDescriptor` as defined by
{{SAML2-METADATA}} with the OP-as-SP Entity ID as its `entityID`. This
metadata:

* MUST include an `SPSSODescriptor` element;
* MUST include at least one `AssertionConsumerService` endpoint using the
  HTTP-POST binding;
* MUST set `SPSSODescriptor/@AuthnRequestsSigned="true"`, reflecting the
  default-MUST signature requirement on AuthnRequests under this profile
  (see {{authn-request-construction}});
* MUST set `SPSSODescriptor/@WantAssertionsSigned="true"`, reflecting the
  signed-assertion requirement under this profile (see
  {{accepted-saml-inputs}});
* MUST include the key material the OP uses to sign SAML AuthnRequests in
  `KeyDescriptor` elements with `use="signing"`;
* MUST include the key material the OP uses to decrypt encrypted SAML
  elements (if any) in `KeyDescriptor` elements with `use="encryption"`;
* MUST be accessible at an HTTPS URI the OP can publish and maintain; and
* SHOULD be updated whenever the OP's SAML key material changes.

The OP's SAML SP signing and encryption keys MUST NOT be confused with
the OP's JOSE signing keys published at `jwks_uri`. Each set of keys is
discovered through its own protocol's metadata.

## OAuth Authorization Server Metadata {#op-as-metadata}

The OP MUST publish OAuth 2.0 authorization server metadata at the
`.well-known/oauth-authorization-server` location defined by {{RFC8414}}
relative to its issuer identifier, and SHOULD additionally publish at
`.well-known/openid-configuration` per {{OIDC-DISCOVERY}}.

The `saml_idp_entity_id` and `saml_metadata_uri` parameters defined by
{{SAML-OIDC-MIGRATION}} MAY be published in the OP's AS metadata to
declare the OP's upstream SAML IdP relationship. See
{{saml-idp-entity-id-reuse}} for restrictions when multiple IdP-Bindings
are configured.

The OP MAY publish the SAML SP metadata location (its
`EntityDescriptor` URI) using an `AdditionalMetadataLocation` element in
the SAML metadata document, paired with a `saml_metadata_uri` pointer in
the AS metadata, to allow administrators and federation registries to
traverse from either side to the other.

## Capability and Metadata Matrix {#op-capability-matrix}

| Capability | Required metadata | Notes |
| --- | --- | --- |
| OIDC OP interface | OAuth/OIDC AS metadata per {{RFC8414}} / {{OIDC-DISCOVERY}} | Standard OAuth/OIDC discovery. |
| SAML SP role at upstream IdP | SAML `EntityDescriptor` with `SPSSODescriptor` | The OP-as-SP entity ID, ACS endpoint, and key material. |
| Upstream IdP binding (declared) | `saml_idp_entity_id` | Optional. Identifies the SAML IdP backing this OP; omitted in multi-IdP deployments. |
| Upstream IdP metadata pointer | `saml_metadata_uri` | Optional. Points to SAML metadata for the upstream IdP. |
| ACR values producible | `acr_values_supported` | SHOULD list the OIDC `acr` values the OP can produce after applying the IdP-Binding's ACR mapping policy ({{acr-mapping}}). |

## Federation Metadata Coexistence {#federation-metadata-coexistence}

An OP under this profile operates two protocol stacks in parallel.
OAuth 2.0 and OpenID Connect toward relying parties use
`.well-known/oauth-authorization-server` and
`.well-known/openid-configuration` ({{RFC8414}}, {{OIDC-DISCOVERY}}).
SAML 2.0 toward the upstream SAML IdP uses `EntityDescriptor`
documents ({{SAML2-METADATA}}). This profile does not unify the two.
Deployments SHOULD observe the following.

* The OP's OAuth `issuer`, the OP-as-SP SAML Entity ID, and the upstream
  SAML IdP Entity ID are distinct values used in their own protocol
  contexts; incidental string equality does not imply identity.
* JOSE signing keys (`jwks_uri`) and SAML signing/encryption keys
  (`KeyDescriptor`) are discovered through their respective protocol
  metadata. Cross-protocol key reuse is permitted, but each set MUST be
  published in the metadata of the protocol that uses it.
* The bidirectional pointer described in {{op-as-metadata}} (SAML
  `AdditionalMetadataLocation` <-> AS `saml_metadata_uri`) is
  OPTIONAL but RECOMMENDED so that administrators and federation
  registries can traverse from either side to the other.

# SAML Assertion Validation {#saml-assertion-validation}

This section applies whenever the OP consumes a SAML assertion under this
profile. SAML assertions arrive at the OP's ACS endpoint as part of the
Web Browser SSO flow defined in {{op-authentication-flow}}.

## Accepted SAML Inputs {#accepted-saml-inputs}

The OP MUST only accept SAML assertions issued by SAML IdPs
identified by a configured IdP-Binding. Other issuers MUST be
rejected.

The submitted input MUST be a signed SAML `Response` containing
exactly one signed bearer `Assertion`. The OP MUST validate the
signatures and the assertion content per {{SAML2-CORE}} and
{{SAML2-PROFILES}} against the IdP-Binding's metadata. Validation
MUST be resistant to XML Signature Wrapping ({{xsw-incoming}}).

This profile adds the following constraints:

* `Response/Issuer` and `Assertion/Issuer` MUST equal the configured
  SAML IdP Entity ID;
* `SubjectConfirmationData/@Recipient` and `Response/@Destination`
  MUST equal the OP's ACS URL;
* the OP-as-SP Entity ID MUST appear in every `AudienceRestriction`
  element;
* `SubjectConfirmationData/@InResponseTo` MUST match the
  `AuthnRequest/@ID` issued by the OP for this flow;
* signatures MUST use RSA-SHA-256 or stronger; SHA-1 MUST NOT be
  accepted (see {{outgoing-signature-algorithms}}).

The OP MAY accept `EncryptedAssertion` when it publishes encryption
key material in its SAML SP metadata. The OP MUST decrypt before
validation. The inner `Assertion` MUST be signed independently of
any `Response`-level signature. `EncryptedID` and
`EncryptedAttribute` MUST be rejected.

## Replay and Freshness {#replay-and-freshness}

The OP MUST detect and reject replay of a previously accepted SAML
assertion. The OP MUST track consumed assertion IDs per (SAML IdP
Issuer, `Assertion/@ID`) until the assertion's
`Conditions/@NotOnOrAfter` (or `SubjectConfirmationData/@NotOnOrAfter`,
whichever is later) has passed.

The OP MUST track consumed SAML AuthnRequest IDs and MUST reject a
`Response/@InResponseTo` that does not match an outstanding (sent but
not yet completed) AuthnRequest issued by the OP for the current OIDC
authorization flow. An assertion received with an unknown or already-
consumed `InResponseTo` MUST be rejected.

The OP MUST verify the assertion's `IssueInstant` falls within a
deployment-configured freshness window. In the absence of stricter local
policy, 5 minutes is RECOMMENDED. Assertions outside the window MUST be
rejected.

OPs deployed across multiple nodes MUST share assertion-replay state and
consumed AuthnRequest state across all nodes, or use equivalent
distributed coordination. Pending AuthnRequest correlation context MAY be
carried using {{stateless-correlation}}, but consumed identifiers still
require shared replay protection. Per-node replay stores are insufficient.

## Required Assertion Content {#required-assertion-content}

The OP MUST NOT issue OIDC tokens unless the assertion contains:

* an `AuthnStatement` with `AuthnInstant` and an `AuthnContext` containing
  an `AuthnContextClassRef`; and
* a `Subject` containing a `NameID` (or, when transient NameIDs are
  accepted under {{nameid-policy}}, a release of the SAML V2.0
  `subject-id` or `pairwise-id` attribute from {{SAML2-SUBJ-ID}}).

If the assertion includes an `AttributeStatement`, the OP MUST apply the
IdP-Binding's attribute mapping policy ({{claim-mapping}}) before
populating any OIDC claim.

# OIDC Authentication Flow at the OP {#op-authentication-flow}

This section specifies the flow whose abstract shape is given in
{{overview}}: an OIDC RP initiates authentication at the OP, the OP
authenticates the end-user via the upstream SAML IdP, and the OP
returns OIDC tokens to the RP. The subsections below specify each
step in detail.

## Authorization Request Processing {#authorization-request-processing}

The OP MUST validate the authorization request per {{OIDC-CORE}}
Section 3.1.2.2 and follow {{RFC9700}}. This profile is defined only
for the Authorization Code Flow; other `response_type` values MUST
be rejected.

This profile adds the following:

* `prompt`, `max_age`, and `acr_values` MUST be consumed when
  constructing the upstream SAML AuthnRequest
  ({{authn-request-construction}});
* `login_hint` MAY influence IdP-Binding selection
  ({{idp-binding-selection}}) but MUST NOT override administrative
  routing. The OP MAY forward a hint value only via an
  IdP-Binding-configured mechanism. RP-supplied values MUST NOT be
  placed in `AuthnRequest/Subject/NameID`;
* `ui_locales` MAY be propagated via a configured SAML extension;
  otherwise it affects OP-rendered UI only;
* `claims` ({{OIDC-CORE}} Section 5.5) is a hint. The OP MUST NOT
  release a claim merely because it appears in `claims`;
* `request` and `request_uri` SHOULD be rejected unless explicitly
  configured;
* `id_token_hint` is a hint. The OP MUST NOT skip upstream SAML
  authentication on its basis;
* `display` MAY influence OP-rendered UI and is not propagated;
* the OP MUST maintain integrity-protected correlation state
  binding (RP `client_id`, scopes, PKCE challenge, `nonce`,
  `redirect_uri`, `state`) to the issued `AuthnRequest/@ID`. State
  MAY be server-side or carried per {{stateless-correlation}}.

## IdP-Binding Selection {#idp-binding-selection}

The OP MUST select exactly one IdP-Binding per authentication. Selection
inputs MAY include administrative routing policy, the RP's `acr_values`
or `login_hint`, deployment-specific hints (e.g., the user's email
domain), or interactive home-realm discovery.

When multiple IdP-Bindings could match, the OP MUST resolve the choice
deterministically before constructing the SAML AuthnRequest. The OP MUST
NOT prompt the upstream SAML IdP to make this selection. The OP MUST NOT
allow an OIDC RP to select an IdP-Binding it is not administratively
authorized to consume.

If no IdP-Binding can be selected for an OIDC authorization request, the
OP MUST return an OIDC `invalid_request` error to the RP. The OP MUST
NOT disclose to the RP which IdP-Bindings are configured or why
selection failed.

## SAML AuthnRequest Construction {#authn-request-construction}

The OP constructs `AuthnRequest` per {{SAML2-CORE}} and
{{SAML2-PROFILES}}. This profile adds:

* the `AuthnRequest` MUST be signed. The signature mechanism is
  binding-specific per {{SAML2-BINDINGS}}: detached query-parameter
  signature for HTTP-Redirect, enveloped `ds:Signature` for
  HTTP-POST. The signature MUST use RSA-SHA-256 or stronger;
  SHA-1 MUST NOT be used;
* `Issuer/@Format`, when present, MUST be
  `urn:oasis:names:tc:SAML:2.0:nameid-format:entity`;
* `AssertionConsumerServiceURL` (or Index) MUST identify an ACS URL
  published in the OP's SAML SP metadata;
* `NameIDPolicy` SHOULD be constructed from the IdP-Binding;
* `RequestedAuthnContext` SHOULD be translated from `acr_values`
  when configured. `@Comparison` follows IdP-Binding policy with a
  default SHOULD of `"exact"` (OIDC `acr_values` is unordered);
* `@ForceAuthn="true"` when the OIDC request contains
  `prompt=login` or `max_age=0`;
* when `max_age > 0`, the OP MUST treat the value as an upper
  bound on the received assertion's `AuthnInstant`. On violation,
  the OP MAY re-issue with `@ForceAuthn="true"` or return
  `login_required` ({{flow-error-handling}});
* `@IsPassive="true"` when the OIDC request contains `prompt=none`;
* the OP MUST NOT convey the OIDC RP's `client_id` to the SAML IdP
  in any form that lets the IdP alter behavior per RP, unless
  local policy authorizes it (see {{privacy-considerations}}).

The OP MUST track the issued `AuthnRequest/@ID` for `InResponseTo`
validation and replay protection per {{replay-and-freshness}}.

## Stateless Correlation with RelayState {#stateless-correlation}

This subsection defines an OPTIONAL implementation mechanism for the
correlation state required by {{authorization-request-processing}}.
Deployments MAY ignore this subsection and use server-side
transaction storage instead.

An OP MAY avoid deployment-wide server-side storage for ordinary
authentication transaction context by carrying a protected transaction
object across the OIDC-to-SAML round trip. The transaction object MUST be
integrity protected and confidentiality protected using an authenticated
encryption mechanism such as JWE, or an equivalent construction. It MUST
include an expiration time and enough context to reconstruct and validate
the OIDC authorization request, including the RP `client_id`, registered
`redirect_uri`, requested scope, OIDC `state`, OIDC `nonce`, PKCE code
challenge and method, selected IdP-Binding, OP ACS URL, and SAML
`AuthnRequest/@ID`.

The OP MAY carry this protected object directly in SAML `RelayState` only
when the selected SAML binding and deployment profile safely support the
resulting size. For broad SAML interoperability, deployments need to
account for the HTTP-Redirect binding RelayState size limit defined by
{{SAML2-BINDINGS}}. When the protected object is too large for
`RelayState`, the OP MAY store the protected object in a Secure,
HttpOnly, SameSite cookie scoped to the OP and carry only a transaction
identifier in `RelayState`.

When processing the returned SAML `Response`, the OP MUST verify the
protected transaction object before using it. In particular, the OP MUST
verify that:

* the object decrypts and authenticates successfully;
* the object has not expired;
* the `RelayState` transaction identifier, when used, matches the
  transaction identifier inside the protected object;
* `Response/@InResponseTo` and
  `SubjectConfirmationData/@InResponseTo` match the protected
  `AuthnRequest/@ID`;
* `SubjectConfirmationData/@Recipient` matches the protected OP ACS URL;
* the SAML Issuer matches the selected IdP-Binding; and
* the OIDC `redirect_uri`, `nonce`, PKCE challenge, and requested scope
  used to complete the OIDC flow are taken from the protected object, not
  from unprotected browser input.

This stateless correlation mechanism does not remove replay-detection
requirements. The OP still MUST maintain shared replay state for consumed
SAML assertion IDs, consumed or completed SAML `AuthnRequest/@ID` values,
and redeemed authorization codes, as required by {{replay-and-freshness}}
and {{response-splicing}}. Stateless correlation reduces deployment state
for pending transactions; it does not make the security model stateless.

## Error Handling {#flow-error-handling}

If the SAML IdP returns a SAML error `Response` (a `Response` with a
non-Success `StatusCode`), the OP MUST translate the SAML status to an
OIDC authorization error and return it to the RP. The OP MUST NOT expose
the SAML IdP's internal error codes or descriptions to the RP. Common
SAML status codes SHOULD be mapped as follows:

* `urn:oasis:names:tc:SAML:2.0:status:NoPassive` →
  OIDC `interaction_required` (when `prompt=none` was requested) or
  `login_required` otherwise;
* `urn:oasis:names:tc:SAML:2.0:status:RequestDenied` →
  OIDC `access_denied`;
* `urn:oasis:names:tc:SAML:2.0:status:AuthnFailed` →
  OIDC `login_required`;
* `urn:oasis:names:tc:SAML:2.0:status:NoAuthnContext` →
  OIDC `login_required` (with operational logging of the ACR mismatch);
  and
* any other SAML error → OIDC `server_error` or `temporarily_unavailable`
  as appropriate.

If SAML assertion validation fails for reasons other than an upstream
error (signature failure, audience mismatch, replay, expired, etc.), the
OP MUST return an OIDC `server_error` to the RP and MUST NOT expose the
specific validation failure reason. The validation failure SHOULD be
logged at the OP for diagnostics.

# Local Subject Resolution {#local-subject-resolution}

Before deriving an OIDC `sub` or releasing OIDC claims under this profile,
the OP MUST resolve stable SAML subject input from the validated assertion
to exactly one active Local Account.

The OP resolves the Local Account using stable SAML subject identifiers
from the trusted SAML Issuer. Eligible primary inputs are the
`urn:oasis:names:tc:SAML:attribute:subject-id` attribute, the
`urn:oasis:names:tc:SAML:attribute:pairwise-id` attribute, and stable
`NameID` values permitted by the IdP-Binding. Releasable attributes such
as `mail`, `eduPersonPrincipalName`, display name, or telephone number
MUST NOT by themselves be treated as sufficient evidence to create or
select a Local Account, although they MAY be used as supplemental hints
when local policy has already bound that supplemental value for the
trusted SAML IdP to exactly one Local Account.

If the SAML IdP releases the `urn:oasis:names:tc:SAML:attribute:subject-id`
or `urn:oasis:names:tc:SAML:attribute:pairwise-id` attribute from
{{SAML2-SUBJ-ID}}, the OP SHOULD use that attribute as the primary
resolution key in preference to the NameID, because those attributes
have well-defined stability and reassignment semantics that
implementation-specific NameID handling may lack.

This profile does not define just-in-time provisioning. Any deployment-
specific account creation or activation triggered by a validated SAML
assertion MUST complete before subject identifier derivation and OIDC
token issuance.

If resolution yields no Local Account, more than one candidate Local
Account, or a Local Account that is disabled, suspended, deprovisioned,
or otherwise not eligible to authenticate, the OP MUST fail the
authentication and MUST NOT issue OIDC tokens.

Once a Local Account has been resolved for a given SAML Issuer and stable
subject input, the OP MUST continue to resolve the same input to the same
Local Account unless an authorized administrative remapping occurs.

# Subject Identifier Mapping {#subject-identifier-mapping}

The OIDC `sub` claim returned to each relying party is the primary
subject continuity point for OIDC RPs. The OP MUST derive `sub` from the
Local Account in a way that preserves the OIDC `subject_type` semantics
the RP registered (public or pairwise per {{OIDC-CORE}} Section 8).

## Pairwise `sub` Derivation {#pairwise-sub}

When the RP's registered `subject_type` is `pairwise`, the OP MUST derive
`sub` deterministically from a stable input bound to the Local Account
and a relying-party context (`sector_identifier_uri` value or the RP's
`client_id` per {{OIDC-CORE}}). The OP MUST use a non-reassignable, non-
reversible derivation function; the specific algorithm is implementation-
defined as permitted by {{OIDC-CORE}}.

The OP MUST NOT expose a pairwise `sub` derived for one RP context to a
different RP context.

## Public `sub` Derivation {#public-sub}

When the RP's registered `subject_type` is `public`, the OP MUST return
the same stable `sub` value for the same Local Account to all such RPs.
The OP MUST mint a public `sub` value as either a randomly generated
opaque identifier persisted in the Local Account, or a deterministic
derivation from a stable Local Account identifier combined with an
OP-held secret salt (so that the public `sub` cannot be reconstructed
from public SAML identifiers alone).

## NameID Stability and Reassignment {#nameid-stability}

The OP MUST NOT use a SAML NameID with format `transient` as input
to `sub` derivation ({{nameid-policy}}).

When local policy or knowledge of the SAML IdP's behavior indicates
that `persistent` NameID values may be reassigned, the IdP-Binding
MUST configure use of a non-reassignable stable identifier as the
`sub` derivation input. Suitable identifiers include `subject-id`
and `pairwise-id` from {{SAML2-SUBJ-ID}}, `eduPersonUniqueId`, or an
opaque internal account identifier. SAML 2.0 does not preclude
reassignment of `persistent` NameIDs, but {{SAML2-SUBJ-ID}}
identifiers specifically disallow it.

The OP MUST NOT use the `emailAddress` NameID format as input to
`sub` derivation when a more stable identifier is available.

An upstream IdP may re-emit the same identifier for a new principal,
in error or under deployment change. To detect this:

* the OP MUST persist the (source identifier, Local Account)
  mapping when first observed and treat it as authoritative;
* on a mapping-match whose attributes are inconsistent with the
  bound Local Account, the OP MUST NOT silently replace the mapping.
  The OP MUST either fail the authentication
  ({{flow-error-handling}}) or require explicit administrative
  remapping.

For IdP-guaranteed non-reassignable identifiers (`subject-id`,
`pairwise-id`, `eduPersonUniqueId`), the mismatch check is an
integrity assertion that should never fire. For other identifiers,
the OP MUST apply the failure or remapping rule above.

## `sub_id` for SAML Provenance {#sub-id-claim}

The OP MAY include a `sub_id` claim in ID Tokens, UserInfo responses,
and introspection responses to convey the original SAML subject context
to relying parties. When included, `sub_id` MUST use the `saml-nameid`
format and member names defined by {{SAML-OIDC-MIGRATION}}, populated
from the validated SAML assertion. Use of `sub_id` is RECOMMENDED when
the RP needs the original SAML identifier context (for example, for
legacy account linking or audit correlation).

The OP MUST NOT include `sub_id` in tokens released to clients whose
release policy does not include the `saml_subject` scope (defined by
{{SAML-OIDC-MIGRATION}}) or an equivalent administrative permission.

# Claim Mapping {#claim-mapping}

The mappings in this section govern how SAML authentication context and
attributes are translated into OIDC claims for the issued tokens.

## Authentication Event Claims {#authentication-event-claims}

The OP MUST populate the following ID Token claims from the validated
SAML assertion:

* `iss`: the OP's OIDC issuer identifier.
* `sub`: the value derived in {{subject-identifier-mapping}}.
* `aud`: the OIDC RP's `client_id`.
* `iat`, `exp`, `nonce`: per {{OIDC-CORE}} Section 2. `iat` is the
  ID Token issuance time at the OP, not the SAML `AuthnInstant`.
  `exp` MUST NOT derive from `Conditions/@NotOnOrAfter`. When the
  selected `AuthnStatement` carries `SessionNotOnOrAfter` and the ID
  Token represents the same SAML session, `exp` MUST NOT exceed
  `SessionNotOnOrAfter`.
* `auth_time`: the SAML assertion's `AuthnStatement/@AuthnInstant`,
  converted from XML `dateTime` (UTC) to NumericDate (seconds since
  epoch). The OP MUST NOT set `auth_time` to the OP's token issuance
  time unless `AuthnInstant` is unavailable and local policy explicitly
  permits substitution.
* `acr`: derived from `AuthnContext/AuthnContextClassRef` per
  {{acr-mapping}}.
* `amr`: derived from `AuthnContext` and any released `amr`-equivalent
  attributes per {{amr-mapping}}.
* `sid`: when the OP maintains a session and the OIDC client supports
  back-channel or front-channel logout, the OP-scoped session identifier
  for this authentication.
* `session_expiry`: when the selected SAML `AuthnStatement` contains
  `SessionNotOnOrAfter`, the OP MAY emit the OpenID Connect Enterprise
  Extensions `session_expiry` claim ({{OIDC-ENTERPRISE-EXTENSIONS}}).
  When emitted, `session_expiry` MUST be a JSON integer containing the
  number of seconds elapsed since 1970-01-01T00:00:00Z, and its value
  MUST NOT be later than the SAML `SessionNotOnOrAfter` value. The OP
  MUST NOT derive `session_expiry` from the SAML assertion's
  `Conditions/@NotOnOrAfter`, because that value limits assertion
  validity, not the underlying authenticated session.

### ACR Mapping {#acr-mapping}

A SAML `AuthnContext` MAY contain multiple `AuthnContextClassRef`
elements. The OP selects one as authoritative for `acr` in this
order:

1. an IdP-Binding-configured selection rule (highest assurance,
   first listed, match `acr_values`, etc.);
2. otherwise, the value mapping to the highest mapped `acr` under
   the IdP-Binding's ordering policy;
3. otherwise, the OP MUST omit `acr`.

Unselected values MUST NOT be exposed as additional `acr` values.
OIDC `acr` is single-valued.

Given the selected value, the OP determines `acr` as follows:

* if the IdP-Binding defines an `AuthnContextClassRef`-to-`acr`
  mapping, the OP MUST apply it;
* otherwise, if the IdP-Binding permits passthrough, the OP MAY
  return the value unchanged;
* otherwise the OP MUST omit `acr`. If the IdP-Binding configures a
  default, that default MUST be
  `urn:oasis:names:tc:SAML:2.0:ac:classes:unspecified`.

The OP MUST NOT produce an `acr` value asserting higher assurance
than the selected `AuthnContextClassRef` warrants.

### AMR Mapping {#amr-mapping}

The OP MAY populate `amr` {{RFC8176}} from concrete authentication
evidence in the SAML assertion. The OP MUST NOT infer `amr` values from
the `AuthnContextClassRef` alone; `amr` MUST be populated only when
the SAML IdP releases attributes (or `AuthnContextDecl`) that identify
specific authentication methods. The OP MUST NOT overstate
authentication assurance by inflating `amr` values.

## Attribute Claims {#attribute-claims}

When the issued OIDC token release includes claims sourced from SAML
attributes, the OP MUST apply the IdP-Binding's attribute mapping policy
combined with the OIDC client's release allowlist. The OP MUST NOT
release a claim to an RP unless both:

* the IdP-Binding's policy maps the SAML attribute to the corresponding
  OIDC claim, and
* the OIDC client's release policy permits that claim.

Common mappings from SAML attributes to OIDC claims, parallel to the
inverse mappings in {{SAML-OIDC-BRIDGE}}, are listed in
{{SAML-OIDC-MIGRATION}} and apply unchanged under this profile.

Attribute processing MUST follow these rules:

* a single-valued SAML `Attribute` with one `AttributeValue` maps to a
  scalar OIDC claim value;
* a multi-valued SAML `Attribute` MAY map to a JSON array OIDC claim
  value when the target claim is array-valued;
* boolean SAML `AttributeValue` strings (`"true"`, `"false"`) SHOULD map
  to boolean OIDC claims when the target claim is boolean-typed;
* the OP MUST NOT expose the SAML `NameID` value as an OIDC claim other
  than `sub` or (when permitted) `sub_id`; and
* the OP MUST NOT release a claim that the IdP-Binding policy does not
  map, even when the SAML assertion contains the underlying attribute.

The OP MUST NOT infer email verification (`email_verified=true`) from
the presence of an `eduPersonPrincipalName`, `mail`, or
`emailAddress`-format NameID alone. `email_verified=true` MUST be
released only when the SAML IdP indicates verified email status through
a configured attribute.

# Session Termination and Revocation {#session-termination-and-revocation}

An OIDC token issued under this profile represents a moment in the
underlying SAML-authenticated session at the upstream IdP. That SAML
session may end at the IdP before the OIDC token would otherwise expire,
and each OIDC RP's own session is independent of either. This section
defines how the OP connects SAML-side session signals to the OIDC tokens
it has issued.

SAML Single Logout (SLO) and OpenID Connect logout are not equivalent
mechanisms, and this profile does not guarantee parity between them.
Deployments that need coordinated logout across both protocols MUST
design that coordination outside this profile; this profile defines only
the OP's local obligations once it has learned that the underlying SAML
session has ended.

## Session Binding {#session-binding}

The OP MUST maintain a session correlation record binding each OIDC
session it issues to the upstream SAML session that authorized it. The
record MUST include the SAML `SessionIndex` (when the IdP emitted one)
and the OP's OIDC session identifier (`sid`).

When the SAML assertion contains a `SessionIndex` in the
`AuthnStatement`, the OP MUST use that value as the SAML session
identifier in the correlation record. When `SessionIndex` is absent, the
OP MUST establish its own session identifier at the time of assertion
consumption.

The OIDC `sid` claim ({{OIDC-FC-LOGOUT}}, {{OIDC-BC-LOGOUT}}) is the
OP-minted identifier above. The OP MUST NOT reuse the SAML
`SessionIndex` directly as `sid`, because `SessionIndex` is scoped to
the upstream IdP and is not guaranteed unique at the OIDC issuer.

This profile does not define OIDC Session Management 1.0
(`session_state`, `check_session_iframe`). OPs MAY additionally
implement it; any binding to upstream SAML session state is
deployment-defined. RPs needing SAML-aware liveness signals SHOULD
use {{OIDC-BC-LOGOUT}} or {{OIDC-FC-LOGOUT}} notifications driven by
SAML SLO events ({{saml-single-logout}}).

## Session Termination {#session-termination}

The OP MUST treat the earlier of the following as the operative end of
the SAML-authorized session for a given OIDC session:

* the assertion's `SessionNotOnOrAfter` value, when present in the
  authoritative `AuthnStatement`;
* any authoritative SAML SLO signal from the upstream IdP for the bound
  `SessionIndex`;
* an administrative or risk-based revocation at the OP; or
* expiry of the OIDC session per OP policy.

When the SAML-authorized session has ended, the OP:

* MUST treat any further OIDC token issuance for that session (including
  refresh-token use) as requiring reauthentication;
* SHOULD propagate the termination to OIDC RPs through
  {{OIDC-BC-LOGOUT}} and {{OIDC-FC-LOGOUT}} for RPs that have registered
  for those mechanisms; and
* MAY rely on token expiry alone when neither back-channel nor front-
  channel logout is supported by an affected RP.

OAuth 2.0 Token Revocation {{RFC7009}} applies to OIDC tokens issued
under this profile without modification.

## SAML Single Logout {#saml-single-logout}

Support for SAML Single Logout (SLO) at the OP-as-SP role is OPTIONAL.
When the OP supports SLO:

* the OP's SAML SP metadata SHOULD include a `SingleLogoutService`
  endpoint;
* upon receiving a SAML `LogoutRequest` from the upstream IdP, the OP
  MUST validate the request signature against the IdP's key material,
  applying the XSW rules in {{xsw-incoming}}; MUST verify
  `LogoutRequest/Issuer` corresponds to a configured IdP-Binding; MUST
  verify that `LogoutRequest/Subject/NameID` or `SessionIndex`
  corresponds to an OP-managed session originated from that IdP;
  and MUST terminate the corresponding OP session, propagating to OIDC
  RPs as in {{session-termination}};
* the OP MUST return a signed `LogoutResponse` even if RP-side
  propagation fails. Signing follows {{outgoing-signature-algorithms}}.
  The signature mechanism is binding-specific (detached query-string
  for HTTP-Redirect, enveloped XML for HTTP-POST, SOAP message
  signature for SOAP);
* the OP MUST support the HTTP-Redirect binding for `LogoutResponse`
  and MAY additionally support HTTP-POST or SOAP;
* the OP MAY initiate `LogoutRequest` toward the upstream IdP on
  OIDC RP-Initiated Logout when no other RPs share the session.
  OP-initiated upstream SLO may invalidate the upstream session at
  every SP in that session, including SPs unrelated to this OP.
  Deployments SHOULD weigh that federation-wide impact when enabling
  this behavior.

# Multi-IdP Deployments {#multi-idp-deployments}

Deployments MAY configure multiple IdP-Bindings at a single OP to
federate to multiple regional or vendor SAML IdPs. The
multi-IdP-specific rules are distributed across this document and
summarized here as a reading aid: AS-metadata omission of
`saml_idp_entity_id` ({{saml-idp-entity-id-reuse}}); deterministic
IdP-Binding selection ({{idp-binding-selection}}); per-IdP-Binding
isolation of validation and Local Account resolution
({{saml-assertion-validation}}, {{local-subject-resolution}}).

This profile additionally permits per-IdP issuer aliasing:

* the OP MAY operate distinct OIDC `issuer` values per IdP-Binding.
  Each alias issuer MUST publish its own discovery document and
  MAY publish a distinct `jwks_uri`. Each alias issuer's
  `saml_idp_entity_id` MAY identify the single backing
  IdP-Binding.
* when per-IdP issuer aliasing is not used, the OP MAY emit a
  deployment-defined claim carrying the IdP-Binding identifier.
  This profile does not define such a claim.


# Security Considerations {#security-considerations}

## Threat Model {#threat-model}

This profile assumes the following:

* The OP and the SAML IdP are operated by, or under the administrative
  control of, the same authority. They share a trust boundary but are
  network-separated components that communicate through their respective
  protocols.
* OIDC RPs trust the OP under their existing OIDC client registrations;
  the OP does not establish new federation trust toward RPs as a
  consequence of using a SAML IdP upstream.
* `saml_idp_entity_id` bindings at the OP are established under
  administrative control, not through self-asserted dynamic
  registration.

In-scope attacker capabilities and the rules that mitigate them:

* obtaining a SAML assertion in transit (XSS at the upstream IdP,
  intermediary compromise, log leakage). Mitigated by the three-way
  binding ({{binding-and-issuance-model}}),
  audience/recipient/`InResponseTo` validation
  ({{accepted-saml-inputs}}), and replay rules
  ({{replay-and-freshness}}).
* forging, replaying, or manipulating a SAML `Response` toward the
  OP. Mitigated by signature, audience, recipient, freshness, and
  replay rules in {{saml-assertion-validation}}.
* XML Signature Wrapping against the SAML `Response` or `Assertion`.
  See {{xsw-incoming}}.
* splicing a SAML response into a different OIDC request context.
  Mitigated by `InResponseTo` and OP-side state binding
  ({{authorization-request-processing}}).
* binding an OIDC client registration that triggers cross-IdP-Binding
  subject overlap. Mitigated by multi-IdP isolation
  ({{multi-idp-deployments}}).
* compromising the OP-as-SP signing key. See {{op-key-compromise}}.

Out of scope: compromise of the OP or the SAML IdP themselves;
infrastructure threats not specific to this profile; end-user account
compromise upstream of the SAML IdP.

## SAML Signature Risks {#saml-signature-risks}

### XML Signature Wrapping on Incoming SAML {#xsw-incoming}

The OP consumes signed `Response` and `Assertion` messages from the
upstream IdP, and MAY consume `LogoutRequest` messages. The OP MUST
validate those signatures in a manner resistant to XML Signature
Wrapping. Specifically:

* the OP MUST verify that each element referenced by a signature's
  `Reference/@URI` is the same element being processed and is the
  unique signed element in the document being acted upon; and
* the OP MUST reject any document structure where these bindings cannot
  be confirmed unambiguously.

Implementations SHOULD use established SAML processing libraries that
have been tested against known XSW attack patterns.

### Outgoing AuthnRequest Signature Algorithms {#outgoing-signature-algorithms}

The OP MUST sign every `AuthnRequest` and `LogoutRequest` it issues
toward the upstream IdP with RSA-SHA-256 or stronger. SHA-1 signature
algorithms MUST NOT be used. Deployments SHOULD also reject MD5-based
digests, RSA keys below 2048 bits, and elliptic curves below 128-bit
security, and SHOULD track current cryptographic guidance (NIST, ENISA)
for further adjustments.

## SAML Response Splicing {#response-splicing}

A browser-based attacker may deliver a SAML `Response` captured from
one OIDC flow to the OP's ACS in another pending flow. The OP MUST
prevent this by:

* binding the issued `AuthnRequest/@ID` to the OIDC request context
  ({{authorization-request-processing}}). The bound context MUST
  include the OIDC `state`. Integrity protection of the binding is
  required (for example, via the authenticated transaction object in
  {{stateless-correlation}});
* validating that `SubjectConfirmationData/@InResponseTo` matches
  the pending `AuthnRequest/@ID`; and
* tracking outstanding `AuthnRequest/@ID` values and rejecting any
  `Response` whose `InResponseTo` is consumed or unknown.

## Metadata Fetching and SSRF {#metadata-ssrf}

The OP fetches the upstream IdP's SAML metadata to discover the
`SingleSignOnService` endpoint and verification keys. Fetches whose
target URI is influenced by configuration data carry a Server-Side
Request Forgery risk against internal services. Implementations MUST
restrict fetches to URIs that use the HTTPS scheme and MUST NOT follow
redirects to non-HTTPS URIs. Deployments SHOULD additionally constrain
permitted fetch targets to a pre-approved allowlist of hosts.

The upstream SAML metadata URI is administratively configured at the OP
and is not exposed to dynamic registration by OIDC RPs under this
profile.

## Stateless Correlation State {#stateless-correlation-security}

When the OP uses stateless correlation ({{stateless-correlation}}) to
carry OIDC authorization request context across the SAML round trip:

* the protected transaction object MUST be integrity-protected and
  confidentiality-protected with an authenticated encryption mechanism;
  authenticated-encryption keys MUST be managed with the same operational
  security as OIDC issuer signing keys, including rotation policy;
* the OP MUST enforce a short expiration on the transaction object (in
  the absence of stricter local policy, no longer than the SAML
  AuthnRequest freshness window in {{replay-and-freshness}}) and MUST
  reject expired objects;
* the OP MUST NOT trust any value extracted from the protected
  transaction object until decryption and authentication succeed; in
  particular, the OP MUST NOT use the carried `redirect_uri`, `nonce`,
  PKCE challenge, or selected IdP-Binding to complete the OIDC flow
  unless those values originated from the authenticated object; and
* when the protected object is stored in a cookie scoped to the OP
  rather than carried in `RelayState`, the cookie MUST be marked
  `Secure`, `HttpOnly`, and `SameSite=None`. `SameSite=Lax` and
  `SameSite=Strict` MUST NOT be used: the SAML Response is delivered
  to the OP's ACS endpoint as a cross-site POST and Lax/Strict
  cookies are not sent. The cookie MUST be cleared on flow
  completion or expiry. Deployments that cannot use cross-site
  cookies MUST instead carry the object in `RelayState` or keep
  server-side state keyed by a small `RelayState` token.

Stateless correlation does not remove the replay-protection
requirements in {{replay-and-freshness}}; the shared replay state for
consumed assertions and consumed `AuthnRequest/@ID` values is still
required.

## OP-as-SP Key Compromise {#op-key-compromise}

The OP-as-SP role holds signing keys (for AuthnRequests and
LogoutRequests) and potentially decryption keys (for EncryptedAssertions
when accepted). Compromise of these keys allows an attacker to:

* forge SAML AuthnRequests to the upstream IdP, potentially obtaining
  assertions for arbitrary users (depending on the IdP's authentication
  policy); and
* decrypt captured `EncryptedAssertion` content if the OP accepts
  encrypted assertions.

Deployments MUST protect OP-as-SP keys with the same operational
security as OIDC OP signing keys. Key rotation MUST publish both old
and new keys in the OP's SAML SP metadata long enough for the upstream
IdP to refresh its cached metadata; in the absence of stricter local
policy, at least 24 hours is RECOMMENDED.

## OIDC RP Authorization {#oidc-rp-authorization}

A valid SAML assertion is necessary but not sufficient for OIDC
token issuance; the authorization basis is defined normatively in
{{authorization-assertion-model}}. Common attack patterns that
attempt to short-circuit that rule include treating the assertion
as authorization to grant `offline_access`, or broadening the set
of OIDC RPs that consume an IdP-Binding based on assertion
contents. The authorization-basis rules in
{{authorization-assertion-model}} forbid both.

## Open Redirect {#open-redirect}

The OP performs two browser redirections per flow: outbound from the OP
to the upstream SAML IdP, and inbound from the OP back to the OIDC RP.
Both MUST be validated against pre-registered values:

* the outbound redirection target MUST equal the
  `SingleSignOnService` endpoint URL in the IdP-Binding's SAML metadata;
  it MUST NOT be derived from OIDC request parameters; and
* the inbound redirection target (the OIDC `redirect_uri`) MUST exactly
  match a registered value for the OIDC RP, as required by {{OIDC-CORE}}.

`RelayState` MUST be treated as opaque and MUST NOT be used as a target
URL.

## Protocol Confusion and Transport Security {#protocol-confusion-and-transport}

Mixing SAML trust inputs and OIDC trust inputs creates a risk of
protocol confusion. Implementations MUST validate SAML signatures using
SAML metadata and MUST validate JOSE objects (ID Tokens, JAR, DPoP,
etc.) using OIDC/JOSE key discovery. A key published in one metadata
format is NOT automatically valid in the other.

The OP MUST apply transport security to its authorization endpoint,
token endpoint, userinfo endpoint, ACS endpoint, SAML SLO endpoint, and
all other endpoints used by this profile, in accordance with the
underlying OAuth, OpenID Connect, and SAML specifications.

The OP's SAML metadata URI, the upstream IdP's SAML metadata URI, the
OP's `jwks_uri`, and the OP's discovery endpoints MUST use the HTTPS
scheme. The OP MUST NOT follow redirects from HTTPS to non-HTTPS when
fetching metadata or keys.

# Privacy Considerations {#privacy-considerations}

This profile concentrates identity data flowing from the SAML IdP and
re-releases it through OIDC tokens. The OP sits between two trust
contexts that RPs and end-users would otherwise see as independent, and
its design choices affect the privacy properties of both.

Deployments SHOULD configure pairwise `sub` for OIDC RPs that do not
require cross-RP correlation. Pairwise `sub` derived per
{{pairwise-sub}} ensures that an RP cannot correlate its user population
with that of another RP at the same OP by comparing `sub` values.

The OP MUST NOT release more SAML attributes to an OIDC RP than would
have been released in a direct SAML federation with an equivalent
policy, unless a separate policy explicitly authorizes broader
disclosure. The attribute release allowlist defined for each OIDC client
is the privacy-relevant boundary, and the OP MUST NOT widen it
implicitly because the SAML assertion happens to carry additional
attributes.

The OP SHOULD NOT convey the OIDC RP's identity (`client_id`, requested
scopes, etc.) to the upstream SAML IdP in a way that allows the IdP to
build an audit trail linking end-users to specific OIDC RPs, unless
local policy explicitly permits this and the privacy implications have
been assessed. Doing so could allow the SAML IdP to profile which OIDC
RPs each user accesses.

Independently of any RP-identity conveyance, the OP topology exposes
some user-activity signal to the upstream IdP. Each authorization
request requiring a fresh SAML authentication produces an
`AuthnRequest`, so the IdP learns authentication frequency and
timing even without RP identifiers. Deployments SHOULD weigh this
when sizing the SAML session reuse window and SHOULD document the
characteristic in privacy notices when federation boundaries cross
organizations.

The `sub_id` claim ({{sub-id-claim}}) exposes the original SAML
`NameID` context, including the SAML IdP entityID and any `NameID`
qualifiers. Deployments SHOULD release `sub_id` only when the RP
genuinely needs the original SAML context (legacy account linking, audit
correlation). A SAML `NameID` in `emailAddress` format carries
email-equivalent sensitivity and MUST be handled accordingly.

OP-side audit logs combine SAML and OIDC protocol identifiers (SAML
Issuer, NameID, `SessionIndex`, OIDC `sub`, `client_id`) and are
uniquely positioned to deanonymize pairwise `sub` values. The OP SHOULD
retain such logs for the minimum period required by deployment policy
and SHOULD apply the access controls appropriate for that combined data.

The OP's published SAML SP metadata (`SPSSODescriptor`) discloses the
OP's ACS URL, signing key, and (when emitted) `EncryptedAssertion`
encryption key. In federations, this metadata may be aggregated and
distributed widely. Deployments SHOULD treat the SAML SP metadata as
public deployment topology information and SHOULD NOT include
contact, organization, or internal-host detail beyond what the
federation requires.

# IANA Considerations {#iana-considerations}

This document requests no new IANA registrations. The
`session_expiry` claim referenced in {{authentication-event-claims}}
is defined and registered by {{OIDC-ENTERPRISE-EXTENSIONS}}.

This document reuses existing registrations with consistent
semantics:


* `saml_idp_entity_id` and `saml_metadata_uri` (AS metadata) from
  {{SAML-OIDC-MIGRATION}} are used unchanged for the upstream SAML
  IdP binding;
* the `sub_id` JWT claim from {{RFC9493}}, the `saml-nameid` Subject
  Identifier Format value from {{SAML-OIDC-MIGRATION}}, and the
  `saml_subject` OAuth scope value from {{SAML-OIDC-MIGRATION}} are
  reused unchanged for SAML provenance;
* OAuth, OpenID Connect, and SAML 2.0 protocol values are governed
  by their respective specifications and not modified here.

--- back

# Examples

This appendix is non-normative. Examples use the `private_key_jwt`
client authentication method as defined by {{OIDC-CORE}}; token values
and client assertion JWTs are shortened placeholders. All examples
assume an OP with the following configuration:

* OP OIDC issuer: `https://op.example.com`
* OP authorization endpoint: `https://op.example.com/authorize`
* OP token endpoint: `https://op.example.com/token`
* OP-as-SP SAML entity ID: `https://op.example.com/saml/sp`
* OP ACS URL: `https://op.example.com/saml/acs`

and an IdP-Binding for an upstream SAML IdP:

* SAML IdP entity ID: `https://idp.example.com/saml/idp`
* SAML IdP SSO URL: `https://idp.example.com/saml/sso`
* NameID format: `urn:oasis:names:tc:SAML:2.0:nameid-format:persistent`
* Attribute mapping: `mail` → `email`, `displayName` → `name`,
  `givenName` → `given_name`, `sn` → `family_name`

## OP Authorization Server Metadata

A non-normative excerpt of the OP's AS metadata published at
`https://op.example.com/.well-known/openid-configuration`:

~~~ json
{
  "issuer": "https://op.example.com",
  "authorization_endpoint": "https://op.example.com/authorize",
  "token_endpoint": "https://op.example.com/token",
  "userinfo_endpoint": "https://op.example.com/userinfo",
  "jwks_uri": "https://op.example.com/.well-known/jwks.json",
  "response_types_supported": ["code"],
  "subject_types_supported": ["public", "pairwise"],
  "id_token_signing_alg_values_supported": ["RS256", "ES256"],
  "token_endpoint_auth_methods_supported": [
    "private_key_jwt",
    "tls_client_auth"
  ],
  "saml_idp_entity_id": "https://idp.example.com/saml/idp",
  "saml_metadata_uri":
    "https://idp.example.com/saml/idp/metadata.xml"
}
~~~

## OIDC Authorization Code Flow with Upstream SAML

An OIDC RP (`calendar-app`) initiates authentication by redirecting
Alice's browser to the OP's authorization endpoint:

~~~ http
GET /authorize?response_type=code
  &client_id=calendar-app
  &redirect_uri=https%3A%2F%2Fcalendar.example.com%2Fcb
  &scope=openid%20profile%20email
  &state=oidc-state-abc
  &nonce=oidc-nonce-xyz
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256
  HTTP/1.1
Host: op.example.com
~~~

The OP selects its IdP-Binding, constructs a signed SAML
`AuthnRequest`, and redirects Alice's browser to the upstream IdP
(`AuthnRequest` shown deflated/unsigned for brevity):

~~~ xml
<samlp:AuthnRequest
    ID="_op-authnreq-7c1a"
    Version="2.0"
    IssueInstant="2026-05-19T12:00:00Z"
    Destination="https://idp.example.com/saml/sso"
    AssertionConsumerServiceURL="https://op.example.com/saml/acs"
    ProtocolBinding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST">
  <saml:Issuer>https://op.example.com/saml/sp</saml:Issuer>
  <samlp:NameIDPolicy
      Format="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent"
      AllowCreate="true"/>
  <ds:Signature>...</ds:Signature>
</samlp:AuthnRequest>
~~~

After Alice authenticates at the upstream IdP, the IdP posts a signed
SAML `Response` to the OP's ACS (assertion shown, simplified):

~~~ xml
<saml:Assertion
    ID="_idp-assertion-9f4b"
    IssueInstant="2026-05-19T12:00:05Z"
    Version="2.0">
  <saml:Issuer>https://idp.example.com/saml/idp</saml:Issuer>
  <ds:Signature>...</ds:Signature>
  <saml:Subject>
    <saml:NameID
        Format="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent"
        NameQualifier="https://idp.example.com/saml/idp"
        SPNameQualifier="https://op.example.com/saml/sp">
      _persistent-2bd3a4f9c0
    </saml:NameID>
    <saml:SubjectConfirmation
        Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
      <saml:SubjectConfirmationData
          Recipient="https://op.example.com/saml/acs"
          InResponseTo="_op-authnreq-7c1a"
          NotOnOrAfter="2026-05-19T12:05:00Z"/>
    </saml:SubjectConfirmation>
  </saml:Subject>
  <saml:Conditions
      NotBefore="2026-05-19T11:59:55Z"
      NotOnOrAfter="2026-05-19T12:05:00Z">
    <saml:AudienceRestriction>
      <saml:Audience>https://op.example.com/saml/sp</saml:Audience>
    </saml:AudienceRestriction>
  </saml:Conditions>
  <saml:AuthnStatement
      AuthnInstant="2026-05-19T11:59:30Z"
      SessionIndex="_idp-sess-c7b1"
      SessionNotOnOrAfter="2026-05-19T20:00:00Z">
    <saml:AuthnContext>
      <saml:AuthnContextClassRef>
        urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
      </saml:AuthnContextClassRef>
    </saml:AuthnContext>
  </saml:AuthnStatement>
  <saml:AttributeStatement>
    <saml:Attribute Name="urn:oid:0.9.2342.19200300.100.1.3"
                    FriendlyName="mail">
      <saml:AttributeValue>alice@example.com</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute Name="urn:oid:2.5.4.42" FriendlyName="givenName">
      <saml:AttributeValue>Alice</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute Name="urn:oid:2.5.4.4" FriendlyName="sn">
      <saml:AttributeValue>Ng</saml:AttributeValue>
    </saml:Attribute>
  </saml:AttributeStatement>
</saml:Assertion>
~~~

The OP validates the assertion, resolves Alice's Local Account, derives
the pairwise `sub` for `calendar-app`, maps the SAML attributes to OIDC
claims, and completes the OIDC flow by redirecting Alice's browser to
the RP's `redirect_uri` with an authorization code. The RP exchanges
the code at the OP's token endpoint and receives an ID Token whose
payload contains:

~~~ json
{
  "iss": "https://op.example.com",
  "sub": "_pairwise-calendar-app-3a8f1cd2",
  "aud": "calendar-app",
  "exp": 1779192300,
  "iat": 1779188700,
  "auth_time": 1779188370,
  "nonce": "oidc-nonce-xyz",
  "acr":
    "urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport",
  "sid": "_op-sess-1f7a2b",
  "session_expiry": 1779220800,
  "email": "alice@example.com",
  "given_name": "Alice",
  "family_name": "Ng",
  "name": "Alice Ng",
  "sub_id": {
    "format": "saml-nameid",
    "issuer": "https://idp.example.com/saml/idp",
    "nameid": "_persistent-2bd3a4f9c0",
    "nameid_format": "urn:oasis:names:tc:SAML:2.0:nameid-format:persistent",
    "name_qualifier": "https://idp.example.com/saml/idp",
    "sp_name_qualifier": "https://op.example.com/saml/sp"
  }
}
~~~

# Paired Profile Summary

This appendix is non-normative.

The following table summarizes how this profile relates to its
companions {{SAML-OIDC-MIGRATION}} and {{SAML-OIDC-BRIDGE}}. The three
profiles together cover the protocol surfaces needed for a phased SAML
deprecation.

| Dimension | This profile | {{SAML-OIDC-MIGRATION}} | {{SAML-OIDC-BRIDGE}} |
|---|---|---|---|
| Direction | SAML upstream → OIDC downstream | SAML input → OIDC token | OIDC input → SAML output |
| Authentication authority | SAML IdP | SAML IdP | OpenID Provider (OP) |
| Translation component | OP (acting as SAML SP) | OAuth AS / OpenID Provider | Bridge (SAML IdP facade) |
| Input credential | SAML Assertion (via Web SSO) | SAML Assertion (via Token Exchange or introspection) | OIDC ID Token (via authorization code or Token Exchange) |
| Output credential | OIDC ID Token, access token, claims | OIDC ID Token, access token, refresh token | SAML Assertion |
| Subject input | SAML NameID | SAML NameID | OIDC `sub` |
| Subject output | OIDC `sub` (pairwise or public) | OIDC `sub` (pairwise or public) | SAML NameID (pairwise or public) |
| ACR direction | SAML `AuthnContextClassRef` → OIDC `acr` | SAML `AuthnContextClassRef` → OIDC `acr` | OIDC `acr` → SAML `AuthnContextClassRef` |
| auth_time direction | SAML `AuthnInstant` → OIDC `auth_time` | SAML `AuthnInstant` → OIDC `auth_time` | OIDC `auth_time` → SAML `AuthnInstant` |
| Phase in SAML deprecation | Enables OP adoption while SAML is authoritative | Modernizes individual SPs to OIDC clients | Continues to serve SAML SPs after OP becomes authoritative |
