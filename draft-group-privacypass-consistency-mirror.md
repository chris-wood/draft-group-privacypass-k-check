---
title: "Checking Resource Consistency with HTTP Mirrors"
abbrev: "Checking Resource Consistency with HTTP Mirrors"
category: std

docname: draft-group-privacypass-consistency-mirror-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Privacy Pass"
keyword:
 - HTTP
 - consistency check
 - mirror
venue:
  group: "Privacy Pass"
  type: "Working Group"
  mail: "privacy-pass@ietf.org"
  github: "chris-wood/draft-group-privacypass-K-Check"

author:
 -
    ins: B. Beurdouche
    name: Benjamin Beurdouche
    organization: Inria & Mozilla
    email: ietf@beurdouche.com
 -
    ins: M. Finkel
    name: Matthew Finkel
    org: Apple Inc.
    email: sysrqb@apple.com
 -
    ins: S. Valdez
    name: Steven Valdez
    org: Google LLC
    email: svaldez@chromium.org
 -
    ins: C. A. Wood
    name: Christopher A. Wood
    org: Cloudflare
    email: caw@heapingbits.net
 -
    ins: T. Pauly
    name: Tommy Pauly
    org: Apple Inc.
    email: tpauly@apple.com

normative:
  PRIVACYPASS: I-D.ietf-privacypass-architecture
  PRIVACYPASS-ISSUANCE: I-D.ietf-privacypass-protocol
  CONSISTENCY: I-D.ietf-privacypass-key-consistency
  OHTTP: I-D.ietf-ohai-ohttp

informative:
  DOUBLE-CHECK: I-D.schwartz-ohai-consistency-doublecheck


--- abstract

This document describes the mirror protocol, an HTTP-based protocol for fetching mirrored
HTTP resources. The primary use case for the mirror protocol is to support HTTP resource
consistency checks in protocols that require clients have a consistent view of some
protocol-specific resource (typically, a public key) for security or privacy reasons,
including Privacy Pass and Oblivious HTTP. To that end, this document also describes how
to use the mirror protocol to implement these consistency checks.

--- middle

# Introduction

Privacy-enhancing protocols such as Privacy Pass {{PRIVACYPASS}} and Oblivious HTTP {{OHTTP}}
require clients to obtain and use a public key for execution. In Privacy Pass,
public keys are used by clients when issuing and redeeming tokens for anonymous
authorization. In Oblivious HTTP (OHTTP), clients use public keys to encrypt
messages to a gateway server.

Deployments of protocols such as Privacy Pass and OHTTP requires that very large sets of clients
share the same key, or even that all clients globally share the same key. This is because the privacy properties depend on the client
anonymity set size. In other words, the key that's used determines the set to which
a particular client belongs. Using a unique, client-specific key would yield an anonymity set
of size one, therefore violating the desired privacy goals of the system. Clients that
use the same key as one another are said to have a consistent view of the key.

{{CONSISTENCY}} describes this notion of consistency in more detail. It also outlines
several designs that can be used as the basis for consistency systems. This document is
a concrete instantiation of one of those designs, "Shared Cache Discovery". In particular,
this document describes the mirror protocol, which is a protocol for fetching a cached copy of
an HTTP resource from so-called mirrors. In this context, a mirror is an HTTP resource that
fetches and caches copies of other HTTP resources that can be returned to clients. In turn,
clients can then use these cached resource copies for consistency checks, i.e., to compare
their expected representation of the resource against that which the mirrors provide.

The mirror protocol can be run one or more times by an application to achieve the desired
consistency criteria. For example, the mirror protocol can be run once with a trusted mirror,
or more than once with many, potentially less trusted mirrors, and used for determining
consistency. {{integration}} provides general guidance for using the mirror protocol for
consistency checks, along with specific profiles of the protocol for Privacy Pass and OHTTP
in {{profile-privacypass}} and {{profile-ohttp}}, respectively.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Terminology

The following terms are used throughout this document:

- Resource: A HTTP resource identified by a URL.
- Normalized resource representation: A unique or otherwise protocol-specific representation
  that is derived from an HTTP resource. The process of normalization is specific to a
  protocol and the resource in question.
- Mirror: A HTTP resource that fetches and caches HTTP resources.

# Mirror Protocol

The mirror protocol is a simple HTTP-based protocol similar to a reverse proxy. Each mirror
resource, henceforth referred to as a mirror, is identified by a Mirror URI Template {{!RFC6570}}.
The scheme for the Mirror URI Template MUST be "https". The Mirror URI Template uses the Level
3 encoding defined {{Section 1.2 of RFC6570}} and contains one variables: "target", which is the
percent-encoded URL of a HTTP resource to be mirrored. Example Mirror URI Templates are shown below.

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
and returns it to the client in a response. The mirror response includes a Cache-Control header
with "max-age" directive set to that of the cached response.

Otherwise, mirrors send a GET request to the target resource URL, copying the Accept header from
the client request if present. If this request fails, the mirror returns a 4xx error to the client.
If this request suceeeds, the mirror checks it for validity. The response is considered valid and stored
in the mirror's cache if the following criteria are met:

1. The response can be cached according to the rules in {{Section 3 of CACHING}}. In particular,
   if the request had a Vary header, this is used in determining whether the mirror's response is valid.
1. The Cache-Control header is present, has a "max-age" response directive that is
   greater than or equal to MIN_VALIDITY_WINDOW, and does not have a "no-store" or "private" directive.

Mirrors purge this cache when the response is no longer valid according to the
Cache-Control headers.

To complete the client request, the mirror then encodes the response using Binary
HTTP {{BHTTP}} and returns it to the client in a response. The mirror response incldues a
Cache-Control header with "max-age" directive set to that of the cached response.

Clients recover the target's mirrored response by Binary HTTP decoding the mirror response
content.

## Mirror Request and Response Example

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

# Using Mirrors for Consistency Checks {#integration}

Clients can use mirrors to implement consistency checks for a candidate HTTP
resource. In particular, in possession of the target URL at which the resource
was obtained, as well as an authoritative representation of the resource, clients
can check to see if this resource is consistent with that of the mirror's as follows:

1. Send a mirror request to the mirror for the target URL. If the request fails, fail
   this consistency check.
1. Otherwise, compute the first valid representation of the resource based on the mirror's response.
1. Compare the computed representation to the input resource representation. If they do not match,
   fail this consistency check. Otherwise, this consistency check succeeds.

The benefits of using the mirror protocol to check consistency depend on a multitude of
factors, including, but not limited to, the number of clients interacting with a particular
mirror, whether or not the mirror is trustworthy, and application requirements for dealing
with consistency check failures.

## Handling Consistency Failures

If a consistency check fails because the mirrored resource did not match, the client
MUST NOT use the original resource. For cases where the check failed because the
client was unable to communicate with the mirror, client policy dictates whether or
not to assume the resource is consistent. Client behavior for what to do in the case
of inconsistency can vary depending on the protocol, availability of alternative services,
and client policy.

If the client has multiple options for equivalent services, it can choose to fall back
from a service that failed a consistency check to one that passed all consistency checks.
For example, if a client has the option of using one of a set of Privacy Pass token
issuers, it can choose an issuer that passes all consistency checks.

If the service that failed the consistency check is an optional optimization for the client,
the client can simply choose to not use the service. For example, if a Privacy Pass token is
used to avoid showing the user a CAPTCHA, but the Privacy Pass token issuer fails the
consistency check, the client can fall back to showing the user a CAPTCHA.

For cases where the client has no alternate services to use, and the service is
required in order to perform user-facing functionality, the client SHOULD report the
error in a visible way that presents the error to the user or an administrator. This
functionality can be similar to how invalid TLS certificates are reported.

## Selecting Mirror Servers

In many of these systems where the mirror protocol might be used, including common
configurations for Privacy Pass and OHTTP, there is already a party who is necessarily
trusted to protect the user's privacy, and whose operational availability is already a
prerequisite for using the system. In OHTTP, this is the Relay; in Privacy Pass it might
be the Attester (in Split Mode) or a transport proxy.

When such a party exists, it is RECOMMENDED that they operate a mirror service
for their users, and that clients do not use any other mirrors for the purposes of
consistency checks. This avoids revealing any metadata about the client's activity
to additional parties and reduces the likelihood of an outage. More information for
implementing this check in the context of Privacy Pass and OHTTP is provided in
{{profile-privacypass}} and {{profile-ohttp}}, respectively.

