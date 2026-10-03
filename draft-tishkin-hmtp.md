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
  RFC9460:
  RFC9530:
  RFC6749:
  RFC7617:
  RFC6750:

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


# Architecture


# Discovery


# Data Model


# Message Transfer


# Delivery Status and Errors


# Authentication and Signing


# SMTP Fallback and Interoperability


# Deployment and Transition Considerations


# Implementation Status


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
