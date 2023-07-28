---
title: "HTTP Upgrade Requires Patience"
abbrev: "HTTP Upgrade Requires Patience"
category: std

docname: draft-schwartz-httpbis-upgrade-patience-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "HTTPBIS"
keyword:
venue:
  github: "bemasc/http-upgrade"
updates: 9110, 9298

author:
 -
    fullname: Benjamin M. Schwartz
    organization: Meta Platforms, Inc.
    email: ietf@bemasc.net

normative:

informative:


--- abstract

The HTTP/1.1 Upgrade mechanism allows the client to request a change to a new protocol.  This document clarifies that, after requesting such an upgrade, the client must wait for a response before sending any more data.


--- middle

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Background {#background}

In HTTP/1.1, a client is permitted to send an "Upgrade" request header field ({{!RFC9110, Section 7.8}}) to indicate that it would like to use this connection for a protocol other than HTTP/1.1.  The server replies with a "101 (Switching Protocols)" status code if it accepts the protocol change.  However, that specification also permits the server to reject the upgrade request:

> A server MAY ignore a received Upgrade header field if it wishes to continue using the current protocol on that connection.

This rejection of the upgrade is common, and can happen for a variety of reasons:

* The server does not support any of the client's indicated Upgrade Tokens (i.e., the client's proposed new protocols), so it continues to use HTTP/1.1.
* The server knows that an upgrade to the offered protocol will not provide any improvement over HTTP/1.1 for this request to this resource, so it chooses to respond in HTTP/1.1.
* The server requires the client to authenticate before upgrading the protocol, so it replies with the status code "401 (Authentication Required)" and provides a challenge in an "Authorization" response header ({{!RFC9110, Section 11.6.2}}).
* The resource has moved, so the server replies with a 3XX redirect status code ({{!RFC9110, Section 3.4}}).

After rejecting the upgrade, the server will continue to interpret subsequent bytes on that connection in accordance with HTTP/1.1.  This document explores the implications of this behavior.

# Inferred Requirements for HTTP Clients {#requirements}

This section documents requirements for clients of protocols that rely on HTTP Upgrade.

## Patience

If the server accepts the upgrade, it interprets the subsequent bytes in accordance with the new protocol.  If it rejects the upgrade, it interprets those bytes as HTTP/1.1.  However, the client doesn't know which interpretation the server will take until it receives the server's response status code.  To prevent protocol confusion, clients MUST NOT send any data after an HTTP/1.1 request until it learns whether the upgrade was accepted.

Protocol confusion is not limited to incidental failures: it can also lead to serious security problems.  Request smuggling attacks are possible when the HTTP client is passing data to the server from an untrusted party, as is common in HTTP.  In the event of protocol confusion, this data could be misparsed as an HTTP request from the client, potentially authenticated by a TLS client certificate.  This could allow the attacker to exercise privileges that are reserved for the client.

A related category of attacks uses request smuggling to exploit vulnerabilities in the server's request parsing logic.  These attacks apply when the HTTP client is trusted, but the payload data source is not.

## Tolerance

If the server rejects the Upgrade, the connection can continue to be used for HTTP/1.1.  There is no requirement to close the connection in response to an Upgrade rejection, and keeping the connection open has performance advantages if additional HTTP requests to this server are likely.  In these situations, clients SHOULD NOT close the connection in response to a rejected upgrade.

# Impact on Existing Upgrade Tokens

At the time of writing, there are four distinct Upgrade Tokens that are registered, associated with published documents, and not marked obsolete.
This section considers the impact of the requirements from {{requirements}} on each of these Upgrade Tokens.

## "HTTP" ({{!RFC9110, Section 2.5}})

{{!RFC9110}} is the source of the requirement quoted in {{background}}.  However, {{!RFC9110}} additionally states:

> A client cannot begin using an upgraded protocol on the connection until it has completely sent the request message (i.e., the client can't change the protocol it is sending in the middle of a message).

That text is hereby updated as follows:

> A client cannot begin using an upgraded protocol on the connection until it has completely sent the request message and received confirmation that the server has accepted the upgrade.

## "TLS" {{?RFC2817}}

{{?RFC2817}} correctly highlights the possibility of the server rejecting the upgrade.  The requirements of this document apply to any use of the "TLS" Upgrade Token, but no change is required in {{?RFC2817}}.

## "WebSocket"/"websocket" {{?RFC6455}}{{?RFC8441}}

{{Section 4.1 of ?RFC6455}} says:

> Once the client's opening handshake has been sent, the client MUST wait for a response from the server before sending any further data.

This advice is consistent with the requirements in {{requirements}}.

## "connect-udp" {{!RFC9298}}

{{Section 5 of !RFC9298}} says:

> A client MAY optimistically start sending UDP packets in HTTP Datagrams before receiving the response to its UDP proxying request.

However, in HTTP/1.1, this "proxying request" is an HTTP Upgrade request.  This upgrade is likely to be rejected in certain circumstances, such as when the proxy responds with a "407 (Proxy Authentication Required)" status code.  Additionally, the contents of the "connect-udp" protocol stream can include untrusted material (i.e., the UDP packets, which might come from other applications on the client device).  This creates the possibility of request smuggling attacks.  To avoid these concerns, this text is updated as follows:

> When using HTTP/2 or later, a client MAY ...

{{Section 3.3 of !RFC9298}} describes the requirement for a successful proxy setup response, including upgrading to the "connect-udp" protocol, and says:

> If any of these requirements are not met, the client MUST treat this proxying attempt as failed and abort the connection.

This text is hereby updated as follows:

> If any of these requirements are not met, the client MUST treat this proxying attempt as failed.  If the "Upgrade" response header field is absent, the client MAY reuse the connection for further HTTP/1.1 requests; otherwise it MUST abort the underlying connection.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
