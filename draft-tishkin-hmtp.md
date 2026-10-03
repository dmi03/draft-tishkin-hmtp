---
title: "HTTP Mail Transfer Protocol"
abbrev: "HTMP"
category: std
docname: draft-tishkin-hmtp-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: ART
keyword:
 - email
 - mail submission
 - HTTP
 - REST
venue:
  github: "dmi03/draft-tishkin-hmtp"
smart_quotes: no

author:
 -
    ins: D. Tishkin
    name: Dmitrii Tishkin
    organization: Independent
    email: hello@dmi03.com

normative:
  RFC5321:
  RFC5322:
  RFC2045:
  RFC6376:
  RFC8259:
  RFC8615:
  RFC9110:
  RFC9421:
  RFC9457:
  RFC9530:
  RFC6749:
  RFC7617:
  RFC6750:
  RFC5598:
  RFC9460:
  RFC5890:
  RFC9113:
  RFC9525:
  RFC9111:
  RFC9728:

informative:
  RFC3207:
  RFC3463:
  RFC6409:
  RFC7208:
  RFC7489:
  RFC7672:
  RFC8461:
  RFC8551:
  RFC8620:
  RFC8621:
  RFC9051:
  RFC9580:
  RFC8552:
  RFC8792:
  RFC6186:
  RFC9114:
  RFC4033:


--- abstract

This document specifies the HTTP Mail Transfer Protocol (HMTP), a
protocol for the submission and transfer of Internet mail over HTTPS.
HMTP covers both the submission of messages by clients to their mail
service and the transfer of messages between mail servers.  Each
message is carried unmodified in the existing Internet Message Format
inside a JSON envelope that holds the information needed for
delivery.  Requests are authenticated by signatures bound to the
sending domain, using keys already published for DomainKeys
Identified Mail (DKIM), which allows receivers to identify senders
independently of their IP addresses.  Servers discover HMTP endpoints
through DNS and fall back to the Simple Mail Transfer Protocol (SMTP)
when a peer does not support HMTP, so that HMTP can be deployed
incrementally alongside existing mail infrastructure.

--- middle

# Introduction

The Simple Mail Transfer Protocol (SMTP) {{RFC5321}} has carried
Internet mail for several decades and remains the only widely
deployed protocol for transferring messages between mail servers.
Message submission by clients {{RFC6409}} uses the same protocol.
Over time, a number of mechanisms have been layered on top of SMTP
to address requirements that did not exist when it was designed:
opportunistic encryption with STARTTLS {{RFC3207}}, sender
authorization with SPF {{RFC7208}}, message signing with DKIM
{{RFC6376}}, policy alignment with DMARC {{RFC7489}}, and
downgrade protection with MTA-STS {{RFC8461}} and DANE {{RFC7672}}.
Each of these mechanisms is useful, but together they form a complex
system that is difficult to deploy and operate correctly.

## Problem Statement

This document is motivated by the following properties of SMTP-based
mail transfer:

Dependence on IP addresses and port 25:
: Receivers commonly make trust decisions based on the IP address of
  the connecting server.  Many cloud and hosting providers restrict
  outbound connections to port 25, and new senders find it difficult
  to establish reputation for their addresses.  IP-based reputation
  also works poorly when many unrelated senders share addresses in
  cloud environments.  As a result, small operators often have to
  relay mail through third-party sending services.

Optional transport security:
: STARTTLS is negotiated in plaintext and can be removed by an active
  attacker.  Certificate validation is frequently not performed.
  Additional mechanisms such as MTA-STS and DANE are required to
  prevent downgrade attacks, and they are not universally deployed.

Limited error reporting:
: SMTP reply codes and free-form text {{RFC3463}} provide limited
  structured information about why a message was rejected, which
  makes automated handling of failures difficult.

Inefficient transfer of large content:
: All content, including large attachments, is transferred inline
  within the message.  The same content is transferred again for
  every receiving domain, and there is no standard way to refer to
  content stored elsewhere with integrity protection.

Separate authentication mechanisms:
: Sender authentication relies on several independent mechanisms
  (SPF, DKIM, and DMARC) that are evaluated separately and can give
  conflicting results.

## Overview of HMTP

HMTP uses HTTP {{RFC9110}} over TLS as its transport and a JSON
{{RFC8259}} envelope to carry delivery information.  Its main
properties are:

- Transport security is mandatory.  HMTP is only defined over HTTPS
  with certificate validation, so no plaintext mode exists and no
  downgrade is possible within the protocol.

