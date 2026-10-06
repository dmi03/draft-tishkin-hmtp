---
title: "HTTP Mail Transfer Protocol"
abbrev: "HMTP"
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
  RFC2045:
  RFC3030:
  RFC3339:
  RFC3461:
  RFC3463:
  RFC3464:
  RFC3553:
  RFC3986:
  RFC4033:
  RFC4648:
  RFC5321:
  RFC5322:
  RFC5598:
  RFC5890:
  RFC6152:
  RFC6376:
  RFC6409:
  RFC6531:
  RFC6532:
  RFC6585:
  RFC6749:
  RFC6750:
  RFC7493:
  RFC7617:
  RFC8126:
  RFC8259:
  RFC8463:
  RFC8552:
  RFC8615:
  RFC8689:
  RFC9110:
  RFC9111:
  RFC9113:
  RFC9421:
  RFC9457:
  RFC9460:
  RFC9525:
  RFC9530:
  RFC9728:

informative:
  RFC2034:
  RFC2046:
  RFC3207:
  RFC4865:
  RFC4954:
  RFC7208:
  RFC7489:
  RFC6522:
  RFC6797:
  RFC7505:
  RFC7672:
  RFC8098:
  RFC8301:
  RFC8461:
  RFC8551:
  RFC8601:
  RFC8620:
  RFC8621:
  RFC8792:
  RFC9051:
  RFC9114:
  RFC9580:


--- abstract

This document specifies the HTTP Mail Transfer Protocol (HMTP), a
protocol for the submission and transfer of Internet mail over HTTPS.
HMTP covers both the submission of messages by clients to their mail
service and the transfer of messages between mail servers.  Each
message is carried unmodified in the existing Internet Message Format
inside a JSON envelope that holds the information needed for
delivery.  Large messages and attachments can be transferred by
reference, with their integrity protected by cryptographic hashes.
Requests are authenticated by signatures bound to the sending domain,
using keys published with the DomainKeys Identified Mail (DKIM) key
publication mechanism, which allows receivers to identify senders
independently of their IP addresses.  Servers discover HMTP endpoints
through DNS.  To allow incremental deployment alongside existing mail
infrastructure, a server can optionally fall back to the Simple Mail
Transfer Protocol (SMTP) when a peer does not support HMTP.

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
: SMTP reply codes {{RFC5321}} and enhanced status codes {{RFC3463}},
  accompanied by free-form text, provide limited structured
  information about why a message was rejected, which makes automated
  handling of failures difficult.

Inefficient transfer of large content:
: All content, including large attachments, is transferred inline
  within the message.  The same content is transferred again for
  every receiving domain, and there is no standard way to refer to
  content stored elsewhere with integrity protection.

Separate authentication mechanisms:
: Sender authentication relies on several independent mechanisms
  (SPF, DKIM, and DMARC) that are evaluated separately and can give
  conflicting results.

No protection against duplicate delivery:
: If the connection is lost after a server has accepted a message
  but before the client has received the reply, the client cannot
  tell whether the message was accepted.  Retrying the transaction
  can result in the message being delivered twice.

## Overview of HMTP

HMTP uses HTTP {{RFC9110}} over TLS as its transport and a JSON
{{RFC8259}} envelope to carry delivery information.  Its main
properties are:

- Transport security is mandatory.  HMTP is only defined over HTTPS
  with certificate validation, so no plaintext mode exists and no
  downgrade is possible within the protocol.  A domain can
  additionally publish a policy that prevents senders from falling
  back to SMTP.

- The message itself is not changed.  Messages are carried in the
  Internet Message Format {{RFC5322}} with MIME {{RFC2045}} exactly
  as produced by the sender.  Servers only prepend trace header
  fields, as SMTP servers do.  Existing DKIM signatures and
  end-to-end protection such as OpenPGP {{RFC9580}} or S/MIME
  {{RFC8551}} therefore remain valid.

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

- Content may be transferred by reference.  A message, or any part
  of it such as an attachment, may be provided as a URL together
  with its size and cryptographic hash.  The receiving server
  retrieves such content directly from where it is stored, so that
  large content does not have to be carried in every request.

- Requests are idempotent.  Each request carries an identifier that
  allows the receiving server to recognize a retried request and
  avoid delivering the same message twice.

- Endpoints are discovered through DNS.  A single SVCB {{RFC9460}}
  record identifies the HMTP endpoint of a domain, and a well-known
  URI {{RFC8615}} describes the capabilities of the server.

- Errors are structured.  Failures are reported using Problem Details
  for HTTP APIs {{RFC9457}}, with a defined mapping to SMTP enhanced
  status codes.

- Deployment is incremental.  HMTP is a complete protocol on its own
  and does not depend on SMTP.  An implementation can optionally
  deliver messages using SMTP when a receiving domain does not
  publish an HMTP endpoint.  Because the message is carried
  unmodified, such a fallback requires no conversion of the message,
  provided that the SMTP server supports the SMTP extensions that the
  message requires.

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


# Conventions and Definitions {#conventions}

{::boilerplate bcp14-tagged}

This document uses the terminology of the Internet Mail Architecture
{{RFC5598}}.  The terms "JSON object", "member", and "array" are used
as defined in {{RFC8259}}.  The terms "signature", "signer",
"verifier", "covered components", and "signature parameters" are used
as defined in {{RFC9421}}.  The Content-Digest header field is defined
in {{RFC9530}}.

The following terms are defined for use in this document:

Message:
: An Internet message in the format defined by {{RFC5322}},
  including any MIME {{RFC2045}} structure.  HMTP transfers the
  Message without modification, except for the prepending of trace
  header fields described in {{trace}} and the changes permitted at
  submission in {{message-validation}}.

Envelope:
: The JSON object that accompanies a Message in an HMTP request and
  carries the information needed for delivery, such as the envelope
  sender and the envelope recipients.  The Envelope is distinct from
  the header fields of the Message.  It is specified in {{envelope}}.

Envelope Sender:
: The address to which delivery status notifications are sent,
  carried in the Envelope.  It corresponds to the SMTP "MAIL FROM"
  address {{RFC5321}}.  The Envelope Sender may be empty, which
  corresponds to the SMTP null reverse-path "<>".

Envelope Recipient:
: An address to which the Message is to be delivered, carried in the
  Envelope.  It corresponds to an SMTP "RCPT TO" address {{RFC5321}}.

Sending Domain:
: The domain on whose behalf an HMTP transfer request is signed,
  as identified by the key used to sign the request (see
  {{signing}}).

Client:
: A Mail User Agent (MUA), or another program acting on behalf of a
  user, that submits Messages to a Submission Server using HMTP.

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

Capabilities Document:
: The JSON document that describes the HMTP Endpoints, limits,
  policy, and extensions of a host (see {{capabilities-document}}).

Content Object:
: A JSON object that carries content, such as a Message, either
  inline, by reference, or as a sequence of segments (see
  {{content-object}}).

Content Reference:
: A Content Object that describes content which is not carried in
  the request itself but is retrieved from an HTTPS URI, together
  with its size and cryptographic hash.

Transfer Identifier:
: The identifier chosen by the sender of an HMTP request that allows
  the recipient of the request to recognize a retried request (see
  {{idempotency}}).

A single server MAY act in more than one of these roles.

In examples, long lines are wrapped as described in {{RFC8792}}.
HTTP messages in examples are shown in HTTP/1.1 syntax for
readability; the semantics of HMTP are independent of the HTTP
version in use.  Unless stated otherwise, base64-encoded data,
digests, and signature values in examples are abbreviated or
illustrative and are not computed over the example content.


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
                               | (SMTP, optional)    +-----------+
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
   a Sending Server that supports SMTP fallback delivers the Message
   using SMTP as described in {{smtp-fallback}}.

4. The Receiving Server verifies the signature of the request,
   applies its local policy, and accepts or rejects the Message for
   each Envelope Recipient.

5. If the request contains Content References, the Receiving Server
   retrieves the referenced content and verifies its size and hash,
   either before it responds or after it has accepted the Message
   (see {{reference-retrieval}}).

6. The Receiving Server prepends a trace header field to the Message
   and delivers the Message to the recipient mailbox or relays it
   further.

## Coexistence with SMTP {#coexistence}

HMTP does not change the use of MX records {{RFC5321}} or the
operation of existing SMTP servers.  A domain can publish an HMTP
Endpoint in addition to its MX records, and a server can support both
protocols at the same time.  Because the Message is carried
unmodified, a Message can travel over a path that combines HMTP and
SMTP hops without conversion.

Support for SMTP is not required for an HMTP implementation.  A
server that implements only HMTP is fully conformant to this
document; it can exchange mail only with domains that publish an HMTP
Endpoint.

# Discovery {#discovery}

A server or Client locates the HMTP Endpoint of a domain in two
steps.  It first queries DNS for the HMTP service record of the
domain ({{dns-record}}) and then retrieves the Capabilities Document
from the target host ({{capabilities-document}}).

## DNS Record {#dns-record}

A domain that supports HMTP publishes one or more SVCB resource
records {{RFC9460}} at the following owner name:

~~~
_hmtp.<domain>
~~~

where `<domain>` is the domain part of an email address, in the form
of A-labels {{RFC5890}} for internationalized domain names.  The
"_hmtp" label is a globally scoped underscored node name
{{RFC8552}}.  It does not indicate a transport protocol, which is
determined by the "alpn" SvcParam.

Addresses whose domain part is an address literal {{RFC5321}} have no
associated DNS name; HMTP is not used for such addresses.

The records are interpreted according to {{RFC9460}} with the
following rules:

- AliasMode records are processed as specified in {{RFC9460}}.  An
  AliasMode record whose TargetName is "." indicates that the domain
  does not provide HMTP ({{Section 2.5.1 of RFC9460}}).

- In ServiceMode records, the TargetName identifies the host that
  provides the HMTP Endpoint.  Because the owner name of the record
  is not a host name, the TargetName of a ServiceMode record MUST NOT
  be ".".  A ServiceMode record whose TargetName is "." is unusable.

- The "alpn" SvcParam MUST be present and MUST include "h2".  It MAY
  also include "h3".  Servers MUST support HTTP/2 {{RFC9113}} and MAY
  support HTTP/3 {{RFC9114}}.  A ServiceMode record without an "alpn"
  SvcParam, or whose "alpn" SvcParam does not include "h2", is
  unusable.

- The "port" SvcParam indicates the TCP or UDP port of the HMTP
  Endpoint.  If it is absent, port 443 is used.

- Other SvcParams, such as "ipv4hint" and "ipv6hint", are used as
  defined in {{RFC9460}} and in the documents that define them.  A
  record that lists an unsupported key in the "mandatory" SvcParam is
  unusable, as specified in {{RFC9460}}.

- If multiple ServiceMode records are present, they are tried in
  order of priority as specified in {{RFC9460}}.

A Sending Server uses the domain of each Envelope Recipient to locate
the Receiving Server.  A Client uses the domain of its own address to
locate its Submission Server.  A Client MAY instead be configured
with the URI of the Capabilities Document of its Submission Server.

When connecting to the target host, the server or Client MUST
validate the TLS certificate of the host against the TargetName, as
described in {{RFC9525}}.  DNS responses SHOULD be validated using
DNSSEC {{RFC4033}} when available.  See {{security-dns}} for the
consequences of an unauthenticated DNS response.

The result of the DNS query is interpreted as follows:

- If the query returns one or more usable ServiceMode records, the
  domain supports HMTP.

- If the query returns a response indicating that the name does not
  exist or that no SVCB records exist for it, if the only record is
  an AliasMode record whose TargetName is ".", or if all returned
  records are unusable, the domain does not support HMTP.  A Sending
  Server then proceeds as described in {{smtp-fallback}}.

- If the query fails for any other reason, such as a timeout or a
  server failure, the result MUST be treated as a temporary failure.
  The server MUST NOT conclude that the domain does not support HMTP.

Results of DNS queries MAY be cached for the duration of their TTL.

## Capabilities Document {#capabilities-document}

The Capabilities Document describes the HMTP Endpoint of a host.  It
is retrieved with an HTTP GET request to the well-known URI
{{RFC8615}} "/.well-known/hmtp" on the target host and port
identified by the DNS record.  The request carries no
authentication, and servers MUST NOT require authentication or a
signature to retrieve the Capabilities Document.

The response is a JSON object with the media type "application/json"
and contains the following members:

versions:
: REQUIRED.  A JSON object whose member names identify the versions
  of HMTP supported by the server and whose values are Version
  Objects.  Version names are strings of decimal digits without
  leading zeros; a higher number denotes a later version.  This
  document defines version "1".  When the server and the sender of a
  request have more than one version in common, the sender SHOULD use
  the highest of them.  The object MUST contain at least one member.

  A Version Object contains the following member:

  endpoints:
  : REQUIRED.  A JSON object whose member names identify the
    interactions supported in this version and whose values are
    Endpoint Objects ({{endpoint-objects}}).  This document defines
    the following members:

    - "transfer": the Transfer Endpoint Object, which describes the
      endpoint for transfer ({{transfer}});
    - "submission": the Submission Endpoint Object, which describes
      the endpoint for submission ({{submission}}) and the related
      identities endpoint ({{identities}}).

    At least one of "transfer" and "submission" MUST be present.
    Additional endpoints MAY be defined by capabilities
    ({{capabilities}}) and are registered as described in
    {{iana-endpoints}}.

limits:
: OPTIONAL.  A JSON object describing limits of the server.  All
  values are non-negative integers.  This document defines the
  following members:

  - "maxMessageSize": the maximum size of a Message in octets, after
    the reconstruction of all segments described in
    {{content-object}}.
  - "maxRequestSize": the maximum size of the content of an HMTP
    request in octets, that is, of the JSON document itself.  Content
    that is transferred by reference does not count toward this
    limit.
  - "maxRecipients": the maximum number of Envelope Recipients in one
    request.  The value MUST NOT be less than 100.
  - "maxIdLength": the maximum length of a Transfer Identifier in
    characters (see {{envelope}}).  The value MUST NOT be less than
    22 and MUST NOT be greater than 255.  If this member is absent,
    the maximum length is 64.

  The absence of any other limit means that the server does not
  advertise it; the server can still reject a request that exceeds a
  limit it applies, as described in {{problem-details}}.

