---
title: TLS Encrypted Client Hello
abbrev: TLS Encrypted Client Hello
docname: draft-ietf-tls-esni-latest
category: std

ipr: trust200902
area: General
workgroup: tls
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
       ins: E. Rescorla
       name: Eric Rescorla
       organization: RTFM, Inc.
       email: ekr@rtfm.com

 -
       ins: K. Oku
       name: Kazuho Oku
       organization: Fastly
       email: kazuhooku@gmail.com

 -
       ins: N. Sullivan
       name: Nick Sullivan
       organization: Cloudflare
       email: nick@cloudflare.com

 -
       ins: C. A. Wood
       name: Christopher A. Wood
       organization: Cloudflare
       email: caw@heapingbits.net


normative:
  RFC2119:
  RFC7918:

--- abstract

This document describes a mechanism in Transport Layer Security (TLS) for
encrypting a ClientHello message under a server public key.

--- middle

# Introduction {#intro}

DISCLAIMER: This draft is work-in-progress and has not yet seen significant (or
really any) security analysis. It should not be used as a basis for building
production systems.

Although TLS 1.3 {{!RFC8446}} encrypts most of the handshake, including the
server certificate, there are several ways in which an on-path attacker can
learn private information about the connection. The plaintext Server Name
Indication (SNI) extension in ClientHello messages, which leaks the target
domain for a given connection, is perhaps the most sensitive, unencrypted
information in TLS 1.3.

The target domain may also be visible through other channels, such as plaintext
client DNS queries, visible server IP addresses (assuming the server does not
use domain-based virtual hosting), or other indirect mechanisms such as traffic
analysis. DoH {{?RFC8484}} and DPRIVE {{?RFC7858}}
{{?RFC8094}} provide mechanisms for clients to conceal DNS lookups from network
inspection, and many TLS servers host multiple domains on the same IP address.
In such environments, the SNI remains the primary explicit signal used to
determine the server's identity.

The TLS Working Group has studied the problem of protecting the SNI, but has
been unable to develop a completely generic solution.
{{?RFC8744}} provides a description of the problem space and
some of the proposed techniques. One of the more difficult problems is "Do not
stick out" ({{?RFC8744}}, Section 3.4): if only sensitive or
private services use SNI encryption, then SNI encryption is a signal that a
client is going to such a service. For this reason, much recent work has focused
on concealing the fact that the SNI is being protected. Unfortunately, the
result often has undesirable performance consequences, incomplete coverage, or
both.

The protocol specified by this document takes a different approach. It assumes
that private origins will co-locate with or hide behind a provider (reverse
proxy, application server, etc.) that protects sensitive ClientHello parameters,
including the SNI, for all of the domains it hosts. These co-located servers
form an anonymity set wherein all elements have a consistent configuration,
e.g., the set of supported application protocols, ciphersuites, TLS versions,
and so on. Usage of this mechanism reveals that a client is connecting to a
particular service provider, but does not reveal which server from the anonymity
set terminates the connection. Thus, it leaks no more than what is already
visible from the server IP address.

This document specifies a new TLS extension, called Encrypted Client Hello
(ECH), that allows clients to encrypt their ClientHello to a supporting server.
This protects the SNI and other potentially sensitive fields, such as the ALPN
list {{?RFC7301}}. This extension is only supported with (D)TLS 1.3 {{!RFC8446}}
and newer versions of the protocol.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here. All TLS
notation comes from {{RFC8446}}, Section 3.

# Overview

This protocol is designed to operate in one of two topologies illustrated below,
which we call "Shared Mode" and "Split Mode".

## Topologies

~~~~
                +---------------------+
                |                     |
                |   2001:DB8::1111    |
                |                     |
Client <----->  | private.example.org |
                |                     |
                | public.example.com  |
                |                     |
                +---------------------+
                        Server
