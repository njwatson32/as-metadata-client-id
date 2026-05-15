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

--- abstract

This specification extends the OAuth 2.0 Authorization Server Metadata [RFC8414]
by defining an optional `client_id` parameter for the metadata endpoint request.
This parameter enables authorization servers to dynamically tailor the returned
metadata on a per-client basis, facilitating gradual rollouts of new features or
configuration changes.

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
identity before revealing sensitive metadata could alternatively require client
authentication on the metadata request. However, doing so deviates from the
standard unauthenticated discovery pattern and might require dedicated
credentials or signed assertions, significantly increasing client implementation
complexity.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO

