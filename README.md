# HTTP Mail Transfer Protocol (HMTP)

This is the working area for the individual Internet-Draft
"HTTP Mail Transfer Protocol", `draft-tishkin-hmtp`.

HMTP is a protocol for the submission and transfer of Internet mail over
HTTPS. It covers both the submission of messages by clients to their mail
service and the transfer of messages between mail servers, and is designed to
be deployed incrementally alongside SMTP.

| | |
|---|---|
| Editor's Copy | [HTML](https://dmi03.github.io/draft-tishkin-hmtp/) |
| Published Draft | [draft-tishkin-hmtp-00](https://datatracker.ietf.org/doc/draft-tishkin-hmtp/) ([HTML](https://www.ietf.org/archive/id/draft-tishkin-hmtp-00.html), [TXT](https://www.ietf.org/archive/id/draft-tishkin-hmtp-00.txt)) |
| Compare | [Editor's Copy vs. latest published version](https://author-tools.ietf.org/iddiff?url1=draft-tishkin-hmtp&url2=https://dmi03.github.io/draft-tishkin-hmtp/draft-tishkin-hmtp.txt) |

The Editor's Copy is built automatically by GitHub Actions from the `main`
branch and always reflects the latest changes. The published draft is the
version submitted to the IETF.

## Overview

- **Mandatory transport security.** HMTP is only defined over HTTPS with
  certificate validation. There is no plaintext mode, and a downgrade policy
  lets a domain prevent fallback to SMTP.
- **Unmodified messages.** Messages are carried in the Internet Message Format
  (RFC 5322) exactly as produced by the sender, inside a JSON envelope that
  holds the delivery information. Existing DKIM signatures, OpenPGP, and
  S/MIME remain valid.
- **Domain-based authentication.** Every transfer request is signed with HTTP
  Message Signatures (RFC 9421), using keys published in DNS in the same way as
  DKIM keys. Receivers identify senders by domain rather than by IP address.
- **Content by reference.** Large messages and attachments can be provided as
  URLs with their size and hash and are retrieved by the receiving server, so
  that they do not have to be carried in every request.
- **Idempotent requests.** Each request carries a Transfer Identifier, so a
  retried request is not delivered twice.
- **Discovery through DNS.** An SVCB record at `_hmtp.<domain>` locates the
  endpoint, and `/.well-known/hmtp` describes its capabilities.
- **Structured errors.** Failures are reported as Problem Details (RFC 9457)
  with a defined mapping to SMTP enhanced status codes.
- **Extensibility.** New functionality is added through capabilities, without
  changing the base protocol.
- **Optional SMTP fallback.** HMTP is self-contained. An implementation can
  additionally fall back to SMTP for domains that do not support HMTP yet.

## Example

A Sending Server transfers a message to the receiving domain:

```http
POST /hmtp/v1/transfer HTTP/1.1
Host: mx.example.net
Content-Type: application/json
Content-Digest: sha-256=:...:
Signature-Input: hmtp=("@method" "@target-uri" "content-type" "content-digest");created=1767268800;expires=1767269100;keyid="hmtp2026._domainkey.example.com";alg="ed25519";tag="hmtp"
Signature: hmtp=:...:

{
  "envelope": {
    "id": "3q2-7wEXAMPLEd9Kc1fQ0g",
    "from": "alice@example.com",
    "to": [{"address": "bob@example.net"}]
  },
  "message": {
    "data": "RnJvbTogQWxpY2UgPGFsaWNlQGV4YW1wbGUuY29tPg0K..."
  }
}
```

The receiving server reports the result for each recipient:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "recipients": [
    {"address": "bob@example.net", "result": "accepted"}
  ]
}
```

More examples, including submission, transfer by reference, SMTP fallback, and
delivery status notifications, are in the draft.

## Contributing

See the [guidelines for contributions](CONTRIBUTING.md).

Comments and proposed changes are welcome as
[issues](https://github.com/dmi03/draft-tishkin-hmtp/issues) and
[pull requests](https://github.com/dmi03/draft-tishkin-hmtp/pulls) in this
repository, or by email to the author at <hello@dmi03.com>.

Contributions can be made by editing the
[draft source](draft-tishkin-hmtp.md) directly in the GitHub web interface.

## Command Line Usage

Formatted text and HTML versions of the draft can be built using `make`.

```sh
$ make
```

Command line usage requires that you have the necessary software installed.
See [the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).

## License

See [LICENSE.md](LICENSE.md).
