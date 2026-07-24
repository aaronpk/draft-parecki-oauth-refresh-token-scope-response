---
title: "OAuth 2.0 Refresh Token Scope"
abbrev: "OAuth RT Scope"
category: std

docname: draft-parecki-oauth-refresh-token-scope-response-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - refresh-token
 - scope
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  github: "aaronpk/draft-parecki-oauth-refresh-token-scope-response"
  latest: "https://aaronpk.github.io/draft-parecki-oauth-refresh-token-scope-response/draft-parecki-oauth-refresh-token-scope-response.html"

author:
 -
    fullname: Aaron Parecki
    organization: Okta
    email: aaron@parecki.com

normative:
  I-D.ietf-oauth-v2-1:

informative:
  I-D.ietf-oauth-security-topics:
  RFC8705:
  RFC9449:

...

--- abstract

This specification defines a new OAuth 2.0 token response parameter,
`refresh_token_scope`, that indicates the scope authorized for a
refresh token when it differs from the scope of the access token
issued alongside it. This allows clients to discover that a refresh
token carries broader authorization than the initial access token,
enabling just-in-time requests for elevated access without requiring
a new authorization flow.


--- middle

# Introduction

The OAuth 2.0 Authorization Framework {{I-D.ietf-oauth-v2-1}} defines a token
endpoint response that includes a `scope` parameter indicating the
scope of the issued access token. When an authorization server issues
a refresh token alongside an access token, the refresh token may be
authorized for a broader set of scopes than the initial access token.
{{I-D.ietf-oauth-v2-1}} does not define a mechanism for the authorization server
to communicate this broader authorization to the client.

This gap creates a practical problem: a client that receives a refresh
token has no reliable way to determine whether that refresh token can
be used to obtain access tokens with scopes beyond those reflected in
the initial access token's `scope` parameter. Without this
information, a client that needs elevated access must either
speculatively attempt a refresh with a larger scope (and handle
potential failure), or initiate an entirely new authorization flow,
even when the user or policy has already approved the broader scope.

Consider the following scenarios:

* In an authorization code flow, a user may approve a broad set of
  scopes during the initial authorization. The authorization server
  issues a minimal access token for the immediate operation but
  includes a refresh token authorized for the full approved scope.
  The client has no way to know that it may request additional scopes
  via the refresh token grant without attempting it.

* In a client credentials flow, an authorization server may issue a
  refresh token pre-authorized for a range of operations, while the
  initial access token is scoped only to a particular operation. The
  client service may need to perform higher-privilege operations later
  and could use the refresh token to obtain the appropriate access
  token on demand.

This specification defines a new token response parameter,
`refresh_token_scope`, that the authorization server includes when it
issues a refresh token. The parameter communicates the scope
authorized for the refresh token, enabling clients to make informed
decisions about when and how to request elevated access using the
refresh token grant.

This mechanism provides a simpler alternative to dynamic
challenge-response protocols for many just-in-time authorization use
cases. Rather than defining new error codes, roundtrips, and challenge
sessions, an authorization server can pre-authorize a refresh token
with elevated scope and rely on the existing refresh token grant flow,
extended by this specification, to deliver the right access token at
the right time.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms "access token", "authorization server",
"client", "refresh token", "scope", "token endpoint", and "token
response" as defined by {{I-D.ietf-oauth-v2-1}}.


# The refresh_token_scope Response Parameter

This specification defines the following token response parameter for
use in OAuth 2.0 token endpoint responses.

`refresh_token_scope`:
: OPTIONAL. The scope authorized for the issued refresh token,
  expressed as a space-separated list of scope values as described in
  Section 1.4.1 of {{I-D.ietf-oauth-v2-1}}. This parameter MUST be included in the
  token response when a refresh token is issued and its authorized
  scope differs from the access token scope indicated by the `scope`
  parameter. This parameter MAY be included even when the refresh
  token scope is identical to the access token scope.

When this parameter is present, it indicates the set of scopes that
the client MAY request when using the refresh token at the token
endpoint. The authorization server is not obligated to grant all
scopes listed in `refresh_token_scope` on any individual refresh
request; it MAY apply additional policy, credentials requirements, or
other conditions when deciding whether to honor a particular scope
request.


# Authorization Server Behavior

An authorization server that issues a refresh token with an authorized
scope that differs from the issued access token scope MUST include the
`refresh_token_scope` parameter in the token response.

An authorization server that issues a refresh token with the same
authorized scope as the issued access token SHOULD include the
`refresh_token_scope` parameter in the token response for clarity, but
MAY omit it.

