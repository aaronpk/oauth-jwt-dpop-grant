---
title: "OAuth 2.0 JWT Authorization Grant with DPoP Binding"
abbrev: "JWT Authorization Grant with DPoP"
category: std

docname: draft-parecki-oauth-jwt-dpop-grant-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
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
  RFC6755:
  RFC6749:
  RFC7519:
  RFC7521:
  RFC7523:
  RFC7638:
  RFC7800:
  I-D.ietf-oauth-rfc7523bis:
  RFC9449:

informative:
  RFC8414:
  RFC8693:
  I-D.ietf-oauth-identity-assertion-authz-grant:

...

--- abstract

This specification defines a new OAuth 2.0 authorization grant type
for exchanging a JSON Web Token (JWT) assertion that is bound to a key
for an access token. To use the assertion, the client must prove
possession of that key using the Demonstrating Proof of Possession (DPoP)
mechanism. Unlike the JWT bearer grant, a stolen assertion cannot be
used by a party that does not hold the private key.

--- middle

# Introduction

The JSON Web Token (JWT) Profile for OAuth 2.0 Client Authentication
and Authorization Grants {{RFC7523}} defines the use of a JWT as an
authorization grant, using the grant type `urn:ietf:params:oauth:grant-type:jwt-bearer`.
This grant type describes the use of a JWT authorization grant as a
bearer token, which is susceptible to reuse by any party that obtains one.

OAuth 2.0 Demonstrating Proof of Possession (DPoP) {{RFC9449}} defines
a mechanism to bind access tokens to a specific cryptographic key.
This prevents the token from being used by any party that does not
have access to the private key.

This specification extends the proof-of-possession concept to the
authorization grant itself. It defines a new grant type,
`urn:ietf:params:oauth:grant-type:jwt-dpop`, for cases where the JWT
assertion is already bound to a DPoP key. To exchange the assertion
for an access token, the client must provide a DPoP proof demonstrating
possession of the key to which the assertion is bound. This makes the
JWT assertion a sender-constrained credential.

A motivating use case is the Identity Assertion JWT Authorization Grant
{{I-D.ietf-oauth-identity-assertion-authz-grant}}, in which a client
obtains a JWT assertion from an identity provider, typically using
Token Exchange {{RFC8693}}, and presents it to the authorization server
of a different trust domain. Binding that assertion to a DPoP key
prevents it from being used by any party other than the client that
requested it, and allows the resulting access token to be bound to the
same key.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology

This specification uses the terminology of {{RFC6749}}, {{RFC7521}}, {{RFC7523}}, and {{RFC9449}}.

DPoP-bound JWT assertion:
: A JWT assertion that contains a `cnf` claim with a `jkt` confirmation
  method member identifying the public key to which the assertion is bound.

# Obtaining a DPoP-Bound JWT Assertion {#issuance}

This specification does not define how a client obtains a DPoP-bound
JWT assertion. Any mechanism may be used, provided that the issuer of
the assertion binds it to a key controlled by the client. This section
describes requirements on the issuer, and one way of meeting them.

The issuer of a DPoP-bound JWT assertion:

* MUST include a `cnf` claim containing a `jkt` confirmation method
  member, whose value is the JWK SHA-256 Thumbprint {{RFC7638}} of the
  client's public key, as described in {{Section 6.1 of RFC9449}}.
* SHOULD verify that the client possesses the private key corresponding
  to that public key before issuing the assertion.
* MUST NOT include more than one `jkt` value, or any other confirmation
  method member alongside `jkt`.

When the assertion is obtained from an OAuth token endpoint, such as by
Token Exchange {{RFC8693}}, the client can demonstrate possession of the
key by including a `DPoP` header field in that request, as defined in
{{Section 5 of RFC9449}}. The issuer validates the DPoP proof and uses
the thumbprint of the public key in the proof as the value of
`cnf.jkt` in the issued assertion. The client then uses the same key
when presenting the assertion as described in {{parameters}}.

For example, when obtaining an Identity Assertion JWT Authorization
Grant (ID-JAG) as described in {{I-D.ietf-oauth-identity-assertion-authz-grant}},
the client includes a `DPoP` header field in the Token Exchange request
to the identity provider, and the identity provider includes the
resulting `cnf.jkt` value in the issued ID-JAG.

The following non-normative example shows the decoded claims of a
DPoP-bound ID-JAG:

    {
      "iss": "https://idp.example.com",
      "sub": "U019488227",
      "aud": "https://as.example.net",
      "client_id": "f53f191f9311af35",
      "exp": 1793400300,
      "iat": 1793400000,
      "jti": "9a8b3cb8-7a1e-4d2c-8c5e-0f3e6a2c7b1d",
      "cnf": {
        "jkt": "GXIEs4thM-cVZkW7mUs9-9R731DmI7aeeBFm4VYCMUA"
      }
    }

# HTTP Parameter Bindings for Transporting Assertions

The OAuth Assertion Framework {{RFC7521}} defines generic HTTP
parameters for transporting assertions (a.k.a. security tokens)
during interactions with a token endpoint.  This section defines
specific parameters and treatments of those parameters for use with
DPoP-bound JWT assertions.

## Using DPoP-Bound JWT Assertions as Authorization Grants {#parameters}

To use a DPoP-bound JWT assertion as an authorization grant, the client makes an
access token request as defined in {{Section 4.1 of RFC7521}}
with the following specific parameter values and encodings.

`grant_type`:
: REQUIRED - The value MUST be `urn:ietf:params:oauth:grant-type:jwt-dpop`

`assertion`:
: REQUIRED - A single JWT, as defined in {{RFC7519}}, that contains a
  `cnf` claim as described in {{processing}}.

`scope`:
: OPTIONAL - The `scope` parameter may be used, as defined in {{Section 4.1 of RFC7521}},
  to indicate the requested scope.

Authentication of the client is optional, as described in
{{Section 3.2.1 of RFC6749}} and consequently, the
`client_id` is only needed when a form of client authentication that
relies on the parameter is used.

The client MUST also include a `DPoP` header field as defined in {{Section 4 of RFC9449}},
which constitutes a proof of possession for the key to which the assertion is bound.

The following example demonstrates an access token request with a
DPoP-bound JWT assertion as an authorization grant (with extra line
breaks for display purposes only). The complete request, with decoded
values, is shown in {{complete-example}}.

    POST /token HTTP/1.1
    Host: as.example.net
    Content-Type: application/x-www-form-urlencoded
    DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt0eSI6
      [...omitted for brevity...]

    grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-dpop
    &assertion=eyJ0eXAiOiJvYXV0aC1pZC1qYWcrand0IiwiYWxnIjoiRVMyNTYi
    [...omitted for brevity...]

# JWT Format and Processing Requirements {#processing}

The authorization server MUST validate the JWT according to the criteria
below. Application of additional restrictions and policy are at the
discretion of the authorization server.

1. The authorization server MUST validate the DPoP proof in the `DPoP`
   header field as described in {{Section 4.3 of RFC9449}}.
2. The authorization server MUST validate the JWT assertion according
   to the processing rules in {{Section 3 of RFC7523}} (including {{Section 3.1 of RFC7523}}) and {{Section 4 of I-D.ietf-oauth-rfc7523bis}}.
3. The authorization server MUST verify that the JWT assertion contains
   a `cnf` claim as defined in {{RFC7800}}, and that the `cnf` claim
   contains a `jkt` confirmation method member as defined in
   {{Section 6.1 of RFC9449}}.
4. The authorization server MUST verify that the public key in the `jwk`
   JOSE Header Parameter of the DPoP proof matches the key to which the
   JWT assertion is bound. This is done in the same way a resource server
   confirms that a DPoP-bound access token matches the key of a DPoP proof
   (check 12 in {{Section 4.3 of RFC9449}}), using the JWK Thumbprint
   confirmation method of {{Section 6.1 of RFC9449}}: compute the
   base64url-encoded JWK SHA-256 Thumbprint {{RFC7638}} of that public
   key, and compare it to the value of `cnf.jkt` in the assertion.

