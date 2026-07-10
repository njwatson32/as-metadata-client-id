---

title: "OAuth Authorization Server Metadata Client ID Parameter"
abbrev: "OAuth AS Metadata Client ID Param"
category: info

docname: draft-watson-oauth-as-metadata-client-id-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - oauth
 - metadata
 - client_id
 - authorization server
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "njwatson32/as-metadata-client-id"
  latest: "https://njwatson32.github.io/as-metadata-client-id/draft-watson-oauth-as-metadata-client-id.html"

author:
 -
    fullname: Nicholas Watson
    organization: Google, LLC
    email: nwatson@google.com

normative:
  RFC6749:
  RFC8414:
  RFC9111:

informative:
  RFC7009:
  RFC7662:
  RFC9126:

--- abstract

This specification extends the OAuth 2.0 Authorization Server Metadata [RFC8414]
by defining an optional `client_id` parameter for the metadata endpoint request.
This parameter enables authorization servers to dynamically tailor the returned
metadata on a per-client basis, facilitating gradual rollouts of new features or
configuration changes. It also defines the `client_specific_metadata_endpoint`
metadata parameter, which advertises a separate authenticated endpoint where a
client can retrieve a confidential variant of its metadata, without affecting
the unauthenticated, cacheable retrieval pattern of the well-known metadata
endpoint.

--- middle

# Introduction

OAuth 2.0 Authorization Server Metadata [RFC8414] defines a mechanism for an
authorization server to publish its configuration details at a well-known
location. Typically, this metadata document is static and homogeneous for all
clients consuming it.

However, in complex production environments, authorization servers frequently
undergo configuration updates or feature additions where a global rollout
carries significant risk. Deploying a change simultaneously for all clients
makes it difficult to conduct phased rollouts, canary testing, or per-client
deprecations without risking wide-scale disruptions.

This specification introduces an optional `client_id` parameter to the
authorization server metadata request URL. Providing the client identifier
allows the authorization server to tailor the returned metadata document to the
specific client requesting it, enabling phased deployments and targeted
configuration delivery.

Some client-specific metadata is confidential: an endpoint for an unannounced
feature, or configuration that reveals a commercial relationship between the
authorization server and a client. Requiring client authentication on the
well-known endpoint would protect such values, but it deviates from the
standard unauthenticated discovery pattern and interacts poorly with shared
caches such as CDNs. This specification therefore separates the two concerns.
The well-known metadata endpoint remains unauthenticated and cacheable and
returns only metadata the authorization server is willing to publish. A
separate client-specific metadata endpoint
({{client-specific-metadata-endpoint}}), discovered through the unauthenticated
document, requires authentication and returns the complete metadata document
for a client, including confidential values. Clients opt in to the
authenticated variant by calling that endpoint; clients that need only public
metadata never contact it.

# Requirements Notation and Conventions

{::boilerplate bcp14-tagged}

# Terminology

"Client" and "Authorization Server" are used as specified in OAuth 2.0
[RFC6749].

# Metadata Request with Client ID

Clients MAY include the `client_id` query parameter when making an HTTP GET
request to the authorization server metadata endpoint (e.g.,
`/.well-known/oauth-authorization-server`).

## Parameter Definition

client_id
:   OPTIONAL. The client identifier as described in Section 2.2 of [RFC6749].

## Authorization Server Behavior

When the `client_id` parameter is present, an authorization server MAY customize
the metadata response (including response headers like `Cache-Control`) based on
the specific client. For example, an authorization server could:

*   Return new endpoints or supported scopes only to clients that are
    participating in a beta testing program.
*   Continue serving legacy authentication methods to backwards-incompatible
    clients while newer clients receive an updated configuration.
*   Gradually shift traffic to a new signing key or token endpoint URL for
    specific clients over time.

If the authorization server does not recognize the `client_id` or chooses not to
provide customized metadata, it MUST ignore the parameter and return the
standard, global authorization server metadata. This ensures full backward
compatibility.

## Client Behavior

[RFC8414] does not specify authorization server behavior when query parameters
are present, so it's possible that some authorization servers that don't adopt
this specification will return an error when given a `client_id` parameter.
Clients SHOULD gracefully handle HTTP 400 errors and retry their request without
the `client_id` parameter.

## Open Question: Indicate support for this specification?

Should the authorization server indicate its support for this specification?

1.  No.
    *   It's unclear what a client would do with this information other than
        make bad assumptions. Knowing whether AS metadata was specifically
        tailored shouldn't change a client's behavior.