The value of `refresh_token_scope` MUST accurately reflect the full
set of scopes for which the refresh token is authorized. The
authorization server MUST NOT include scope values in
`refresh_token_scope` that are not authorized for the refresh token.

When the authorization server receives a refresh token grant request
that includes a `scope` parameter, it MUST verify that all requested
scope values are within the authorized scope of the presented refresh
token. If the requested scope exceeds the refresh token's authorized
scope, the authorization server MUST return an `invalid_scope` error
as defined in Section 3.2.4 of {{I-D.ietf-oauth-v2-1}}.


# Client Behavior

A client that receives a token response containing both a
`refresh_token` and a `refresh_token_scope` parameter MUST treat the
`refresh_token_scope` value as the authoritative indicator of the
scopes available via the refresh token grant.

A client that needs an access token with a scope that is listed in
`refresh_token_scope` but not in the current access token's scope
SHOULD use the refresh token grant to request an access token with
the desired scope, rather than initiating a new authorization flow.

A client MUST NOT request scopes via the refresh token grant that are
not listed in the `refresh_token_scope` parameter. Doing so will
result in an `invalid_scope` error from the authorization server.

A client that does not receive a `refresh_token_scope` parameter in
the token response MUST assume that the refresh token is authorized
only for the scope indicated by the `scope` parameter (or, if the
`scope` parameter is also absent, the scope originally requested by
the client per Section 3.2.3 of {{I-D.ietf-oauth-v2-1}}).


# Use in the Authorization Code Flow

In the authorization code flow, the resource owner authorizes a set
of scopes during the authorization request. As required by
{{I-D.ietf-oauth-v2-1}}, clients MUST use PKCE with the authorization
code flow. The authorization server may issue an initial access token
scoped to only what the client immediately requires, while issuing a
refresh token authorized for the full set of scopes approved by the
resource owner.

This pattern is useful when:

* The client performs an initial low-privilege operation (e.g.,
  reading a user profile) and may later need higher-privilege access
  (e.g., modifying account settings).

* The authorization server wishes to minimize the privilege of tokens
  in active circulation while preserving the user's broad approval for
  later use via the refresh token.

* Policies require that certain high-privilege access tokens be
  short-lived and obtained fresh via the refresh token grant at the
  time of use.

## Example

The client initiates an authorization request for a broad set of
scopes:

~~~ http
GET /authorize?response_type=code
  &client_id=s6BhdRkqt3
  &scope=profile%20payments%3Aread%20payments%3Awrite
  &redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256 HTTP/1.1
Host: authorization-server.example
~~~

After the user approves the request, the client exchanges the
authorization code for tokens:

~~~ http
POST /token HTTP/1.1
Host: authorization-server.example
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https%3A%2F%2Fclient.example.com%2Fcb
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
~~~

The authorization server issues an access token scoped to `profile`
only, alongside a refresh token authorized for all approved scopes:

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "2YotnFZFEjr1zCsicMWpAA",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "profile",
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
  "refresh_token_scope": "profile payments:read payments:write"
}
~~~

The client now knows that when it later needs to initiate a payment,
it can obtain a suitable access token using the refresh token grant:

~~~ http
POST /token HTTP/1.1
Host: authorization-server.example
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
&scope=payments%3Awrite
~~~

The authorization server issues an access token for the requested
scope, as it is within the refresh token's authorized scope:

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "8xLOxBtZp8",
  "token_type": "Bearer",
  "expires_in": 300,
  "scope": "payments:write",
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
  "refresh_token_scope": "profile payments:read payments:write"
}
~~~


# Use in the Client Credentials Flow

Section 4.2 of {{I-D.ietf-oauth-v2-1}} states that a refresh token
SHOULD NOT be included in the response to a client credentials grant.
This guidance reflects the typical case where the client can simply
re-authenticate to obtain a new access token. However,
{{I-D.ietf-oauth-v2-1}} does not prohibit issuing a refresh token for
a client credentials grant, and there are deployment scenarios where
doing so is appropriate.

In machine-to-machine contexts, a client may be pre-authorized for a
range of operations that span different privilege levels. Rather than
issuing a single long-lived, high-privilege access token, the
authorization server can issue:

* A minimal-privilege access token for the current operation, and

* A refresh token authorized for the broader set of operations the
  client may need to perform.

The `refresh_token_scope` parameter enables this pattern by informing
the client of what it may request later. The client requests only the
scope it needs for the current operation and uses the refresh token to
obtain higher-privilege access tokens on demand, without
re-authenticating to the token endpoint from scratch.

