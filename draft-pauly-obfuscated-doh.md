---
title: "Obfuscated DNS Over HTTPS"
abbrev: Obfuscated DoH
docname: draft-pauly-obfuscated-doh-latest
date:
category: std

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: T. Pauly
    name: Tommy Pauly
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: tpauly@apple.com
  -
    ins: C. Wood
    name: Chris Wood
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: cawood@apple.com
  -
    ins: E. Kinnear
    name: Eric Kinnear
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: ekinnear@apple.com
    
normative:
    ADNS:
      title: "Adaptive DNS: Improving Privacy of Name Resolution"
      authors:
        -
          T. Pauly
    RRTYPE:
      title: Associated Trusted Resolver Records
      authors:
        -
          T. Pauly

--- abstract

This document describes an extension to DNS Over HTTPS (DoH) that allows obfuscation
of client addresses via proxying. This improves privacy of DNS operations by not allowing
any one server entity to be aware of both the client IP address and the content of DNS
queries and answers.

--- middle

# Introduction

DNS Over HTTPS (DoH) {{!RFC8484}} defines a mechanism to allow DNS messages to be
transmitted in encrypted HTTP messages. This provides improved confidentiality and authentication
for DNS interactions in various circumstances.

While DoH can prevent eavesdroppers from directly reading the contents of DNS exchanges, it does
not allow clients to send DNS queries and receive answers from servers without revealing
their local IP address, and thus information about the identity or location of the client.

Proposals such as Oblivious DNS ({{?I-D.annee-dprive-oblivious-dns}}) allow increased privacy
by not allowing any single DNS server to be aware of both the client IP address and the
message contents.

This document defines an Obfuscated DoH, an extension to DoH that allows for a proxied mode
of resolution, in which DNS messages are encrypted in such a way that no DoH server
can independently read both the client IP address and the DNS message contents.

This mechanism is intended to be used as one option for resolving privacy-sensitive content
in a broader context of Adaptive DNS {{ADNS}}.

## Specification of Requirements

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP 14
{{?RFC2119}} {{?RFC8174}} when, and only when,
they appear in all capitals, as shown here.

# Terminology

This document defines the following terms:

Obfuscation Proxy:
: A resolution server that proxies encrypted client DNS queries to another resolution server that
will be able to decrypt the query (the Obfuscation Target).

Obfuscation Target:
: A resolution server that receives encrypted client DNS queries via an Obfuscation Proxy.

# Deployment Requirements

Obfuscated DoH requires, at a minimum:

- Two DoH servers, where one can act as an Obfuscation Proxy, and the other can act as an
Obfuscation Target.
- Public keys for encrypt DNS queries that are passed from a client through a proxy
to a target.
- Client ability to generate one-time-use symmetric keys to encrypt DNS responses.

One mechanism for discovering and privisioning the DoH URI Templates and public keys
is a DNS resource record, NS2 {{RRTYPE}}.

# HTTP Exchange

Unlike direct resolution, obfuscated hostname resolution over DoH involves three parties:

1. The Client, which generates queries.
2. The Obfuscation Proxy, which is a resolution server that receives encrypted queries from the client
and passes them on to another resolution server.
3. The Obfuscation Target, which is a resolution server that receives proxied queries from the client
via the Obfuscation Proxy.

## HTTP Request

Obfuscated DoH queries are created by the Client, and sent to the Obfuscation Proxy using the
hostname in the proxy's DoH URI Template. Obfuscated queries MUST use the POST method,
in which the encrypted query blob is sent as the HTTP message body.

Clients MUST set an HTTP Content-Type header to "application/obfuscated-dns-message"
to indicate that this request is an obfuscated query intended for proxying. Clients also SHOULD
set this same value for the HTTP Accept header.

The :authority psuedo-header MUST indicate the hostname of the Obfuscation Target (not the Obfuscation
Proxy that initially receives the request), and the :path psuedo-header MUST conform to the path specified
by the Obfuscation Target's DoH URI Template.

Note that the authority specified in the request will not match the certificate of the Obfuscation Proxy.

Upon receiving a POST request that contains a "application/obfuscated-dns-message" Content-Type,
the DoH server looks at the :authority and :path psuedo-headers. If the fields match the DoH server's
own hostname and configured path, then it is the target of the query, and can decrypt the query {{encryption}}.
If the fields do not match the local server, then the server is acting as an Obfuscation Proxy. If it is
a proxy, it is expected to open an HTTPS connection to the Obfuscation Target based on the
authority identified in the :authority psuedo-header, and send the request on to the target.

## HTTP Request Example

The following example shows how a client requests that an Obfuscation Proxy, "dnsproxy.example.net",
forwards an encrypted message to "dnstarget.example.net".

~~~
:method = POST
:scheme = https
:authority = dnstarget.example.net
:path = /dns-query
accept = application/obfuscated-dns-message
content-type = application/obfuscated-dns-message
content-length = 106

<Bytes containing the encrypted payload for an Obfuscated DNS query>
~~~

The Obfuscation Proxy then sends the exact same request on to the Obfuscation Target, without modification.

## HTTP Response

The response to an obfuscated query is generated by the Obfuscation Target. It MUST set the
Content-Type HTTP header to "application/obfuscated-dns-message" for all successful responses.
The body of the response contains a DNS message that is encrypted with the client's symmetric key {{encryption}}.

All other aspects of the HTTP response and error handling are inherited from standard DoH.

## HTTP Response Example

The following example shows a response that can be sent from an Obfuscation Target
to a client via an Obfuscation Proxy.

~~~
:status = 200
content-type = application/obfuscated-dns-message
content-length = 154

<Bytes containing the encrypted payload for an Obfuscated DNS response>
~~~

# Obfuscated DNS Message Format {#encryption}

# Security Considerations

# IANA Considerations

This document registers a new media type, "application/obfuscated-dns-message".

Type name: application

Subtype name: obfuscated-dns-message

Required parameters: N/A

Optional parameters: N/A

Encoding considerations: This is a binary format, containing encrypted DNS
requests and responses, as defined in this document.

Security considerations: See this document. The content is an encrypted DNS
message, and not executable code.

Interoperability considerations: This document specifies format of
conforming messages and the interpretation thereof.

Published specification: This document.

Applications that use this media type: This media type is intended
to be used by clients wishing to obfuscate their DNS queries when
using DNS over HTTPS.

Additional information: None

Person and email address to contact for further information: See
Authors' Addresses section

Intended usage: COMMON

Restrictions on usage: None

Author: IETF

Change controller: IETF

# Acknowledgments

This work is inspired by Oblivious DNS {{?I-D.annee-dprive-oblivious-dns}}. Thanks to all of the
authors of that document.