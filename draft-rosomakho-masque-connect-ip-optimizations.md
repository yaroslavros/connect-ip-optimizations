---
title: "Reusable templates and checksum offload for CONNECT-IP"
abbrev: "Reusable templates and checksum offload for CONNECT-IP"
category: std

docname: draft-rosomakho-masque-connect-ip-optimizations-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Multiplexed Application Substrate over QUIC Encryption"
keyword:
 - template
 - checksum
 - connect-ip
venue:
  group: "Multiplexed Application Substrate over QUIC Encryption"
  type: "Working Group"
  mail: "masque@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/masque/"
  github: "yaroslavros/connect-ip-optimizations"
  latest: "https://yaroslavros.github.io/connect-ip-optimizations/draft-rosomakho-masque-connect-ip-optimizations.html"

author:
 -
    fullname: Yaroslav Rosomakho
    organization: Zscaler
    email: yrosomakho@zscaler.com

normative:

informative:

--- abstract

This document defines extensions to the CONNECT-IP protocol (RFC 9484) that improve the efficiency of datagram transmission by introducing reusable templates and checksum offload capabilities.

Reusable templates allow endpoints to associate Context Identifiers with static portions of packet headers, enabling datagrams to omit repeated byte sequences while remaining self-contained and stateless on the wire. Checksum offload enables endpoints to delegate computation of transport-layer checksums to the receiver by signaling the relevant offsets within the reconstructed packet.

These optimisations reduce per-packet overhead, processing cost, and effectively increase the usable maximum transmission unit (MTU) when CONNECT-IP datagrams are encapsulated in QUIC DATAGRAM frames.

--- middle

# Introduction

The CONNECT-IP method {{!CONNECT-IP=RFC9484}} allows an HTTP client to establish an IP tunnel through an HTTP proxy and exchange IP packets using either HTTP/3 Datagrams specified in {{Section 2.1 of !HTTP-DATAGRAMS=RFC9297}} or DATAGRAM capsules specified in {{Section 3.5 of HTTP-DATAGRAMS}}. Each packet is carried in full, including all transport and network headers, which provides simplicity and interoperability but incurs per-packet overhead due to the repeated transmission of largely invariant header fields.

This document introduces two optional extensions, Reusable Templates and Checksum Offload, that optimise CONNECT-IP datagram transmission without changing the existing protocol semantics.

Reusable templates allow endpoints to associate a Context Identifier with a reusable packet layout consisting of static and variable byte regions. Once a template has been installed using reliable Capsules, datagrams referencing the same Context Identifier carry only the variable portions of the packet. This reduces the size of transmitted datagrams and processing overhead, while maintaining on-the-wire statelessness and compatibility with intermediaries that are unaware of these optimisations.

Checksum offload enables endpoints to delegate computation of transport-layer checksums to the receiver by identifying the checksum field offset and the start of the checksum coverage within the reconstructed packet. This mirrors hardware checksum-offload behavior used on network interfaces and tunnel devices, reducing per-packet CPU cost for encapsulating or decapsulating CONNECT-IP traffic.

When CONNECT-IP datagrams are encapsulated in QUIC DATAGRAM frames, these optimisations also increase the effective maximum transmission unit (MTU) by reducing the number of bytes carried inside each QUIC packet.