2.  Add a global boolean flag `per_client_as_metadata_supported`.
    *   This only seems useful if there are many AS Metadata deployments today
        that will reject requests with query parameters. But also there's not
        much difference between (a) calling AS metadata without a client_id,
        seeing `per_client_as_metadata_supported`, and retrying with client_id
        and (b) following [Client Behavior] above. If anything I'd expect (b)
        to be better in practice because most AS aren't going to fail when given
        an unrecognized URL param.
3.  Echo the `client_id` back as a field in the AS Metadata response.
    *   It's unclear what benefits this provides over Option 2, but it has the
        additional downside of preventing the AS from serving a static file for
        non-customized clients.

# Client-Specific Metadata Endpoint {#client-specific-metadata-endpoint}

The `client_id` parameter preserves the unauthenticated retrieval pattern of
[RFC8414]: any party can fetch the tailored document, and responses remain
cacheable by shared caches. That pattern is the right default, but it cannot
serve metadata that the authorization server considers confidential for a
specific client.

This specification keeps the well-known endpoint unauthenticated and instead
defines a separate, authenticated client-specific metadata endpoint. The
endpoint is published and protected the same way as the token revocation
[RFC7009] and token introspection [RFC7662] endpoints: it is advertised in
authorization server metadata together with its supported client
authentication methods, and the client authenticates to it directly using its
registered client authentication method. The rationale for a separate
endpoint, rather than an authenticated variant of the well-known URL, is
discussed in {{design-rationale}}.

## Endpoint Discovery {#endpoint-discovery}

Authorization servers that offer confidential client-specific metadata
advertise the endpoint with the following authorization server metadata
parameter:

{: newline="true"}
client_specific_metadata_endpoint
:   OPTIONAL. URL of the authorization server's client-specific metadata
    endpoint ({{client-specific-metadata-request}}). The URL MUST use the
    `https` scheme and MUST be distinct from the well-known metadata URL.

client_specific_metadata_endpoint_auth_methods_supported
:   OPTIONAL. JSON array containing a list of client authentication methods
    supported by the client-specific metadata endpoint. The values are those
    used with the token endpoint, as defined for
    `token_endpoint_auth_methods_supported` in Section 2 of [RFC8414]. If
    omitted, the default is `client_secret_basic`.

client_specific_metadata_endpoint_auth_signing_alg_values_supported
:   OPTIONAL. JSON array containing a list of the JWS signing algorithms
    (`alg` values) supported by the client-specific metadata endpoint for the
    signature on the JWT used to authenticate the client for the
    `private_key_jwt` and `client_secret_jwt` authentication methods. The
    value `none` MUST NOT be used.

These parameters parallel the advertisement of the token revocation [RFC7009]
and token introspection [RFC7662] endpoints in [RFC8414], so an authorization
server can publish and protect all three endpoints the same way.

The parameter's presence in a metadata response tells the client that an
authenticated variant of its metadata is available; its absence tells the
client that the unauthenticated response is complete. An authorization server
MAY include the parameter only in responses tailored to clients for which
confidential metadata exists.

For example, the unauthenticated metadata response for
`?client_id=s6BhdRkqt3` could include:

~~~ json
{
  "issuer": "https://as.example.com",
  "authorization_endpoint": "https://as.example.com/authorize",
  "token_endpoint": "https://as.example.com/token",
  "client_specific_metadata_endpoint":
    "https://as.example.com/client-metadata",
  "client_specific_metadata_endpoint_auth_methods_supported":
    ["private_key_jwt"]
}
~~~

## Metadata Request {#client-specific-metadata-request}

The client requests its metadata with an HTTP POST request to the
client-specific metadata endpoint with a `Content-Type` of
`application/x-www-form-urlencoded`, authenticating exactly as it would at
the token endpoint using its registered client authentication method
(Section 2.3 of [RFC6749]). This mirrors the token revocation [RFC7009] and
token introspection [RFC7662] endpoints.

The request defines no parameters of its own; the authenticated client
identity determines whose metadata is returned. The request body is empty
except for parameters carried by the client authentication method itself,
such as `client_assertion` and `client_assertion_type` for the
`private_key_jwt` method:

~~~ http
POST /client-metadata HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded

client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A
client-assertion-type%3Ajwt-bearer&
client_assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6IjE2In0.eyJpc3Mi...
~~~

(Line breaks within the request body are for display purposes only.)

Only confidential clients can retrieve confidential metadata. A client that
cannot authenticate cannot be distinguished from an impersonator, so no
metadata served to it can remain confidential.

## Metadata Response {#client-specific-metadata-response}

A successful response is a complete authorization server metadata document as
defined in Section 3.2 of [RFC8414], with global values, public
client-specific values, and confidential client-specific values already merged
by the authorization server. The document MUST be usable on its own; clients
do not merge it with the unauthenticated document, so no client-side
precedence rules are needed.

The response MUST include a `Cache-Control` header field containing the
`no-store` directive so that it is never stored by caches
({{caching-considerations}}).

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
~~~
~~~ json
{
  "issuer": "https://as.example.com",
  "authorization_endpoint": "https://as.example.com/authorize",
  "token_endpoint": "https://as.example.com/token",
  "client_specific_metadata_endpoint":
    "https://as.example.com/client-metadata",
  "client_specific_metadata_endpoint_auth_methods_supported":
    ["private_key_jwt"],
  "pushed_authorization_request_endpoint":
    "https://as.example.com/beta/par"
}
~~~

In this example, the authorization server is piloting pushed authorization
requests [RFC9126] with this client. The
`pushed_authorization_request_endpoint` value is confidential and appears only
in the authenticated document, not in any unauthenticated metadata response.

## Error Responses {#client-specific-metadata-errors}

A request whose client authentication is missing or invalid is answered as
the token endpoint would answer it: with the `invalid_client` error defined
in Section 5.2 of [RFC6749], using HTTP status code 401 and a
`WWW-Authenticate` challenge when the client attempted to authenticate
through the `Authorization` header field. The error response MUST be
identical whether or not the client has any client-specific metadata, so that
the endpoint does not act as an oracle for which clients receive tailored
configuration. A successfully authenticated client always receives a complete
document, even when it contains no confidential values.

Unlike the well-known endpoint, the client-specific metadata endpoint MUST NOT
fall back to returning global or public metadata when authentication fails.
Fallback is a property of the unauthenticated endpoint only; here it would let
a client mistake the public variant for its complete configuration.

# Caching Considerations {#caching-considerations}

The two endpoints have opposite caching goals. This specification keeps cache
behavior a static property of each URL rather than a function of request
credentials, so that shared caches such as CDNs can be configured with simple
per-path rules.

## Well-Known Metadata Endpoint {#caching-well-known}

Responses from the well-known endpoint, with or without the `client_id`
parameter, remain cacheable by private and shared caches under the normal HTTP
rules [RFC9111]. Because the cache key includes the query string, tailored
responses are cached per client identifier. Authorization servers SHOULD set
an explicit `Cache-Control` header field (for example, `public, max-age=3600`)
with a freshness lifetime chosen for how quickly rollout changes need to
propagate.

The authorization server MUST NOT vary the content of a well-known metadata
response based on credentials presented with the request and MUST NOT emit
`Vary: Authorization` for it. HTTP forbids shared caches from reusing
responses to requests that carried an `Authorization` header field unless the
response explicitly permits it (Section 3.5 of [RFC9111]), but CDN
configurations frequently normalize or ignore that header field, and handling
of `Vary: Authorization` is inconsistent across implementations. A single
misconfiguration would serve one client's confidential response from a shared
cache to all parties. Keeping the well-known URL exclusively public removes
this failure mode.

## Client-Specific Metadata Endpoint {#caching-client-specific}

Responses from the client-specific metadata endpoint MUST NOT be stored by
caches. Requests use the POST method, whose responses are not reusable by
caches without explicit freshness information [RFC9111], and the mandatory
`Cache-Control: no-store` header field
({{client-specific-metadata-response}}) makes the exclusion explicit. Because
the endpoint is a dedicated URL, a CDN in front of the authorization server
can additionally be configured to bypass caching for the path with a static
rule; no credential-dependent cache configuration is required.

Because every request reaches the origin, authorization servers MAY apply
rate limits to the endpoint. Clients SHOULD retain a retrieved document and
reuse it rather than fetching it for each protocol interaction, refreshing it
on the same schedule they would apply to the well-known metadata document.

# Security Considerations

Providing tailored metadata introduces nuances to the threat model of
authorization server configuration discovery.

## Unauthenticated Requests vs. Client Authentication

The Authorization Server Metadata endpoint defined in [RFC8414] is typically
accessed via unauthenticated HTTP GET requests over TLS. Accepting a bare
`client_id` query parameter without any form of client authentication carries
certain tradeoffs:

