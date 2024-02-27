---
title: "Cross-Domain Mutually Trusted Authorization"
abbrev: "Mutually Trusted Authz"
category: std

docname: draft-parecki-oauth-mutually-trusted-authorization-grant-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - cross-domain
 - authorization
 - enterprise
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "aaronpk/draft-parecki-oauth-mutually-trusted-authorization-grant"
  latest: "https://aaronpk.github.io/draft-parecki-oauth-mutually-trusted-authorization-grant/draft-parecki-oauth-mutually-trusted-authorization-grant.html"

author:
 -
    fullname: Aaron Parecki
    organization: Okta
    email: aaron@parecki.com
 -
    fullname: Karl McGuinness
    organization: Okta
    email: public@karlmcguinness.com

normative:
  RFC6749:
  RFC7521:
  RFC7523:
  RFC8693:
  RFC8725:
  I-D.ietf-oauth-identity-chaining:
  IANA.MediaTypes:
  IANA.OAuth.Parameters:
  RFC6838:
  RFC2046:

informative:
  RFC9470:


--- abstract

This specification provides a mechanism for an application to obtain an access token for a third-party application through a mutually trusted identity provider using Token Exchange {{RFC8693}}.

--- middle

# Introduction

Enterprises often have hundreds of SaaS applications.  SaaS applications often have integrations to other SaaS applications that are critical to the application experience and jobs to be done.  When a SaaS app needs to request an access token on-behalf of a user to a 3rd party SaaS integration's API, the end-user needs to complete an interactive delegated OAuth 2.0 ceremony and consent.  The SaaS application is not in the same security or policy domain as the 3rd party SaaS integration.

It is industry best practice for an enterprise to connect their ecosystem of SaaS applications to their Identity Provider (IdP) to centralize identity and access management capabilites for the organization.  End-users get a better experience (SSO) and administrators get better security outcomes such multi-factor authentication and zero-trust.  SaaS applications today enable the administrator to establish trust with an IdP for user authentication but typically don't allow the administrator to trust the IdP for API authorization.

The draft specification Identity Chaining Across Trust Domains ({{I-D.ietf-oauth-identity-chaining}}) defines how to request a JWT authorization grant from from an Authorization Server and exchange it for an Access Token at another Authorization Server. The specification is an application of a combination of OAuth 2.0 Token Exchange {{RFC8693}} and JSON Web Token (JWT) Profile for OAuth 2.0 Client Authentication and Authorization Grants {{RFC7523}}. The draft supports multiple different use cases by leaving many details of the token exchange request and JWT authorization grant unspecified. This specification defines the additional details necessary to support interoperable implementations for this specific use case.

This specification enables federation for Authorization Servers across policy or administrative boundaries. The same enterprise IdP for example that is trusted by applications for SSO can be extended to broker access to APIs.  This enables the enterprise to centralize more access decisions across their SaaS ecosystem and provides better end-user experience for users that need to connect multiple applications via OAuth 2.0.

This specification extends {{I-D.ietf-oauth-identity-chaining}}, enabling applications to request access to 3rd party applications using backchannel operations that don't interupt the end user's interactive application experience.  Its also useful for deployments where SSO is based on SAML and not using OpenID Connect.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Roles

Client
: Application that wants to obtain an OAuth 2.0 access token on behalf of a signed-in user to an external/3rd party application's API (Resource Server below) that is managed by the same enterprise IdP. In {{I-D.ietf-oauth-identity-chaining}}, this is the Client in trust domain A.

Resource Application
: Application that provides an OAuth 2.0 Protected Resource that is used across an enterprise's SaaS ecosystem. In {{I-D.ietf-oauth-identity-chaining}}, this is the Protected Resource in trust domain B.

Mutually-Trusted Authorization Server (IdP)
: Organization's Identity Provider that is trusted by a set of applications in an enterprise's app ecosystem for identity and access management. In {{I-D.ietf-oauth-identity-chaining}}, this is the Authorization Server in trust domain A, which is also trusted by the Authorization Server of the Protected Resource in trust domain B.

# Overview

The example flow is for an enterprise `acme`


    | Role     | App URL | Tenant URL   | Description |
    | -------- | -------- | -------- | ----------- |
    | Client | `https://wiki.app` | `https://acme.wiki.app` | SaaS Wiki app that embeds content from best-of-breed SaaS apps |
    | Resource Application   | `https://chat.app` | `https://acme.chat.app` | SaaS chat and communication app |
    | Identity Provider      | `https//idp.cloud`   | `https://acme.idp.cloud` | Cloud Identity Provider |


1. User logs in to the Client via SSO with the Enterprise IdP (SAML or OIDC)
2. Client requests a JWT Authorization Grant for the Resource Application from the IdP
3. Client exchanges the JWT Authorization Grant for an access token at the Resource Application's token endpoint

## Preconditions

- Client has a registered OAuth 2.0 Client with the IdP Authorization Server
- Client has a registered OAuth 2.0 Client with the Resource Applicaiton
- Enterprise has established a trust relationship between their IdP and the Client for SSO and Mutually-Trusted Authorization Grant
- Enterprise has established a trust relationship between their IdP and the Resource Application for SSO and Mutually-Trusted Authorization Grant
- Enterprise has granted the Client to act on-behalf of users for the Resource Application with a set of scopes


# User Authentication

The Client initiates an authentication request with the tenant's trusted Enterprise IdP using SAML or OIDC.

The following is an example using SAML 2.0

    302 Redirect
    Location: https://acme.idp.cloud/SAML2/SSO/Redirect?SAMLRequest={base64AuthnRequest}&RelayState=DyXvaJtZ1BqsURRC

The user authenticates with the IdP and post backs an assertion to the Client

Note: The Enterprise IdP may enforce security controls such as multi-factor authentication before granting the user access to the Client.


    POST /SAML2/SSO/ACS HTTP/1.1
    Host: https://acme.wiki.app/SAML2/ACS
    Content-Type: application/x-www-form-urlencoded
    Content-Length: nnn

    SAMLResponse={AuthnResponse}&RelayState=DyXvaJtZ1BqsURRC


# Token Exchange

The Client makes a Token Exchange {{RFC8693}} request to the IdP's Token Endpoint with the following parameters:

* `requested_token_type=urn:ietf:params:oauth:token-type:mtag`
* `resource` - The token endpoint of the Resource Application.
* `scope` - The space-separated list of scopes at the Resource Application to include in the token
* `subject_token` - The SSO assertion (SAML or OpenID Connect ID Token) for the target end-user
* `subject_token_type` - For SAML2 Assertion: `urn:ietf:params:oauth:token-type:saml2`, or OpenID Connect ID Token: `urn:ietf:params:oauth:token-type:id_token`
* Client authentication (e.g. `client_id` and `client_secret`, or the more secure `private_key_jwt` method using `client_assertion` and `client_assertion_type`)

For example:

    POST /oauth2/token HTTP/1.1
    Host: acme.idp.cloud
    Content-Type: application/x-www-form-urlencoded

    grant_type=urn:ietf:params:oauth:grant-type:token-exchange
    &requested_token_type=urn:ietf:params:oauth:token-type:mtag-jwt
    &resource=https://acme.chat.app/oauth2/token
    &scope=chat.read+chat.history
    &subject_token=PHNhbWw6QXNzZXJ0aW9uCiAgeG1sbnM6c2FtbD0idXJuOm9hc2lzOm5hbWVzOnRjOlNBTUw6Mi4wOmFzc2VydGlvbiIKICBJRD0iaWRlbnRpZmllcl8zIgogIFZlcnNpb249IjIuMCIKICBJc3N1ZUluc3RhbnQ9IjIwMjMtMDYtMDVUMDk6MjA6MDVaIj4KICA8c2FtbDpJc3N1ZXI-aHR0cHM6Ly9hY21lLmlkcC5jbG91ZDwvc2FtbDpJc3N1ZXI-CiAgPGRzOlNpZ25hdHVyZQogICAgeG1sbnM6ZHM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvMDkveG1sZHNpZyMiPi4uLjwvZHM6U2lnbmF0dXJlPgogIDxzYW1sOlN1YmplY3Q-CiAgICA8c2FtbDpOYW1lSUQKICAgICAgRm9ybWF0PSJ1cm46b2FzaXM6bmFtZXM6dGM6U0FNTDoxLjE6bmFtZWlkLWZvcm1hdDplbWFpbEFkZHJlc3MiPgogICAgICBrYXJsQGFjbWUuY29tCiAgICA8L3NhbWw6TmFtZUlEPgogICAgPHNhbWw6U3ViamVjdENvbmZpcm1hdGlvbgogICAgICBNZXRob2Q9InVybjpvYXNpczpuYW1lczp0YzpTQU1MOjIuMDpjbTpiZWFyZXIiPgogICAgICA8c2FtbDpTdWJqZWN0Q29uZmlybWF0aW9uRGF0YQogICAgICAgIEluUmVzcG9uc2VUbz0iNmM5ODMwZTQtMTMzMi00ZjQ5LWFkZTAtZjI0ZjYxMTk2ZDdlIgogICAgICAgIFJlY2lwaWVudD0iaHR0cHM6Ly9hY21lLndpa2kuYXBwL1NBTUwyL0FDUyIKICAgICAgICBOb3RPbk9yQWZ0ZXI9IjIwMjMtMDYtMDVUMDk6MjU6MDVaIi8-CiAgICA8L3NhbWw6U3ViamVjdENvbmZpcm1hdGlvbj4KICA8L3NhbWw6U3ViamVjdD4KICA8c2FtbDpDb25kaXRpb25zCiAgICBOb3RCZWZvcmU9IjIwMjMtMDYtMDVUMDk6MTU6MDVaIgogICAgTm90T25PckFmdGVyPSIyMDIzLTA2LTA1VDA5OjI1OjA1WiI-CiAgICA8c2FtbDpBdWRpZW5jZVJlc3RyaWN0aW9uPgogICAgICA8c2FtbDpBdWRpZW5jZT5odHRwczovL2FjbWUud2lraS5hcHA8L3NhbWw6QXVkaWVuY2U-CiAgICA8L3NhbWw6QXVkaWVuY2VSZXN0cmljdGlvbj4KICA8L3NhbWw6Q29uZGl0aW9ucz4KICA8c2FtbDpBdXRoblN0YXRlbWVudAogICAgQXV0aG5JbnN0YW50PSIyMDIzLTA2LTA1VDA5OjIwOjAwWiIKICAgIFNlc3Npb25JbmRleD0iMzcxMWVjZDYtN2Y5NC00NWM3LTgxYzUtNDkyNjI1NDg0NWYzIj4KICAgIDxzYW1sOkF1dGhuQ29udGV4dD4KICAgICAgPHNhbWw6QXV0aG5Db250ZXh0Q2xhc3NSZWY-CiAgICAgICAgdXJuOm9hc2lzOm5hbWVzOnRjOlNBTUw6Mi4wOmFjOmNsYXNzZXM6UGFzc3dvcmRQcm90ZWN0ZWRUcmFuc3BvcnQKICAgICA8L3NhbWw6QXV0aG5Db250ZXh0Q2xhc3NSZWY-CiAgICA8L3NhbWw6QXV0aG5Db250ZXh0PgogIDwvc2FtbDpBdXRoblN0YXRlbWVudD4KPC9zYW1sOkFzc2VydGlvbj4
    &subject_token_type=urn:ietf:params:oauth:token-type:saml2
    &client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
    &client_assertion=eyJhbGciOiJSUzI1NiIsImtpZCI6IjIyIn0.

## Processing Rules

The IdP validates the subject token, and checks that the audience of the subject token matches the `client_id` of the client authentication of the request.

The IdP evaluates administrator-defined policy for the token exchange request and determines if the application (client) should be granted access to act on behalf of the subject for the target audience & scopes.

IdP may also introspect the authentication context described in the SSO assertion to determine if step-up authentication is required.

## Response

If access is granted, the IdP will return a signed Mutually-Trusted Authorization Grant JWT in the token exchange response defined in Section 2.2 of {{RFC8693}}:

    HTTP/1.1 200 OK
    Content-Type: application/json
    Cache-Control: no-store
    Pragma: no-cache

    {
      "access_token": "eyJhbGciOiJIUzI1NiIsI...",
      "issued_token_type": "urn:ietf:params:oauth:token-type:mtag-jwt",
      "token_type": "N_A",
      "scope": "chat.read chat.history",
      "expires_in": 300
    }

* `access_token` - The Mutually-Trusted Authorization Grant JWT. Token Exchange requires the `access_token` response parameter for historical reasons, even though this is not an access token.
* `issued_token_type` - `urn:ietf:params:oauth:token-type:mtag-jwt`
* `token_type` - `N_A` (Required by the Token Exchange spec)
* `scope` - The list of scopes granted by the IdP. This may be fewer scopes than the application requested based on various policies in the IdP.
* `expires_in` - The lifetime in seconds of the authorization grant.

### Error Response

On an error condition, the IdP returns an OAuth 2.0 Token Error response as defined in Section 5.2 of {{RFC6749}}, e.g:

    HTTP/1.1 400 Bad Request
    Content-Type: application/json
    Cache-Control: no-store

    {
      "error": "invalid_grant",
      "error_description": "Audience validation failed"
    }


## Mutually-Trusted Authorization Grant JWT {#jwt-authorization-grant}

The Mutually-Trusted Authorization Grant JWT is issued by the IdP `https://acme.idp.cloud` for the requested audience `https://acme.chat.app` and includes the following claims:

* `iss` - The IdP `issuer` URL
* `sub` - The User ID at the IdP
* `aud` - Token endpoint of the Resource Application's authorization server
* `azp` - Client ID as registered with the Resource Application's authorization server.
* `exp` -
* `iat` -
* `scopes` - Array of scopes at the Resource Application granted to the Client
* `jti` - Unique ID of this JWT

The `typ` of the JWT indicated in the JWT header MUST be `oauth-mtag+jwt`.

An example JWT shown with expanded header and payload claims is below:

    {
      "typ": "oauth-mtag+jwt"
    }
    .
    {
      "jti": "9e43f81b64a33f20116179",
      "iss": "https://acme.idp.cloud",
      "sub": "U019488227",
      "aud": "https://acme.chat.app/oauth2/token",
      "azp": "f53f191f9311af35",
      "exp": 1311281970,
      "iat": 1311280970,
      "scopes" : [ "chat.read" , "chat.history" ]
    }
    .
    signature

Notes:

* If the IdP is multi-tenant, and uses the same `issuer` for all tenants, the Resource Application will already have IdP-specific logic to determine the tenant from OIDC/SAML (e.g. `hd` in Google) and will need to use that if the IdP also has only one client registration for the Resource Application
* `sub` should be an opaque ID, as `iss`+`sub` is unique. The IdP might want to also include the user's email here, which it should do as a new `email` claim. This would let the app dedupe existing users who may have an account with an email address but have not done SSO yet


# Access Token Request {#token-request}

The Client makes an access token request to the Resource Application's token endpoint using the previously obtained Mutually-Trusted Authorization Grant as a JWT Assertion {{RFC7523}}.

* `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer`
* `assertion` - The JWT authorization grant obtained in the previous step
* Client Authentication - the Client authenticates with its credentials as registered with the Resource Application's authorization server

For example:

    POST /oauth2/token HTTP/1.1
    Host: acme.chat.app
    Authorization: Basic yZS1yYW5kb20tc2VjcmV0v3JOkF0XG5Qx2

    grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
    assertion=eyJhbGciOiJIUzI1NiIsI...


## Processing Rules

All of Section 5.2 of {{RFC7521}} applies, in addition to the following processing rules:

* Validate the JWT `typ` is `oauth-mtag+jwt` (per {{RFC8725}})
* The `aud` claim MUST identify the token endpoint of the Resource Application as the intended audience of the JWT.


## Response

The Resource Application token endpoint responds with an OAuth 2.0 Token Response, e.g.:


    HTTP/1.1 200 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store
    Pragma: no-cache

    {
      "token_type": "Bearer",
      "access_token": "2YotnFZFEjr1zCsicMWpAA",
      "expires_in": 86400,
      "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
    }



# Security Considerations

## Client Authentication

This specification SHOULD only be supported for confidential clients.  Public clients SHOULD redirect the user with an OAuth 2.0 Authorization Request.

## Step-Up Authentication

In the initial token exchange request, the IdP may require step-up authentication for the subject if the authentication context in the subject's assertion does not meet policy requirements. An `insufficient_user_authentication` OAuth error response may be returned to convey the authentication requirements back to the client similar to [OAuth 2.0 Step-up Authentication Challenge Protocol](https://www.rfc-editor.org/rfc/rfc9470.html)


    HTTP/1.1 400 Bad Request
    Content-Type: application/json
    Cache-Control: no-store

    {
      "error": "insufficient_user_authentication",
      "error_description": "Subject doesn't meet authentication requirements",
      "max_age: "5"
    }


The Client would need to redirect the user back to the IdP to obtain a new assertion that meets the requirements and retry the token exchange.

TBD: It may make more sense to request the JWT authorization grant as an additional `response_type` on the authorization request if using OIDC for SSO when performing a step-up to skip the need for additional token exchange round-trip.


# IANA Considerations

## Media Types

This section registers `oauth-mtag+jwt`, a new media type {{RFC2046}} in the "Media Types" registry {{IANA.MediaTypes}} in the manner described in {{RFC6838}}. It can be used to indicate that the content is a Mutually-Trusted Aauthorization Grant JWT.


## OAuth URI Registration

This section registers `urn:ietf:params:oauth:token-type:mtag-jwt` in the "OAuth URI" subregistry of the "OAuth Parameters" registry {{IANA.OAuth.Parameters}}.

* URN: urn:ietf:params:oauth:token-type:mtag-jwt
* Common Name: Token type URI for a Mutually-Trusted Authorization Grant JWT
* Change Controller: IESG
* Specification Document: This document



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