policy:
: OPTIONAL.  A JSON object that expresses the downgrade policy of the
  domains served by this host.  It is specified in {{policy}}.

capabilities:
: OPTIONAL.  A JSON object whose member names identify supported
  extensions and whose values contain parameters of each extension.
  Extensions are specified as described in {{capabilities}}.  Their
  names are either registered with IANA or URIs that serve only as
  unique names ({{capability-names}}).

Recipients of the Capabilities Document MUST ignore members they do
not understand.

### Endpoint Objects {#endpoint-objects}

An Endpoint Object is a JSON object that describes one endpoint and
the parameters that apply to it.  Every Endpoint Object contains the
following member:

uri:
: REQUIRED.  The URI of the endpoint.  The URI MUST use the "https"
  scheme and MAY be relative, in which case it is resolved against
  the URI of the Capabilities Document {{RFC3986}}.

The specification of an endpoint, or a capability ({{capabilities}}),
can define further members of an Endpoint Object.  Because recipients
ignore members they do not understand, new parameters can be added to
an endpoint without changing the structure of the Capabilities
Document.

The Transfer Endpoint Object has no members other than "uri" in this
document.  Transfer requests are authenticated by signatures
({{signing}}), so no authentication parameters apply to it.

The Submission Endpoint Object contains the following members in
addition to "uri":

identities:
: OPTIONAL.  The URI of the identities endpoint ({{identities}}),
  subject to the same rules as "uri".  The identities endpoint accepts
  the same authentication as the submission endpoint.

authentication:
: REQUIRED.  A non-empty array of strings listing the authentication
  schemes accepted by the submission endpoint and the identities
  endpoint, as described in {{client-authentication}}.

oauthResourceMetadata:
: OPTIONAL.  The URI of the OAuth 2.0 Protected Resource Metadata
  {{RFC9728}} of the Submission Server, subject to the same rules as
  "uri".  If this member is absent, Clients locate the metadata as
  specified in {{RFC9728}}.  The resource identifier of a Submission
  Server is the origin of its submission endpoint.

### Retrieval and Caching

Servers SHOULD include caching information in the response, and
recipients MAY cache the document as specified in {{RFC9111}}.

The result of retrieving the Capabilities Document is interpreted as
follows:

- If the Capabilities Document cannot be retrieved because of a
  connection failure, a TLS failure (including a failure to validate
  the certificate of the host), or a 5xx status code, the result MUST
  be treated as a temporary failure.

- If the server responds with any other error status code, if the
  document is not valid, or if it lists no version that the server or
  Client supports, the server or Client MUST NOT use HMTP with this
  host and continues with the next ServiceMode record, if any.  If no
  usable host remains, the domain is treated as not supporting HMTP.

### Example {#capabilities-document-example}

The following example shows the DNS record and the Capabilities
Document of a domain whose mail is handled by a provider:

~~~ dns
_hmtp.example.com.  3600 IN SVCB 1 hmtp.provider.example. (
                               alpn=h2,h3 )
~~~

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=86400

{
  "versions": {
    "1": {
      "endpoints": {
        "transfer": {
          "uri": "/hmtp/v1/transfer"
        },
        "submission": {
          "uri": "/hmtp/v1/submission",
          "identities": "/hmtp/v1/identities",
          "authentication": ["basic", "bearer"],
          "oauthResourceMetadata":
            "/.well-known/oauth-protected-resource"
        }
      }
    }
  },
  "limits": {
    "maxMessageSize": 107374182400,
    "maxRequestSize": 52428800,
    "maxRecipients": 1000,
    "maxIdLength": 64
  },
  "policy": {
    "mode": "enforce",
    "maxAge": 604800
  },
  "capabilities": {}
}
~~~

# Data Model {#data-model}

All JSON documents defined by this document MUST conform to the
Internet JSON (I-JSON) profile {{RFC7493}}.  Member names are
case-sensitive.  Unless stated otherwise, recipients of a JSON
document MUST ignore members they do not understand, and a member
whose value is null is treated as if it were absent.  Timestamps are
strings in the "date-time" format of {{Section 5.6 of RFC3339}} and
MUST use the UTC offset "Z".  Sizes are integers counting octets.

The content of a submission or transfer request is a JSON object,
called the Request Object, with the following members:

capabilities:
: OPTIONAL.  An array of strings naming the capabilities that the
  request uses and that the recipient of the request is required to
  understand, as described in {{capabilities}}.  The array MUST NOT
  contain duplicates.  If this member is absent, it is equivalent to
  an empty array.

envelope:
: REQUIRED.  The Envelope ({{envelope}}).

message:
: REQUIRED.  A Content Object ({{content-object}}) whose content is
  the Message.

The response to a successful request is a Response Object, specified
in {{transfer-response}}.

## Envelope {#envelope}

The Envelope is a JSON object with the following members:

id:
: REQUIRED.  The Transfer Identifier of the request.  It is a string
  of at least 1 character consisting only of the characters of the
  base64url alphabet ({{Section 5 of RFC4648}}): "A" to "Z", "a" to
  "z", "0" to "9", "-", and "_".  Its length MUST NOT exceed the
  "maxIdLength" limit of the recipient of the request (see
  {{capabilities-document}}).  The sender SHOULD include at least 128
  bits of randomness, for example 16 random octets encoded in
  base64url without padding, which yields 22 characters.  A request
  whose Transfer Identifier does not meet these requirements is
  rejected with the "invalid-request" problem type.

  A Transfer Identifier is not globally unique and is never
  interpreted on its own.  The recipient of a request stores and
  compares it only together with its scope ({{idempotency}}):

  - for a transfer request, the Sending Domain, that is, the domain
    of the key that signed the request as established by the
    verification procedure ({{verification}}).  The host name and IP
    address of the Sending Server are not part of the scope, so that
    a retry is recognized even if it is sent from another host of the
    same Sending Domain;
  - for a submission request, the account of the authenticated
    Client.

  The sender MUST generate Transfer Identifiers that are unique
  within their scope.  Identical Transfer Identifiers in different
  scopes, for example from two different Sending Domains, identify
  unrelated requests.

from:
: REQUIRED.  The Envelope Sender.  The value is either a "Mailbox" as
  defined in {{Section 4.1.2 of RFC5321}}, without the enclosing
  angle brackets, or the empty string, which denotes the null
  reverse-path.  If "smtpUtf8" is true, the address MAY use the
  syntax extended by {{RFC6531}}.

to:
: REQUIRED.  A non-empty array of Recipient Objects, one for each
  Envelope Recipient.  The array MUST NOT contain two Recipient
  Objects with the same address.  Its length MUST NOT exceed the
  "maxRecipients" limit of the recipient of the request, if one is
  advertised.

dsn:
: OPTIONAL.  A JSON object that carries the per-message parameters of
  the Delivery Status Notification (DSN) extension {{RFC3461}}.  It
  has the following members, both OPTIONAL:

  - "ret": the string "full" or "hdrs", with the semantics of the
    RET parameter of {{RFC3461}};
  - "envid": a string with the semantics of the ENVID parameter of
    {{RFC3461}}.  The value is carried in decoded form, without the
    "xtext" encoding used in SMTP, and is subject to the same length
    and character restrictions as in {{RFC3461}}.

body:
: OPTIONAL.  The type of the body of the Message.  The value is one
  of "7bit", "8bitmime", and "binarymime", with the semantics of the
  BODY parameter values "7BIT", "8BITMIME" {{RFC6152}}, and
  "BINARYMIME" {{RFC3030}}, respectively.  The default is "7bit".
  The sender MUST set this member to a value that correctly describes
  the Message.

smtpUtf8:
: OPTIONAL.  A boolean that is true if any address in the Envelope or
  any header field of the Message contains non-ASCII characters, as
  permitted by {{RFC6531}} and {{RFC6532}}.  The default is false.
  The sender MUST set this member to true in that case.

requireTls:
: OPTIONAL.  A boolean that, when true, requests that the Message be
  transferred only over connections that are protected by TLS with
  certificate validation on every hop, with the semantics of the
  REQUIRETLS extension {{RFC8689}}.  The default is false.  HMTP
  hops always satisfy this requirement; SMTP hops are governed by
  {{RFC8689}}.

A Recipient Object is a JSON object with the following members:

address:
: REQUIRED.  The Envelope Recipient.  The value is a "Mailbox" as
  defined in {{Section 4.1.2 of RFC5321}}, without the enclosing
  angle brackets, or, if "smtpUtf8" is true, as extended by
  {{RFC6531}}.

dsn:
: OPTIONAL.  A JSON object that carries the per-recipient parameters
  of the DSN extension {{RFC3461}}.  It has the following members,
  both OPTIONAL:

  - "notify": an array of strings with the semantics of the NOTIFY
    parameter of {{RFC3461}}.  The array contains either the single
    value "never", or one or more of the values "success", "failure",
    and "delay";
  - "orcpt": a string with the semantics of the ORCPT parameter of
    {{RFC3461}}, consisting of an address type, a semicolon, and the
    original recipient address, for example
    "rfc822;bob@example.net".  The value is carried in decoded form,
    without the "xtext" encoding used in SMTP.

When the domain part of an address is compared, for example to
determine alignment ({{alignment}}) or to perform discovery, it is
first converted to A-labels and then compared without regard to case.

The following example shows an Envelope:

~~~ json
{
  "id": "3q2-7wEXAMPLEd9Kc1fQ0g",
  "from": "bounces+7f3a@example.com",
  "to": [
    {"address": "bob@example.net"},
    {
      "address": "carol@example.net",
      "dsn": {
        "notify": ["failure", "delay"],
        "orcpt": "rfc822;carol@example.net"
      }
    }
  ],
  "dsn": {"ret": "hdrs", "envid": "QQ314159"},
  "body": "8bitmime"
}
~~~

## Content Object {#content-object}

A Content Object carries a sequence of octets, called its content.
It takes one of three forms, distinguished by the members present.
A Content Object MUST contain exactly one of the members "data",
"uri", and "segments".

Inline:
: The content is carried in the request.  The Content Object has the
  following member:

  data:
  : REQUIRED.  The content, encoded using base64 as defined in
    {{Section 4 of RFC4648}}, with padding and without line breaks.

Reference:
: The content is retrieved from a URI.  A Content Object of this form
  is a Content Reference and has the following members:

  uri:
  : REQUIRED.  An absolute URI {{RFC3986}} with the "https" scheme
    from which the content is retrieved, as described in
    {{reference-retrieval}}.

  size:
  : REQUIRED.  The size of the retrieved representation data in
    octets, before any transformation specified by "encoding".

  digest:
  : REQUIRED.  A Digest Object for the retrieved representation data,
    before any transformation specified by "encoding".

  expires:
  : REQUIRED.  A timestamp until which the sender guarantees that the
    content can be retrieved from the URI.  The timestamp MUST be at
    least 24 hours after the time at which the request is sent and
    SHOULD be at least 7 days after it.

  encoding:
  : OPTIONAL.  The name of a transformation that is applied to the
    retrieved representation data to produce the content, as
    described in {{attachments}}.  If this member is absent, the
    content is the retrieved representation data itself.

Composite:
: The content is the concatenation of the content of a sequence of
  segments.  The Content Object has the following members:

  segments:
  : REQUIRED.  A non-empty array of Content Objects, each of which is
    either of the inline or of the reference form.  A segment MUST
    NOT itself be of the composite form.

  size:
  : REQUIRED.  The size of the content, that is, the sum of the sizes
    of the content of all segments.

The content of a composite Content Object is reconstructed by
concatenating the content of its segments in the order in which they
appear in the array.  No octets are added between segments.  The
integrity of the reconstructed content follows from the integrity of
each segment: inline segments are covered by the Content-Digest of
the request, and reference segments by their "digest" member.
Recipients MUST support composite Content Objects with at least 64
segments.

A Digest Object is a JSON object whose member names are hash
algorithm keys from the "Hash Algorithms for HTTP Digest Fields"
registry established by {{RFC9530}} and whose values are the hashes
computed with these algorithms, encoded using base64 as defined in
{{Section 4 of RFC4648}}.  A Digest Object MUST contain at least one
of the members "sha-256" and "sha-512", and MAY contain additional
members.  Recipients MUST support "sha-256" and "sha-512", MUST
ignore algorithms they do not support, and MUST verify the hash of
every supported algorithm that is present.  If a Digest Object
contains no supported algorithm, the request is rejected with the
"invalid-request" problem type.

The "maxMessageSize" limit ({{capabilities-document}}) applies to the
size of the reconstructed Message.  Because the size of every segment
is known before any content is retrieved, the recipient of a request
can determine whether the limit is exceeded without retrieving any
content.

The following example shows a Message carried inline and the same
Message carried as a single Content Reference:

~~~ json
{
  "data": "RnJvbTogQWxpY2UgPGFsaWNlQGV4YW1wbGUuY29tPg0K..."
}
~~~

~~~ json
{
  "uri": "https://files.example.com/m/7f3a91c2e4d8",
  "size": 52428800,
  "digest": {
    "sha-256": "rbJw2yTgX0Gj3nVQb1Zu4eKcN9mF8sPq7LdW6xHtA0E="
  },
  "expires": "2026-01-08T12:00:00Z"
}
~~~

### Retrieval of Referenced Content {#reference-retrieval}

The recipient of a request retrieves the content of a Content
Reference with an HTTP GET request to its URI.  The following rules
apply:

