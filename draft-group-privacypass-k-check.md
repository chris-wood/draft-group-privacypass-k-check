---
title: "The K-Check Protocol for HTTP Resource Consistency"
abbrev: "K-Check"
category: std

docname: draft-group-privacypass-k-check-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Privacy Pass"
keyword:
 - token
 - extensions
venue:
  group: "Privacy Pass"
  type: "Working Group"
  mail: "privacy-pass@ietf.org"
  github: "chris-wood/draft-group-privacypass-K-Check"

author:
 -
    fullname: J. Doe
    organization: Mars
    email: jdoe@weylandyutani.com

normative:

informative:


--- abstract

TODO Abstract


--- middle

# Introduction

This document describes a protocol mechanism based on {{?DoubleCheck=I-D.schwartz-ohai-consistency-doublecheck}}
for checking that an HTTP resource is consistent with the view of one or more mirrors. In this
context, a mirror is an HTTP server that fetches and caches copies of an HTTP resource
for clients to use for consistency checks. More specifically, clients obtain copies of
a desired resource from a mirror and then compare those copies to their resource.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Terminology

The following terms are used throughout this document:

- Resource: A HTTP resource identified by a URL.
- Normalized resource: A normalized HTTP resource representation is a unique or otherwise
  protocol-specific representation that is derived from an HTTP resource. The process
  of normalization is specific to a protocol and the resource in question.
- Mirror: A HTTP server that fetches and caches HTTP resources.

# Mirror Protocol

The mirror protocol is a simple protocol similar to a reverse-proxy. Each mirror is identified
by a Mirror URI Template {{!RFC6570}}. The scheme for the Mirror URI Template MUST be "https".
The Mirror URI Template uses the Level 3 encoding defined {{Section 1.2 of RFC6570}} and contains
one variables: "target", which is the percent-encoded URL of a HTTP resource to be mirrored.
Example Mirror URI Templates are shown below.

~~~
https://mirror.example/mirror{?target}
https://mirror.example/{target}
~~~

The Mirror URI Template MUST contain the "target" variable exactly once. The variable
MUST be within the path or query components of the URI.

In addition, each mirror is configured with a MIN_VALIDITY_WINDOW parameter, which is an integer
indicating the minimum time for resources the mirror will cache according to their "max-age"
response directive. We refer to the validity window of the mirror response as the period of
time determined by the Cache-Control headers as the response.

Clients send requests to mirror resources after being configured with their corresponding
Mirror URI Template. Clients MUST ignore configurations that do not conform to this template.

Upon receipt of a mirror request, mirrors validate the incoming request. If the request is invalid
or malformed, e.g., the "target" parameter is not a correctly encoded URL, the mirror aborts
and returns a a 4xx (Client Error) to the client. The mirror SHOULD check that the target resource
identified by the "target" parameter is allowed by policy, e.g., so that it is not abused to
fetch arbitrary resources. One way to implement this check is via an allowlist of target URLs.

If the request is valid and allowed, the mirror checks to see if it has a cached version of the
resource identified by the target URL. Mirrors can provide a cached response to a client request
if the following criteria are met:

1. The target URL matches that of a cached response.
1. The cached response is fresh according to its Cache-Control header (see {{Section 4.2 of ?CACHING=RFC9111}}).

If both criteria are met, the mirror encodes the cached response using Binary HTTP {{!BHTTP=RFC9292}}
and returns it to the client in a response. The mirror response incldues a Cache-Control header
with "max-age" directive set to that of the cached response.

Otherwise, mirrors send a GET request to the target resource URL, copying the Accept header from
the client request if present. If this request fails, the mirror returns a 4xx error to the client.
Otherwise, the response to a mirror request is the content that was contained in the target resource.
If this request suceeeds, the mirror checks it for validity. The response is considered valid and stored
in the mirror's cache if the following criteria are met:

1. The response can be cached according to the rules in {{Section 3 of CACHING}}. In particular,
   if the request had a Vary header, this is used in determining whether the mirror's response is valid.
1. The Cache-Control header is present, has a "max-age" response directive that is
   greater than or equal to MIN_VALIDITY_WINDOW, and does not have a "no-store" or "private" directive.

If the response is valid, the response is stored in the mirror's cache. Mirrors purge this cache when
the response is no longer valid according to the Cache-Control headers.

To complete the client request, the mirror then encodes the response using Binary
HTTP {{BHTTP}} and returns it to the client in a response. The mirror response incldues a
Cache-Control header with "max-age" directive set to that of the cached response.

## Mirror Request and Respnose Example

The following example shows two mirror request and response examples. The first one yields a mirror
cache miss and the second one yields a mirror cache hit. The Mirror URI Template is
"https://mirror.example/mirror{?target}", and the target URL is
"https://issuer.example/.well-known/private-token-issuer-directory".

The first client request to the mirror might be the following.

~~~
:method = GET
:scheme = https
:authority = mirror.example
:path = /mirror?target=https%3A%2F%2Fissuer.example%2F.well-known%2Fprivate-token-issuer-directory
accept = application/private-token-issuer-directory
~~~

Upon receipt, the mirror decodes the "target" parameter, inspects its cache for a copy of the
resource, and then constructs a HTTP request to the target URL to fetch the content. If present,
the relay copies the Accept header from the client request to the request sent to the target.
This mirror request to the target might be the following.

~~~
:method = GET
:scheme = https
:authority = target.example
:path = /.well-known/private-token-issuer-directory
accept = application/private-token-issuer-directory
~~~

The target response is then returned to the mirror, like so:

~~~
:status = 200
content-type = application/private-token-issuer-directory
content-length = ...
cache-control: max-age=3600

<Bytes containing a private token issuer directory>
~~~

The mirror caches this response content for the target URL, encodes it using Binary HTTP {{BHTTP}},
and then returns the response to the client:

~~~
:status = 200
content-length = ...
cache-control: max-age=3600

<Bytes containing the target's BHTTP-encoded response>
~~~

When a second client asks for the same request by the mirror it can be served with the cached
copy. The second client's request might be the following:

~~~
:method = GET
:scheme = https
:authority = mirror.example
:path = /mirror?target=https%3A%2F%2Fissuer.example%2F.well-known%2Fprivate-token-issuer-directory
~~~

The mirror validates the request, locates the cached copy of the
"https://issuer.example/.well-known/private-token-issuer-directory" content, and then
returns it to the client without updating its cached copy.

~~~
:status = 200
content-length = ...
cache-control: max-age=3600

<Bytes containing the target's BHTTP-encoded response>
~~~

# K-Check

Clients are configured with the URLs for one or more mirror servers. Each URL identifies an API
endpoint that clients use to obtain mirrored copies of a resource.

The input to K-Check is a candidate HTTP resource, a target URL at which the resource
was obtained, and a normalized representation of the input resource. To check this
resource, the client runs the following steps for each configured mirror.

1. Send a mirror request to the mirror for the target URL. If the request fails, fail
   this mirror check.
1. Otherwise, compute normalized values for the resource based on the mirror’s response.
1. Compare the normalized values to the input normalized representation. If none match,
   fail this mirror check. Otherwise, if there exists one matching resource normalized
   representation, this mirror check succeeds.

If all mirror checks succeed, the client outputs success. Otherwise, the client has
detected an inconsistency and outputs fail.

[[OPEN ISSUE: Can mirrors somehow communicate the number of “active users” to clients? How would mirrors determine client uniqueness? And finally, if mirrors did this accurately, how would clients use this information?]]

## Privacy Pass Profile

Clients are given as input an issuer token key from an origin server and want to check
whether it is consistent with the key that is given to other clients. Let the input key
be denoted token_key and its identifier be token_key_id. Clients are also given as input
the name of the issuer, from which they can construct the target URL for the issuer
directory. If clients have already checked this issuer’s token key, i.e., they’ve
previously run K-Check, they can simply reuse the result up to its expiration. Otherwise,
clients invoke K-Check in parallel with the issuance protocol.

Each issuer directory can yield one or more normalized representations that clients use
in the K-Check protocol. For example, given a mirrored token directory resource like the
following:

~~~
{
  "issuer-request-uri": "https://issuer.example.net/request",
  "token-keys": [
    {
      "token-type": 2,
      "token-key": "MI...AB",
    },
    {
      "token-type": 2,
      "token-key": "MI...AQ",
    }
  ]
}
~~~

Clients would take each “token-key” parameter, base64url-decode the result, and then
derive a token key identifier as defined in the specification. Each token key identifier
is a normalized representation of the resource.

## Oblivious HTTP Profile

Clients can run K-Check for OHTTP in several ways depending on the deployment. In practice,
common deployments are as follows:

1. Clients are configured with gateway configurations; and
1. Clients fetch gateway configurations before use.

In both cases, clients begin with a gateway configuration and want to check it for consistency.
In OHTTP, there is exactly one normalized resource representation for a gateway configuration –
the configuration itself. Before using the configuration to encrypt a binary HTTP message to
the gateway, clients can run K-Check with their configured mirrors to ensure that this
configuration is correct for the given gateway.

# Consistency Properties

The consistency properties of K-Check are as follows:

1. With honest mirrors, clients that successfully check a resource are assured that they
   share the same copy of the resource with the union of mirror clients for each configured mirror.
1. Consistency only holds for the period of time of the minimum mirror validity window.
1. With at least one dishonest mirror, the probability of discovering an inconsistency is 1 - (1 / 2^(k-1)).
   This is the probability that each individual mirror check succeeds in the mirror protocol.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

This document is based on the {{DoubleCheck}} protocol from Benjamin Schwartz.