~~~~
{: #shared-mode title="Shared Mode Topology"}

In Shared Mode, the provider is the origin server for all the domains whose DNS
records point to it. In this mode, the TLS connection is terminated by the
provider.

~~~~
           +--------------------+     +---------------------+
           |                    |     |                     |
           |   2001:DB8::1111   |     |   2001:DB8::EEEE    |
Client <----------------------------->|                     |
           | public.example.com |     | private.example.com |
           |                    |     |                     |
           +--------------------+     +---------------------+
            Client-Facing Server            Backend Server
~~~~
{: #split-mode title="Split Mode Topology"}

In Split Mode, the provider is not the origin server for private domains.
Rather, the DNS records for private domains point to the provider, and the
provider's server relays the connection back to the origin server, who
terminates the TLS connection with the client. Importantly, service provider
does not have access to the plaintext of the connection.

In the remainder of this document, we will refer to the ECH-service provider as
the "client-facing server" and to the TLS terminator as the "backend server".
These are the same entity in Shared Mode, but in Split Mode, the client-facing
and backend servers are physically separated.

## Encrypted ClientHello (ECH)

ECH allows the client to encrypt sensitive ClientHello extensions, e.g., SNI,
ALPN, etc., under the public key of the client-facing server. This requires the
client-facing server to publish the public key and metadata it uses for ECH for
all the domains for which it serves directly or indirectly (via Split Mode).
This document defines the format of the ECH encryption public key and metadata,
referred to as an ECH configuration, and delegates DNS publication details to
{{!HTTPS-RR=I-D.ietf-dnsop-svcb-https}}, though other delivery mechanisms are
possible. In particular, if some of the clients of a private server are
applications rather than Web browsers, those applications might have the public
key and metadata preconfigured.

When a client wants to establish a TLS session with the backend server, it
constructs its ClientHello as indicated in {{real-ech}}. We will refer to
this as the ClientHelloInner message. The client encrypts this message using
the public key of the ECH configuration. It then constructs a new ClientHello,
the ClientHelloOuter, with innocuous values for sensitive extensions, e.g., SNI,
ALPN, etc., and with an "encrypted_client_hello" extension, which this document
defines ({{encrypted-client-hello}}). The extension's payload carries the
encrypted ClientHelloInner and specifies the ECH configuration used for
encryption. Finally, it sends ClientHelloOuter to the server.

Upon receiving the ClientHelloOuter, a TLS server takes one of the following
actions:

1. If it does not support ECH, it ignores the "encrypted_client_hello" extension
   and proceeds with the handshake as usual, per {{RFC8446}}, Section 4.1.2.
1. If it is a client-facing server for the ECH protocol, but cannot decrypt the
   extension, then it terminates the handshake using the ClientHelloOuter. This
   is referred to as "ECH rejection". When ECH is rejected, the client-facing
   server sends an acceptable ECH configuration in its EncryptedExtensions
   message.
1. If it supports ECH and decrypts the extension, it forwards the
   ClientHelloInner to the backend server, who terminates the connection. This
   is referred to as "ECH acceptance".

Upon receiving the server's response, the client determines whether or not ECH
was accepted and proceeds with the handshake accordingly. (See
{{client-behavior}} for details.)

The primary goal of ECH is to ensure that connections to servers in the same
anonymity set are indistinguishable from one another. Moreover, it should
achieve this goal without affecting any existing security properties of TLS 1.3.
See {{goals}} for more details about the ECH security and privacy goals.

# Encrypted ClientHello Configuration {#ech-configuration}

ECH uses draft-08 of HPKE for public key encryption {{!I-D.irtf-cfrg-hpke}}.
The ECH configuration is defined by the following `ECHConfig` structure.

~~~~
    opaque HpkePublicKey<1..2^16-1>;
    uint16 HpkeKemId;  // Defined in I-D.irtf-cfrg-hpke
    uint16 HpkeKdfId;  // Defined in I-D.irtf-cfrg-hpke
    uint16 HpkeAeadId; // Defined in I-D.irtf-cfrg-hpke

    struct {
        HpkeKdfId kdf_id;
        HpkeAeadId aead_id;
    } HpkeSymmetricCipherSuite;

    struct {
        uint8 config_id;
        HpkeKemId kem_id;
        HpkePublicKey public_key;
        HpkeSymmetricCipherSuite cipher_suites<4..2^16-4>;
    } HpkeKeyConfig;

    struct {
        HpkeKeyConfig key_config;
        uint8 maximum_name_length;
        opaque public_name<1..255>;
        Extension extensions<0..2^16-1>;
    } ECHConfigContents;

    struct {
        uint16 version;
        uint16 length;
        select (ECHConfig.version) {
          case 0xfe0a: ECHConfigContents contents;
        }
    } ECHConfig;
~~~~

The structure contains the following fields:

version
: The version of ECH for which this configuration is used. Beginning with
draft-08, the version is the same as the code point for the
"encrypted_client_hello" extension. Clients MUST ignore any `ECHConfig`
structure with a version they do not support.

length
: The length, in bytes, of the next field.

contents
: An opaque byte string whose contents depend on the version. For this
specification, the contents are an `ECHConfigContents` structure.

The `ECHConfigContents` structure contains the following fields:

key_config
: A `HpkeKeyConfig` structure carrying the configuration information associated
with the HPKE public key. Note that this structure contains the `config_id`
field, which applies to the entire ECHConfigContents. Sites MUST NOT publish
two different `ECHConfigContents` values with the same `HpkeKeyConfig` value.
The RECOMMENDED technique for choosing `config_id` is to choose a random byte.
This process is repeated if this config_id matches that of any valid ECHConfig,
which could include any ECHConfig that has been recently removed from active
use.

maximum_name_length
: The longest name of a backend server, if known. If this value is not known it
can be set to zero, in which case clients SHOULD use the inner ClientHello
padding scheme described below. That could happen if wildcard names are in use,
or if names can be added or removed from the anonymity set during the lifetime
of a particular ECH configuration.

public_name
: The non-empty name of the client-facing server, i.e., the entity trusted to
update the ECH configuration. This is used to correct misconfigured clients, as
described in {{handle-server-response}}. This value MUST NOT begin or end with
an ASCII dot and MUST be parsable as a dot-separated sequence of LDH labels, as
defined in {{!RFC5890}}, Section 2.3.1. Clients MUST ignore any `ECHConfig`
structure whose `public_name` does not meet these criteria. Note that these
criteria are incomplete; they incidentally rule out textual representations of
IPv6 addresses (see {{!RFC3986}}, Section 3.2.2), but do not exclude IPv4
addresses in standard dotted-decimal or other non-standard notations such as
octal and hexadecimal (see {{RFC3986}}, Section 7.4). If `public_name` contains
a literal IPv4 or IPv6 address, the client SHOULD ignore the `ECHConfig` to
avoid sending a non-compliant "server_name" extension on the ClientHelloOuter
(see {{!RFC6066}}, Section 3).

extensions
: A list of extensions that the client must take into consideration when
generating a ClientHello message. These are described below
({{config-extensions}}).

The `HpkeKeyConfig` structure contains the following fields:

config_id
: A one-byte identifier for the given HPKE key configuration. This is used by
clients to indicate the key used for ClientHello encryption.

kem_id
: The HPKE KEM identifier corresponding to `public_key`. Clients MUST ignore any
`ECHConfig` structure with a key using a KEM they do not support.

public_key
: The HPKE public key used by the client to encrypt ClientHelloInner.

cipher_suites
: The list of HPKE KDF and AEAD identifier pairs clients can use for encrypting
ClientHelloInner.

The client-facing server advertises a sequence of ECH configurations to clients,
serialized as follows.

~~~~
    ECHConfig ECHConfigList<1..2^16-1>;
~~~~

The `ECHConfigList` structure contains one or more `ECHConfig` structures in
decreasing order of preference. This allows a server to support multiple
versions of ECH and multiple sets of ECH parameters.

## Configuration Extensions {#config-extensions}

ECH configuration extensions are used to provide room for additional
functionality as needed. See {{config-extensions-guidance}} for guidance on
which types of extensions are appropriate for this structure.

The format is as defined in {{RFC8446}}, Section 4.2.
The same interpretation rules apply: extensions MAY appear in any order, but
there MUST NOT be more than one extension of the same type in the extensions
block. An extension can be tagged as mandatory by using an extension type
codepoint with the high order bit set to 1. A client that receives a mandatory
extension they do not understand MUST reject the `ECHConfig` content.

Clients MUST parse the extension list and check for unsupported mandatory
extensions. If an unsupported mandatory extension is present, clients MUST
ignore the `ECHConfig`.

# The "encrypted_client_hello" Extension {#encrypted-client-hello}

The encrypted ClientHelloInner is carried in an "encrypted_client_hello"
extension, defined as follows:

~~~
    enum {
       encrypted_client_hello(0xfe0a), (65535)
    } ExtensionType;
~~~

When offered by the client, the extension appears only in the ClientHelloOuter.
The payload MUST have the following structure:

~~~~
    struct {
       HpkeSymmetricCipherSuite cipher_suite;
       uint8 config_id;
       opaque enc<1..2^16-1>;
       opaque payload<1..2^16-1>;
    } ClientECH;
~~~~

config_id
: The ECHConfigContents.key_config.config_id for the chosen ECHConfig.

cipher_suite
: The cipher suite used to encrypt ClientHelloInner. This MUST match a value
provided in the corresponding `ECHConfigContents.cipher_suites` list.

enc
: The HPKE encapsulated key, used by servers to decrypt the corresponding
`payload` field.

payload
: The serialized and encrypted ClientHelloInner structure, encrypted using HPKE
as described in {{real-ech}}.

When the client offers the "encrypted_client_hello" extension, the server MAY
include an "encrypted_client_hello" extension in its EncryptedExtensions message
with the following payload:

~~~
    struct {
       ECHConfigList retry_configs;
    } ServerECH;
~~~

retry_configs
: An ECHConfigList structure containing one or more ECHConfig structures, in
decreasing order of preference, to be used by the client in subsequent
connection attempts. These are known as the server's "retry configurations".

This document also defines the "ech_required" alert, which the client MUST send
when it offered an "encrypted_client_hello" extension that was not accepted by
the server. (See {{alerts}}.)

## Encoding the ClientHelloInner {#encoding-inner}

Some TLS 1.3 extensions can be quite large, thus repeating them in the
ClientHelloInner and ClientHelloOuter can lead to an excessive overall size.
One pathological example is "key_share" with post-quantum
algorithms. To reduce the impact of duplicated extensions, the client
may use the "ech_outer_extensions" extension.

~~~
    enum {
       ech_outer_extensions(0xfd00), (65535)
    } ExtensionType;

    ExtensionType OuterExtensions<2..254>;
~~~

OuterExtensions consists of one or more ExtensionType values, each of which
reference an extension in ClientHelloOuter.

When sending ClientHello, the client first computes ClientHelloInner, including
any PSK binders. It then computes a new value, the EncodedClientHelloInner, by
first making a copy of ClientHelloInner. It then replaces the
legacy\_session\_id field with an empty string.

The client then MAY substitute extensions which it knows will be duplicated in
ClientHelloOuter. To do so, the client removes and replaces extensions from
EncodedClientHelloInner with a single "ech_outer_extensions" extension. Removed
extensions MUST be ordered consecutively in ClientHelloInner. The list of outer
extensions, OuterExtensions, includes those which were removed from
EncodedClientHelloInner, in the order in which they were removed.

Finally, EncodedClientHelloInner is serialized as a ClientHello structure,
defined in Section 4.1.2 of {{RFC8446}}. Note this does not include the
four-byte header included in the Handshake structure.

The client-facing server computes ClientHelloInner by reversing this process.
First it makes a copy of EncodedClientHelloInner and copies the
legacy_session_id field from ClientHelloOuter. It then looks for an
"ech_outer_extensions" extension. If found, it replaces the extension with the
corresponding sequence of extensions in the ClientHelloOuter. If any referenced
extensions are missing or if "encrypted_client_hello" appears in the list, the
server MUST abort the connection with an "illegal_parameter" alert.

The "ech_outer_extensions" extension is only used for compressing the
ClientHelloInner. It MUST NOT be sent in either ClientHelloOuter or
ClientHelloInner.

Note that it is possible to implement decoding of the EncodedClientHelloInner in
a way that creates a denial-of-service vulnerability. Specifically, the server
needs to check that each extension in the OuterExtensions list appears in the
ClientHelloOuter. The naive strategy would require O(N\*M) time, where N is the
number of extensions in the ClientHelloOuter and M is the number of extensions
in the OuterExtensions list. Malicious clients could exploit this behavior in
order to cause excessive work for the server, possibly making it unavailable.
This problem can be mitigated by representing OuterExtensions in a way that
allows it to be searched more quickly. For example, the runtime can be improved
to O(N\*log(M)) by sorting the OuterExtensions and using binary search to access
it.

## Authenticating the ClientHelloOuter {#authenticating-outer}

To prevent a network attacker from modifying the reconstructed ClientHelloInner
(see {{flow-clienthello-malleability}}), ECH authenticates ClientHelloOuter by
computing ClientHelloOuterAAD as described below and passing it in as the
associated data for HPKE sealing and opening operations. ClientHelloOuterAAD has
the following structure:

~~~
   struct {
      HpkeSymmetricCipherSuite cipher_suite;
      uint8 config_id;
      opaque enc<1..2^16-1>;
      opaque outer_hello<1..2^24-1>;
   } ClientHelloOuterAAD;
~~~

The first three parameters are equal to, respectively, the
`ClientECH.cipher_suite`, `ClientECH.config_id`, and `ClientECH.enc` fields of
the payload of the "encrypted_client_hello" extension. The last parameter,
`outer_hello`, is computed by serializing ClientHelloOuter with the
"encrypted_client_hello" extension set to the empty string, i.e., the
`extension_data` list has zero length. Note this serialization does not include
the four-byte header included in the Handshake structure.

The decompression process in {{encoding-inner}} forbids
"encrypted_client_hello" in OuterExtensions. This ensures the unauthenticated
portion of ClientHelloOuter is not incorporated into ClientHelloInner.

# Client Behavior {#client-behavior}

Clients that implement the ECH extension behave in one of two ways: either they
offer a real ECH extension, as described in {{real-ech}}; or they send a GREASE
ECH extension, as described in {{grease-ech}}. Clients of the latter type do not
negotiate ECH. Instead, they generate a dummy ECH extension that is ignored by
the server. (See {{dont-stick-out}} for an explanation.) The client offers ECH
if it is in possession of a compatible ECH configuration and sends GREASE ECH
otherwise.

## Offering ECH {#real-ech}

To offer ECH, the client first chooses a suitable ECHConfig from the server's
ECHConfigList. To determine if a given `ECHConfig` is suitable, it checks that
it supports the KEM algorithm identified by `ECHConfig.contents.kem_id`, at
least one KDF/AEAD algorithm identified by `ECHConfig.contents.cipher_suites`,
and the version of ECH indicated by `ECHConfig.contents.version`. Once a
suitable configuration is found, the client selects the cipher suite it will
use for encryption. It MUST NOT choose a cipher suite or version not advertised
by the configuration. If no compatible configuration is found, then the client
SHOULD proceed as described in {{grease-ech}}.

Next, the client constructs the ClientHelloInner message just as it does a
standard ClientHello, with the exception of the following rules:

1. It MUST NOT offer to negotiate TLS 1.2 or below. This is necessary to ensure
   the backend server does not negotiate a TLS version that is incompatible with
   ECH.
1. It MUST NOT offer to resume any session for TLS 1.2 and below.
1. It SHOULD contain TLS padding {{!RFC7685}} as described in {{padding}}.
1. If it intends to compress any extensions (see {{encoding-inner}}), it MUST
   order those extensions consecutively.
1. It MUST include the "ech_is_inner" extension as defined in
   {{is-inner}}. (This requirement is not applicable when the
   "encrypted_client_hello" extension is generated as described in
   {{grease-ech}}.)

The client then constructs EncodedClientHelloInner as described in
{{encoding-inner}}. Finally, it constructs the ClientHelloOuter message just as
it does a standard ClientHello, with the exception of the following rules:

1. It MUST offer to negotiate TLS 1.3 or above.
1. If it compressed any extensions in EncodedClientHelloInner, it MUST copy the
   corresponding extensions from ClientHelloInner.
1. It MUST ensure that all extensions or parameters in ClientHelloInner that
   might change in response to receiving HelloRetryRequest match that in
   ClientHelloOuter. See {{client-hrr}} for more information.
1. It MUST copy the legacy\_session\_id field from ClientHelloInner. This
   allows the server to echo the correct session ID for TLS 1.3's compatibility
   mode (see Appendix D.4 of {{RFC8446}}) when ECH is negotiated.
1. It MAY copy any other field from the ClientHelloInner except
   ClientHelloInner.random. Instead, It MUST generate a fresh
   ClientHelloOuter.random using a secure random number generator. (See
   {{flow-client-reaction}}.)
1. It MUST include an "encrypted_client_hello" extension with a payload
   constructed as described below.
1. The value of `ECHConfig.contents.public_name` MUST be placed in the
   "server_name" extension.
1. When the client offers the "pre_shared_key" extension in ClientHelloInner, it
   SHOULD also include a GREASE "pre_shared_key" extension in ClientHelloOuter,
   generated in the manner described in {{grease-psk}}. The client MUST NOT use
   this extension to advertise a PSK to the client-facing server. (See
   {{flow-clienthello-malleability}}.) When the client includes a GREASE
   "pre_shared_key" extension, it MUST also copy the "psk_key_exchange_modes"
   from the ClientHelloInner into the ClientHelloOuter.
1. When the client offers the "early_data" extension in ClientHelloInner, it
   MUST also include the "early_data" extension in ClientHelloOuter. This
   allows servers that reject ECH and use ClientHelloOuter to safely ignore any
   early data sent by the client per {{RFC8446}}, Section 4.2.10.

[[OPEN ISSUE: We currently require HRR-sensitive parameters to match in
ClientHelloInner and ClientHelloOuter in order to simplify client-side
logic in the event of HRR. See
https://github.com/tlswg/draft-ietf-tls-esni/pull/316
for more information. We might also solve this by including an explicit
signal in HRR noting ECH acceptance. We need to decide if inner/outer
variance is important for HRR-sensitive parameters, and if so, how to
best deal with it without complicated client logic.]]

The client might duplicate non-sensitive extensions in both messages. However,
implementations need to take care to ensure that sensitive extensions are not
offered in the ClientHelloOuter. See {{outer-clienthello}} for additional
guidance.

To encrypt EncodedClientHelloInner, the client first computes
ClientHelloOuterAAD as described in {{authenticating-outer}}. Note this
requires the "encrypted_client_hello" be computed after all other extensions.
In particular, this is possible because the "pre_shared_key" extension is
forbidden in ClientHelloOuter.

The client then generates the HPKE encryption context and computes the
encapsulated key, context, and payload as:

~~~
    pkR = Deserialize(ECHConfig.contents.public_key)
    enc, context = SetupBaseS(pkR,
                              "tls ech" || 0x00 || ECHConfig)
    payload = context.Seal(ClientHelloOuterAAD,
                           EncodedClientHelloInner)
~~~

Note that the HPKE functions Deserialize and SetupBaseS are those which match
`ECHConfig.contents.kem_id` and the AEAD/KDF used with `context` are those which
match the client's chosen preference from `ECHConfig.contents.cipher_suites`.
The `info` parameter to SetupBaseS is the concatenation of "tls ech", a zero
byte, and the serialized ECHConfig.

The value of the "encrypted_client_hello" extension in the ClientHelloOuter is
a `ClientECH` with the following values:

- `config_id`, the identifier corresponding to the chosen ECHConfig structure;
- `cipher_suite`, the client's chosen cipher suite;
- `enc`, as computed above; and
- `payload`, as computed above.

If optional configuration identifiers (see {{optional-configs}})) are used,
`config_id` SHOULD be set to a randomly generated byte. Unless specified by the
application using (D)TLS or externally configured on both sides,
implementations MUST set the field as specified in
{{encrypted-client-hello}}.

### ClientHelloInner Indication Extension {#is-inner}

If, in a ClientHello, the "encrypted_client_hello" extension is not present and
an "ech_is_inner" extension is present, the ClientHello is a
ClientHelloInner. This extension MUST only be sent in the ClientHello message.

~~~
    enum {
       ech_is_inner(0xda09), (65535)
    } ExtensionType;
~~~

The "extension_data" field of the "ech_is_inner" extension is zero
length.

Backend servers (as described in {{server-behavior}}) MUST support the
"ech_is_inner" extension.

### GREASE PSK {#grease-psk}

When offering ECH, the client is not permitted to advertise PSK identities in
the ClientHelloOuter. However, the client can send a "pre_shared_key" extension
in the ClientHelloInner. In this case, when resuming a session with the client,
the backend server sends a "pre_shared_key" extension in its ServerHello. This
would appear to a network observer as if the the server were sending this
extension without solicitation, which would violate the extension rules
described in {{RFC8446}}. Sending a GREASE "pre_shared_key" extension in the
ClientHelloOuter makes it appear to the network as if the extension were
negotiated properly.

The client generates the extension payload by constructing an `OfferedPsks`
structure (see {{RFC8446}}, Section 4.2.11) as follows. For each PSK identity
advertised in the ClientHelloInner, the client generates a random PSK identity
with the same length. It also generates a random, 32-bit, unsigned integer to
use as the `obfuscated_ticket_age`. Likewise, for each inner PSK binder, the
client generates random string of the same length.

If the server replies with a "pre_shared_key" extension in its SeverHello, then
the client MUST abort the handshake with an "illegal_parameter" alert.

### Recommended Padding Scheme {#padding}

This section describes a deterministic padding mechanism based on the following
observation: individual extensions can reveal sensitive information through
their length. Thus, each extension in the inner ClientHello may require
different amounts of padding. This padding may be fully determined by the
client's configuration or may require server input.

By way of example, clients typically support a small number of application
profiles. For instance, a browser might support HTTP with ALPN values
["http/1.1, "h2"] and WebRTC media with ALPNs ["webrtc", "c-webrtc"]. Clients
SHOULD pad this extension by rounding up to the total size of the longest ALPN
extension across all application profiles. The target padding length of most
ClientHello extensions can be computed in this way.

In contrast, clients do not know the longest SNI value in the client-facing
server's anonymity set without server input. For the "server_name" extension
with length D, clients SHOULD use the server's length hint L
(ECHConfig.contents.maximum_name_length) when computing the padding as follows:

1. If L >= D, add L - D bytes of padding. This rounds to the server's
   advertised hint, i.e., ECHConfig.contents.maximum_name_length.
2. Otherwise, let P = 31 - ((D - 1) % 32), and add P bytes of padding, plus an
   additional 32 bytes if D + P < L + 32. This rounds D up to the nearest
   multiple of 32 bytes that permits at least 32 bytes of length ambiguity.

In addition to padding ClientHelloInner, clients and servers will also need to
pad all other handshake messages that have sensitive-length fields. For example,
if a client proposes ALPN values in ClientHelloInner, the server-selected value
will be returned in an EncryptedExtension, so that handshake message also needs
to be padded using TLS record layer padding.

### Handling the Server Response {#handle-server-response}

As described in {{server-behavior}}, the server MAY either accept ECH and use
ClientHelloInner or reject it and use ClientHelloOuter. In handling the server's
response, the client's first step is to determine which value was used. The
client presumes acceptance if the last 8 bytes of ServerHello.random are equal
to the first 8 bytes of `accept_confirmation` as defined in {{backend-server}}.
Otherwise, it presumes rejection.

#### Accepted ECH

If the server used ClientHelloInner, the client proceeds with the connection as
usual, authenticating the connection for the true server name.

#### Rejected ECH

If the server used ClientHelloOuter, the client proceeds with the handshake,
authenticating for ECHConfig.contents.public_name as described in
{{auth-public-name}}. If authentication or the handshake fails, the client MUST
return a failure to the calling application. It MUST NOT use the retry
configurations.

Otherwise, if both authentication and the handshake complete successfully, the
client MUST abort the connection with an "ech_required" alert. It then
processes the "retry_configs" field from the server's "encrypted_client_hello"
extension.

If at least one of the values contains a version supported by the client, it can
regard the ECH keys as securely replaced by the server. It SHOULD retry the
handshake with a new transport connection, using the retry configurations
supplied by the server. The retry configurations may only be applied to the
retry connection. The client MUST continue to use the previously-advertised
configurations for subsequent connections. This avoids introducing pinning
concerns or a tracking vector, should a malicious server present
client-specific retry configurations in order to identify the client in a
subsequent ECH handshake.

If none of the values provided in "retry_configs" contains a supported version,
the client can regard ECH as securely disabled by the server. As below, it
SHOULD then retry the handshake with a new transport connection and ECH
disabled.

If the field contains any other value, the client MUST abort the connection with
an "illegal_parameter" alert.

If the server negotiates an earlier version of TLS, or if it does not provide an
"encrypted_client_hello" extension in EncryptedExtensions, the client proceeds
with the handshake, authenticating for ECHConfig.contents.public_name as
described in {{auth-public-name}}. If an earlier version was negotiated, the
client MUST NOT enable the False Start optimization {{RFC7918}} for this
handshake. If authentication or the handshake fails, the client MUST return a
failure to the calling application. It MUST NOT treat this as a secure signal to
disable ECH.

Otherwise, when the handshake completes successfully with the public name
authenticated, the client MUST abort the connection with an "ech_required"
alert. The client can then regard ECH as securely disabled by the server. It
SHOULD retry the handshake with a new transport connection and ECH disabled.

Clients SHOULD implement a limit on retries caused by "ech_retry_request" or
servers which do not acknowledge the "encrypted_client_hello" extension. If the
client does not retry in either scenario, it MUST report an error to the calling
application.

#### Authenticating for the Public Name {#auth-public-name}

When the server rejects ECH or otherwise ignores "encrypted_client_hello"
extension, it continues with the handshake using the plaintext "server_name"
extension instead (see {{server-behavior}}). Clients that offer ECH then
authenticate the connection with the public name, as follows:

- The client MUST verify that the certificate is valid for
  ECHConfig.contents.public_name. If invalid, it MUST abort the connection with
  the appropriate alert.

- If the server requests a client certificate, the client MUST respond with an
  empty Certificate message, denoting no client certificate.

Note that authenticating a connection for the public name does not authenticate
it for the origin. The TLS implementation MUST NOT report such connections as
successful to the application. It additionally MUST ignore all session tickets
and session IDs presented by the server. These connections are only used to
trigger retries, as described in {{handle-server-response}}. This may be
implemented, for instance, by reporting a failed connection with a dedicated
error code.

### Handling HelloRetryRequest {#client-hrr}

As required in {{real-ech}}, clients offering ECH MUST ensure that all
extensions or parameters that might change in response to receiving a
HelloRetryRequest have the same values in ClientHelloInner and
ClientHelloOuter. That is, if a HelloRetryRequest causes a parameter to be
changed, the same change is applied to both ClientHelloInner and
ClientHelloOuter. Applicable parameters include:

1. TLS 1.3 {{!RFC8446}} ciphersuites in the ClientHello.cipher_suites list.
1. The "key_share" and "supported_groups" extensions {{RFC8446}}. (These
extensions may be copied from ClientHelloOuter into ClientHelloInner as
described in {{real-ech}}.)
1. Versions in the "supported_versions" extension, excluding TLS 1.2 and
earlier. Note the ClientHelloOuter MAY include these older versions, while the
ClientHelloInner MUST omit them.

Future extensions that might change across first and second ClientHello messages
in response to a HelloRetryRequest MUST have the same value.

If the server sends a HelloRetryRequest in response to the ClientHello, the
client sends a second updated ClientHello per the rules in {{RFC8446}}.
However, at this point, the client does not know whether the server processed
ClientHelloOuter or ClientHelloInner, and MUST regenerate both values to be
acceptable. Note: if ClientHelloOuter and ClientHelloInner use different groups
for their key shares or differ in some other way, then the HelloRetryRequest
may actually be invalid for one or the other ClientHello, in which case a fresh
ClientHello MUST be generated, ignoring the instructions in HelloRetryRequest.
Otherwise, the usual rules for HelloRetryRequest processing apply.

The client encodes the second ClientHelloInner as in {{encoding-inner}}, using
the second ClientHelloOuter for any referenced extensions. It then encrypts
the new EncodedClientHelloInner value as a second message with the previous
HPKE context:

~~~
    payload = context.Seal(ClientHelloOuterAAD,
                           EncodedClientHelloInner)
~~~

ClientHelloOuterAAD is computed as described in {{authenticating-outer}}, but
again using the second ClientHelloOuter. Note that the HPKE context maintains a
sequence number, so this operation internally uses a fresh nonce for each AEAD
operation. Reusing the HPKE context avoids an attack described in
{{flow-hrr-hijack}}.

The client then modifies the "encrypted_client_hello" extension in
ClientHelloOuter as follows:

- `config_id` is unchanged and contains the `config_id` corresponding to
  the client's chosen ECHConfig.
- `cipher_suite` is unchanged and contains the client's chosen HPKE cipher
  suite.
- `enc` is replaced with the empty string.
- `payload` is replaced with the value computed above.

If the client offered ECH in the first ClientHello, then it MUST offer ECH in
the second. Likewise, if the client did not offer ECH in the first ClientHello,
then it MUST NOT not offer ECH in the second.

## GREASE ECH {#grease-ech}

If the client attempts to connect to a server and does not have an ECHConfig
structure available for the server, it SHOULD send a GREASE {{?RFC8701}}
"encrypted_client_hello" extension in the first ClientHello as follows:

- Set the `config_id` field to a random byte.

- Set the `cipher_suite` field to a supported HpkeSymmetricCipherSuite. The
  selection SHOULD vary to exercise all supported configurations, but MAY be
  held constant for successive connections to the same server in the same
  session.

- Set the `enc` field to a randomly-generated valid encapsulated public key
  output by the HPKE KEM.

- Set the `payload` field to a randomly-generated string of L+C bytes, where C
  is the ciphertext expansion of the selected AEAD scheme and L is the size of
  the EncodedClientHelloInner the client would compute when offering ECH, padded
  according to {{padding}}.

When sending a second ClientHello in response to a HelloRetryRequest, the
client copies the entire "encrypted_client_hello" extension from the first
ClientHello.

[[OPEN ISSUE: The above doesn't match HRR handling for either ECH acceptance or
rejection. See issue https://github.com/tlswg/draft-ietf-tls-esni/issues/358.]]

If the server sends an "encrypted_client_hello" extension, the client MUST check
the extension syntactically and abort the connection with a "decode_error" alert
if it is invalid. It otherwise ignores the extension and MUST NOT use the retry
keys.

[[OPEN ISSUE: if the client sends a GREASE "encrypted_client_hello" extension,
should it also send a GREASE "pre_shared_key" extension? If not, GREASE+ticket
is a trivial distinguisher. See issue #384.]]

Offering a GREASE extension is not considered offering an encrypted ClientHello
for purposes of requirements in {{client-behavior}}. In particular, the client
MAY offer to resume sessions established without ECH.

# Server Behavior {#server-behavior}

Servers that support ECH play one of two roles, depending on which of the
"ech_is_inner" ({{is-inner}}) and "encrypted_client_hello"
({{encrypted-client-hello}}) extensions are present in the ClientHello:

* If both the "ech_is_inner" and "encrypted_client_hello" extensions are
  present in the ClientHello, the backend server MUST abort with an
  "illegal_parameter" alert.

* If only the "encrypted_client_hello" extension is present, the server acts as
  a client-facing server and proceeds as described in {{client-facing-server}}
  to extract a ClientHelloInner, if available.

* If only the "ech_is_inner" extension is present and the
  "encrypted_client_hello" extension is not present, the server acts as a
  backend server and proceeds as described in {{backend-server}}.

* If neither extension is present, the server completes the handshake normally,
  as described in {{RFC8446}}.

## Client-Facing Server {#client-facing-server}

Upon receiving an "encrypted_client_hello" extension in an initial
ClientHello, the client-facing server determines if it will accept ECH, prior
to negotiating any other TLS parameters. Note that successfully decrypting the
extension will result in a new ClientHello to process, so even the client's TLS
version preferences may have changed.

If the client offers the "ech_is_inner" extension ({{is-inner}})
in addition to the "encrypted_client_hello" extension, the server MUST abort
with an "illegal_parameter" alert.

First, the server collects a set of candidate ECHConfig values. This list is
determined by one of the two following methods:

1. Compare ClientECH.config_id against identifiers of each known ECHConfig
   and select the ones that match, if any, as candidates.
2. Collect all known ECHConfig values as candidates, with trial decryption
   below determining the final selection.

Some uses of ECH, such as local discovery mode, may randomize the
ClientECH.config_id since it can be used as a tracking vector. In such cases,
the second method should be used for matching ClientECH to known ECHConfig. See
{{optional-configs}}. Unless specified by the application using (D)TLS or
externally configured on both sides, implementations MUST use the first method.

The server then iterates over the candidate ECHConfig values, attempting to
decrypt the "encrypted_client_hello" extension:

The server verifies that the ECHConfig supports the cipher suite indicated by
the ClientECH.cipher_suite and that the version of ECH indicated by the client
matches the ECHConfig.version. If not, the server continues to the next
candidate ECHConfig.

Next, the server decrypts ClientECH.payload, using the private key skR
corresponding to ECHConfig, as follows:

~~~
    context = SetupBaseR(ClientECH.enc, skR,
                         "tls ech" || 0x00 || ECHConfig)
    EncodedClientHelloInner = context.Open(ClientHelloOuterAAD,
                                           ClientECH.payload)
~~~

ClientHelloOuterAAD is computed from ClientHelloOuter as described in
{{authenticating-outer}}. The `info` parameter to SetupBaseS is the
concatenation "tls ech", a zero byte, and the serialized ECHConfig. If
decryption fails, the server continues to the next candidate ECHConfig.
Otherwise, the server reconstructs ClientHelloInner from
EncodedClientHelloInner, as described in {{encoding-inner}}. It then stops
iterating over the candidate ECHConfig values.

Upon determining the ClientHelloInner, the client-facing server then checks
that the message includes the "ech_is_inner" extension, omits the
"encrypted_client_hello" extension, and does not offer TLS 1.2 or below
versions. If any of these checks fails, the client-facing server MUST
abort with an "illegal_parameter" alert.

If these checks succeed, the client-facing server then forwards the
ClientHelloInner to the appropriate backend server, which proceeds as in
{{backend-server}}. If the backend server responds with a HelloRetryRequest,
the client-facing server forwards it, decrypts the client's second
ClientHelloOuter using the procedure in {{server-hrr}}, and forwards the
resulting second ClientHelloInner. The client-facing server forwards all other
TLS messages between the client and backend server unmodified.

Otherwise, if all candidate ECHConfig values fail to decrypt the extension, the
client-facing server MUST ignore the extension and proceed with the connection
using ClientHelloOuter. This connection proceeds as usual, except the server
MUST include the "encrypted_client_hello" extension in its EncryptedExtensions
with the "retry_configs" field set to one or more ECHConfig structures with
up-to-date keys. Servers MAY supply multiple ECHConfig values of different
versions. This allows a server to support multiple versions at once.

Note that decryption failure could indicate a GREASE ECH extension (see
{{grease-ech}}), so it is necessary for servers to proceed with the connection
and rely on the client to abort if ECH was required. In particular, the
unrecognized value alone does not indicate a misconfigured ECH advertisement
({{misconfiguration}}). Instead, servers can measure occurrences of the
"ech_required" alert to detect this case.

### Handling HelloRetryRequest {#server-hrr}

After sending or forwarding a HelloRetryRequest, the client-facing server does
not repeat the steps in {{client-facing-server}} with the second
ClientHelloOuter. Instead, it continues with the ECHConfig selection from the
first ClientHelloOuter as follows:

If the client-facing server accepted ECH, it checks the second ClientHelloOuter
also contains the "encrypted_client_hello" extension. If not, it MUST abort the
handshake with a "missing_extension" alert. Otherwise, it checks that
ClientECH.cipher_suite and ClientECH.config_id are unchanged, and that
ClientECH.enc is empty. If not, it MUST abort the handshake with an
"illegal_parameter" alert.

Finally, it decrypts the new ClientECH.payload as a second message with the
previous HPKE context:

~~~
    EncodedClientHelloInner = context.Open(ClientHelloOuterAAD,
                                           ClientECH.payload)
~~~

ClientHelloOuterAAD is computed as described in {{authenticating-outer}}, but
using the second ClientHelloOuter. If decryption fails, the client-facing
server MUST abort the handshake with a "decrypt_error" alert. Otherwise, it
reconstructs the second ClientHelloInner from the new EncodedClientHelloInner
as described in {{encoding-inner}}, using the second ClientHelloOuter for
any referenced extensions.

The client-facing server then forwards the resulting ClientHelloInner to the
backend server. It forwards all subsequent TLS messages between the client and
backend server unmodified.

If the client-facing server rejected ECH, or if the first ClientHello did not
include an "encrypted_client_hello" extension, the client-facing server
proceeds with the connection as usual. The server does not decrypt the
second ClientHello's ClientECH.payload value, if there is one.

[[OPEN ISSUE: If the client-facing server implements stateless HRR, it has no
way to send a cookie, short of as-yet-unspecified integration with the
backend server. Stateful HRR on the client-facing server works fine, however.
See issue https://github.com/tlswg/draft-ietf-tls-esni/issues/333.]]

## Backend Server {#backend-server}

Upon receipt of an "ech_is_inner" extension in a ClientHello, if the backend
server negotiates TLS 1.3 or higher, then it MUST confirm ECH acceptance to the
client by computing its ServerHello as described here.

The backend server begins by generating a message ServerHelloECHConf, which is
identical in content to a ServerHello message with the exception that
ServerHelloECHConf.random is equal to 24 random bytes followed by 8 zero bytes.
It then computes an 8-byte string

~~~
   accept_confirmation = HKDF-Expand-Label(0,
      "ech accept confirmation",
      Transcript-Hash(ClientHelloInner...ServerHelloECHConf),
      8)
~~~

where HKDF-Expand-Label and Transcript-Hash are as defined in {{RFC8446}},
Section 7.1, "0" indicates a string of Hash.length bytes set to zero, and
ClientHelloInner...ServerHelloECHConf refers to the sequence of handshake
messages beginning with the first ClientHelloInner and ending with
ServerHelloECHConf. (Note that ClientHelloInner and ServerHelloECHConf
messages replace the ClientHello and ServerHello messages in the transcript
hash sequence as specified in Section 4.1.1 of {{RFC8446}}. Finally, the
backend server constructs its ServerHello message so that it is equal to
ServerHelloECHConf but with the last 8 bytes of ServerHello.random set to
`accept_confirmation`.

The backend server MUST NOT perform this operation if it negotiated TLS 1.2 or
below. Note that doing so would overwrite the downgrade signal for TLS 1.3 (see
{{RFC8446}}, Section 4.1.3).

The "ech_is_inner" is expected to have an empty payload. If the payload is
non-empty (i.e., the length of the "extension_data" field is non-zero) then the
backend server MUST abort the handshake with an "illegal_parameter" alert.

# Compatibility Issues

Unlike most TLS extensions, placing the SNI value in an ECH extension is not
interoperable with existing servers, which expect the value in the existing
plaintext extension. Thus server operators SHOULD ensure servers understand a
given set of ECH keys before advertising them. Additionally, servers SHOULD
retain support for any previously-advertised keys for the duration of their
validity

However, in more complex deployment scenarios, this may be difficult to fully
guarantee. Thus this protocol was designed to be robust in case of
inconsistencies between systems that advertise ECH keys and servers, at the cost
of extra round-trips due to a retry. Two specific scenarios are detailed below.

## Misconfiguration and Deployment Concerns {#misconfiguration}

It is possible for ECH advertisements and servers to become inconsistent. This
may occur, for instance, from DNS misconfiguration, caching issues, or an
incomplete rollout in a multi-server deployment. This may also occur if a server
loses its ECH keys, or if a deployment of ECH must be rolled back on the server.

The retry mechanism repairs inconsistencies, provided the server is
authoritative for the public name. If server and advertised keys mismatch, the
server will respond with ech_retry_requested. If the server does not understand
the "encrypted_client_hello" extension at all, it will ignore it as required by
{{RFC8446}}; Section 4.1.2. Provided the server can present a certificate valid
for the public name, the client can safely retry with updated settings, as
described in {{handle-server-response}}.

Unless ECH is disabled as a result of successfully establishing a connection to
the public name, the client MUST NOT fall back to using unencrypted
ClientHellos, as this allows a network attacker to disclose the contents of this
ClientHello, including the SNI. It MAY attempt to use another server from the
DNS results, if one is provided.

## Middleboxes

A more serious problem is MITM proxies which do not support this extension.
{{RFC8446}}, Section 9.3 requires that such proxies remove any extensions they
do not understand. The handshake will then present a certificate based on the
public name, without echoing the "encrypted_client_hello" extension to the
client.

Depending on whether the client is configured to accept the proxy's certificate
as authoritative for the public name, this may trigger the retry logic described
in {{handle-server-response}} or result in a connection failure. A proxy which
is not authoritative for the public name cannot forge a signal to disable ECH.

A non-conformant MITM proxy which instead forwards the ECH extension,
substituting its own KeyShare value, will result in the client-facing server
recognizing the key, but failing to decrypt the SNI. This causes a hard failure.
Clients SHOULD NOT attempt to repair the connection in this case.

# Compliance Requirements {#compliance}

In the absence of an application profile standard specifying otherwise,
a compliant ECH application MUST implement the following HPKE cipher suite:

- KEM: DHKEM(X25519, HKDF-SHA256) (see {{!I-D.irtf-cfrg-hpke}}, Section 7.1)
- KDF: HKDF-SHA256 (see {{!I-D.irtf-cfrg-hpke}}, Section 7.2)
- AEAD: AES-128-GCM (see {{!I-D.irtf-cfrg-hpke}}, Section 7.3)

# Security Considerations

## Security and Privacy Goals {#goals}

ECH considers two types of attackers: passive and active. Passive attackers can
read packets from the network, but they cannot perform any sort of active
behavior such as probing servers or querying DNS. A middlebox that filters based
on plaintext packet contents is one example of a passive attacker. In contrast,
active attackers can also write packets into the network for malicious purposes,
such as interfering with existing connections, probing servers, and querying
DNS. In short, an active attacker corresponds to the conventional threat model
for TLS 1.3 {{RFC8446}}.

Given these types of attackers, the primary goals of ECH are as follows.

1. Use of ECH does not weaken the security properties of TLS without ECH.
1. TLS connection establishment to a host with a specific ECHConfig and TLS
   configuration is indistinguishable from a connection to any other host with
   the same ECHConfig and TLS configuration. (The set of hosts which share the
   same ECHConfig and TLS configuration is referred to as the anonymity set.)

Client-facing server configuration determines the size of the anonymity set. For
example, if a client-facing server uses distinct ECHConfig values for each host,
then each anonymity set has size k = 1. Client-facing servers SHOULD deploy ECH
in such a way so as to maximize the size of the anonymity set where possible.
This means client-facing servers should use the same ECHConfig for as many hosts
as possible. An attacker can distinguish two hosts that have different ECHConfig
values based on the ClientECH.config_id value. This also means public
information in a TLS handshake is also consistent across hosts. For example, if
a client-facing server services many backend origin hosts, only one of which
supports some cipher suite, it may be possible to identify that host based on
the contents of unencrypted handshake messages.

Beyond these primary security and privacy goals, ECH also aims to hide, to some
extent, the fact that it is being used at all. Specifically, the GREASE ECH
extension described in {{grease-ech}} does not change the security properties of
the TLS handshake at all. Its goal is to provide "cover" for the real ECH
protocol ({{real-ech}}), as a means of addressing the "do not stick out"
requirements of {{?RFC8744}}. See {{dont-stick-out}} for details.

## Unauthenticated and Plaintext DNS {#plaintext-dns}

In comparison to {{?I-D.kazuho-protected-sni}}, wherein DNS Resource Records are
signed via a server private key, ECH records have no authenticity or provenance
information. This means that any attacker which can inject DNS responses or
poison DNS caches, which is a common scenario in client access networks, can
supply clients with fake ECH records (so that the client encrypts data to them)
or strip the ECH record from the response. However, in the face of an attacker
that controls DNS, no encryption scheme can work because the attacker can
replace the IP address, thus blocking client connections, or substituting a
unique IP address which is 1:1 with the DNS name that was looked up (modulo DNS
wildcards). Thus, allowing the ECH records in the clear does not make the
situation significantly worse.

Clearly, DNSSEC (if the client validates and hard fails) is a defense against
this form of attack, but DoH/DPRIVE are also defenses against DNS attacks by
attackers on the local network, which is a common case where ClientHello and SNI
encryption are desired. Moreover, as noted in the introduction, SNI encryption
is less useful without encryption of DNS queries in transit via DoH or DPRIVE
mechanisms.

## Client Tracking

A malicious client-facing server could distribute unique, per-client ECHConfig
structures as a way of tracking clients across subsequent connections. On-path
adversaries which know about these unique keys could also track clients in this
way by observing TLS connection attempts.

The cost of this type of attack scales linearly with the desired number of
target clients. Moreover, DNS caching behavior makes targeting individual users
for extended periods of time, e.g., using per-client ECHConfig structures
delivered via HTTPS RRs with high TTLs, challenging. Clients can help mitigate
this problem by flushing any DNS or ECHConfig state upon changing networks.

## Optional Configuration Identifiers and Trial Decryption {#optional-configs}

Optional configuration identifiers may be useful in scenarios where clients and
client-facing servers do not want to reveal information about the client-facing
server in the "encrypted_client_hello" extension. In such settings, clients
send a randomly generated config_id in the ClientECH. Servers in these settings
must perform trial decryption since they cannot identify the client's chosen
ECH key using the config_id value. As a result, support for optional
configuration identifiers may exacerbate DoS attacks. Specifically, an
adversary may send malicious ClientHello messages, i.e., those which will not
decrypt with any known ECH key, in order to force wasteful decryption. Servers
that support this feature should, for example, implement some form of rate
limiting mechanism to limit the damage caused by such attacks.

## Outer ClientHello {#outer-clienthello}

Any information that the client includes in the ClientHelloOuter is visible to
passive observers. The client SHOULD NOT send values in the ClientHelloOuter
which would reveal a sensitive ClientHelloInner property, such as the true
server name. It MAY send values associated with the public name in the
ClientHelloOuter.

In particular, some extensions require the client send a server-name-specific
value in the ClientHello. These values may reveal information about the
true server name. For example, the "cached_info" ClientHello extension
{{?RFC7924}} can contain the hash of a previously observed server certificate.
The client SHOULD NOT send values associated with the true server name in the
ClientHelloOuter. It MAY send such values in the ClientHelloInner.

A client may also use different preferences in different contexts. For example,
it may send a different ALPN lists to different servers or in different
application contexts. A client that treats this context as sensitive SHOULD NOT
send context-specific values in ClientHelloOuter.

Values which are independent of the true server name, or other information the
client wishes to protect, MAY be included in ClientHelloOuter. If they match
the corresponding ClientHelloInner, they MAY be compressed as described in
{{encoding-inner}}. However, note the payload length reveals information about
which extensions are compressed, so inner extensions which only sometimes match
the corresponding outer extension SHOULD NOT be compressed.

Clients MAY include additional extensions in ClientHelloOuter to avoid
signaling unusual behavior to passive observers, provided the choice of value
and value itself are not sensitive. See {{dont-stick-out}}.

## Related Privacy Leaks

ECH requires encrypted DNS to be an effective privacy protection mechanism.
However, verifying the server's identity from the Certificate message,
particularly when using the X509 CertificateType, may result in additional
network traffic that may reveal the server identity. Examples of this traffic
may include requests for revocation information, such as OCSP or CRL traffic, or
requests for repository information, such as authorityInformationAccess. It may
also include implementation-specific traffic for additional information sources
as part of verification.

Implementations SHOULD avoid leaking information that may identify the server.
Even when sent over an encrypted transport, such requests may result in indirect
exposure of the server's identity, such as indicating a specific CA or service
being used. To mitigate this risk, servers SHOULD deliver such information
in-band when possible, such as through the use of OCSP stapling, and clients
SHOULD take steps to minimize or protect such requests during certificate
validation.

Attacks that rely on non-ECH traffic to infer server identity in an ECH
connection are out of scope for this document. For example, a client that
connects to a particular host prior to ECH deployment may later resume a
connection to that same host after ECH deployment. An adversary that observes
this can deduce that the ECH-enabled connection was made to a host that the
client previously connected to and which is within the same anonymity set.

## Attacks Exploiting Acceptance Confirmation

To signal acceptance, the backend server overwrites 8 bytes of its
ServerHello.random with a value derived from the ClientHelloInner.random. (See
{{backend-server}} for details.) This behavior increases the likelihood of the
ServerHello.random colliding with the ServerHello.random of a previous session,
potentially reducing the overall security of the protocol. However, the
remaining 24 bytes provide enough entropy to ensure this is not a practical
avenue of attack.

On the other hand, the probability that two 8-byte strings are the same is
non-negligible. This poses a modest operational risk. Suppose the client-facing
server terminates the connection (i.e., ECH is rejected or bypassed): if the
last 8 bytes of its ServerHello.random coincide with the confirmation signal,
then the client will incorrectly presume acceptance and proceed as if the
backend server terminated the connection. However, the probability of a false
positive occurring for a given connection is only 1 in 2^64. This value is
smaller than the probability of network connection failures in practice.

Note that the same bytes of the ServerHello.random are used to implement
downgrade protection for TLS 1.3 (see {{RFC8446}}, Section 4.1.3). These
mechanisms do not interfere because the backend server only signals ECH
acceptance in TLS 1.3 or higher.

## Comparison Against Criteria

{{?RFC8744}} lists several requirements for SNI encryption.
In this section, we re-iterate these requirements and assess the ECH design
against them.

### Mitigate Cut-and-Paste Attacks

Since servers process either ClientHelloInner or ClientHelloOuter, and because
ClientHelloInner.random is encrypted, it is not possible for an attacker to "cut
and paste" the ECH value in a different Client Hello and learn information from
ClientHelloInner.

### Avoid Widely Shared Secrets

This design depends upon DNS as a vehicle for semi-static public key
distribution. Server operators may partition their private keys however they
see fit provided each server behind an IP address has the corresponding private
key to decrypt a key. Thus, when one ECH key is provided, sharing is optimally
bound by the number of hosts that share an IP address. Server operators may
further limit sharing by publishing different DNS records containing ECHConfig
values with different keys using a short TTL.

### Prevent SNI-Based Denial-of-Service Attacks

This design requires servers to decrypt ClientHello messages with ClientECH
extensions carrying valid digests. Thus, it is possible for an attacker to force
decryption operations on the server. This attack is bound by the number of valid
TCP connections an attacker can open.

### Do Not Stick Out {#dont-stick-out}

As a means of reducing the impact of network ossification, {{?RFC8744}}
recommends SNI-protection mechanisms be designed in such a way that network
operators do not differentiate connections using the mechanism from connections
not using the mechanism. To that end, ECH is designed to resemble a standard
TLS handshake as much as possible. The most obvious difference is the extension
itself: as long as middleboxes ignore it, as required by {{!RFC8446}}, the rest
of the handshake is designed to look very much as usual.

The GREASE ECH protocol described in {{grease-ech}} provides a low-risk way to
evaluate the deployability of ECH. It is designed to mimic the real ECH protocol
({{real-ech}}) without changing the security properties of the handshake. The
underlying theory is that if GREASE ECH is deployable without triggering
middlebox misbehavior, and real ECH looks enough like GREASE ECH, then ECH
should be deployable as well. Thus, our strategy for mitigating network
ossification is to deploy GREASE ECH widely enough to disincentivize
differential treatment of the real ECH protocol by the network.

Ensuring that networks do not differentiate between real ECH and GREASE ECH may
not be feasible for all implementations. While most middleboxes will not treat
them differently, some operators may wish to block real ECH usage but allow
GREASE ECH. This specification aims to provide a baseline security level that
most deployments can achieve easily, while providing implementations enough
flexibility to achieve stronger security where possible. Minimally, real ECH is
designed to be indifferentiable from GREASE ECH for passive adversaries with
following capabilities:
1. The attacker does not know the ECHConfigList used by the server.
1. The attacker keeps per-connection state only. In particular, it does not
   track endpoints across connections.
1. ECH and GREASE ECH are designed so that the following features do not vary:
   the code points of extensions negotiated in the clear; the length of
   messages; and the values of plaintext alert messages.

This leaves a variety of practical differentiators out-of-scope. including,
though not limited to, the following:
1. the value of the configuration identifier;
1. the value of the outer SNI;
1. use of the "pre_shared_key" extension in the ClientHelloOuter, which is
   permitted in GREASE ECH but not real ECH; [[TODO: Remove this differentiator
   if issue #384 is resolved by a spec change.]]
1. the TLS version negotiated, which may depend on ECH acceptance;
1. client authentication, which may depend on ECH acceptance; and
1. HRR issuance, which may depend on ECH acceptance.

These can be addressed with more sophisticated implementations, but some
mitigations require coordination between the client and server. These
mitigations are out-of-scope for this specification.

### Maintain Forward Secrecy

This design is not forward secret because the server's ECH key is static.
However, the window of exposure is bound by the key lifetime. It is RECOMMENDED
that servers rotate keys frequently.

### Enable Multi-party Security Contexts

This design permits servers operating in Split Mode to forward connections
directly to backend origin servers. The client authenticates the identity of
the backend origin server, thereby avoiding unnecessary MiTM attacks.

Conversely, assuming ECH records retrieved from DNS are authenticated, e.g.,
via DNSSEC or fetched from a trusted Recursive Resolver, spoofing a
client-facing server operating in Split Mode is not possible. See
{{plaintext-dns}} for more details regarding plaintext DNS.

Authenticating the ECHConfig structure naturally authenticates the included
public name. This also authenticates any retry signals from the client-facing
server because the client validates the server certificate against the public
name before retrying.

### Support Multiple Protocols

This design has no impact on application layer protocol negotiation. It may
affect connection routing, server certificate selection, and client certificate
verification. Thus, it is compatible with multiple application and transport
protocols. By encrypting the entire ClientHello, this design additionally
supports encrypting the ALPN extension.

## Padding Policy

Variations in the length of the ClientHelloInner ciphertext could leak
information about the corresponding plaintext. {{padding}} describes a
RECOMMENDED padding mechanism for clients aimed at reducing potential
information leakage.

## Active Attack Mitigations

This section describes the rationale for ECH properties and mechanics as
defenses against active attacks. In all the attacks below, the attacker is
on-path between the target client and server. The goal of the attacker is to
learn private information about the inner ClientHello, such as the true SNI
value.

### Client Reaction Attack Mitigation {#flow-client-reaction}

This attack uses the client's reaction to an incorrect certificate as an oracle.
The attacker intercepts a legitimate ClientHello and replies with a ServerHello,
Certificate, CertificateVerify, and Finished messages, wherein the Certificate
message contains a "test" certificate for the domain name it wishes to query. If
the client decrypted the Certificate and failed verification (or leaked
information about its verification process by a timing side channel), the
attacker learns that its test certificate name was incorrect. As an example,
suppose the client's SNI value in its inner ClientHello is "example.com," and
the attacker replied with a Certificate for "test.com". If the client produces a
verification failure alert because of the mismatch faster than it would due to
the Certificate signature validation, information about the name leaks. Note
that the attacker can also withhold the CertificateVerify message. In that
scenario, a client which first verifies the Certificate would then respond
similarly and leak the same information.

~~~
 Client                         Attacker               Server
   ClientHello
   + key_share
   + ech         ------>      (intercept)     -----> X (drop)

                             ServerHello
                             + key_share
                   {EncryptedExtensions}
                   {CertificateRequest*}
                          {Certificate*}
                    {CertificateVerify*}
                 <------
   Alert
                 ------>
~~~
{: #flow-diagram-client-reaction title="Client reaction attack"}

ClientHelloInner.random prevents this attack. In particular, since the attacker
does not have access to this value, it cannot produce the right transcript and
handshake keys needed for encrypting the Certificate message. Thus, the client
will fail to decrypt the Certificate and abort the connection.

### HelloRetryRequest Hijack Mitigation {#flow-hrr-hijack}

This attack aims to exploit server HRR state management to recover information
about a legitimate ClientHello using its own attacker-controlled ClientHello.
To begin, the attacker intercepts and forwards a legitimate ClientHello with an
"encrypted_client_hello" (ech) extension to the server, which triggers a
legitimate HelloRetryRequest in return. Rather than forward the retry to the
client, the attacker, attempts to generate its own ClientHello in response based
on the contents of the first ClientHello and HelloRetryRequest exchange with the
result that the server encrypts the Certificate to the attacker. If the server
used the SNI from the first ClientHello and the key share from the second
(attacker-controlled) ClientHello, the Certificate produced would leak the
client's chosen SNI to the attacker.

~~~
 Client                         Attacker                   Server
   ClientHello
   + key_share
   + ech         ------>       (forward)        ------->
                                              HelloRetryRequest
                                                    + key_share
                              (intercept)       <-------

                              ClientHello
                              + key_share'
                              + ech'           ------->
                                                    ServerHello
                                                    + key_share
                                          {EncryptedExtensions}
                                          {CertificateRequest*}
                                                 {Certificate*}
                                           {CertificateVerify*}
                                                     {Finished}
                                                <-------
                         (process server flight)
~~~
{: #flow-diagram-hrr-hijack title="HelloRetryRequest hijack attack"}

This attack is mitigated by using the same HPKE context for both ClientHello
messages. The attacker does not possess the context's keys, so it cannot
generate a valid encryption of the second inner ClientHello.

If the attacker could manipulate the second ClientHello, it might be possible
for the server to act as an oracle if it required parameters from the first
ClientHello to match that of the second ClientHello. For example, imagine the
client's original SNI value in the inner ClientHello is "example.com", and the
attacker's hijacked SNI value in its inner ClientHello is "test.com". A server
which checks these for equality and changes behavior based on the result can be
used as an oracle to learn the client's SNI.

### ClientHello Malleability Mitigation {#flow-clienthello-malleability}

This attack aims to leak information about secret parts of the encrypted
ClientHello by adding attacker-controlled parameters and observing the server's
response. In particular, the compression mechanism described in
{{encoding-inner}} references parts of a potentially attacker-controlled
ClientHelloOuter to construct ClientHelloInner, or a buggy server may
incorrectly apply parameters from ClientHelloOuter to the handshake.

To begin, the attacker first interacts with a server to obtain a resumption
ticket for a given test domain, such as "example.com". Later, upon receipt of a
ClientHelloOuter, it modifies it such that the server will process the
resumption ticket with ClientHelloInner. If the server only accepts resumption
PSKs that match the server name, it will fail the PSK binder check with an
alert when ClientHelloInner is for "example.com" but silently ignore the PSK
and continue when ClientHelloInner is for any other name. This introduces an
oracle for testing encrypted SNI values.

~~~
      Client              Attacker                       Server

                                    handshake and ticket
                                       for "example.com"
                                       <-------->

      ClientHello
      + key_share
      + ech
      + ech_outer_extensions(pre_shared_key)
      + pre_shared_key
                  -------->
                        (intercept)
                        ClientHello
                        + key_share
                        + ech
                           + ech_outer_extensions(pre_shared_key)
                        + pre_shared_key'
                                          -------->
                                                         Alert
                                                         -or-
                                                   ServerHello
                                                            ...
                                                      Finished
                                          <--------
~~~
{: #tls-clienthello-malleability title="Message flow for malleable ClientHello"}

This attack may be generalized to any parameter which the server varies by
server name, such as ALPN preferences.

ECH mitigates this attack by only negotiating TLS parameters from
ClientHelloInner and authenticating all inputs to the ClientHelloInner
(EncodedClientHelloInner and ClientHelloOuter) with the HPKE AEAD. See
{{authenticating-outer}}. An earlier iteration of this specification only
encrypted and authenticated the "server_name" extension, which left the overall
ClientHello vulnerable to an analogue of this attack.

# IANA Considerations

## Update of the TLS ExtensionType Registry

IANA is requested to create the following three entries in the existing registry
for ExtensionType (defined in {{!RFC8446}}):

1. encrypted_client_hello(0xfe0a), with "TLS 1.3" column values set to
   "CH, EE", and "Recommended" column set to "Yes".
2. ech_is_inner (0xda09), with "TLS 1.3" column values set to
   "CH", and "Recommended" column set to "Yes".
3. ech_outer_extensions(0xfd00), with the "TLS 1.3" column values set to "",
   and "Recommended" column set to "Yes".

## Update of the TLS Alert Registry {#alerts}

IANA is requested to create an entry, ech_required(121) in the existing registry
for Alerts (defined in {{!RFC8446}}), with the "DTLS-OK" column set to
"Y".

# ECHConfig Extension Guidance {#config-extensions-guidance}

Any future information or hints that influence ClientHelloOuter SHOULD be
specified as ECHConfig extensions. This is primarily because the outer
ClientHello exists only in support of ECH. Namely, it is both an envelope for
the encrypted inner ClientHello and enabler for authenticated key mismatch
signals (see {{server-behavior}}). In contrast, the inner ClientHello is the
true ClientHello used upon ECH negotiation.

--- back


# Alternative SNI Protection Designs

Alternative approaches to encrypted SNI may be implemented at the TLS or
application layer. In this section we describe several alternatives and discuss
drawbacks in comparison to the design in this document.

## TLS-layer

### TLS in Early Data

In this variant, TLS Client Hellos are tunneled within early data payloads
belonging to outer TLS connections established with the client-facing server.
This requires clients to have established a previous session -— and obtained
PSKs —- with the server. The client-facing server decrypts early data payloads
to uncover Client Hellos destined for the backend server, and forwards them
onwards as necessary. Afterwards, all records to and from backend servers are
forwarded by the client-facing server -- unmodified. This avoids double
encryption of TLS records.

Problems with this approach are: (1) servers may not always be able to
distinguish inner Client Hellos from legitimate application data, (2) nested
0-RTT data may not function correctly, (3) 0-RTT data may not be supported --
especially under DoS -- leading to availability concerns, and (4) clients must
bootstrap tunnels (sessions), costing an additional round trip and potentially
revealing the SNI during the initial connection. In contrast, encrypted SNI
protects the SNI in a distinct Client Hello extension and neither abuses early
data nor requires a bootstrapping connection.

### Combined Tickets

In this variant, client-facing and backend servers coordinate to produce
"combined tickets" that are consumable by both. Clients offer combined tickets
to client-facing servers. The latter parse them to determine the correct backend
server to which the Client Hello should be forwarded. This approach is
problematic due to non-trivial coordination between client-facing and backend
servers for ticket construction and consumption. Moreover, it requires a
bootstrapping step similar to that of the previous variant. In contrast,
encrypted SNI requires no such coordination.

## Application-layer

### HTTP/2 CERTIFICATE Frames

In this variant, clients request secondary certificates with CERTIFICATE_REQUEST
HTTP/2 frames after TLS connection completion. In response, servers supply
certificates via TLS exported authenticators
{{!I-D.ietf-tls-exported-authenticator}} in CERTIFICATE frames. Clients use a
generic SNI for the underlying client-facing server TLS connection. Problems
with this approach include: (1) one additional round trip before peer
authentication, (2) non-trivial application-layer dependencies and interaction,
and (3) obtaining the generic SNI to bootstrap the connection. In contrast,
encrypted SNI induces no additional round trip and operates below the
application layer.

# Acknowledgements

This document draws extensively from ideas in {{?I-D.kazuho-protected-sni}}, but
is a much more limited mechanism because it depends on the DNS for the
protection of the ECH key. Richard Barnes, Christian Huitema, Patrick McManus,
Matthew Prince, Nick Sullivan, Martin Thomson, and David Benjamin also provided
important ideas and contributions.

# Change Log

> **RFC Editor's Note:** Please remove this section prior to publication of a
> final version of this document.

Issue and pull request numbers are listed with a leading octothorp.

## Since draft-ietf-tls-esni-09

- Finalize HPKE dependency (#390)

- Move from client-computed to server-chosen, one-byte config
  identifier (#376, #381)

- Rename ECHConfigs to ECHConfigList (#391)

- Clarify some security and privacy properties (#385, #383)