If the DPoP proof is missing or invalid (step 1), the authorization server
MUST return an error response as described in {{Section 5 of RFC9449}}
(`invalid_dpop_proof`), or {{Section 8 of RFC9449}} (`use_dpop_nonce`) if
a nonce is required.
If any of the remaining validation steps fail, the authorization server MUST
return an `invalid_grant` error response.

## DPoP Nonces and Replay {#replay}

The authorization server MAY require the client to include a nonce in
the DPoP proof, as described in {{Section 8 of RFC9449}}. In that case
the authorization server responds to a request without a valid nonce
with a `use_dpop_nonce` error and a `DPoP-Nonce` header field, and the
client retries the request with a new DPoP proof containing the
provided nonce and the same assertion.

The authorization server SHOULD perform replay detection on the `jti`
claim of the DPoP proof, as described in {{Section 11.1 of RFC9449}}.
The authorization server MAY also reject assertions whose `jti` has
already been used, as described in {{Section 3 of RFC7523}}.

If the authorization server records assertion `jti` values to detect
replay, it MUST NOT record the assertion's `jti` as used when the request
is rejected because of an invalid or missing DPoP proof or nonce.
Otherwise a client following the nonce retry described above would have
its assertion rejected as a replay. Performing the validation steps in
the order given in {{processing}} achieves this.

## Use with the JWT Bearer Grant Type {#jwt-bearer}

An authorization server that supports the grant type defined in this
specification MUST NOT accept a DPoP-bound JWT assertion as an
authorization grant using the `urn:ietf:params:oauth:grant-type:jwt-bearer`
grant type defined in {{RFC7523}}, and MUST return an `invalid_grant`
error response if one is presented. This applies whether or not the
request includes a `DPoP` header field.

An authorization server that supports this specification SHOULD include
`urn:ietf:params:oauth:grant-type:jwt-dpop` in the `grant_types_supported`
value of its authorization server metadata {{RFC8414}}.

# Access Token Response

If the request is valid, the authorization server issues an access
token. The issued access token SHOULD be DPoP-bound to the key from the
DPoP proof, which is the same key to which the assertion is bound. In
this case the `token_type` in the response MUST be `DPoP`, as described
in {{Section 5 of RFC9449}}.

The authorization server MAY instead issue a bearer access token, for
example when the protected resources for which the token is intended
do not support DPoP. In this case the `token_type` MUST be `Bearer`. See
{{bearer-access-tokens}} for the security implications.

If the authorization server issues a refresh token, the refresh token
is subject to the binding requirements of {{Section 5 of RFC9449}}: a
refresh token issued to a public client MUST be bound to the key from
the DPoP proof. The authorization server SHOULD NOT issue a refresh
token whose lifetime extends beyond what is permitted by the policy
under which the assertion was issued.


# Security Considerations

The security considerations described within the following
specifications are all applicable to this document:
"Assertion Framework for OAuth 2.0 Client Authentication and Authorization Grants" {{RFC7521}},
"JSON Web Token (JWT) Profile for OAuth 2.0 Client Authentication and Authorization Grants" {{RFC7523}},
"Updates to OAuth 2.0 JSON Web Token (JWT) Client Authentication and Assertion-Based Authorization Grants" {{I-D.ietf-oauth-rfc7523bis}},
"OAuth 2.0 Demonstrating Proof of Possession (DPoP)" {{RFC9449}},
"The OAuth 2.0 Authorization Framework" {{RFC6749}},
and "JSON Web Token (JWT)" {{RFC7519}}.

## Downgrade to the JWT Bearer Grant

The binding of a DPoP-bound JWT assertion only provides protection if
the authorization server enforces it. An attacker that obtains a
DPoP-bound JWT assertion, but not the corresponding private key, could
present the assertion using the `urn:ietf:params:oauth:grant-type:jwt-bearer`
grant type instead of the grant type defined in this specification. An
authorization server that processes the assertion only according to
{{RFC7523}} would ignore the `cnf` claim and issue an access token,
defeating the binding entirely.

Authorization servers that support this specification prevent this
attack by rejecting DPoP-bound JWT assertions presented with the
`jwt-bearer` grant type, as required by {{jwt-bearer}}.

Authorization servers that do not implement this specification are not
aware of this requirement and may accept a DPoP-bound JWT assertion as
a bearer assertion. For this reason, an issuer of DPoP-bound JWT
assertions SHOULD NOT issue one for use at an authorization server
unless it knows that the authorization server supports this
specification, for example because the authorization server advertises
`urn:ietf:params:oauth:grant-type:jwt-dpop` in its `grant_types_supported`
metadata {{RFC8414}}, or by prior configuration.


## Issuing Bearer Access Tokens {#bearer-access-tokens}

When the authorization server issues a bearer access token in response
to this grant, the protection provided by the DPoP binding ends at the
token endpoint. The assertion cannot be used by a party that does not
hold the private key, but the resulting access token can be used by any
party that obtains it. Authorization servers SHOULD issue DPoP-bound
access tokens whenever the intended protected resources support them.

# Privacy Considerations

The `jkt` value in a DPoP-bound JWT assertion is a stable identifier of
the client's key. When a client uses the same key with an identity
provider, with one or more authorization servers, and with the protected
resources to which DPoP-bound access tokens are presented, each of these
parties observes the same key, either in DPoP proofs or as `cnf.jkt` in
assertions and access tokens. Colluding parties could use this to
correlate the client's activity across them, beyond what is already
possible from the identity of the client or the user.

To limit this, a client MAY use a different DPoP key for each
authorization server at which it presents DPoP-bound JWT assertions.
Since the assertion is bound to the key used when requesting it (see
{{issuance}}), the client needs to choose the key it will use at the
target authorization server before requesting the assertion. A client
SHOULD NOT reuse a DPoP key across unrelated users or sessions where
doing so would let those otherwise unlinked activities be correlated.

This specification does not change the claims about the user that are
conveyed in the assertion, and the privacy considerations of the
specification defining the assertion, such as
{{I-D.ietf-oauth-identity-assertion-authz-grant}}, continue to apply.

# IANA Considerations

## OAuth URI Registration

This specification requests registration of the following value in the
"OAuth URI" registry established by {{RFC6755}}.

* URN: `urn:ietf:params:oauth:grant-type:jwt-dpop`
* Common Name: DPoP-bound JWT Authorization Grant
* Change Controller: IESG
* Specification Document(s): this document


--- back

# Complete Example {#complete-example}

This appendix contains a complete example of an access token request
using a DPoP-bound ID-JAG. The signatures in this example can be
verified using the public keys shown. Line breaks within JWTs and
within the canonical JWK below are for display purposes only.

## Identity Provider Key

The ID-JAG is signed by the identity provider using the following
key, shown as a JWK:

    {
      "kty": "EC",
      "crv": "P-256",
      "x": "RQAT9mqfwKatL9zm1ckKHIyDvjt45YePri8FEw3bbBw",
      "y": "do5FLL-GZqbHgCQlkNs-qpJC6fWghRQsAwrl9cPD73M",
      "kid": "idp-key-1",
      "use": "sig",
      "alg": "ES256"
    }

## DPoP-Bound JWT Assertion

The decoded JOSE header of the ID-JAG:

    {
      "typ": "oauth-id-jag+jwt",
      "alg": "ES256",
      "kid": "idp-key-1"
    }