This pattern provides meaningful security benefits in environments
where access token leakage is a concern: high-privilege tokens are
only obtained and circulated when needed, and their lifetimes can be
kept short relative to the refresh token's lifetime.

Authorization servers that choose to issue refresh tokens in client
credentials responses SHOULD apply the same refresh token security
requirements described in Section 7 of {{I-D.ietf-oauth-v2-1}} and in
{{I-D.ietf-oauth-security-topics}}. In particular, refresh token
rotation SHOULD be applied, and the refresh token SHOULD be bound to
the client using mTLS {{RFC8705}} or DPoP {{RFC9449}}.

## Example

A client service requests an access token for a read operation:

~~~ http
POST /token HTTP/1.1
Host: authorization-server.example
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

grant_type=client_credentials
&scope=records%3Aread
~~~

The authorization server issues an access token for the requested
scope and a refresh token authorized for a broader set of scopes:

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "2YotnFZFEjr1zCsicMWpAA",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "records:read",
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
  "refresh_token_scope": "records:read records:write records:delete"
}
~~~

When the client later needs to perform a write operation, it uses the
refresh token to obtain a suitable access token:

~~~ http
POST /token HTTP/1.1
Host: authorization-server.example
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

grant_type=refresh_token
&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
&scope=records%3Awrite
~~~

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "9dEJolkQr2",
  "token_type": "Bearer",
  "expires_in": 300,
  "scope": "records:write",
  "refresh_token": "9tCvLOkF0XG5Qx2pE7iWIA",
  "refresh_token_scope": "records:read records:write records:delete"
}
~~~


# Refresh Token Grant Behavior

When a client uses a refresh token to request a new access token, the
response MUST include a `refresh_token_scope` parameter if a new
refresh token is issued and its authorized scope differs from the
newly issued access token scope. The authorized scope of the new
refresh token MUST NOT exceed the authorized scope of the presented
refresh token.

If the client omits the `scope` parameter from the refresh token grant
request, the authorization server issues an access token with the same
scope as the previously issued access token for that refresh token,
consistent with Section 4.3 of {{I-D.ietf-oauth-v2-1}}.

If the client includes a `scope` parameter in the refresh token grant
request, the authorization server MUST verify the requested scope is
within the authorized scope of the presented refresh token. If the
requested scope is valid, the authorization server issues an access
token for the requested scope. The authorization server MAY also issue
a new refresh token; if it does, the `refresh_token_scope` of the new
token MUST NOT exceed the `refresh_token_scope` of the presented
token.


# Security Considerations

## Scope Escalation

The `refresh_token_scope` parameter informs the client of available
scopes but does not by itself grant those scopes. The authorization
server retains full authority to deny any refresh token grant request
based on current policy, the state of the client's authentication
credentials, risk signals, or any other factor. The presence of a
scope value in `refresh_token_scope` is not a guarantee that the
authorization server will issue an access token for that scope on
demand.

Implementors SHOULD NOT treat `refresh_token_scope` as an irrevocable
commitment. Authorization servers MAY narrow or revoke refresh token
scope at any time in response to policy changes or security events.

## Refresh Token Protection

Because a refresh token carrying broad scope authorization is a
high-value credential, implementors MUST follow the refresh token
security requirements in Section 10 of
{{I-D.ietf-oauth-security-topics}}. Sender-constraining the refresh
token using mTLS {{RFC8705}} or DPoP {{RFC9449}} is strongly
RECOMMENDED, particularly in deployments where the refresh token is
authorized for elevated or sensitive scopes.

Refresh token rotation (issuing a new refresh token on each use and
invalidating the previous one) limits the window of exposure if a
refresh token is compromised.

## Client Credentials and Refresh Tokens

When an authorization server issues a refresh token in response to a
client credentials grant, it MUST ensure that the client is
confidential and capable of maintaining the confidentiality of the
refresh token. Refresh tokens MUST NOT be issued for public clients
using the client credentials grant.

The authorization server SHOULD bind the refresh token to the
client's authentication credential (e.g., via DPoP key binding or
mTLS) to prevent use of the refresh token by a party other than the
authenticated client.


# IANA Considerations

This document defines the following value for the IANA "OAuth
Parameters" registry of {{IANA.oauth-parameters}} established by
{{I-D.ietf-oauth-v2-1}}.

Parameter Name:
: `refresh_token_scope`

Parameter Usage Location:
: token response

Change Controller:
: IETF

Specification Document:
: This document, Section 3


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.


