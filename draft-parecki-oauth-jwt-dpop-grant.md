---
title: "OAuth 2.0 JWT Authorization Grant with DPoP Token Binding"
abbrev: "JWT Authorization Grant with DPoP"
category: std

docname: draft-parecki-oauth-jwt-dpop-grant-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: Security
workgroup: "Web Authorization Protocol"
keyword:
 - jwt
 - dpop
 - authorization
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "aaronpk/oauth-jwt-dpop-grant"
  latest: "https://drafts.aaronpk.com/oauth-jwt-dpop-grant/draft-parecki-oauth-jwt-dpop-grant.html"

author:
 -
    fullname: Aaron Parecki
    organization: Okta
    email: aaron@parecki.com

normative:
  RFC6749:
  RFC7519:
  RFC7521:
  RFC7523:
  I-D.ietf-oauth-rfc7523bis:
  RFC9449:

informative:

...

--- abstract

This specification defines a new OAuth 2.0 authorization grant type
that uses a JSON Web Token (JWT) assertion to request an access token
that is bound to a specific key using the Demonstration of Proof-of-Possession (DPoP)
mechanism. This provides a higher level of security than a simple bearer token,
as the client must prove possession of the key to use the access token.

--- middle

# Introduction

The JSON Web Token (JWT) Profile for OAuth 2.0 Client Authentication
and Authorization Grants {{RFC7523}} defines the use of a JWT as an
authorization grant, using the grant type `urn:ietf:params:oauth:grant-type:jwt-bearer`.
This grant type describes the use of a JWT authorization grant as a
bearer token, which is susceptible to reuse by any party that obtains one.

OAuth 2.0 Demonstration of Proof-of-Possession at the Application
Layer (DPoP) {{RFC9449}} defines a mechanism to bind access tokens to a
specific cryptographic key. This prevents the token from being used by
any party that does not have access to the private key.

This specification extends the proof-of-possession concept to the
authorization grant itself. It defines a new grant type,
`urn:ietf:params:oauth:grant-type:jwt-dpop`, for cases where the JWT
assertion is already bound to a DPoP key. To exchange the assertion
for an access token, the client must provide a DPoP proof demonstrating
possession of the key to which the assertion is bound. This makes the
JWT assertion a sender-constrained credential.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology

This specification uses the terminology of {{RFC6749}}, {{RFC7521}}, {{RFC7523}}, and {{RFC9449}}.

# HTTP Parameter Bindings for Transporting Assertions

The OAuth Assertion Framework {{RFC7521}} defines generic HTTP
parameters for transporting assertions (a.k.a. security tokens)
during interactions with a token endpoint.  This section defines
specific parameters and treatments of those parameters for use with
JWT DPoP-Bound Tokens.

## Using DPoP-Bound JWTs as Authorization Grants

To use a DPoP-bound JWT as an authorization grant, the client uses an
access token request as defined in {{Section 4 of RFC7521}}
with the following specific parameter values and encodings.

`grant_type`:
: REQUIRED - The value MUST be `urn:ietf:params:oauth:grant-type:jwt-dpop`

`assertion`:
: REQUIRED - A single JWT, as defined in {{RFC7519}}, that contains a
  `cnf` claim as described in {{cnf}}.

`scope`:
: OPTIONAL - The `scope` parameter may be used, as defined in {{RFC7521}},
  to indicate the requested scope.

Authentication of the client is optional, as described in
{{Section 3.2.1 of RFC6749}} and consequently, the
`client_id` is only needed when a form of client authentication that
relies on the parameter is used.

The client MUST also include a DPoP header as defined in {{Section 4 of RFC9449}},
which constitutes a proof of possession for the key to which the assertion is bound.

The following example demonstrates an access token request with a JWT
as an authorization grant (with extra line breaks for display
purposes only):

    POST /token HTTP/1.1
    Host: as.example.com
    Content-Type: application/x-www-form-urlencoded
    DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrI...

    grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-dpop
    &assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6IjE2In0.
    eyJpc3Mi[...omitted for brevity...].
    J9l-ZhwP[...omitted for brevity...]

# JWT Format and Processing Requirements

The authorization server MUST validate the JWT according to the criteria
below. Application of additional restrictions and policy are at the
discretion of the authorization server.

1. The authorization server MUST validate the DPoP proof in the DPoP
   header as described in {{Section 4 of RFC9449}}. The `htm` claim
   of the DPoP JWT MUST be `POST`, and the `htu` claim must match
   the token endpoint URL.
2. The authorization server MUST validate the JWT assertion according
   to the processing rules in {{Section 3.1 of RFC7523}} and {{Section 4 of I-D.ietf-oauth-rfc7523bis}}.
3. The authorization server MUST verify that the JWT assertion contains
   a `cnf` claim as defined in {{RFC7800}}. This `cnf`
   claim MUST contain a `jwk` property representing a public key.
4. The authorization server MUST verify that the public key in the
   `jwk` property of the `cnf` claim of the JWT assertion exactly
   matches the public key in the `jwk` header of the DPoP proof.

If any of these validation steps fail, the authorization server MUST
return an `invalid_grant` error response.

## Access Token Response

If the request is valid, the authorization server issues an access
token. The issued access token SHOULD also be DPoP-bound to the same
key from the DPoP proof. In this case, the `token_type` of the access
token MUST be `DPoP`, and the response MUST include a `token_type`
parameter with the value `DPoP`. If a bearer token is issued, the
`token_type` MUST be `Bearer`.


# Security Considerations

The security considerations described within the following
specifications are all applicable to this document:
"Assertion Framework for OAuth 2.0 Client Authentication and Authorization Grants" {{RFC7521}},
"JSON Web Token (JWT) Profile for OAuth 2.0 Client Authentication and Authorization Grants" {{RFC7523}},
"Updates to OAuth 2.0 JSON Web Token (JWT) Client Authentication and Assertion-Based Authorization Grants" {{I-D.ietf-oauth-rfc7523bis}},
"OAuth 2.0 Demonstrating Proof of Possession (DPoP)" {{RFC9449}},
"The OAuth 2.0 Authorization Framework" {{RFC6749}},
and "JSON Web Token (JWT)" {{RFC7519}}.


# IANA Considerations

## OAuth URI Registration

This specification requests registration of the following value in the
"OAuth URI" registry established by {{RFC6755}}.

* URN: `urn:ietf:params:oauth:grant-type:jwt-dpop`
* Common Name: DPoP-bound JWT Authorization Grant
* Change Controller: IESG
* Specification Document(s): this document


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.