- The recipient MUST validate the TLS certificate of the host as
  described in {{RFC9525}}.

- The recipient MUST NOT send credentials, such as cookies or an
  Authorization header field, in the retrieval request.  It MAY sign
  the retrieval request as described in {{retrieval-signing}}.

- The recipient MAY follow redirections, but only to URIs with the
  "https" scheme, and SHOULD NOT follow more than 5 redirections.

- The recipient MAY use range requests to resume an interrupted
  retrieval.

- The representation data is the content of the response after any
  content coding has been removed.  If its size differs from "size",
  or any supported hash in "digest" does not match, the retrieval has
  failed permanently with the "content-mismatch" problem type.  The
  recipient SHOULD stop retrieving as soon as the received data
  exceeds "size".

- A connection failure, a TLS failure, or a response with status code
  408, 429, or 5xx is a temporary failure, and the recipient retries
  the retrieval.  A response with any other error status code is a
  permanent failure with the "content-unavailable" problem type.

The recipient of a request chooses, for each request, when it
retrieves referenced content:

Before responding:
: The recipient retrieves and verifies all Content References before
  it sends the response, which then has the status code 200 (OK).  If
  retrieval fails temporarily, it responds
  with the "content-unavailable" problem type and a temporary
  enhanced status code.  If retrieval fails permanently, it rejects
  the request with the corresponding problem type.

After responding:
: The recipient accepts the Message for some or all Envelope
  Recipients and responds with the status code 202 (Accepted), which
  indicates that retrieval is still pending (see
  {{transfer-response}}).  By doing so, it accepts
  responsibility for retrieving the content and MUST complete the
  retrieval before the earliest "expires" timestamp of the Content
  References.  If the retrieval fails permanently, or cannot be
  completed before that time, the recipient MUST treat the Message as
  undeliverable for all Envelope Recipients for which it was
  accepted and generate delivery status notifications as described
  in {{dsn}}.

This choice allows a recipient to retrieve small content
synchronously while accepting very large Messages without keeping the
request open for the duration of the retrieval.  In either case, a
server MUST NOT deliver a Message to a recipient mailbox, and MUST
NOT make any part of it available to the recipient, before the
Message has been completely reconstructed and verified.  A server
MUST NOT make the time of retrieval depend on actions of the
recipient of the Message, such as opening the Message (see
{{privacy}}).

The sender of a request MUST keep the content of every Content
Reference available at its URI until its "expires" timestamp, unless
every request containing the Content Reference has received a
response with the status code 200 (OK), which indicates that the
retrieval is complete or was not needed.  After a response with the
status code 202 (Accepted), the sender keeps the content available
until its "expires" timestamp.

A server that relays a Message MAY pass Content References on to the
next hop unchanged, without retrieving them, if at least 24 hours
remain before their "expires" timestamp.  Otherwise, it MUST retrieve
the content and either carry it inline or provide it through Content
References of its own.

A recipient MAY reject a request that contains a Content Reference
whose "expires" timestamp is less than 24 hours after the time of
receipt, with the "invalid-request" problem type.

## Attachments {#attachments}

A composite Content Object allows a sender to transfer parts of a
Message, such as the body of an attachment, by reference, while the
reconstructed Message remains identical, octet for octet, to the
Message as produced by its originator.  DKIM signatures and other
signatures computed over the Message therefore remain valid.

To transfer an attachment by reference, the sender splits the Message
into three segments: an inline segment containing everything up to
the first octet of the body of the MIME body part, a reference segment
containing the body of the body part, and an inline segment
containing the remainder of the Message.  Following {{RFC2046}}, the
CRLF that precedes a boundary delimiter belongs to the delimiter and
therefore to the following inline segment.  A Message may contain any
number of such references, subject to the limit on segments described
in {{content-object}}.

Attachments are usually encoded with the base64
Content-Transfer-Encoding {{RFC2045}}, while the content stored at a URI is usually the
original, unencoded file.  The "encoding" member allows a Content
Reference to refer to the unencoded file.  This document defines the
following encoding:

base64:
: The retrieved representation data is encoded using base64 as
  defined in {{Section 6.8 of RFC2045}}.  The encoded output is
  divided into lines of exactly 76 characters, except that the last
  line contains the remaining 1 to 76 characters.  Lines are
  separated by CRLF.  No CRLF follows the last line.  If the
  representation data is empty, the content is empty.

A sender MUST use the "base64" encoding only if the body of the MIME
body part in the Message is exactly the output of this
transformation.  Senders that wish to use this encoding therefore
produce base64-encoded body parts with lines of 76 characters, which
is the most common practice.  The size of the content produced by
the "base64" encoding from n octets of representation data is
4 * ceil(n / 3) characters plus 2 octets for each line separator.

Additional encodings can be registered as described in
{{iana-encodings}}.  A sender MUST NOT use an encoding other than
"base64" unless the recipient of the request advertises support for
it through a capability.

The following example shows a Message with a 700 MiB attachment.  The
first segment contains the header section and the MIME structure up
to the body of the attachment, which is followed by a CRLF and the
closing boundary delimiter "--b1--" in the last segment:

~~~ json
{
  "segments": [
    {
      "data": "RnJvbTogQWxpY2UgPGFsaWNlQGV4YW1wbGUuY29tPg0K..."
    },
    {
      "uri": "https://files.example.com/f/9b1c4e7a",
      "size": 734003200,
      "digest": {
        "sha-256": "Yk3qV0sJpRm9T2cFhWxN6dL1bA8eQu7ZgKi4oPyXwE0="
      },
      "expires": "2026-01-08T12:00:00Z",
      "encoding": "base64"
    },
    {
      "data": "DQotLWIxLS0NCg=="
    }
  ],
  "size": 1004426786
}
~~~

In this example, the first segment contains 1342 octets, the second
segment produces 1004425434 octets after the "base64" encoding, and
the last segment contains 10 octets.

The same URI can be used in requests to any number of Receiving
Servers, so that the attachment is uploaded by the originator only
once.  Receiving Servers that retrieve the same content for several
Messages can recognize it by its digest.  Messages that are protected
end to end, such as encrypted OpenPGP or S/MIME messages, can only be
transferred by reference as a whole.

## Capabilities {#capabilities}

Capabilities allow HMTP to be extended without changing this
document.  A capability can define new members of the Request Object,
the Envelope, the Recipient Object, Content Objects, the Response
Object, and the Capabilities Document; new endpoints; new encodings;
and new problem types.  Examples of functionality that could be
defined as capabilities include the transfer of several Messages in
one request, a status endpoint for submitted Messages, and the recall
of Messages that have not yet been delivered.

### Capability Names {#capability-names}

A capability is identified by its name.  A name takes one of two
forms:

Registered name:
: A string of 1 to 64 characters that consists of lowercase ASCII
  letters, digits, and hyphens, starts with a letter, and does not
  end with a hyphen, for example "future-release".  A registered name
  MUST be registered in the "HMTP Capabilities" registry
  ({{iana-capabilities}}) before it is used, which requires a
  publicly available specification (see {{iana}}).  Implementations
  MUST NOT use a name of this form that is not registered.
  Registered names are intended for capabilities that are meant to
  be implemented interoperably by independent parties.

URI:
: An absolute URI {{RFC3986}}, for example
  "https://vendor.example.com/hmtp/future-release".  Such a URI is only a
  unique name for the capability.  It is compared with other names
  character by character, with case sensitivity, and is never
  dereferenced: implementations MUST NOT retrieve it as part of
  protocol processing, and it does not need to resolve to anything.
  Uniqueness follows from the control of the party that defines the
  capability over the authority component of the URI, typically a
  domain name it owns.  URIs require no registration with IANA and
  are intended for private, vendor-specific, and experimental
  capabilities.  The URI MAY point to human-readable documentation,
  but this has no significance for the protocol.

A capability that starts as a URI and is later standardized receives
a registered name.  The two names identify distinct capabilities; a
server MAY advertise both during a transition period, and a request
lists the one whose definition it follows.

### Advertisement and Use {#capability-use}

A server advertises the capabilities it supports in the "capabilities"
member of its Capabilities Document.  The value of each member is a
JSON object containing the parameters of the capability, as defined by
its specification; a capability without parameters has an empty
object as its value.

The specification of a capability states whether the capability is
"must-understand".  A must-understand capability changes the meaning
of a request in a way that a recipient that does not support it
cannot safely ignore.  The following rules apply:

- A sender MUST NOT use a capability that is not advertised by the
  recipient of the request, except that members defined by a
  capability that is not must-understand MAY be included regardless,
  because recipients that do not support them ignore them.

- A sender MUST list every must-understand capability that a request
  uses in the "capabilities" member of the Request Object.

- A recipient MUST reject a request that lists a capability it does
  not support with the "unsupported-capability" problem type.

- A recipient MAY include members defined by a capability in its
  response if the capability is listed in the request or is not
  must-understand.  Recipients of responses MUST ignore members they
  do not understand.

To prevent collisions between members defined by independent parties,
the following rules apply to the members that a capability adds to
JSON objects defined by this document:

- A capability with a registered name defines member names directly.
  The designated experts ensure that these names do not collide with
  names defined by this document or by other registered capabilities.

- A capability identified by a URI adds exactly one member to each
  object it extends.  The name of this member is the URI of the
  capability, and its value is a JSON object that contains all
  members defined by the capability for that object.

- A capability with a registered name that defines a new endpoint
  registers the endpoint name ({{iana-endpoints}}) and is advertised
  with an Endpoint Object in the "endpoints" member of a Version
  Object.  A capability identified by a URI advertises the URIs of
  its endpoints in its own parameters instead.

### Specifying a Capability {#capability-spec}

The specification of a capability defines at least the following:

- its name and whether it is must-understand;
- its parameters in the Capabilities Document, including their types
  and default values;
- the members it adds to the Request Object, the Envelope, the
  Recipient Object, Content Objects, the Response Object, and
  Endpoint Objects, and their meaning;
- any new endpoints, encodings, and problem types;
- the behavior of senders and recipients, including the behavior of
  a server that relays a Message that uses the capability to a next
  hop that does not support it, either over HMTP or over SMTP; and
- its security and privacy considerations.

### Example {#capability-example}

This section shows a complete example of a hypothetical
must-understand capability that allows the sender to request that a
Message be held by the Receiving Server and released for delivery no
earlier than a given time, similar to the SMTP FUTURERELEASE extension
{{RFC4865}}.  The capability is defined by a vendor and is therefore
identified by a URI.  Its specification could read as follows:

Name:
: "https://vendor.example/hmtp/future-release".

Must-understand:
: Yes.  A recipient that ignored the capability would deliver the
  Message immediately.

Parameters:
: "maxInterval": REQUIRED.  The maximum number of seconds between the
  receipt of a request and the release time that the server accepts.

Envelope members:
: "releaseAt": REQUIRED.  A timestamp before which the recipient MUST
  NOT deliver the Message to a recipient mailbox.  If it is more than
  "maxInterval" seconds after the time of receipt, the recipient
  rejects the request with the "invalid-request" problem type.

Response members:
: None.

Relaying:
: A server that relays the Message before the release time MUST NOT
  pass it to a next hop that does not advertise the capability; it
  holds the Message itself until the release time instead.

A server that supports the capability advertises it in its
Capabilities Document:

~~~ json
{
  "versions": {
    "1": {
      "endpoints": {
        "transfer": {
          "uri": "/hmtp/v1/transfer"
        }
      }
    }
  },
  "capabilities": {
    "https://vendor.example/hmtp/future-release": {
      "maxInterval": 604800
    }
  }
}
~~~

A sender that uses the capability lists it in the request and adds
the member defined by the capability to the Envelope, under the name
of the capability:

~~~ json
{
  "capabilities": [
    "https://vendor.example/hmtp/future-release"
  ],
  "envelope": {
    "id": "Rt5yU8iO1pA4sD7fG0hJ2k",
    "from": "alice@example.com",
    "to": [{"address": "bob@example.net"}],
    "https://vendor.example/hmtp/future-release": {
      "releaseAt": "2026-01-02T09:00:00Z"
    }
  },
  "message": {
    "data": "RnJvbTogQWxpY2UgPGFsaWNlQGV4YW1wbGUuY29tPg0K..."
  }
}
~~~

The URI "https://vendor.example/hmtp/future-release" in this example
is never retrieved; it only names the capability.

A server that does not support the capability rejects the request:

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "urn:ietf:params:hmtp:error:unsupported-capability",
  "title": "Unsupported capability",
  "status": 400,
  "detail": "https://vendor.example/hmtp/future-release",
  "smtpStatus": "5.5.4",
  "smtpReply": 555
}
~~~

If the same capability were standardized and registered under the
name "future-release", the server would advertise
`"future-release": {"maxInterval": 604800}`, the request would list
"future-release" in its "capabilities" member, and the Envelope would
contain the member `"releaseAt": "2026-01-02T09:00:00Z"` directly.

# Message Transfer {#transfer}

A Sending Server transfers a Message to a Receiving Server by sending
a transfer request to the transfer endpoint of the Receiving Server.
All Envelope Recipients in one request MUST have domains for which
discovery yielded the same transfer endpoint.  A Sending Server sends
a separate request for each transfer endpoint.

## Request {#transfer-request}

A transfer request is an HTTP POST request {{RFC9110}} to the
transfer endpoint, with the following properties:

- The content of the request is a Request Object ({{data-model}}),
  and the Content-Type header field is "application/json".

- The request MUST contain a Content-Digest header field {{RFC9530}}
  computed over the content of the request, using the "sha-256" or
  "sha-512" algorithm.

- The request MUST be signed on behalf of the Sending Domain as
  described in {{signing}}, using the Signature-Input and Signature
  header fields {{RFC9421}}.

- If the Envelope Sender is not empty, its domain MUST be aligned
  with the Sending Domain as described in {{alignment}}.

- The request does not use HTTP authentication.  Receiving Servers
  ignore any Authorization header field in a transfer request.

- The content of the request MAY use a content coding only if the
  Receiving Server has indicated support for it in an
  Accept-Encoding header field, as described in
  {{Section 12.5.3 of RFC9110}}.

A Sending Server SHOULD reuse connections for several requests and
MAY send several requests concurrently on one connection, within the
limits that the Receiving Server sets, such as the
SETTINGS_MAX_CONCURRENT_STREAMS setting of HTTP/2 {{RFC9113}}.  This
document does not define the transfer of several Messages in one
request.

Sending Servers SHOULD wait at least 10 minutes for the response
after the request has been sent completely, consistent with the
timeouts recommended in {{Section 4.5.3.2 of RFC5321}}.

The following example shows a transfer request:

~~~ http-message
NOTE: '\' line wrapping per RFC 8792

POST /hmtp/v1/transfer HTTP/1.1
Host: mx.example.net
Content-Type: application/json
Content-Digest: sha-256=:Cw2CRUGnwBRSPpU7qzJnP6Fk2zBdJ8u8mXh7sQe1zYI=:
Signature-Input: hmtp=("@method" "@target-uri" "content-type" \
  "content-digest");created=1767268800;expires=1767269100;\
  keyid="hmtp2026._domainkey.example.com";alg="ed25519";tag="hmtp"
Signature: hmtp=:3iXk1Lw3P1pZq9xH0M2eR8vUe6bJ4yD7nT5aC0fK2gS9\
  hV1mQ4wL8oE6rB3tY7uI0pA5sD2fG9hJ1kL3zX8cVQ==:

{
  "envelope": {
    "id": "3q2-7wEXAMPLEd9Kc1fQ0g",
    "from": "bounces+7f3a@example.com",
    "to": [
      {"address": "bob@example.net"},
      {"address": "carol@example.net"}
    ]
  },
  "message": {
    "data": "RnJvbTogQWxpY2UgPGFsaWNlQGV4YW1wbGUuY29tPg0K..."
  }
}
~~~

## Processing by the Receiving Server {#transfer-processing}

A Receiving Server processes a transfer request as follows:

1. It verifies that the request uses the POST method and that the
   media type and size of the content are acceptable.  It rejects a
   request whose content exceeds its "maxRequestSize" limit with the
   "request-too-large" problem type.

2. It verifies the signature and the Content-Digest of the request
   as described in {{verification}}, including the alignment of the
   Envelope Sender ({{alignment}}).

3. It parses the Request Object and validates it against
   {{data-model}}.  It rejects an invalid Request Object with the
   "invalid-request" problem type.

4. It checks the Transfer Identifier as described in
   {{idempotency}}.  If the request is a retry of a request that has
   already been processed, it returns the stored response and does
   not process the request again.

5. It checks that it supports every capability listed in the
   "capabilities" member ({{capabilities}}).

6. It verifies that the size of the Message does not exceed its
   "maxMessageSize" limit and that the number of Envelope Recipients
   does not exceed its "maxRecipients" limit.

7. It determines, for each Envelope Recipient, whether it accepts the
   Message for that recipient.  For an Envelope Recipient whose
   domain it does not serve, and for which it has no agreement to
   relay, it rejects the recipient with the "domain-not-served"
   problem type.  Other decisions are a matter of local policy.

8. It retrieves referenced content as described in
   {{reference-retrieval}}, either before responding or after
   responding.

9. It prepends trace header fields as described in {{trace}}, stores
   the Message for each accepted Envelope Recipient, and sends the
   response.

A Receiving Server MAY perform these steps in a different order, as
long as the result is the same.  In particular, it MAY start
verifying the signature before the content of the request has been
received completely.

## Response {#transfer-response}

If the Receiving Server has processed the request, it responds with a
Response Object and one of the following status codes, even if it has
rejected the Message for some or all Envelope Recipients:

200 (OK):
: The request has been processed completely.  Either the request
  contains no Content References, or all referenced content has been
  retrieved and verified, or the Message has not been accepted for any
  Envelope Recipient.

202 (Accepted):
: The Message has been accepted for at least one Envelope Recipient,
  but the retrieval of referenced content is still pending.  The
  Receiving Server has accepted responsibility for completing the
  retrieval later, as described in {{reference-retrieval}}.  A
  Receiving Server MUST NOT use this status code for a request that
  contains no Content References.

In both cases, the results for the individual Envelope Recipients are
final: a recipient with the result "accepted" remains accepted, and
any later failure, including a failure of the pending retrieval, is
reported through a delivery status notification ({{dsn}}).  The status
code only tells the sender whether it still has to keep the
referenced content available.

If the request as a whole fails, the Receiving Server responds with a
4xx or 5xx status code and a problem details object as described in
{{problem-details}}; in that case, the Message has not been accepted
for any Envelope Recipient.

The Response Object is a JSON object with the media type
"application/json" and the following members:

queueId:
: OPTIONAL.  A string assigned by the Receiving Server that
  identifies the Message in its systems, such as a queue identifier.
  It is intended for logging and diagnostics.

recipients:
: REQUIRED.  An array of Recipient Result Objects, exactly one for
  each Envelope Recipient, in the same order as in the "to" member of
  the Envelope.

A Recipient Result Object is a JSON object with the following members:

address:
: REQUIRED.  The address of the Envelope Recipient, as given in the
  request.

result:
: REQUIRED.  One of the following strings:

  - "accepted": the Receiving Server has accepted responsibility for
    delivering or relaying the Message to this recipient, with the
    semantics of {{Section 6.1 of RFC5321}}.  It MUST NOT lose the
    Message, and if it later fails to deliver it, it generates a
    delivery status notification as described in {{dsn}}.
  - "deferred": the Message was not accepted for this recipient
    because of a temporary condition.  The sender MAY retry
    delivery to this recipient later.
  - "rejected": the Message was not accepted for this recipient
    because of a permanent condition.  The sender MUST NOT retry
    delivery to this recipient.

problem:
: REQUIRED if "result" is "deferred" or "rejected", and absent
  otherwise.  A problem details object, as described in
  {{problem-details}}, that describes the reason.  Its "smtpStatus"
  member MUST have the class 4 if "result" is "deferred" and the
  class 5 if "result" is "rejected".

The following example shows a response in which the Message was
accepted for one recipient and rejected for another:

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json

{
  "queueId": "4Y7mKq2ZtR",
  "recipients": [
    {
      "address": "bob@example.net",
      "result": "accepted"
    },
    {
      "address": "carol@example.net",
      "result": "rejected",
      "problem": {
        "type": "urn:ietf:params:hmtp:error:unknown-recipient",
        "title": "Unknown recipient",
        "detail": "The mailbox carol@example.net does not exist.",
        "smtpStatus": "5.1.1",
        "smtpReply": 550
      }
    }
  ]
}
~~~

## Retries and Idempotency {#idempotency}

The Transfer Identifier allows a Receiving Server to recognize a
request that is retried because the sender did not receive the
response, for example because the connection was lost after the
Receiving Server had accepted the Message.

The scope of a Transfer Identifier is the Sending Domain for transfer
requests, and the authenticated account of the Client for submission
requests (see {{envelope}}).  The key under which a Receiving Server
stores and looks up a Transfer Identifier is therefore the pair
(Sending Domain, Transfer Identifier) for transfer requests and the
pair (account, Transfer Identifier) for submission requests.  The
Sending Domain is the one established by verifying the signature of
the request ({{verification}}), never a value taken from the content
of the request.  A sender MUST NOT use the same Transfer Identifier
within one scope for requests with different content.

When a Receiving Server sends a 200 (OK) or 202 (Accepted) response,
it MUST store the key, a hash of the content of the request, and the
response, and MUST retain them for at least 24 hours.  When it
receives a request whose key matches a stored entry:

- If the hash of the content of the request matches, the Receiving
  Server MUST NOT process the request again and MUST return the stored
  response.  If the stored response has the status code 202
  (Accepted) and the retrieval of referenced content has been
  completed in the meantime, it MAY return the status code 200 (OK)
  with the same Response Object instead.  Senders MUST NOT retry
  requests solely to learn whether a retrieval has been completed.

- Otherwise, it MUST reject the request with the "id-conflict"
  problem type.

If a request with the same scope and Transfer Identifier is still
being processed, the Receiving Server MUST NOT process the second
request concurrently; it MAY respond with the "server-unavailable"
problem type and a Retry-After header field.

A request that fails as a whole does not need to be stored, because
the Message has not been accepted for any recipient.

A sender retries a request as follows:

- If the sender has not received a response, or has received a
  response indicating a temporary failure of the whole request, it
  MAY send the identical request again, with the same Transfer
  Identifier and the same content.  It signs each attempt anew.  It
  SHOULD send the retry within 24 hours of the first attempt, so
  that the Receiving Server can recognize it.

- For Envelope Recipients with the result "deferred", the sender
  sends a new request containing only those recipients, with a new
  Transfer Identifier.

- The sender MUST NOT retry a request, or deliver to an Envelope
  Recipient, after a permanent failure.

A failure of the whole request is temporary if the "smtpStatus" member
of the problem details object has the class 4.  If the response does
not contain a problem details object with an "smtpStatus" member, the
status codes 408, 429, and 5xx, as well as connection failures, are
temporary, and all other status codes are permanent.

The sender MUST honor a Retry-After header field {{RFC9110}} in the
response.  Otherwise, the intervals between retries and the time
after which the sender gives up SHOULD follow the guidance of
{{Section 4.5.4.1 of RFC5321}}.  When the sender gives up, it
generates a delivery status notification as described in {{dsn}}.
Before each retry, the sender repeats discovery if its cached results
have expired.

## Trace Information {#trace}

A server that accepts responsibility for a Message, whether through
transfer or submission, MUST prepend a Received header field to the
Message.  The field follows the syntax of {{Section 4.4 of RFC5321}}
and has the following content:

- The "from" clause contains, for a transfer request, the Sending
  Domain, followed by the IP address of the sender of the request in
  the "TCP-info" part, for example
  "from example.com (hmtp-out.example.com [192.0.2.25])".  For a
  submission request, it contains the IP address of the Client as an
  address literal, followed by the same address literal in the
  "TCP-info" part, for example "from [198.51.100.7] ([198.51.100.7])".
  For privacy reasons, a Submission Server MAY use the address literal
  "[127.0.0.1]" in both places instead of the IP address of the
  Client.

- The "by" clause contains the host name of the server.

- The "with" clause contains "HMTP" for a transfer request and
  "HMTPA" for a submission request (see {{iana-transmission-types}}).

- The "id" clause, if present, contains the "queueId" of the
  response.

- The "for" clause SHOULD be included only if the request has a
  single Envelope Recipient (see {{privacy}}).

A server MAY also prepend other trace header fields, such as an
Authentication-Results header field {{RFC8601}} that records the
result of the verification described in {{verification}}.

Prepending header fields is the only modification that servers make
to a Message during transfer.  When the Message is represented as a
Content Object, a server prepends header fields by adding an inline
segment that contains them before the existing content; existing
Content References can thus be passed on unchanged.

A Receiving Server SHOULD reject a Message that contains more
Received header fields than a locally configured limit with the
"loop-detected" problem type, as described in
{{Section 6.3 of RFC5321}}.

The following example shows a Received header field added by a
Receiving Server:

~~~
Received: from example.com (hmtp-out.example.com [192.0.2.25])
        by mx.example.net with HMTP id 4Y7mKq2ZtR;
        Thu, 01 Jan 2026 12:00:01 +0000
~~~

# Message Submission {#submission}

A Client submits a Message by sending a submission request to the
submission endpoint of its Submission Server.  A submission request
is an HTTP POST request with the same content as a transfer request
(see {{transfer-request}}), with the following differences:

- The Client authenticates using HTTP authentication as described in
  {{client-authentication}}.  The request does not need to be signed,
  and the Content-Digest header field is OPTIONAL.

- The Envelope Recipients can have any domain.

The Submission Server processes the request as described in
{{transfer-processing}}, except that it authenticates the Client
instead of verifying a signature, accepts Envelope Recipients in any
domain, verifies the sender addresses as described in
{{sender-authorization}}, and validates the Message as described in
{{message-validation}}.  It responds with a Response
Object as described in {{transfer-response}}.  The result "accepted"
indicates that the Submission Server has accepted responsibility for
delivering the Message to the recipient; the outcome of the final
delivery is reported through delivery status notifications
({{dsn}}).

The Transfer Identifier of a submission request is scoped to the
authenticated account, as described in {{idempotency}}, so that a
Client can safely retry a submission after a lost response.

## Client Authentication {#client-authentication}

The "authentication" member of the Submission Endpoint Object
({{endpoint-objects}}) lists the HTTP authentication schemes
{{RFC9110}} accepted by the Submission Server for the submission and
identities endpoints, using the scheme names from the "HTTP Authentication Scheme
Registry".  Scheme names are compared without regard to case.  This
document uses the following schemes:

basic:
: The "Basic" scheme {{RFC7617}}.  The user-id is the name of the
  account at the Submission Server.  Servers that support this scheme
  SHOULD include the "charset" parameter with the value "UTF-8" in
  their challenges.  This scheme allows existing SMTP AUTH
  credentials, including application-specific passwords, to be used
  with HMTP.

bearer:
: The "Bearer" scheme {{RFC6750}}.  Access tokens are obtained using
  OAuth 2.0 {{RFC6749}}.  The Client discovers the authorization
  servers and the supported scopes from the OAuth 2.0 Protected
  Resource Metadata {{RFC9728}} of the Submission Server.

A Submission Server MUST support at least one of these schemes and
SHOULD support "bearer".  Clients SHOULD support both.

If a submission request or a request to the identities endpoint does
not contain valid credentials, the server responds with the status
code 401 (Unauthorized), a WWW-Authenticate header field containing a
challenge for each accepted scheme, and a problem details object with
the "authentication-required" problem type if no credentials were
provided, or the "authentication-failed" problem type if the provided
credentials are not valid.  A challenge for the "Bearer" scheme
SHOULD include the "resource_metadata" parameter defined in
{{RFC9728}}.  If a bearer token is valid but lacks a required scope,
the server responds as specified in {{RFC6750}}.

Servers SHOULD limit the rate of failed authentication attempts.
Credentials for submission MUST NOT be sent to a transfer endpoint.

The following example shows a rejected submission request:

~~~ http-message
NOTE: '\' line wrapping per RFC 8792

HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="hmtp", charset="UTF-8"
WWW-Authenticate: Bearer realm="hmtp", resource_metadata=\
  "https://hmtp.provider.example/.well-known/oauth-protected-resource"
Content-Type: application/problem+json

{
  "type": "urn:ietf:params:hmtp:error:authentication-required",
  "title": "Authentication required",
  "status": 401,
  "smtpStatus": "5.7.0",
  "smtpReply": 530
}
~~~

## Authorization of Sender Addresses {#sender-authorization}

A Submission Server determines the set of identities that the
authenticated Client is authorized to use, as returned by the
identities endpoint ({{identities}}).  It MUST verify that each of
the following addresses is covered by one of these identities:

- the Envelope Sender, unless it is empty;
- each address in the From header field of the Message;
- the address in the Sender header field of the Message, if present.

An empty Envelope Sender is permitted for Messages that require it,
such as message disposition notifications {{RFC8098}}.

If any of these addresses is not authorized, the Submission Server
rejects the request with the "sender-not-authorized" problem type.

When the Submission Server acts as a Sending Server for the Message,
it signs the transfer requests with a key of a Sending Domain that is
aligned with the Envelope Sender ({{alignment}}).  A Submission Server
therefore only includes in the identities of a Client addresses for
which it can produce such signatures.

## Message Validation and Modification {#message-validation}

A Submission Server MUST verify that the Message conforms to
{{RFC5322}}, and in particular that it contains exactly one Date
header field and exactly one From header field.  It MUST reject a
Message that does not conform with the "invalid-request" problem
type, except as follows.

A Submission Server MAY add a Date header field or a Message-ID header
field if either is missing, as permitted by {{Section 8 of RFC6409}}.
Besides adding these fields, prepending trace header fields
({{trace}}), and prepending DKIM-Signature header fields {{RFC6376}},
a Submission Server MUST NOT modify the Message.  The Client is
responsible for any other content of the Message, including the
removal of Bcc header fields ({{Section 3.6.3 of RFC5322}}).

## Identities {#identities}

The identities endpoint returns the identities that the authenticated
Client is authorized to use as originator addresses.  Its URI is given
by the "identities" member of the Submission Endpoint Object
({{endpoint-objects}}).  A Client sends an HTTP GET request to the
endpoint, authenticated as described in
{{client-authentication}}.  The server responds with the status code
200 (OK) and a JSON object with the media type "application/json" and
the following member:

identities:
: REQUIRED.  An array of Identity Objects.

An Identity Object is a JSON object with the following members:

email:
: REQUIRED.  An address that the Client may use.  If the local part of
  the address is the single character "*", as in "*@example.org",
  the Client may use any address in that domain, following the
  convention of {{Section 6 of RFC8621}}.

name:
: OPTIONAL.  A display name associated with the address, which the
  Client can use in the From header field.

The server SHOULD include an ETag header field in the response and
support conditional requests {{RFC9110}}.  The response is specific to
the authenticated account and MUST be marked accordingly for caches,
for example with "Cache-Control: private".

~~~ http-message
GET /hmtp/v1/identities HTTP/1.1
Host: hmtp.provider.example
Authorization: Bearer mF_9.B5f-4.1JqM

HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: private
ETag: "a7c3"

{
  "identities": [
    {"email": "alice@example.com", "name": "Alice Example"},
    {"email": "*@sales.example.com"}
  ]
}
~~~

# Authentication and Signing {#signing}

Every transfer request is signed by the Sending Server using HTTP
Message Signatures {{RFC9421}}.  The public keys are published in DNS
using the key record format and location defined by DKIM {{RFC6376}},
so that domains can use their existing DNS infrastructure and
delegation practices.  Successful verification establishes the
Sending Domain, which the Receiving Server can use for reputation and
policy decisions.

## Signing Keys {#signing-keys}

A signing key is identified by a selector and a domain, as in DKIM
({{Section 3.1 of RFC6376}}).  The public key is published as a DKIM
key record ({{Section 3.6.1 of RFC6376}}) in a TXT record at the
following name:

~~~
<selector>._domainkey.<domain>
~~~

The tags of the key record are interpreted as follows:

- "k" (key type): the value "ed25519" {{RFC8463}} corresponds to the
  signature algorithm "ed25519", and the value "rsa" corresponds to
  the signature algorithm "rsa-v1_5-sha256", both as defined in
  {{Section 3.3 of RFC9421}}.  Records with other key types MUST NOT
  be used for HMTP.

- "p" (public key data): an empty value means that the key has been
  revoked.  RSA keys MUST have a modulus of at least 2048 bits.

- "s" (service type): the record can be used for HMTP only if this
  tag is absent, or if its value includes "*" or "hmtp".  The service
  type "hmtp" is registered in {{iana-dkim}}.

- "h" (acceptable hash algorithms): if present for a record of key
  type "rsa", the value MUST include "sha256".

- "t" (flags): if the flag "y" is present, the domain is testing the
  key.  In accordance with the meaning of this flag in {{RFC6376}},
  a verifier MUST NOT treat a request signed with such a key
  differently from an unsigned request, and therefore rejects it.

A domain SHOULD use a dedicated selector for HMTP with the service
type "hmtp" only.  Such a key record is ignored by DKIM verifiers,
which keeps keys for HMTP request signing separate from keys for DKIM
message signing (see {{security-keys}}).  A domain MAY instead use an
existing DKIM key whose record permits all service types; this
simplifies deployment but forgoes this separation.

Signers and verifiers MUST implement both "ed25519" and
"rsa-v1_5-sha256".  Signers SHOULD use "ed25519".

To replace a key, a domain publishes a new key record under a new
selector, starts signing with the new key, and keeps the old key
record published for at least the maximum lifetime of a signature
plus the TTL of the old record before revoking or removing it.

## Signature Construction {#signature-construction}

The Sending Server creates a signature as specified in
{{Section 3.1 of RFC9421}}, with the following requirements:

- The covered components MUST include "@method", "@target-uri",
  "content-type", and "content-digest".  The Content-Digest header
  field MUST be present in the request, as required in
  {{transfer-request}}.

- The signature parameters ({{Section 2.3 of RFC9421}}) MUST include
  "created", "expires", "keyid", "alg", and "tag":

  - "created" is the time at which the signature was created.
  - "expires" MUST NOT be more than 300 seconds after "created".
  - "keyid" is the DNS name of the key record, in the form
    "<selector>._domainkey.<domain>", where the domain is in A-label
    form, without a trailing dot.  The domain in this name is the
    Sending Domain.
  - "alg" is the signature algorithm that corresponds to the key type
    of the key record.
  - "tag" is the string "hmtp".

- The signature label is not significant; "hmtp" is RECOMMENDED.

A request MAY carry more than one signature with the tag "hmtp", for
example during a key rollover or a change of algorithm.  Other
signatures that the request carries are ignored for the purposes of
HMTP.

The following example shows the signature-related header fields of a
transfer request:

~~~ http-message
NOTE: '\' line wrapping per RFC 8792

Content-Digest: sha-256=:Cw2CRUGnwBRSPpU7qzJnP6Fk2zBdJ8u8mXh7sQe1zYI=:
Signature-Input: hmtp=("@method" "@target-uri" "content-type" \
  "content-digest");created=1767268800;expires=1767269100;\
  keyid="hmtp2026._domainkey.example.com";alg="ed25519";tag="hmtp"
Signature: hmtp=:3iXk1Lw3P1pZq9xH0M2eR8vUe6bJ4yD7nT5aC0fK2gS9\
  hV1mQ4wL8oE6rB3tY7uI0pA5sD2fG9hJ1kL3zX8cVQ==:
~~~

The corresponding key record is:

~~~ dns
hmtp2026._domainkey.example.com. 3600 IN TXT (
    "v=DKIM1; k=ed25519; s=hmtp; "
    "p=11qYAYKxCrfVS/7TyWQHOg7hcvPapiMlrwIaaPcHURo=" )
~~~

### Signing of Retrieval Requests {#retrieval-signing}

A Receiving Server MAY sign the requests with which it retrieves
referenced content ({{reference-retrieval}}), using a key of its own
domain published as described in {{signing-keys}}.  Such a signature
covers at least "@method" and "@target-uri", contains the signature
parameters required in {{signature-construction}}, and uses the tag
"hmtp-retrieval".  A server that hosts referenced content MAY use such
signatures to restrict access to the content, for example to the
domains of the Envelope Recipients.  A Receiving Server that does not
sign its retrieval requests can only retrieve content that does not
require such signatures.

## Verification Procedure {#verification}

The Receiving Server verifies a transfer request as follows.  Unless
stated otherwise, a failure of any step causes the request to be
rejected with the "signature-invalid" problem type.

1. It selects the signatures in the request whose "tag" parameter is
   "hmtp".  If there is none, verification fails.  If there are
   several, it processes each of them until one is verified
   successfully.

2. It verifies that the covered components and signature parameters
   meet the requirements of {{signature-construction}} and that it
   supports the algorithm given in "alg".

3. It verifies that "created" is not later than the current time and
   that "expires" is not earlier than the current time, both
   evaluated at the time the header section of the request was
   received, allowing for a clock skew of no more than 60 seconds,
   and that "expires" is not more than 300 seconds after "created".

4. It verifies that "keyid" has the form
   "<selector>._domainkey.<domain>", where the selector has the syntax
   defined in {{RFC6376}}.  Because a selector cannot contain the label
   "_domainkey", the Sending Domain is the part of the name that
   follows the first "_domainkey" label.

5. It queries DNS for TXT records at the name given in "keyid".  If
   the query fails temporarily, it rejects the request with the
   "key-unavailable" problem type, which is a temporary failure.  If
   the name does not exist, or if the response does not contain
   exactly one valid key record, verification fails.  DNS responses
   SHOULD be validated using DNSSEC {{RFC4033}} when available.

6. It verifies that the key record meets the requirements of
   {{signing-keys}} and that its key type corresponds to "alg".

7. It verifies the signature as specified in
   {{Section 3.2 of RFC9421}}.

8. It verifies that the Content-Digest header field matches the
   content of the request as specified in {{RFC9530}}.  If it does
   not, it rejects the request with the "digest-mismatch" problem
   type.

9. It verifies the alignment of the Envelope Sender with the Sending
   Domain as described in {{alignment}}.

If verification fails, the response MAY include an Accept-Signature
header field as described in {{RFC9421}}.

A Receiving Server that terminates TLS in a reverse proxy MUST ensure
that the covered components reach the verifier unchanged, or perform
verification in the proxy.

## Sender Alignment {#alignment}

The domain of a non-empty Envelope Sender is aligned with the Sending
Domain if it is equal to the Sending Domain or is a subdomain of it.
For example, the Envelope Sender "bounces@mail.example.com" is
aligned with the Sending Domain "example.com", but not with the
Sending Domain "example.org".

A Receiving Server MUST reject a transfer request in which the
Envelope Sender is not empty and is not aligned with the Sending
Domain, with the "sender-not-aligned" problem type, unless the
Receiving Server has a local agreement that authorizes the Sending
Domain to relay Messages for other domains.  Such agreements are used,
for example, with inbound filtering services and backup servers that
relay Messages to the Receiving Server on behalf of the recipient
domain (see {{gateways}}).

If the Envelope Sender is empty, any Sending Domain is acceptable,
and the Receiving Server applies its local policy based on the
Sending Domain.

Alignment authenticates the Envelope Sender's domain, not the author
of the Message.  The domains in the From header field continue to be
authenticated by DKIM and DMARC at the message level.

A server that forwards Messages to another domain, such as a mailing
list or a forwarding service, uses an Envelope Sender in a domain for
which it can produce aligned signatures.  As in SMTP, such a server
typically rewrites the Envelope Sender so that delivery status
notifications are returned to it.

## Third-Party Senders {#third-party}

A domain can authorize a third party, such as an email service
provider, to send Messages with Envelope Senders in its domain.
Because HMTP keys are published in the same way as DKIM keys, the
same delegation methods apply:

Delegation of a selector:
: The domain publishes a CNAME record at a selector name that points
  to a key record maintained by the provider, for example:

  ~~~ dns
  esp1._domainkey.example.com. 3600 IN CNAME (
      example-com.keys.esp.example. )
  ~~~

  The provider signs with "keyid" set to
  "esp1._domainkey.example.com"; the Sending Domain is "example.com".
  The domain can revoke the authorization at any time by removing the
  CNAME record.

Publication of a provider key:
: The domain publishes, under a selector of its own, a public key
  whose private key is held by the provider.

Provider domain:
: The provider uses Envelope Senders in its own domain and signs as
  that domain.  The reputation of the Sending Domain then accrues to
  the provider rather than to its customer.

Because signatures are not bound to IP addresses, a Sending Server
can send requests through any network path, including forward
proxies and HTTP intermediaries, provided that the covered components
and the content of the request are not modified on the way.

# Delivery Status and Errors

HMTP reports failures in two ways: synchronously, through problem
details objects in responses, and asynchronously, through delivery
status notifications.

## Problem Details {#problem-details}

When a request fails as a whole, the server responds with a 4xx or
5xx status code and a problem details object {{RFC9457}} with the
media type "application/problem+json".  When the Message is not
accepted for an individual Envelope Recipient, the reason is given as
a problem details object in the "problem" member of the Recipient
Result Object ({{transfer-response}}); in that case, the "status"
member MAY be omitted and is ignored if present.

Problem types defined for HMTP have URIs of the form
"urn:ietf:params:hmtp:error:<name>" and are registered as described
in {{iana-problem-types}}.  Servers MAY use other problem type URIs
as permitted by {{RFC9457}}.

This document defines the following extension members of problem
details objects:

smtpStatus:
: REQUIRED in problem details objects sent by HMTP servers.  An
  enhanced mail system status code {{RFC3463}} in the form
  "class.subject.detail".  The class MUST be 4 (persistent transient
  failure) or 5 (permanent failure).  The class determines whether the
  failure is temporary or permanent, also for problem types that are
  unknown to the recipient of the problem details object.

smtpReply:
: OPTIONAL.  An SMTP reply code {{RFC5321}} that corresponds to the
  failure.  It is used when the failure is reported over SMTP, for
  example by a gateway ({{gateways}}) or in a delivery status
  notification ({{dsn}}).  If it is absent, the reply code from
  {{tab-mapping}} is used for registered problem types, and 451 or 550
  otherwise, depending on the class of "smtpStatus".

The "detail" member SHOULD be suitable for inclusion in a delivery
status notification sent to the Envelope Sender.  The language of
human-readable members can be indicated with the Content-Language
header field.

Responses with the status codes 429 (Too Many Requests) {{RFC6585}}
and 503 (Service Unavailable) SHOULD include a Retry-After header
field.

The following example shows a request that was rejected as a whole:

~~~ http-message
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json
Content-Language: en

{
  "type": "urn:ietf:params:hmtp:error:sender-not-aligned",
  "title": "Envelope Sender not aligned with Sending Domain",
  "status": 403,
  "detail": "example.org is not aligned with example.com.",
  "smtpStatus": "5.7.1",
  "smtpReply": 550
}
~~~

## Mapping to SMTP Status Codes {#smtp-mapping}

{{tab-mapping}} lists the problem types defined by this document,
whether they apply to a whole request ("Request"), to an individual
Envelope Recipient ("Recipient"), or to both, and the HTTP status
code, enhanced status code, and SMTP reply code to be used with them.
Where two codes are given, the first is used for a temporary and the
second for a permanent failure.

