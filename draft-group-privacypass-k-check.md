---
title: "The K-Check Protocol for HTTP Resource Consistency"
abbrev: "K-Check"
category: std

docname: draft-group-privacypass-K-Check-latest
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

The mirror protocol is a simple protocol similar to a reverse-proxy. The mirror API accepts
POST requests with content that contains the URL of the target HTTP resource. (We use POST
here because it is generally bad practice to put content into GET requests, and POST
responses can be cacheable.) Upon receipt, mirrors see if they have a cached version of
the resource identified by the URL locally. If so, they return it to the client.
Otherwise, mirrors send a GET request to the target resource URL. If this request fails,
the mirror returns an error to the client. Otherwise, the response to a mirror request is
the content that was contained in the target resource. With the exception of Cache-Control
headers, the mirror response does not replicate any of the headers that may have been attached
to the HTTP message carrying the target resource.

[[OPEN ISSUE: Should there be some mandatory or RECOMMENDED minimum validity window of resources?]]

We refer to the set of clients that interact with a mirror as mirror clients. We refer to
the validity window of the mirror response as the period of time determined by the Cache-Control
headers as the response.

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

1. Clients are configured with gateway configuration; and
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