The decoded claims of the ID-JAG:

    {
      "iss": "https://idp.example.com",
      "sub": "U019488227",
      "aud": "https://as.example.net",
      "client_id": "f53f191f9311af35",
      "exp": 1793400300,
      "iat": 1793400000,
      "jti": "9a8b3cb8-7a1e-4d2c-8c5e-0f3e6a2c7b1d",
      "cnf": {
        "jkt": "GXIEs4thM-cVZkW7mUs9-9R731DmI7aeeBFm4VYCMUA"
      }
    }

The encoded and signed ID-JAG:

    eyJ0eXAiOiJvYXV0aC1pZC1qYWcrand0IiwiYWxnIjoiRVMyNTYiLCJraWQiOiJp
    ZHAta2V5LTEifQ.
    eyJpc3MiOiJodHRwczovL2lkcC5leGFtcGxlLmNvbSIsInN1YiI6IlUwMTk0ODgy
    MjciLCJhdWQiOiJodHRwczovL2FzLmV4YW1wbGUubmV0IiwiY2xpZW50X2lkIjoi
    ZjUzZjE5MWY5MzExYWYzNSIsImV4cCI6MTc5MzQwMDMwMCwiaWF0IjoxNzkzNDAw
    MDAwLCJqdGkiOiI5YThiM2NiOC03YTFlLTRkMmMtOGM1ZS0wZjNlNmEyYzdiMWQi
    LCJjbmYiOnsiamt0IjoiR1hJRXM0dGhNLWNWWmtXN21VczktOVI3MzFEbUk3YWVl
    QkZtNFZZQ01VQSJ9fQ.
    c_PUFFNDDHvgWUxpCQ_jsiMaP4ZU0xXqkWFQ5lGHT_WYsjlAfOboUOSxuHR-_GjM
    P3cSmUETAr_ux2ivS828xw

## DPoP Proof

The decoded JOSE header of the DPoP proof, which contains the client's
public key:

    {
      "typ": "dpop+jwt",
      "alg": "ES256",
      "jwk": {
        "kty": "EC",
        "crv": "P-256",
        "x": "xSYHiPmBtVKQAzs79FRVe1dI7Y_Lu93I67WxbSsg9P0",
        "y": "zHt5qFsvN2EYcSWf5hq1GqpcqmhONiB9W_YIgvsYqmQ"
      }
    }

The decoded claims of the DPoP proof:

    {
      "jti": "e1j3V_bKic8-LAEB",
      "htm": "POST",
      "htu": "https://as.example.net/token",
      "iat": 1793400005
    }

The encoded and signed DPoP proof:

    eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt0eSI6IkVD
    IiwiY3J2IjoiUC0yNTYiLCJ4IjoieFNZSGlQbUJ0VktRQXpzNzlGUlZlMWRJN1lf
    THU5M0k2N1d4YlNzZzlQMCIsInkiOiJ6SHQ1cUZzdk4yRVljU1dmNWhxMUdxcGNx
    bWhPTmlCOVdfWUlndnNZcW1RIn19.
    eyJqdGkiOiJlMWozVl9iS2ljOC1MQUVCIiwiaHRtIjoiUE9TVCIsImh0dSI6Imh0
    dHBzOi8vYXMuZXhhbXBsZS5uZXQvdG9rZW4iLCJpYXQiOjE3OTM0MDAwMDV9.
    S0ygnxypg5vIkyq7HCRz03MNafRa1Hk1NFsCGHbjFc4mrRQic9X7TpBxMy2C1BTT
    6fjbCEFS5J_GJKuwdGxSZA

## Thumbprint Comparison

The authorization server constructs the canonical JWK of the public
key in the `jwk` header of the DPoP proof, as described in
{{Section 3 of RFC7638}}:

    {"crv":"P-256","kty":"EC",
    "x":"xSYHiPmBtVKQAzs79FRVe1dI7Y_Lu93I67WxbSsg9P0",
    "y":"zHt5qFsvN2EYcSWf5hq1GqpcqmhONiB9W_YIgvsYqmQ"}

The base64url-encoded SHA-256 hash of this value is:

    GXIEs4thM-cVZkW7mUs9-9R731DmI7aeeBFm4VYCMUA

This matches the value of `cnf.jkt` in the ID-JAG, so the assertion
is bound to the key used to sign the DPoP proof.

## Access Token Request

The resulting access token request:

    POST /token HTTP/1.1
    Host: as.example.net
    Content-Type: application/x-www-form-urlencoded
    DPoP:
      eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt0eSI6IkVD
      IiwiY3J2IjoiUC0yNTYiLCJ4IjoieFNZSGlQbUJ0VktRQXpzNzlGUlZlMWRJN1lf
      THU5M0k2N1d4YlNzZzlQMCIsInkiOiJ6SHQ1cUZzdk4yRVljU1dmNWhxMUdxcGNx
      bWhPTmlCOVdfWUlndnNZcW1RIn19.
      eyJqdGkiOiJlMWozVl9iS2ljOC1MQUVCIiwiaHRtIjoiUE9TVCIsImh0dSI6Imh0
      dHBzOi8vYXMuZXhhbXBsZS5uZXQvdG9rZW4iLCJpYXQiOjE3OTM0MDAwMDV9.
      S0ygnxypg5vIkyq7HCRz03MNafRa1Hk1NFsCGHbjFc4mrRQic9X7TpBxMy2C1BTT
      6fjbCEFS5J_GJKuwdGxSZA

    grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-dpop
    &assertion=
    eyJ0eXAiOiJvYXV0aC1pZC1qYWcrand0IiwiYWxnIjoiRVMyNTYiLCJraWQiOiJp
    ZHAta2V5LTEifQ.
    eyJpc3MiOiJodHRwczovL2lkcC5leGFtcGxlLmNvbSIsInN1YiI6IlUwMTk0ODgy
    MjciLCJhdWQiOiJodHRwczovL2FzLmV4YW1wbGUubmV0IiwiY2xpZW50X2lkIjoi
    ZjUzZjE5MWY5MzExYWYzNSIsImV4cCI6MTc5MzQwMDMwMCwiaWF0IjoxNzkzNDAw
    MDAwLCJqdGkiOiI5YThiM2NiOC03YTFlLTRkMmMtOGM1ZS0wZjNlNmEyYzdiMWQi
    LCJjbmYiOnsiamt0IjoiR1hJRXM0dGhNLWNWWmtXN21VczktOVI3MzFEbUk3YWVl
    QkZtNFZZQ01VQSJ9fQ.
    c_PUFFNDDHvgWUxpCQ_jsiMaP4ZU0xXqkWFQ5lGHT_WYsjlAfOboUOSxuHR-_GjM
    P3cSmUETAr_ux2ivS828xw

# Acknowledgments
{:numbered="false"}

The authors would like to thank the following people for their contributions and reviews of this specification: Filip Skokan, Karl McGuinness.


# Document History
{:numbered="false"}

\[\[ To be removed from the final specification ]]

-02

* Rewrote the abstract to describe binding of the assertion rather than the access token.
* Defined "DPoP-bound JWT assertion" and used the term consistently.
* Use DPoP error codes for DPoP proof validation failures.
* Moved Access Token Response to a top-level section.
* Added a section describing requirements on the issuer of a DPoP-bound JWT assertion, with an example.
* Added the Identity Assertion JWT Authorization Grant as a motivating use case.
* Added Privacy Considerations.
* Added a complete, verifiable example in an appendix.
* Added guidance on DPoP nonces and replay detection, including not consuming the assertion on a nonce retry.
* Clarified when bearer access tokens may be issued, and the binding of refresh tokens.
* Added requirement to reject DPoP-bound JWT assertions presented with the `jwt-bearer` grant type, and security considerations for downgrade attacks.
* Clarified that the DPoP proof's `jwk` is compared to `cnf.jkt` by computing its JWK SHA-256 Thumbprint, as a resource server does for DPoP-bound access tokens.

-01

* Changed DPoP check to use `jkt` string instead of `jwk` object to be able to use simple string comparison.