1.  **Ease of Deployment:** The primary advantage of accepting a bare
    `client_id` is simplicity and compatibility with existing HTTP GET metadata
    retrieval patterns. Clients can fetch metadata early in their lifecycle,
    even before holding or presenting any credentials.
2.  **Information Disclosure:** Because the request is unauthenticated, an
    attacker or unauthorized third party could query the metadata endpoint with
    arbitrary client identifiers. This could allow the attacker to learn
    specific client configurations, such as which clients have access to beta
    features or legacy authentication methods. In most environments, metadata is
    considered public information, but care SHOULD be taken if specific metadata
    values reveal sensitive business logic or proprietary features.

Authorization servers that require strong assurance of the requesting client's
identity before revealing certain metadata SHOULD NOT require client
authentication on the well-known metadata request. Instead, they SHOULD serve
those values only from the client-specific metadata endpoint
({{client-specific-metadata-endpoint}}), which reuses the client's existing
registered credentials rather than requiring dedicated ones. Values the
authorization server considers confidential for a client MUST NOT appear in
unauthenticated metadata responses. Serving public and confidential variants
of the same URL, distinguished by request credentials, is NOT RECOMMENDED for
the caching reasons given in {{caching-considerations}} and because a client
whose credentials are rejected could silently receive the public variant and
mistake it for its complete configuration.

Because the unauthenticated document is cached, a client can briefly observe a
stale public document alongside a fresh authenticated one. Authorization
servers SHOULD keep each metadata value in exactly one of the two variants so
that skew between caches cannot produce contradictory values.

## Metadata Integrity

Metadata documents served through CDNs or other intermediaries can be
protected against modification in transit or in cache with signed metadata, as
defined in Section 2.1 of [RFC8414]. An authorization server MAY also include
`signed_metadata` in the client-specific metadata response, giving the client
a verifiable record of the configuration it was issued.

# IANA Considerations

## OAuth Authorization Server Metadata Registration

This specification registers the following metadata names in the IANA "OAuth
Authorization Server Metadata" registry established by [RFC8414]. The change
controller for each is the IESG, and the specification document for each is
{{endpoint-discovery}} of this document.

{: newline="true"}
client_specific_metadata_endpoint
:   Metadata Description: URL of the authorization server's client-specific
    metadata endpoint

client_specific_metadata_endpoint_auth_methods_supported
:   Metadata Description: JSON array containing a list of client
    authentication methods supported by the client-specific metadata endpoint

client_specific_metadata_endpoint_auth_signing_alg_values_supported
:   Metadata Description: JSON array containing a list of the JWS signing
    algorithms supported by the client-specific metadata endpoint for the
    signature on the JWT used to authenticate the client

--- back

# Design Rationale {#design-rationale}

An alternative design would serve both variants from the well-known URL,
returning richer metadata when the request carries valid client credentials.
This specification instead adds a second endpoint, for three reasons.

Public metadata is the common case. The well-known document is fetched
overwhelmingly by parties that need only public metadata, and [RFC8414]
deployments serve it from static files and shared caches. The `client_id`
parameter preserves that profile: a tailored public document is still a
static resource keyed by its URL. A well-known URL whose response depends on
request credentials would burden the common case with the requirements of the
rare one. The two mechanisms therefore layer: the query parameter alone
serves deployments whose tailored metadata is public, and the endpoint is an
additive step for the narrower case of confidential metadata, discoverable
from the metadata document itself and available only to confidential clients.

Hosting infrastructure configures caching per URL. When cache behavior is a
static property of each URL, a CDN serves the well-known path under one cache
rule and bypasses the endpoint path under another; both rules are simple and
fail safe. When cacheability instead depends on request credentials,
correctness rests on `Vary` and `Authorization` handling that is inconsistent
across shared caches, and a single misconfiguration serves one client's
confidential metadata to all parties ({{caching-considerations}}). A
credentialed variant of the well-known URL would also give failed
authentication a poor failure mode: an error on a URL that must remain
unauthenticated, or a silent downgrade to the public document that the client
mistakes for its complete configuration.

Existing OAuth implementations already contain both halves. Retrieving the
well-known document with an additional query parameter requires no changes to
metadata libraries. The endpoint reuses the request shape, client
authentication, and error semantics of token revocation [RFC7009] and token
introspection [RFC7662], so authorization servers can protect it with the
same machinery they already apply to those endpoints, and clients
authenticate with the code they already use at the token endpoint. Neither
side implements a new mechanism; each configures a URL.

# Acknowledgments
{:numbered="false"}

TODO