Both extensions are negotiated at CONNECT-IP establishment and signalled using Capsules on the reliable control stream. When necessary, endpoints can fall back to transmitting complete IP packets using Context ID 0, which represents unoptimised datagrams containing the full IP packet as defined in {{Section 5 of CONNECT-IP}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following terms are used in this document:

Context Identifier (CID):

: A numeric identifier associated with a specific packet reconstruction and processing behavior. A CONNECT-IP tunnel can maintain multiple CIDs in each direction. Context ID 0 always represents transmission of complete, unoptimised IP packets as defined in {{Section 5 of CONNECT-IP}}.

Template:

: A reusable packet layout consisting of a sequence of static and variable segments. Static segments contain bytes omitted from optimized datagrams, while variable segments correspond to bytes still carried in the datagram payload.

Checksum Offload:

: A capability allowing the receiver to compute and finalize the Internet checksum according to {{!INCREMENTAL-CHECKSUM=RFC1624}} for the reconstructed packet, based on two offsets: `Checksum Field Offset` and `Checksum Start Offset`.

Capsule:

: A reliable control-stream message, as defined in {{Section 3 of HTTP-DATAGRAMS}}, used in this specification to signal creation or deletion of Context Identifiers and their associated templates.

# Negotiation of Capabilities {#negotiation}

Endpoints negotiate support for these optimizations when establishing a CONNECT-IP tunnel by using a `connect-ip-optimizations` HTTP header field, whose value is a Structured Field Dictionary as defined in {{Section 3.2 of !STRUCTURED-HTTP=RFC8941}}.

## Header Definition

~~~
connect-ip-optimizations = sf-dictionary
~~~
{: #fig-connect-ip-optimizations-header title="Connect-ip-optimizations header field"}

This document defines the following optional dictionary keys:

`templates` (sf-interger):

: Indicates support for reusable templates and specifies the maximum number of templates that the sender is willing to maintain for templates received from the peer. A positive integer value indicates full support for reusable templates, and the value represents an upper bound on the number of concurrently active templates that can be created by the peer. A value of 0 indicates that the sender supports reusable templates only for its own transmissions but does not accept templates created by the peer. If this member is omitted, reusable templates are not supported in either direction.

`checksum` (sf-boolean):

: Indicates support for checksum offload semantics. A value of `?1` means that checksum offload is supported in both directions. A value of `?0` means that checksum offload may be used for packets sent by this endpoint but not for packets received from the peer. If this member is omitted, checksum offload is not supported in either direction.

Endpoints MUST ignore unknown dictionary members. The absence of a member implies that the corresponding capability is not supported by the sender.

## Negotiation Behavior

If both peers indicate support for templates (that is, `templates` member is present and non-zero in both directions), either endpoint MAY create and delete Context Identifiers and their templates using capsules as described in {{capsules}}.

If both peers indicate support for checksum offload (that is, `checksum=?1` in both directions), either endpoint MAY install checksum offload parameters for specific Context Identifiers using capsules as described in {{optimization-create}}.

Peers of endpoints that advertise support in only one direction (for example, `checksum=?0` or `templates=0`) MUST NOT send capsules that would require those endpoints to maintain state for the corresponding capability.

Capabilities that were not successfully negotiated MUST NOT be used within the tunnel.

## Example

HTTP/3 sample request (client to proxy):

~~~
:method = CONNECT
:protocol = connect-ip
:scheme = https
:path = /.well-known/masque/ip/*/*/
:authority = proxy.example.net
capsule-protocol = ?1
connect-ip-optimizations = templates=20000, checksum=?1
~~~
{: #fig-connect-ip-optimizations-request-example title="CONNECT-IP with connect-ip-optimizations request example"}

HTTP/3 sample response (proxy to client):

~~~
:status = 200
capsule-protocol = ?1
connect-ip-optimizations = templates=65535, checksum=?0
~~~
{: #fig-connect-ip-optimizations-response-example title="CONNECT-IP with connect-ip-optimizations response example"}

In this example, both peers support reusable templates, and checksum offload is supported only for packets sent from the proxy to the client.

After this exchange, both endpoints may define Context Identifiers and associated templates and/or checksum parameters using capsules on the reliable control stream.

# Optimization capsules {#capsules}

This specification defines two capsule types:

OPTIMIZATION_CREATE (capsule type 0x1a768469):

: Creates an immutable optimization context bound to a Context ID.

OPTIMIZATION_DELETE (capsule type 0x1a76846a):

: Retires a CID.

All capsules are sent on the reliable control stream of the CONNECT-IP tunnel.

## Context ID allocation

The Context ID carried in these capsules is encoded as a QUIC variable-length integer defined in {{Section 16 of ?QUIC=RFC9000}}. Even-numbered CIDs are allocated by the client, and odd-numbered CIDs are allocated by the proxy, consistent with {{Section 4 of !CONNECT-UDP=RFC9298}}. CID 0 is reserved for unoptimized raw packets and MUST NOT appear in these capsules.

## OPTIMIZATION_CREATE capsule {#optimization-create}

OPTIMIZATION_CREATE capsule is used to create a new, immutable optimization context for the indicated CID.

~~~
OPTIMIZATION_CREATE Capsule {
  Type (i) = 0x1a768469,
  Length (i),
  Context ID (i),
  Static Segments Length (i),
  Static Segment (..) ...,
  Checksum Field Offset (i)?,
  Checksum Start Offset (i)?,
}
~~~
{: #fig-optimization-create-capsule title="OPTIMIZATION_CREATE Capsule Format"}

OPTIMIZATION_CREATE capsule contains the following fields:

Context ID:

: Context Identifier, encoded as a variable-length integer, defined by this capsule.

Static Segments Length:

: Aggregate length in bytes of all subsequent Static Segments. A value of 0 indicates that no template is used (that is, the payload of datagrams for the given Context ID contains a full reconstructed packet, modulo checksum finishing if configured).

Checksum Field Offset:

: An optional field, encoded as a variable-length integer, containing byte offset of the 16-bit Internet checksum field within the reconstructed packet. This field is omitted if checksum offload is not used.

Checksum Start Offset:

: An optional field, encoded as a variable-length integer, containing byte offset where checksum coverage begins. Coverage runs from this offset to the end of the reconstructed packet. Not included in the capsule if checksum offloading is not used.

The OPTIMIZATION_CREATE capsule contains a sequence of zero or more Static Segments.

~~~
Static Segment {
  Segment Offset (i),
  Segment Length (i),
  Segment Payload (..),
}
~~~
{: #fig-static-segment title="Static Segment Format"}

Each Static Segment contains following fields:

Segment Offset:

: Byte offset from the start of the reconstructed packet, encoded as a variable-length integer

Segment Length:

: Number of bytes in this static segment, encoded as a variable-length integer

Segment Payload:

: the static bytes to insert at the Segment Offset

### Parsing and validation

The receiver parses an OPTIMIZATION_CREATE capsule by reading, in order: the Context ID, the Static Segments Length, zero or more static segments whose encodings consume exactly Static Segments Length bytes, and then two checksum offsets if they are present.

The Context ID MUST be non-zero, even-numbered when created by the client, and odd-numbered when created by the proxy. The Context ID MUST NOT have been used previously. Static Segments Length is the total size, in bytes, of the static segments section including each Segmenet Offset, Segment Length, and Segment Payload.

Each Static Segment consists of a Segment Offset, a Segment Length, and exactly Segment Length octets of Segment Payload. Static segments MUST appear in strictly increasing Segment Offset order and MUST NOT overlap.

After Static Segments Length bytes have been consumed, the capsule either ends immediately or contains exactly two additional fields: Checksum Field Offset followed by Checksum Start Offset. If only one variable-length integer is present, or if any bytes remain after the two length integers, the capsule is malformed.

A receiver that did not negotiate acceptance of checksum offload in its direction as defined in {{negotiation}} MUST treat an OPTIMIZATION_CREATE capsule that includes checksum offsets as an error and MUST follow the error-handling procedure in {{Section 3.3 of HTTP-DATAGRAMS}}. A receiver that has already accepted the maximum number of templates it advertised via the templates member of `connect-ip-optimizations` MUST treat any additional OPTIMIZATION_CREATE capsule containing a template (that is, with Static Segments Length > 0) as an error and MUST follow the same error-handling procedure.

If any of the capsule fields are malformed upon reception, the receiver of the capsule MUST follow the error-handling procedure defined in {{Section 3.3 of HTTP-DATAGRAMS}}.

Per-packet validation uses the reconstruction procedure described in {{reconstruction}}.

## OPTIMIZATION_DELETE capsule

OPTIMIZATION_DELETE capsule is used to indicate that a Context ID previously defined by OPTIMIZATION_CREATE capsule will no longer be used.

~~~
OPTIMIZATION_DELETE Capsule {
  Type (i) = 0x1a76846a,
  Length (i),
  Context ID (i),
}
~~~
{: #fig-optimization-delete-capsule title="OPTIMIZATION_DELETE Capsule Format"}

After sending an OPTIMIZATION_DELETE Capsule, the sender MUST NOT use the Context ID. Upon receipt, the peer retires the indicated context and releases any negotiated template budget consumed by that context if, and only if, the retired context included a template (that is, its OPTIMIZATION_CREATE had a non-zero Static Segments Length). Retiring a context that had no template (for example, checksum-only) does not affect the template budget.

If an OPTIMIZATION_DELETE capsule is received for a Context ID that was not previously and correctly defined by an OPTIMIZATION_CREATE capsule from the peer, the receiver MUST follow the error-handling procedure defined in {{Section 3.3 of HTTP-DATAGRAMS}}.

# Optimized Datagram Operation

This section defines how endpoints construct and consume datagrams once a Context ID has been created with OPTIMIZATION_CREATE capsule.

## Sender behavior

For each packet sent under a Context ID, the sender constructs the datagram payload according to the context's template. If the context has no template (that is, Static Segments Length is 0), the payload MUST be a complete reconstructed packet. If the context has a template, the payload MUST be the concatenation of all variable byte ranges, taken in strictly increasing offset order, corresponding to the gaps not covered by static segments starting at offset 0.

When checksum offload is configured (that is, the context includes Checksum Field Offset and Checksum Start Offset), the sender MUST set the checksum field in the reconstructed packet image to the 16-bit one’s-complement sum of the appropriate pseudo-header before transmission. The two bytes at Checksum Field Offset in the reconstructed image therefore MUST contain this pseudo-header sum; these bytes originates from static bytes in the template as well as from variable bytes in the payload. The final reconstructed image MUST contain the correct preseeding value.

A sender uses Context ID 0 for any one-off packet that does not fit an existing context (for example, a transient change in header layout).

## Receiver behavior {#reconstruction}

A datagram that arrives with a non-zero Context ID that has not been previously and correctly defined by an OPTIMIZATION_CREATE capsule from the peer MUST be dropped.

If the CID has no template (that is, Static Segments Length is 0), the datagram payload is the complete reconstructed packet. If the CID has a template, the receiver reconstructs the packet from offset 0 upward by writing static bytes at their declared offsets and filling all remaining byte positions with bytes consumed from the datagram payload in strictly increasing offset order. Reconstruction continues until all payload bytes have been consumed. If the payload is exhausted before the offset of the last static segment have been filled, the packet MUST be dropped.

When Checksum Field Offset and Checksum Start Offset are present for the CID, the receiver finishes the Internet checksum as follows: it computes the checksum over the byte range starting at Checksum Start Offset and ending at the reconstructed packet length while treating the checksum field as zero during the sum; it then adds (folds) the 16-bit value currently at Checksum Field Offset, performs end-around carry, and writes the final one's-complement result back at Checksum Field Offset. If either checksum offset is greater than or equal to the reconstructed packet length for this packet, the packet MUST be dropped.

# Security Considerations

This specification changes how CONNECT-IP datagrams are constructed but does not weaken transport-layer integrity or confidentiality protections provided by the underlying HTTP mapping. All Capsules travel on the reliable control stream and inherit those protections.

Context state can be abused for resource exhaustion. Endpoints enforce negotiated limits from `connect-ip-optimizations`; they MUST reject creations that exceed the declared template budget and must release budget when a context is retired with OPTIMIZATION_DELETE. Implementations SHOULD bound the number of static segments, validate lengths before allocation and cap per-context memory.

Negotiation and Capsule handling are directional and immutable to reduce desynchronization risk. Even/odd Context ID allocation prevents collisions between endpoints; CID 0 is reserved and must not appear in Capsules. Unknown CIDs must be dropped. Reusing a CID within the same connection after deletion is not permitted; endpoints MUST allocate a fresh CID to change behavior.

# IANA Considerations

## HTTP Capsule Types Registration

This specification registers the following values in the "HTTP Capsule Types" registry:

| Value | Capsule Type
+ --- + --- +
| 0x1a768460 | OPTIMIZATION_CREATE |
| 0x1a76846a | OPTIMIZATION_DELETE |

## HTTP Field Name Registration

This specification registers the following value in the "HTTP Field Name" registry:

- Field Name: connect-ip-optimizations
- Status: provisional (permanent if approved)
- Structured Type: Dictionary
- Reference: This document


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