In some cases, this trusted party can provide consistency enforcement through
a protocol-specific mechanism (e.g., {{?I-D.pw-privacypass-in-band-consistency}}
for Privacy Pass in Split Mode).  Protocol-specific consistency mechanisms may
be preferable to protocol-agnostic consistency checks based on the mirror protocol,
especially if they provide equivalent consistency guarantees with better performance
or reliability.

## Consistency Validity Period

Executing a consistency check provides a client with some assurance that other
clients may be using the same resource {{security-considerations}}. However,
this result only reflects a mirror's view of a resource at a particular point
in time. The client should periodically re-check consistency. The frequency of
re-checking depends on the requirements of the client {{integration}}. Clients
should re-confirm consistency of a cached resource if it is not fresh (see
{{Section 4.2 of ?CACHING=RFC9111}}).

Two strategies for maintaining consistency are:
1. Pre-emptively execute a consistency check for a resource that is expiring soon; and
1. Execute a consistency check of an expired resource at the time of its next use

For strategy 1, clients should avoid executing the check at the time of
expiration. Implementations should decide when re-checking is appropriate
depending on available server capacity and expected client load. For example,
an implementation could configure clients to re-check consistency at some
randomly chosen time within a few hours before the resource's expiration time.

Strategy 2 might be preferrable for a service and resource that is infrequently
used. Clients should consider how this strategy may reveal usage patterns over
time to the mirror.

When the origin server has multiple versions of a resource corresponding to a
URL, it should respond with the resource that is both currently valid and will
remain fresh for the longest amount of time in the future.

## Privacy Pass Profile {#profile-privacypass}

Clients are given as input an issuer token key from an origin server and want to check
whether it is consistent with the key that is given to other clients. Let the input key
be denoted token_key and its identifier be token_key_id. Clients are also given as input
the name of the issuer, from which they can construct the target URL for the issuer
directory. If clients have already checked this issuer’s token key, i.e., they’ve
previously run a consistency check, they can simply reuse the result up to its expiration.
Otherwise, clients invoke a mirror-based consistency check in parallel with the issuance
protocol.

Each issuer directory can yield one or more normalized representations that clients use
in the consistency check. For example, given a mirrored token directory resource like the
following:

~~~
{
  "issuer-request-uri": "https://issuer.example.net/request",
  "token-keys": [
    {
      "token-type": 2,
      "token-key": "MI...AB",
      "not-before": 1686913811,
    },
    {
      "token-type": 2,
      "token-key": "MI...AQ",
    }
  ]
}
~~~

Clients compute the first valid representation of this directory, i.e., the first entry in
the list that the client can use, which might be the key ID of the first key in the
"token-keys" list (depending on the "not-before" value), or the key ID of the second key
in the "token-keys" list. The key ID is computed as defined in {{Section 6.5 of PRIVACYPASS-ISSUANCE}}.

## Oblivious HTTP Profile {#profile-ohttp}

Clients can run consistency checks for OHTTP in several ways depending on the deployment. In practice,
common deployments are as follows:

1. Clients are configured with gateway configurations; and
1. Clients fetch gateway configurations before use.

In both cases, clients begin with a gateway configuration and want to check it for consistency.
In OHTTP, there is exactly one representation for a gateway configuration – the configuration itself.
Before using the configuration to encrypt a binary HTTP message to the gateway, clients can run
a consistency check with their configured mirror(s) to ensure that this configuration is correct
for the given gateway.

# Security Considerations {#security-considerations}

Consistency checks assume that the client-configured set of mirrors is honest. Under this assumption,
the consistency properties of consistency checks based on the mirror protocol are as follows:

1. With honest mirrors, clients that successfully check a resource are assured that they
   share the same copy of the resource with the union of mirror clients for each configured mirror.
1. Consistency only holds for the period of time of the minimum mirror validity window.
1. With at least one dishonest mirror, the probability of discovering an inconsistency is 1 - (1 / 2^(k-1)),
   where k is the number of disjoint consistency checks. This is the probability that each individual
   consistency check succeeds.

Unless all clients share the same configured mirrors, consistency checks using the mirror protocol do
not achieve global consistency as is defined in {{CONSISTENCY}}.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

This document is based on the {{DOUBLE-CHECK}} protocol from Benjamin Schwartz.