| Name | Scope | HTTP | Enhanced Status | Reply |
|---|---|---|---|---|
| invalid-request | Request | 400 | 5.5.2 | 501 |
| unsupported-capability | Request | 400 | 5.5.4 | 555 |
| id-conflict | Request | 409 | 5.5.4 | 501 |
| signature-invalid | Request | 403 | 5.7.0 | 550 |
| key-unavailable | Request | 503 | 4.4.3 | 451 |
| digest-mismatch | Request | 400 | 5.7.7 | 554 |
| sender-not-aligned | Request | 403 | 5.7.1 | 550 |
| authentication-required | Request | 401 | 5.7.0 | 530 |
| authentication-failed | Request | 401 | 5.7.8 | 535 |
| sender-not-authorized | Request | 403 | 5.7.1 | 550 |
| request-too-large | Request | 413 | 5.3.4 | 552 |
| message-too-large | Both | 413 | 5.3.4 | 552 |
| too-many-recipients | Request | 422 | 4.5.3 | 452 |
| rate-limited | Request | 429 | 4.7.0 | 451 |
| server-unavailable | Request | 503 | 4.3.2 | 421 |
| content-unavailable | Request | 502 / 422 | 4.4.1 / 5.6.0 | 451 / 554 |
| content-mismatch | Request | 422 | 5.7.7 | 554 |
| loop-detected | Request | 422 | 5.4.6 | 554 |
| policy-rejected | Both | 403 | 4.7.1 / 5.7.1 | 451 / 550 |
| unknown-recipient | Recipient | - | 5.1.1 | 550 |
| invalid-address | Recipient | - | 5.1.3 | 553 |
| domain-not-served | Recipient | - | 5.1.2 | 550 |
| mailbox-disabled | Recipient | - | 5.2.1 | 550 |
| mailbox-full | Recipient | - | 4.2.2 / 5.2.2 | 452 / 552 |
| routing-failed | Recipient | - | 4.4.4 / 5.4.4 | 451 / 550 |
| conversion-required | Recipient | - | 5.6.3 | 554 |
| tls-required | Recipient | - | 5.7.30 | 550 |
{: #tab-mapping title="HMTP Problem Types"}

The meaning of each problem type is as follows:

invalid-request:
: The request content is not a valid Request Object, the Message does
  not conform to {{RFC5322}}, or another requirement of this document
  is not met.

unsupported-capability:
: The request lists a capability that the server does not support.

id-conflict:
: The Transfer Identifier has already been used for a request with
  different content ({{idempotency}}).

signature-invalid:
: The request has no valid HMTP signature ({{verification}}).

key-unavailable:
: The key record could not be retrieved because of a temporary DNS
  failure.

digest-mismatch:
: The Content-Digest header field does not match the content of the
  request.

sender-not-aligned:
: The Envelope Sender is not aligned with the Sending Domain
  ({{alignment}}).

authentication-required:
: A submission request or identities request carries no credentials.

authentication-failed:
: The credentials of a submission request or identities request are
  not valid.  This problem type corresponds to the SMTP enhanced
  status code 5.7.8 defined in {{RFC4954}}.

sender-not-authorized:
: The Client is not authorized to use an originator address of the
  Message ({{sender-authorization}}).

request-too-large:
: The content of the request exceeds the "maxRequestSize" limit.  The
  sender can transfer part of the Message by reference instead.

message-too-large:
: The size of the Message exceeds a limit of the server or of a
  recipient mailbox.

too-many-recipients:
: The number of Envelope Recipients exceeds the "maxRecipients"
  limit.  The sender MUST NOT repeat the request unchanged; it splits
  the Envelope Recipients into several requests with new Transfer
  Identifiers.

rate-limited:
: The sender has exceeded a rate limit of the server.

server-unavailable:
: The server is temporarily unable to process requests.

content-unavailable:
: Referenced content could not be retrieved
  ({{reference-retrieval}}).

content-mismatch:
: The size or digest of referenced content does not match its Content
  Reference.  This problem type corresponds to the enhanced status
  code for message integrity failures defined in {{RFC3463}}.

loop-detected:
: The Message has passed through too many servers ({{trace}}).

policy-rejected:
: The Message was rejected by the local policy of the server, for
  example on the basis of the reputation of the Sending Domain or the
  content of the Message.

unknown-recipient:
: The recipient mailbox does not exist.

invalid-address:
: The address of the Envelope Recipient is syntactically invalid.

domain-not-served:
: The server does not accept Messages for the domain of the Envelope
  Recipient and has no agreement to relay them.

mailbox-disabled:
: The recipient mailbox exists but does not accept Messages.

mailbox-full:
: The recipient mailbox has exceeded its storage limit.

routing-failed:
: The server was unable to route the Message toward the recipient,
  for example because the recipient domain supports neither HMTP nor,
  for a server that supports fallback, SMTP.

conversion-required:
: Delivery requires a conversion of the Message that the server does
  not perform, for example because an SMTP server used for fallback
  does not support an extension that the Message requires
  ({{when-to-fall-back}}).

tls-required:
: The Envelope requests REQUIRETLS, and the requirement cannot be
  satisfied on the path to the recipient.  The enhanced status code
  is defined in {{RFC8689}}.

## Delivery Status Notifications {#dsn}

When a Message cannot be delivered after it has been accepted, or when
a delivery status notification (DSN) is requested on success or
delay, a DSN is generated as specified in {{RFC3461}} and {{RFC3464}}.
The DSN parameters of the Envelope ({{envelope}}) have the same
semantics as the corresponding SMTP parameters.  In particular:

- A Sending Server generates a DSN for an Envelope Recipient that was
  rejected by the Receiving Server or for which it has given up
  retrying ({{idempotency}}).  The Receiving Server does not generate
  a DSN for a recipient that it rejected in its response.

- A Receiving Server generates a DSN for an Envelope Recipient for
  which it accepted the Message but later failed to deliver it,
  including when the retrieval of referenced content fails after the
  response ({{reference-retrieval}}).

- A DSN is sent to the Envelope Sender of the original Message, with
  an empty Envelope Sender.  No DSN is generated for a Message whose
  Envelope Sender is empty.

- A server that relays a Message passes the DSN parameters on to the
  next hop, whether that hop uses HMTP or SMTP.

The fields of the DSN are filled in as follows:

- The "Reporting-MTA" field contains the host name of the reporting
  server with the type "dns".

- The "Remote-MTA" field, if present, contains the TargetName of the
  HMTP Endpoint or, after a fallback to SMTP ({{smtp-fallback}}), the
  host name of the SMTP server, with the type "dns".

- The "Status" field contains the enhanced status code from the
  "smtpStatus" member of the problem details object.  A Sending
  Server that gives up after retries uses the code 4.4.7 if no more
  specific code is available.

- The "Diagnostic-Code" field uses the diagnostic type "smtp" and
  contains the SMTP reply code, the enhanced status code, and the
  "detail" member of the problem details object, if any.

- The "Original-Recipient" and "Original-Envelope-Id" fields are
  derived from the "orcpt" and "envid" members of the Envelope.

If the Envelope requests the return of the full Message ("ret" is
"full") and the Message is very large, the DSN MAY contain only the
header section of the Message.

DSNs are ordinary Messages and are transferred using HMTP or SMTP like
any other Message.

### Example {#dsn-example}

The following DSN is generated by the Sending Server in the example of
{{fallback-example}} for the rejected Envelope Recipient
"dave@example.org".  It uses the "multipart/report" media type
{{RFC6522}} with a "message/delivery-status" part {{RFC3464}}.
Because the "ret" member of the Envelope is "hdrs", the DSN contains
only the header section of the original Message:

~~~
Date: Thu, 01 Jan 2026 12:00:03 +0000
From: Mail Delivery System <mailer-daemon@hmtp.provider.example>
To: alice@example.com
Subject: Delivery Status Notification (Failure)
Message-ID: <dsn.Xk4mP9qR2sT7@hmtp.provider.example>
MIME-Version: 1.0
Content-Type: multipart/report; report-type=delivery-status;
        boundary="dsn-b1"

--dsn-b1
Content-Type: text/plain; charset=us-ascii

Your message could not be delivered to the following recipient:

  dave@example.org
  550 5.1.1 Recipient address rejected

--dsn-b1
Content-Type: message/delivery-status

Reporting-MTA: dns; hmtp.provider.example
Original-Envelope-Id: QQ314159
Arrival-Date: Thu, 01 Jan 2026 11:59:58 +0000

Original-Recipient: rfc822;dave@example.org
Final-Recipient: rfc822;dave@example.org
Action: failed
Status: 5.1.1
Remote-MTA: dns; mx.example.org
Diagnostic-Code: smtp; 550 5.1.1 <dave@example.org>: Recipient
 address rejected

--dsn-b1
Content-Type: text/rfc822-headers

Received: from [198.51.100.7] ([198.51.100.7])
        by hmtp.provider.example with HMTPA id S-20260101-0043;
        Thu, 01 Jan 2026 11:59:58 +0000
Date: Thu, 01 Jan 2026 11:59:57 +0000
From: Alice <alice@example.com>
To: Dave <dave@example.org>, Erin <erin@example.org>
Subject: Quarterly report
Message-ID: <20260101115957.4f2a@example.com>
MIME-Version: 1.0
Content-Type: text/plain; charset=utf-8
Content-Transfer-Encoding: 8bit

--dsn-b1--
~~~

The fields of the "message/delivery-status" part are derived as
described above: "Original-Envelope-Id" from the "envid" member,
"Original-Recipient" from the "orcpt" member, "Status" and
"Diagnostic-Code" from the SMTP reply, and "Remote-MTA" from the host
name of the SMTP server.  If the failure had been reported by an HMTP
Receiving Server, "Status" would have been taken from the "smtpStatus"
member of the problem details object, "Diagnostic-Code" would have
been composed from its "smtpReply", "smtpStatus", and "detail"
members, and "Remote-MTA" would have contained the TargetName of the
HMTP Endpoint.

The DSN is delivered to the Envelope Sender of the original Message
like any other Message, with an empty Envelope Sender, so that no DSN
is generated if the DSN itself cannot be delivered.  When it is
transferred using HMTP, the Request Object contains the following
Envelope; because the Envelope Sender is empty, any Sending Domain is
acceptable ({{alignment}}):

~~~ json
{
  "id": "nH8sK2dL5fQ9wE3rT6yU1i",
  "from": "",
  "to": [{"address": "alice@example.com"}]
}
~~~

# SMTP Fallback and Interoperability {#smtp-fallback}

HMTP is a self-contained protocol.  Support for SMTP is OPTIONAL and
serves only the interoperability with domains that do not support
HMTP.  A Sending Server that does not support SMTP rejects Envelope
Recipients whose domains do not support HMTP with the "routing-failed"
problem type and a permanent enhanced status code.

## When to Fall Back {#when-to-fall-back}

A Sending Server that supports SMTP MAY deliver a Message to an
Envelope Recipient using SMTP only if both of the following
conditions hold:

1. Discovery ({{discovery}}) has determined that the domain of the
   Envelope Recipient does not support HMTP.

2. The Sending Server has no unexpired downgrade policy with the mode
   "enforce" for that domain ({{policy}}).

A Sending Server MUST NOT fall back to SMTP after a temporary failure
of discovery or of an HMTP request, such as a DNS timeout, a
connection or TLS failure, or a response with a 5xx or 429 status
code.  It retries the delivery using HMTP as described in
{{idempotency}}.  A Sending Server MUST NOT fall back to SMTP after
an HMTP request has been rejected permanently.

When it falls back, the Sending Server delivers the Message as
specified in {{RFC5321}}, with the following requirements:

- It locates the SMTP server using MX records as specified in
  {{Section 5.1 of RFC5321}}.  If the domain publishes a null MX
  record {{RFC7505}}, the Message cannot be delivered.

- It retrieves all referenced content and reconstructs the Message
  before the transfer, because SMTP cannot carry Content References.

- It transmits the Envelope using the parameters of the corresponding
  SMTP extensions: the BODY parameter for "body" {{RFC6152}}
  {{RFC3030}}, the SMTPUTF8 parameter for "smtpUtf8" {{RFC6531}}, the
  RET, ENVID, NOTIFY, and ORCPT parameters for the DSN parameters
  {{RFC3461}}, and the REQUIRETLS parameter for "requireTls"
  {{RFC8689}}.  Values are encoded as "xtext" where {{RFC3461}}
  requires it.

- It MUST NOT convert the Message.  If the SMTP server does not
  support an extension that the Message requires, such as 8BITMIME,
  BINARYMIME, or SMTPUTF8, the delivery to the affected Envelope
  Recipients fails with the "conversion-required" problem type.  If
  "requireTls" is true and the requirements of {{RFC8689}} cannot be
  met, the delivery fails with the "tls-required" problem type.

- If the SMTP server does not support the DSN extension, the Sending
  Server behaves as specified in {{RFC3461}} for relaying to a server
  that does not support it.

- It SHOULD use STARTTLS {{RFC3207}} and SHOULD apply MTA-STS
  {{RFC8461}} or DANE {{RFC7672}} when the domain publishes them.

### Example {#fallback-example}

The Submission Server "hmtp.provider.example" of the domain
"example.com", which also acts as its Sending Server, has accepted a
Message from a Client with the following Envelope:

~~~ json
{
  "id": "Xk4mP9qR2sT7vW1yZ3bC5d",
  "from": "alice@example.com",
  "to": [
    {
      "address": "dave@example.org",
      "dsn": {
        "notify": ["failure", "delay"],
        "orcpt": "rfc822;dave@example.org"
      }
    },
    {"address": "erin@example.org"}
  ],
  "dsn": {"ret": "hdrs", "envid": "QQ314159"},
  "body": "8bitmime"
}
~~~

Discovery for "example.org" shows that the domain does not support
HMTP, because the name "_hmtp.example.org" does not exist.  The
Sending Server has no recorded downgrade policy for "example.org", so
it falls back to SMTP and looks up the MX records of the domain:

~~~ dns
; _hmtp.example.org. IN SVCB  ->  NXDOMAIN
example.org.  3600 IN MX 10 mx.example.org.
~~~

The Sending Server then delivers the Message to "mx.example.org".  In
the following transcript, "C:" denotes lines sent by the Sending
Server and "S:" lines sent by the SMTP server:

~~~
NOTE: '\' line wrapping per RFC 8792

S: 220 mx.example.org ESMTP
C: EHLO hmtp.provider.example
S: 250-mx.example.org
S: 250-8BITMIME
S: 250-DSN
S: 250-ENHANCEDSTATUSCODES
S: 250 STARTTLS
C: STARTTLS
S: 220 2.0.0 Ready to start TLS
   (TLS handshake; the certificate of mx.example.org is validated)
C: EHLO hmtp.provider.example
S: 250-mx.example.org
S: 250-8BITMIME
S: 250-DSN
S: 250 ENHANCEDSTATUSCODES
C: MAIL FROM:<alice@example.com> BODY=8BITMIME RET=HDRS \
   ENVID=QQ314159
S: 250 2.1.0 Sender OK
C: RCPT TO:<dave@example.org> NOTIFY=FAILURE,DELAY \
   ORCPT=rfc822;dave@example.org
S: 550 5.1.1 <dave@example.org>: Recipient address rejected
C: RCPT TO:<erin@example.org>
S: 250 2.1.5 Recipient OK
C: DATA
S: 354 End data with <CR><LF>.<CR><LF>
C: Received: from [198.51.100.7] ([198.51.100.7])
C:         by hmtp.provider.example with HMTPA id S-20260101-0043;
C:         Thu, 01 Jan 2026 11:59:58 +0000
C: Date: Thu, 01 Jan 2026 11:59:57 +0000
C: From: Alice <alice@example.com>
C: To: Dave <dave@example.org>, Erin <erin@example.org>
C: Subject: Quarterly report
C: Message-ID: <20260101115957.4f2a@example.com>
C: MIME-Version: 1.0
C: Content-Type: text/plain; charset=utf-8
C: Content-Transfer-Encoding: 8bit
C:
C: (body of the Message)
C: .
S: 250 2.0.0 Ok: queued as 7HG2Lk
C: QUIT
S: 221 2.0.0 Bye
~~~

The members of the Envelope are mapped to SMTP as follows:

- "from" and the "address" members of the Recipient Objects become
  the arguments of the MAIL FROM and RCPT TO commands.

- "body" with the value "8bitmime" becomes the parameter
  "BODY=8BITMIME".  The SMTP server advertises 8BITMIME, so the
  Message can be transferred without conversion.  If it did not, the
  Sending Server would fail the delivery to both recipients with the
  "conversion-required" problem type instead of converting the
  Message.

- "ret" and "envid" become the parameters "RET=HDRS" and
  "ENVID=QQ314159", and the "dsn" member of the first Recipient
  Object becomes the parameters "NOTIFY=FAILURE,DELAY" and
  "ORCPT=rfc822;dave@example.org".  The values contain no characters
  that require "xtext" encoding.  The SMTP server advertises the DSN
  extension {{RFC3461}}, so the parameters can be transmitted.

- The Message is transmitted octet for octet as it was accepted from
  the Client, including the Received header field that the Submission
  Server prepended ({{trace}}).  The SMTP server advertises
  ENHANCEDSTATUSCODES {{RFC2034}}, so its replies contain enhanced
  status codes.

The SMTP server accepts the Message for "erin@example.org".  It
rejects "dave@example.org" with a permanent failure.  Because the
Recipient Object of "dave@example.org" requests notification on
failure, the Sending Server generates the DSN shown in
{{dsn-example}}.  Had the SMTP server failed with a temporary error,
the Sending Server would have retried later, repeating discovery
before each retry, and would have used HMTP if "example.org" had
started to publish an HMTP Endpoint in the meantime.

## Downgrade Policy {#policy}

Without DNSSEC, an attacker who can modify DNS responses can remove
the SVCB records of a domain and thereby cause Sending Servers to fall
back to SMTP.  A downgrade policy allows a domain to instruct Sending
Servers that have once reached it over HMTP not to fall back to SMTP,
in a way similar to HTTP Strict Transport Security {{RFC6797}} and
MTA-STS {{RFC8461}}.

The policy is expressed in the "policy" member of the Capabilities
Document ({{capabilities-document}}), which is a JSON object with the
following members:

mode:
: REQUIRED.  The string "enforce" or "none".

maxAge:
: REQUIRED.  The number of seconds for which a Sending Server applies
  the policy, as a non-negative integer.  The value MUST NOT exceed
  31557600 (one year).  Sending Servers MUST treat larger values as
  31557600.

Whenever a Sending Server has located a host through discovery for a
domain and has successfully retrieved the Capabilities Document of
the host, it updates its policy state for that domain as follows:

- If the document contains a policy with the mode "enforce", the
  Sending Server records that the policy applies to the domain until
  "maxAge" seconds after the time of retrieval.

- If the document contains a policy with the mode "none", or no
  policy, the Sending Server removes any recorded policy for the
  domain.

While an unexpired policy with the mode "enforce" is recorded for a
domain, the Sending Server MUST treat a discovery result indicating
that the domain does not support HMTP as a temporary failure and MUST
NOT fall back to SMTP.  It retries delivery as described in
{{idempotency}} and, when it gives up, generates a DSN.

Because the Capabilities Document belongs to a host, its policy
applies to all domains that use the host.  A domain that intends to
stop using HMTP first publishes, through its host, a policy with the
mode "none" or a "maxAge" of 0, and keeps its SVCB records published
for at least the previously advertised "maxAge" before removing them.

A Sending Server that does not support SMTP fallback does not need to
record policies.

## SMTP to HMTP Gateways {#gateways}

An SMTP server can relay Messages that it receives over SMTP to an
HMTP Endpoint.  Such a gateway acts as a Sending Server and
constructs the Envelope from the SMTP transaction:

- "from" and the "address" members of the Recipient Objects are
  taken from the MAIL FROM and RCPT TO commands;

- "body", "smtpUtf8", and "requireTls" are taken from the BODY,
  SMTPUTF8, and REQUIRETLS parameters;

- the DSN parameters are taken from the RET, ENVID, NOTIFY, and ORCPT
  parameters, after decoding "xtext".

The Message is the message received over SMTP, including the Received
header field that the gateway prepends as an SMTP server.  A gateway
SHOULD record the results of the authentication checks it performed
on the SMTP transaction, such as SPF, DKIM, and DMARC, in an
Authentication-Results header field {{RFC8601}}.

A gateway signs its requests with a key of its own domain.  For
outbound Messages of the domain that operates the gateway, the
Envelope Sender is aligned as usual.  A gateway that relays Messages
from other domains, such as an inbound filtering service or a backup
server, cannot produce aligned signatures for them.  It is used under
a local agreement with the Receiving Server, as described in
{{alignment}}.

A gateway MAY relay the Message over HMTP before it replies to the
SMTP DATA or BDAT command.  In that case, it translates the result of
the HMTP request into the SMTP reply, using the "smtpReply" and
"smtpStatus" members of the problem details object, and the mapping
in {{tab-mapping}}.  Otherwise, it accepts the Message, relays it
later, and reports failures through DSNs.

# Deployment and Transition Considerations

HMTP is designed to be deployed alongside SMTP, without a flag day.
The following considerations apply:

Receiving mail:
: A domain enables inbound HMTP by publishing an SVCB record and
  serving a Capabilities Document, while keeping its MX records.
  Because the TLS certificate is validated against the TargetName, a
  provider that hosts many domains needs a certificate only for its
  own host name, not for each customer domain.

Sending mail:
: A domain enables outbound HMTP by publishing a key record for HMTP,
  preferably under a dedicated selector with the service type "hmtp",
  and by enabling discovery in its Sending Servers.  Sending Servers
  that support SMTP fall back to it for domains that do not yet
  support HMTP.

Downgrade policy:
: A domain SHOULD start with a short "maxAge", or without a policy,
  and increase it once it is confident that its HMTP Endpoint is
  reliable, because a policy with the mode "enforce" prevents
  delivery over SMTP for its duration.

Reputation:
: Receiving Servers base reputation on the Sending Domain instead of
  IP addresses.  New Sending Domains still need to establish
  reputation, and Receiving Servers MAY apply rate limits to Sending
  Domains without history.

Infrastructure:
: HMTP uses standard HTTP infrastructure, including load balancers,
  reverse proxies, and HTTP libraries.  Operators need to ensure that
  intermediaries preserve the components covered by signatures
  ({{verification}}) and accept request sizes up to the advertised
  "maxRequestSize".  Large content is best transferred by reference,
  so that it can be served from storage optimized for that purpose.

Clients:
: Clients locate their Submission Server through discovery or
  configuration.  Existing SMTP AUTH credentials can be used with the
  "basic" scheme, and Clients can migrate to OAuth 2.0 bearer tokens
  over time.

Operating both protocols:
: A server that supports both protocols can use a single queue,
  because the Message is the same in both protocols and only the
  Envelope representation differs.

# Implementation Status

TODO list implementations.

# Security Considerations {#security}

## Transport Security

All HMTP requests, including the retrieval of the Capabilities
Document and of referenced content, use TLS with certificate
validation.  No plaintext mode exists.  TLS protects the
confidentiality and integrity of each hop, but not of the Message
end to end: every server that handles a Message has access to its
content, as in SMTP.  End-to-end protection requires OpenPGP or
S/MIME.

## DNS and Downgrade Attacks {#security-dns}

The TLS certificate of the HMTP Endpoint is validated against the
TargetName obtained from DNS.  If DNS responses are not authenticated,
an attacker who can modify them can redirect HMTP requests to a host
under its control, for which it can obtain a valid certificate, or can
remove the SVCB records so that a Sending Server falls back to SMTP.
This is the same exposure that MX records have without DNSSEC.
Domains SHOULD sign their zones with DNSSEC, and Sending Servers
SHOULD validate DNS responses.

The downgrade policy ({{policy}}) protects against the removal of
SVCB records after a Sending Server has reached a domain once, but
not on first contact and not against redirection to another host.
The rule that temporary failures never cause a fallback to SMTP
({{when-to-fall-back}}) prevents an attacker from forcing a downgrade
by blocking HTTPS connections.

## Signatures and Replay {#security-replay}

Signatures cover the method, the target URI, the media type, and the
digest of the content, and are therefore bound to the Envelope, the
Message, and the Receiving Server.  A captured request cannot be
replayed to another Receiving Server, because the target URI differs.
A replay to the same Receiving Server is limited by the validity
period of the signature, which is at most 300 seconds, and within that
period by the Transfer Identifier, which causes the Receiving Server
to return the stored response instead of delivering the Message again.
Verifiers need reasonably accurate clocks.

## Key Management {#security-keys}

The security of HMTP depends on the protection of the private keys of
the Sending Domain.  A compromised key allows an attacker to send
requests on behalf of the domain until the key record is revoked.
Domains SHOULD rotate keys regularly as described in
{{signing-keys}}.

When a key is shared between DKIM message signing and HMTP request
signing, a compromise affects both uses.  The data that HMTP signs,
the signature base defined by {{RFC9421}}, has a different structure
from the data signed by DKIM, which makes it difficult to use a
signature from one protocol in the other.  Nevertheless, keys
dedicated to HMTP with the service type "hmtp" are RECOMMENDED.  RSA
keys shorter than 2048 bits, which {{RFC8301}} still permits for
DKIM, are not permitted for HMTP.

## Sender Alignment

Alignment permits a domain to sign for its subdomains.  A Receiving
Server SHOULD NOT accept signatures of domains that are registry
suffixes, such as top-level domains, as authenticating the Envelope
Sender, because such domains do not operate mail services for the
domains registered under them.

An HMTP signature authenticates the Sending Domain and its alignment
with the Envelope Sender.  It does not authenticate the author of the
Message.  Receiving Servers MUST NOT present the Sending Domain to
users as the author of the Message.

## Content References

Retrieving referenced content causes the Receiving Server to send
requests to URIs chosen by the sender.  A Receiving Server MUST NOT
retrieve content from IP addresses that are not globally routable,
such as loopback, private, and link-local addresses, unless it is
explicitly configured to do so, in order to prevent attacks on
internal services.  It SHOULD apply limits to the number of
redirections, the duration of retrievals, and the number of
concurrent retrievals per Sending Domain.

The integrity of referenced content is protected by its digest, which
is covered by the signature of the request.  The size of referenced
content is declared in advance, so a Receiving Server can reject
oversized Messages before retrieving them and can stop a retrieval
that exceeds the declared size.

URIs of referenced content act as bearer capabilities: anyone who
learns them can retrieve the content until it is removed.  Senders
SHOULD use URIs that contain sufficient randomness to be unguessable,
SHOULD remove content after its "expires" timestamp, and MAY require
signed retrieval requests ({{retrieval-signing}}).  Receiving Servers
MUST NOT disclose these URIs to recipients of the Message.

## Denial of Service

HMTP servers are exposed to resource exhaustion through large
requests, many concurrent requests, slow transmission of requests,
and Messages with many segments or recipients.  Servers SHOULD
advertise and enforce limits ({{capabilities-document}}), SHOULD
apply rate limits per Sending Domain and per IP address, and MAY
verify the signature before receiving the content of the request.
Because the identity of the sender is established by the signature,
rate limits and blocking can be applied per Sending Domain rather
than per IP address.

## Client Authentication

Credentials used with the "basic" scheme are passwords and are subject
to guessing and phishing.  Servers SHOULD support bearer tokens,
SHOULD support application-specific passwords for the "basic" scheme,
and SHOULD limit the rate of failed attempts.  Bearer tokens SHOULD be
restricted to the Submission Server as their audience, and Clients
MUST NOT send them to other servers.

A Submission Server MUST authenticate every submission request and
MUST verify the sender addresses ({{sender-authorization}}).  A
transfer endpoint MUST NOT accept Messages for domains it does not
serve, except under a local relay agreement.  Otherwise, the server
would act as an open relay.

## Transfer Identifiers

Transfer Identifiers are scoped to the Sending Domain or to the
authenticated account, so a third party cannot cause a legitimate
request to be suppressed as a duplicate without being able to sign
requests for the same scope.  The limit on the length of Transfer
Identifiers bounds the storage that a Receiving Server needs for them.

# Privacy Considerations {#privacy}

HMTP exposes the same information to the servers that handle a
Message as SMTP does: the Envelope, the Message, and the IP addresses
of the communicating servers.  TLS protects this information from
observers on the network.  The name of the HMTP Endpoint is visible in
the TLS handshake unless Encrypted Client Hello is used, and DNS
queries for SVCB records reveal the domains with which a server
exchanges mail unless encrypted DNS transports are used.

An HMTP request contains all Envelope Recipients that are served by
the same HMTP Endpoint.  As in SMTP, the Receiving Server learns all
of them, including recipients that are not visible in the header
fields of the Message, such as blind carbon copy recipients.
Receiving Servers MUST NOT disclose the list of Envelope Recipients to
recipients of the Message, and include the "for" clause in Received
header fields only for single recipients ({{trace}}).

The retrieval of referenced content reveals to the server that hosts
the content which Receiving Servers retrieve it and when.  Because
retrieval is performed by the Receiving Server and is not triggered
by actions of the recipient ({{reference-retrieval}}), it does not
reveal whether or when the recipient reads the Message.  Senders can
still learn which Receiving Servers serve the recipients, which they
can also learn from DNS.

The identities endpoint reveals the addresses of an account only to
the authenticated Client.  The Capabilities Document is public and
reveals the endpoints and limits of the host, and implicitly the
provider that serves a domain, which can also be learned from DNS.

Receiving Servers store Transfer Identifiers together with a hash of
the request content and the response.  This information SHOULD NOT be
retained longer than needed for the purposes described in
{{idempotency}}.

Submission Servers MAY replace the IP address of the Client in the
Received header field with a placeholder ({{trace}}) to protect the
privacy of the user.

# IANA Considerations {#iana}

## Underscored and Globally Scoped DNS Node Name

IANA is requested to add the following entry to the "Underscored and
Globally Scoped DNS Node Names" registry {{RFC8552}}:

| RR Type | _NODE NAME | Reference |
|---|---|---|
| SVCB | _hmtp | RFC XXXX |

## Well-Known URI

IANA is requested to add the following entry to the "Well-Known URIs"
registry {{RFC8615}}:

URI Suffix:
: hmtp

Change Controller:
: IETF

Specification Document(s):
: {{capabilities-document}} of RFC XXXX

Status:
: permanent

Related Information:
: None

## DKIM Service Type {#iana-dkim}

IANA is requested to add the following entry to the "DKIM Service
Types" registry {{RFC6376}}:

| Type | Reference | Status |
|---|---|---|
| hmtp | RFC XXXX | active |

## Mail Transmission Types {#iana-transmission-types}

IANA is requested to add the following entries to the "Mail
Transmission Types" registry in the "Mail Parameters" registry group:

| WITH protocol type | Description | Reference |
|---|---|---|
| HMTP | HTTP Mail Transfer Protocol, transfer | RFC XXXX |
| HMTPA | HTTP Mail Transfer Protocol, submission with client authentication | RFC XXXX |

## URN Sub-namespace {#iana-urn}

IANA is requested to register the following entry in the "IETF URN
Sub-namespace for Registered Protocol Parameter Identifiers" registry
{{RFC3553}}:

Registry name:
: hmtp

Specification:
: RFC XXXX

Repository:
: The "HTTP Mail Transfer Protocol (HMTP)" registry group defined in
  this document.  URNs of the form
  "urn:ietf:params:hmtp:error:<name>" identify problem types
  registered in the "HMTP Problem Types" registry.

Index value:
: The name of a problem type, as registered in the "HMTP Problem
  Types" registry.

## HMTP Registry Group

IANA is requested to create a new registry group titled "HTTP Mail
Transfer Protocol (HMTP)" containing the registries defined in the
following subsections.  For all of them, the registration policy is
Specification Required {{RFC8126}}.  The designated experts verify
that the specification is publicly available, that the registered
name does not conflict with an existing entry, and that the
specification is consistent with the extension rules of
{{capabilities}}.

### HMTP Capabilities {#iana-capabilities}

The registry contains capabilities with registered names.
Capabilities identified by URIs are not registered (see
{{capability-names}}).  The registry records the following fields for
each capability:

- Name: the registered name of the capability, with the syntax
  defined in {{capability-names}}.
- Description: a brief description.
- Must-Understand: "yes" or "no", as defined in {{capabilities}}.
- Reference: the specification of the capability.
- Change Controller: the party responsible for the capability.

The registry is initially empty.

### HMTP Endpoints {#iana-endpoints}

The registry records the following fields for each endpoint name
used in the "endpoints" member of a Version Object:

- Name: the member name.
- Description: a brief description.
- Reference: the specification of the endpoint and of the members of
  its Endpoint Object.

The initial contents are:

| Name | Description | Reference |
|---|---|---|
| transfer | Transfer of Messages between servers | {{transfer}} and {{endpoint-objects}} of RFC XXXX |
| submission | Submission of Messages by Clients, and identities of an authenticated Client | {{submission}} and {{endpoint-objects}} of RFC XXXX |

### HMTP Content Encodings {#iana-encodings}

The registry records the following fields for each encoding used in
the "encoding" member of a Content Reference:

- Name: the name of the encoding.
- Description: a brief description of the transformation.
- Reference: the specification of the encoding.

The initial contents are:

| Name | Description | Reference |
|---|---|---|
| base64 | Base64 with lines of 76 characters separated by CRLF | {{attachments}} of RFC XXXX |

### HMTP Problem Types {#iana-problem-types}

The registry records the following fields for each problem type:

- Name: the name of the problem type, used in the URN
  "urn:ietf:params:hmtp:error:<name>".
- Scope: "Request", "Recipient", or "Both".
- HTTP Status: the HTTP status code used for request-level failures.
- Enhanced Status: the enhanced status code or codes.
- SMTP Reply: the SMTP reply code or codes.
- Reference: the specification of the problem type.

The initial contents are the problem types listed in {{tab-mapping}},
each with the reference {{smtp-mapping}} of RFC XXXX.


--- back

# Examples {#examples}

This appendix contains complete examples of HMTP exchanges.  As noted
in {{conventions}}, base64 data, digests, and signature
values are abbreviated or illustrative.

## Discovery

A Sending Server that has a Message for "bob@example.net" queries DNS
and retrieves the Capabilities Document of the target host:

~~~ dns
_hmtp.example.net.  3600 IN SVCB 1 mx.example.net. alpn=h2,h3
~~~

~~~ http-message
GET /.well-known/hmtp HTTP/1.1
Host: mx.example.net

HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=86400

{
  "versions": {
    "1": {
      "endpoints": {
        "transfer": {
          "uri": "/hmtp/v1/transfer"
        }
      }
    }
  },
  "limits": {
    "maxMessageSize": 10737418240,
    "maxRequestSize": 26214400,
    "maxRecipients": 500
  },
  "policy": {
    "mode": "enforce",
    "maxAge": 2592000
  }
}
~~~

## Transfer with Referenced Content

The Sending Server transfers a Message with a large attachment by
reference.  The Receiving Server accepts the Message and retrieves the
attachment after responding, which it indicates with the status code
202 (Accepted).  The Sending Server keeps the attachment available
until its "expires" timestamp:

~~~ http-message
NOTE: '\' line wrapping per RFC 8792

POST /hmtp/v1/transfer HTTP/1.1
Host: mx.example.net
Content-Type: application/json
Content-Digest: sha-256=:q1M8rT4vYw7Zb2Nd5Kf0Hs9Lx3Jc6Pg1Ue8Ra4Wi2Oo=:
Signature-Input: hmtp=("@method" "@target-uri" "content-type" \
  "content-digest");created=1767268800;expires=1767269100;\
  keyid="hmtp2026._domainkey.example.com";alg="ed25519";tag="hmtp"
Signature: hmtp=:Zm9vYmFyZXhhbXBsZXNpZ25hdHVyZXZhbHVlbm90cmVh\
  bGx5dmFsaWRqdXN0aWxsdXN0cmF0aXZlb25seWZvcg==:

{
  "envelope": {
    "id": "Hq9vR2xLm4Tz8Kc1Wb6Nd0",
    "from": "alice@example.com",
    "to": [{"address": "bob@example.net"}],
    "body": "7bit"
  },
  "message": {
    "segments": [
      {"data": "RnJvbTogQWxpY2UgPGFsaWNlQGV4YW1wbGUuY29tPg0K..."},
      {
        "uri": "https://files.example.com/f/9b1c4e7a",
        "size": 734003200,
        "digest": {
          "sha-256": "Yk3qV0sJpRm9T2cFhWxN6dL1bA8eQu7ZgKi4oPyXwE0="
        },
        "expires": "2026-01-08T12:00:00Z",
        "encoding": "base64"
      },
      {"data": "DQotLWIxLS0NCg=="}
    ],
    "size": 1004426786
  }
}

HTTP/1.1 202 Accepted
Content-Type: application/json

{
  "queueId": "8Jk2Pq7Xv1",
  "recipients": [
    {"address": "bob@example.net", "result": "accepted"}
  ]
}
~~~

## Submission

A Client submits a Message with Basic authentication.  The Submission
Server accepts responsibility for both recipients and reports the
outcome of the final delivery through DSNs:

~~~ http-message
POST /hmtp/v1/submission HTTP/1.1
Host: hmtp.provider.example
Authorization: Basic YWxpY2U6czNjcjN0LWFwcC1wYXNzd29yZA==
Content-Type: application/json

{
  "envelope": {
    "id": "c5N1qW8eR3tY6uI9oP2aS4",
    "from": "alice@example.com",
    "to": [
      {"address": "bob@example.net"},
      {"address": "dave@example.org"}
    ]
  },
  "message": {
    "data": "RnJvbTogQWxpY2UgPGFsaWNlQGV4YW1wbGUuY29tPg0K..."
  }
}

HTTP/1.1 200 OK
Content-Type: application/json

{
  "queueId": "S-20260101-0042",
  "recipients": [
    {"address": "bob@example.net", "result": "accepted"},
    {"address": "dave@example.org", "result": "accepted"}
  ]
}
~~~

## Retried Request

The connection is lost before the Sending Server receives the
response to the request in {{transfer-request}}.  The Sending Server
sends the identical request again with a new signature.  The Receiving
Server recognizes the Transfer Identifier and returns the stored
response without delivering the Message again:

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json

{
  "queueId": "4Y7mKq2ZtR",
  "recipients": [
    {"address": "bob@example.net", "result": "accepted"},
    {
      "address": "carol@example.net",
      "result": "rejected",
      "problem": {
        "type": "urn:ietf:params:hmtp:error:unknown-recipient",
        "title": "Unknown recipient",
        "smtpStatus": "5.1.1",
        "smtpReply": 550
      }
    }
  ]
}
~~~

TODO examples.

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