- The message itself is not changed.  Messages are carried in the
  Internet Message Format {{RFC5322}} with MIME {{RFC2045}} exactly
  as produced by the sender.  Existing DKIM signatures and end-to-end
  protection such as OpenPGP {{RFC9580}} or S/MIME {{RFC8551}}
  therefore remain valid.

- The envelope is separate from the message.  As in SMTP, the
  envelope sender and recipients are distinct from the header fields
  of the message, which preserves the semantics of blind carbon
  copies, mailing lists, and forwarding.

- Senders are identified by domain.  Each request is signed using
  HTTP Message Signatures {{RFC9421}} together with Digest Fields
  {{RFC9530}}, with keys published in DNS in the same way as DKIM
  keys.  This allows receivers to base trust decisions on domain
  reputation rather than on IP address reputation, and allows
  senders to operate behind proxies and content delivery networks.

- Content may be transferred by reference.  A message or its
  attachments may be provided as URLs together with their size and
  cryptographic hash, so that large content does not have to be
  carried in every request.

- Endpoints are discovered through DNS.  A single SVCB {{RFC9460}}
  record identifies the HMTP endpoint of a domain, and a well-known
  URI {{RFC8615}} describes the capabilities of the server.

- Errors are structured.  Failures are reported using Problem Details
  for HTTP APIs {{RFC9457}}, with a defined mapping to SMTP enhanced
  status codes.

- Deployment is incremental.  When a receiving domain does not
  publish an HMTP endpoint, the sending server delivers the message
  using SMTP.  Because the message is carried unmodified, this
  fallback requires no conversion of the message.

HMTP defines two roles: submission, in which a client hands a message
to its mail service, and transfer, in which one mail server delivers
a message to another.  Clients authenticate to their submission
server using HTTP authentication.  Basic authentication {{RFC7617}}
provides compatibility with existing SMTP AUTH credentials, and
bearer tokens {{RFC6750}} obtained through OAuth 2.0 {{RFC6749}} are
supported for deployments that require stronger authentication.

## Relationship to Other Work

The JSON Meta Application Protocol (JMAP) {{RFC8620}} {{RFC8621}}
provides HTTP-based access to mailboxes and supports message
submission, but it does not define transfer of messages between
mail servers.  HMTP is complementary: a JMAP server can use HMTP to
deliver submitted messages to other domains.

HMTP does not replace DKIM.  DKIM signatures inside the message
continue to protect the message across all hops, including hops that
use SMTP.  HMTP signatures protect each individual HMTP request.

## Non-Goals

The following are outside the scope of this document:

- Defining a new format for messages.  Alternative message formats
  may be defined as extensions.

- Defining spam filtering or reputation algorithms.  These remain a
  matter of local policy at the receiving server.

- Mailbox access and synchronization, which are addressed by IMAP
  {{RFC9051}} and JMAP.

- End-to-end encryption of messages, which continues to be provided
  by existing mechanisms such as OpenPGP and S/MIME.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terminology of the Internet Mail Architecture
{{RFC5598}}.  The terms "JSON object", "member", and "array" are used
as defined in {{RFC8259}}.  The terms "Content-Digest", "signature",
"signer", and "verifier" are used as defined in {{RFC9421}} and
{{RFC9530}}.

The following terms are defined for use in this document:

Message:
: An Internet message in the format defined by {{RFC5322}},
  including any MIME {{RFC2045}} structure.  HMTP transfers the
  Message without modification.

Envelope:
: The JSON object that accompanies a Message in an HMTP request and
  carries the information needed for delivery, such as the envelope
  sender and the envelope recipients.  The Envelope is distinct from
  the header fields of the Message.

Envelope Sender:
: The address to which delivery status notifications are sent,
  carried in the Envelope.  It corresponds to the SMTP "MAIL FROM"
  address {{RFC5321}}.

Envelope Recipient:
: An address to which the Message is to be delivered, carried in the
  Envelope.  It corresponds to an SMTP "RCPT TO" address {{RFC5321}}.

Sending Domain:
: The domain on whose behalf an HMTP request is signed.

Client:
: A Mail User Agent (MUA) that submits Messages to a Submission
  Server using HMTP.

Submission Server:
: A server that accepts Messages from authenticated Clients and
  relays them toward their recipients.  It corresponds to the Message
  Submission Agent (MSA) role described in {{RFC5598}}.

Sending Server:
: A server that transfers a Message to a Receiving Server using
  HMTP.  It corresponds to the Message Transfer Agent (MTA) role
  described in {{RFC5598}}.

Receiving Server:
: A server that accepts Messages from Sending Servers for the domains
  it serves and delivers them to recipients or relays them further.

HMTP Endpoint:
: The HTTPS URI at which a server accepts HMTP requests, as
  identified through discovery (see {{discovery}}).

Content Reference:
: A description of content, such as a Message or an attachment, that
  is not carried in the request itself but is retrieved from a URI,
  together with its size and cryptographic hash.

A single server MAY act in more than one of these roles.

In examples, long lines are wrapped as described in {{RFC8792}}.


# Architecture {#architecture}

HMTP is an application protocol that uses HTTP {{RFC9110}} over TLS
as its transport.  All HMTP requests are HTTP requests sent to an
HMTP Endpoint.

## Roles {#roles}

Submission:
: A Client sends a Message to its Submission Server.  The Client
  authenticates to the Submission Server as described in
  {{submission}}.

Transfer:
: A Sending Server sends a Message to a Receiving Server.  The
  Sending Server signs each request on behalf of the Sending Domain,
  and the Receiving Server verifies the signature as described in
  {{signing}}.

Both interactions use the same Data Model ({{data-model}}).  A
Submission Server typically also acts as a Sending Server for the
Messages it accepts.

## Message Flow {#message-flow}

{{fig-flow}} shows the path of a Message from a Client to a
recipient.

~~~
+--------+  submission   +------------+   transfer   +-----------+
| Client | ------------> | Submission | -----------> | Receiving |
|  (MUA) |    (HMTP)     |   Server   |    (HMTP)    |  Server   |
+--------+               +------------+              +-----------+
                               |                           |
                               | fallback                  v
                               | (SMTP)              +-----------+
                               v                     | Recipient |
                         +------------+              |  Mailbox  |
                         |   SMTP     |              +-----------+
                         |  Server    |
                         +------------+
~~~
{: #fig-flow title="Message Flow"}

1. The Client constructs a Message and an Envelope and submits them
   to its Submission Server.

2. The Submission Server authenticates the Client and verifies that
   the Client is authorized to use the Envelope Sender and the
   originator addresses of the Message.

3. For each recipient domain, the Sending Server performs discovery
   ({{discovery}}).  If the domain publishes an HMTP Endpoint, the
   Sending Server transfers the Message to it using HMTP.  Otherwise,
   the Sending Server delivers the Message using SMTP as described in
   {{smtp-fallback}}.

4. The Receiving Server verifies the signature of the request,
   applies its local policy, and accepts or rejects the Message for
   each Envelope Recipient.

5. If the Message contains Content References, the Receiving Server
   retrieves the referenced content and verifies its hash.

6. The Receiving Server delivers the Message to the recipient
   mailbox or relays it further.

## Coexistence with SMTP {#coexistence}

HMTP does not change the use of MX records {{RFC5321}} or the
operation of existing SMTP servers.  A domain can publish an HMTP
Endpoint in addition to its MX records, and a server can support both
protocols at the same time.  Because the Message is carried
unmodified, a Message can travel over a path that combines HMTP and
SMTP hops without conversion.

# Discovery {#discovery}

A server or Client locates the HMTP Endpoint of a domain in two
steps.  It first queries DNS for the HMTP service record of the
domain ({{dns-record}}) and then retrieves the Capabilities Document
from the target host ({{capabilities-document}}).

## DNS Record {#dns-record}

A domain that supports HMTP publishes an SVCB resource record
{{RFC9460}} at the following owner name:

~~~
_hmtp._tcp.<domain>
~~~

where `<domain>` is the domain part of an email address, in the form
of A-labels {{RFC5890}} for internationalized domain names.

The "_tcp" label is used for consistency with service records of
other mail protocols {{RFC6186}}.  It does not restrict the transport
protocol, which is determined by the "alpn" SvcParam.

The record is interpreted according to {{RFC9460}} with the following
rules:

- The TargetName identifies the host that provides the HMTP Endpoint.
  A TargetName of "." refers to the owner name without the
  "_hmtp._tcp" prefix.

- The "alpn" SvcParam MUST be present and MUST include at least one
  HTTP protocol identifier.  Servers MUST support "h2" {{RFC9113}}
  and MAY support "h3" {{RFC9114}}.

- The "port" SvcParam indicates the TCP or UDP port of the HMTP
  Endpoint.  If it is absent, port 443 is used.

- Other SvcParams, such as "ech", are used as defined in {{RFC9460}}.

- If multiple ServiceMode records are present, they are tried in
  order of priority as specified in {{RFC9460}}.

A Sending Server uses the domain of each Envelope Recipient to locate
the Receiving Server.  A Client uses the domain of its own address to
locate its Submission Server.

When connecting to the target host, the server or Client MUST
validate the TLS certificate of the host against the TargetName, as
described in {{RFC9525}}.  DNS responses SHOULD be validated using
DNSSEC {{RFC4033}} when available.  See {{security}} for the
consequences of an unauthenticated DNS response.

The result of the DNS query is interpreted as follows:

- If the query returns one or more usable SVCB records, the domain
  supports HMTP.

- If the query returns a response indicating that the name does not
  exist or that no SVCB records exist for it, the domain does not
  support HMTP.  A Sending Server then proceeds as described in
  {{smtp-fallback}}.

- If the query fails for any other reason, such as a timeout or a
  server failure, the result MUST be treated as a temporary failure.
  The server MUST NOT conclude that the domain does not support HMTP.

Results of DNS queries MAY be cached for the duration of their TTL.

## Capabilities Document


The Capabilities Document describes the HMTP Endpoint of a host.  It
is retrieved with an HTTP GET request to the well-known URI
{{RFC8615}} "/.well-known/hmtp" on the target host and port
identified by the DNS record.

The response is a JSON object with the media type "application/json"
and contains the following members:

versions:
: REQUIRED.  An array of strings listing the versions of HMTP
  supported by the server.  This document defines version "1".

endpoints:
: REQUIRED.  A JSON object that maps each supported interaction to
  the URI of its endpoint.  This document defines the following
  members:

  - "transfer": the endpoint for transfer ({{transfer}});
  - "submission": the endpoint for submission ({{submission}});
  - "identities": the endpoint that lists the addresses a Client is
    authorized to use ({{identities}}).

  At least one of "transfer" and "submission" MUST be present.  The
  "identities" member MUST NOT be present unless "submission" is
  present.  URIs MUST use the "https" scheme and MAY be relative, in
  which case they are resolved against the URI of the Capabilities
  Document.

authentication:
: REQUIRED if "submission" is present.  An array of strings listing
  the authentication schemes accepted for submission, as described
  in {{client-authentication}}.

oauthResourceMetadata:
: OPTIONAL.  The URI of the OAuth 2.0 Protected Resource Metadata
  {{RFC9728}} of the server.  The URI MUST use the "https" scheme and
  MAY be relative, in which case it is resolved against the URI of
  the Capabilities Document.  If this member is absent, Clients
  locate the metadata as specified in {{RFC9728}}.  The resource
  identifier of a Submission Server is the origin of its submission
  endpoint.

limits:
: OPTIONAL.  A JSON object describing limits of the server.  This
  document defines "maxMessageSize", the maximum size of a Message in
  octets, and "maxRecipients", the maximum number of Envelope
  Recipients in one request.

capabilities:
: OPTIONAL.  A JSON object whose member names identify supported
  extensions and whose values contain parameters of each extension.
  Extensions are registered as described in {{iana}}.

Recipients of the Capabilities Document MUST ignore members they do
not understand.

Servers SHOULD include caching information in the response, and
recipients MAY cache the document as specified in {{RFC9111}}.

If the Capabilities Document cannot be retrieved because of a
connection failure or a 5xx status code, the result MUST be treated
as a temporary failure.  If the server responds with any other error,
or the document is not valid, the server or Client MUST NOT use HMTP
with this host.

The following example shows the DNS records and the Capabilities
Document of a domain whose mail is handled by a provider:

~~~ dns
_hmtp._tcp.example.com.  3600 IN SVCB 1 hmtp.provider.example. (
                                    alpn=h2,h3 )
~~~

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=86400

{
  "versions": ["1"],
  "endpoints": {
    "transfer": "/hmtp/v1/transfer",
    "submission": "/hmtp/v1/submission",
    "identities": "/hmtp/v1/identities"
  },
  "authentication": ["basic", "bearer"],
  "oauthResourceMetadata": "/.well-known/oauth-protected-resource",
  "limits": {
    "maxMessageSize": 52428800,
    "maxRecipients": 1000
  },
  "capabilities": {}
}
~~~

# Data Model {#data-model}


## Envelope


## Content Object


## Attachments


## Capabilities


# Message Transfer {#transfer}


## Request


## Response


## Retries and Idempotency


# Message Submission {#submission}


## Client Authentication {#client-authentication}


## Authorization of Sender Addresses


## Identities {#identities}


# Authentication and Signing {#signing}


## Signature Construction


## Verification Procedure


## Third-Party Senders


# Delivery Status and Errors


## Problem Details


## Mapping to SMTP Status Codes


## Delivery Status Notifications


# SMTP Fallback and Interoperability {#smtp-fallback}


## When to Fall Back


## SMTP to HMTP Gateways


# Deployment and Transition Considerations


# Implementation Status


# Security Considerations {#security}


# Privacy Considerations


# IANA Considerations {#iana}


--- back

# Examples

# Acknowledgments

{:numbered="false"}

TODO acknowledge.
