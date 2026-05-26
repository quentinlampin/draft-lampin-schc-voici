# SCHC Transport Encapsulation (SCHC-TE)

This version of the draft: 2026-05-26

Quentin Lampin
Orange

# Abstract

This document specifies a minimal Transport Encapsulation (SCHC-TE) for the
Static Context Header Compression (SCHC) framework. SCHC-TE provides Session
multiplexing and Context version awareness for deployments where extrinsic
Discriminators are insufficient. The encapsulation is designed for minimal
overhead, reducing to 3 bytes in the common case, while supporting optional
integrity protection and next-protocol identification. SCHC-TE operates in
conjunction with the protocol numbers defined in [DRAFT-PROTOCOL-NUMBERS].

# Status of This Memo

This is an Internet-Draft.

Internet-Drafts are working documents of the Internet Engineering Task Force
(IETF). Note that other groups may also distribute working documents as
Internet-Drafts. The list of current Internet-Drafts is at
https://datatracker.ietf.org/drafts/current/.

Copyright (c) 2026 IETF Trust and the persons identified as the document
authors. All rights reserved.

This document is subject to BCP 78 and the IETF Trust's Legal Provisions
Relating to IETF Documents (https://trustee.ietf.org/license-info) in effect
on the date of publication of this document. Please review these documents
carefully, as they describe your rights and restrictions with respect to this
document. Code Components extracted from this document must include Revised
BSD License text as described in Section 4.e of the Trust Legal Provisions and
the provided license template must be modified as appropriate to refer to the
provided script template instead of the document.


# Table of Contents

{::options toc_from_heading='2' /}

# Introduction

The SCHC framework [RFC8724] provides header compression and optional
fragmentation based on a static Context shared between Endpoints. In the common
deployment — a single Instance per Endpoint over a single link — the mapping
between the link and the Instance is trivial: all Datagrams on the link belong
to that one Instance, and no multiplexing mechanism is needed.

However, several deployment scenarios require a mechanism to distinguish
multiple SCHC Sessions over a shared link:

* An Endpoint hosts multiple Instances serving different Domains or tenants.
* Multiple Sessions share an Ethernet segment or IPv6 link.
* Context updates require version awareness to detect mismatch during
  coordinated transitions.

Extrinsic Discriminators (Device address, PPP connection identity) are
insufficient in these cases. This document specifies a Transport Encapsulation
(SCHC-TE) that provides the required multiplexing and version awareness.

## Requirements Language

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",
"NOT RECOMMENDED", "MAY", and "OPTIONAL" are to be interpreted as described
in BCP 14 [RFC2119] [RFC8174].

# Requirements

The SCHC architecture [DRAFT-SCHC-ARCH] defines the following requirements
for a Transport Encapsulation:

1. **Session identification**: A mechanism to distinguish Sessions and route
   Datagrams to the correct Instance. The identifier (Session ID) is locally
   significant to the link.

2. **Context version awareness**: A mechanism to carry the Context Version
   during update transitions, enabling receivers to detect Datagrams sent with
   a mismatched Context version.

3. **Integrity protection** (optional): A mechanism to detect corruption of the
   Datagram, including the Session ID and the SCHC operation result.

4. **Next-protocol identification** (optional): A mechanism to identify the
   Upper Layer Protocol that was compressed, when the compressed residue elides
   protocol identification information.

Additional requirements derived from deployment considerations:

5. **Minimal overhead**: The common case (Session identification only, no
   optional fields) MUST minimize header bytes. The target is 3 bytes.

6. **Layer independence**: SCHC-TE MUST operate over any link layer that carries
   SCHC traffic, whether identified by an Ethertype, IP Protocol Number, or
   UDP port ([DRAFT-PROTOCOL-NUMBERS]).

# Gap Analysis

Several existing mechanisms can provide multiplexing or labeling. This section
analyzes their suitability for SCHC and identifies the gap that SCHC-TE fills.

## MPLS

MPLS labels provide efficient multiplexing (20-byte label stack) and are widely
deployed in operator networks. However:

* MPLS is not available on all link types relevant to SCHC (LPWAN, PPP,
  low-speed serial links).
* MPLS provides no version awareness mechanism.
* MPLS provides no integrity protection.
* MPLS adds 4 bytes per label (plus optional 4-byte padding), which may be
  excessive for highly constrained deployments.

## UDP Encapsulation

UDP is commonly used for Internet traversal and NAT traversal. The UDP source
port can carry a Session ID:

* The UDP header is 8 bytes, which is larger than the 3-byte SCHC-TE minimal
  header.
* UDP provides no version awareness mechanism.
* Using the UDP source port as Session ID is fragile in the presence of NAT
  (port remapping) and port exhaustion (65535 limit shared with other
  applications).
* UDP checksum is optional and not designed for SCHC Datagram integrity.

UDP remains a valid transport option when NAT traversal is required, but
SCHC-TE provides a more efficient and purpose-built alternative when NAT is
not a concern.

## Raw IP Protocol Number

The SCHC Internet Protocol Number [DRAFT-PROTOCOL-NUMBERS] identifies SCHC
traffic at the IP layer but does not provide:

* Session multiplexing (one protocol number per link, not per Session).
* Context version awareness.
* Integrity protection.

The IP Protocol Number is necessary to identify SCHC traffic but insufficient
for multiplexing.

## SCHC Ethertype

The SCHC Ethertype [DRAFT-PROTOCOL-NUMBERS] identifies SCHC frames on IEEE 802
links but does not provide:

* Session multiplexing (one Ethertype per link, not per Session).
* Context version awareness.
* Integrity protection.

The Ethertype is necessary to identify SCHC traffic at the link layer but
insufficient for multiplexing.

## Summary

| Mechanism       | Multiplexing | Version Awareness | Integrity | Overhead    | Link Coverage |
|-----------------|--------------|-------------------|-----------|-------------|---------------|
| MPLS            | Yes          | No                | No        | 4+ bytes    | Limited       |
| UDP (src port)  | Yes          | No                | No        | 8 bytes     | IP only       |
| IP Protocol Num | No           | No                | No        | 0 bytes     | IP only       |
| Ethertype       | No           | No                | No        | 0 bytes     | IEEE 802 only |
| **SCHC-TE**     | **Yes**      | **Yes**           | **Opt.**  | **3 bytes** | **Any**       |

SCHC-TE fills the gap by providing all four required capabilities with minimal
overhead.

# Header Format

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-----+----------------+--------+
|V|I|N|NPI/R|   Session ID   |        |
+-+-+-+-----+----------------+--------+
+----------------+
|Context Version |  (optional, present if V=1)
+----------------+
+----------------+
|      CRC       |  (optional, present if I=1)
+----------------+
~~~

**Figure 1: SCHC-TE Header**

The first row (3 bytes) is always present. The Context Version row (2 bytes)
is present when V=1. The CRC row (2 bytes) is present when I=1. The SCHC
Datagram follows immediately after the header.

## Fields

* **V** (1 bit): Version flag. When set, the Context Version field is present.
  Used during Context update transitions to allow the receiver to detect version
  mismatch.

* **I** (1 bit): Integrity flag. When set, a CRC-16 field is present and covers
  the Session ID through the end of the SCHC Datagram. When clear, no integrity
  check is carried.

* **N** (1 bit): Next-protocol flag. When set, the next protocol identifier is
  carried in the Reserved/NPI field. Used when the compressed header elides
  protocol identification information.

* **Reserved/NPI** (5 bits): When N=0, reserved and MUST be zero. When N=1,
  the field carries a 5-bit next protocol identifier (values 0-31). Protocol
  numbers above 31 are not supported by this field; profiles that need to
  identify such protocols MUST use an alternative mechanism. The most
  significant bit of the 5-bit field (bit 4) is reserved for a future Endpoint
  ID extension and MUST be zero when N=0.

* **Session ID** (16 bits): Identifier for the SCHC Session. Used by the
  Dispatcher to route the Datagram to the correct Instance. The Session ID
  space is local to the link over which SCHC-TE is carried.

* **Context Version** (16 bits, optional): Present when V=1. Carries the Context
  Version to enable version mismatch detection during Context updates.

* **CRC** (16 bits, optional): Present when I=1. CRC-16/CCITT-FALSE over the
  Session ID field through the end of the SCHC Datagram.

## Minimal Header

When no optional fields are needed (V=0, I=0, N=0), the SCHC-TE header reduces
to 3 bytes:

~~~
 0               1               2
 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
+-+-+-+-----+----------------+
|V|I|N|NPI/R|   Session ID   |
+-+-+-+-----+----------------+
~~~

**Figure 2: Minimal SCHC-TE Header (3 bytes)**

## Header Size Summary

| Configuration              | V | I | N | Size |
|----------------------------|---|---|---|------|
| Session ID only            | 0 | 0 | 0 | 3 B  |
| Session ID + Version       | 1 | 0 | 0 | 5 B  |
| Session ID + CRC           | 0 | 1 | 0 | 5 B  |
| Session ID + Version + CRC | 1 | 1 | 0 | 7 B  |
| Session ID + Next-Proto    | 0 | 0 | 1 | 3 B  |
| All fields                 | 1 | 1 | 1 | 7 B  |

# Session ID Allocation

The Session ID is locally significant to the link. Allocation strategies depend
on the deployment topology:

## P2P Deployments

Session IDs MAY be negotiated between peers during Session establishment, or
assigned by the Domain Manager during provisioning.

## Star Topologies

The Network Gateway assigns Session IDs and communicates them to Devices during
provisioning. The Gateway maintains the Session ID to Instance mapping.

## Mesh and Other Topologies

Session IDs MAY be assigned by the Domain Manager or negotiated between peers.

## Relay Remapping

A relay or gateway translating between links MAY remap Session IDs. The Session
ID space is local to each link segment; there is no requirement for global
uniqueness.

# Context Version Awareness

The V flag and Context Version field provide Context version awareness during
Coordinated Context updates. This section describes how SCHC-TE supports the
three-state update model defined in the SCHC architecture [DRAFT-SCHC-ARCH].

## Stable State (vN)

All Instances in the Session operate on Context Version vN. The V flag is clear
(V=0). The Context Version field is not present.

## Transition State

At least one Instance has moved to vN+1 while at least one remains on vN.
Datagrams carry V=1 with the Context Version field present. A receiver that
detects a version mismatch MAY drop the Datagram and notify the Domain Manager.

The transition window is bounded by Domain Manager coordination. Both sides set
V=1 until they confirm the peer has updated.

## Stable State (vN+1)

All Instances in the Session confirm they are on vN+1. The V flag returns to
clear (V=0). Normal operation resumes.

## Incompatible Updates

For incompatible Context updates (shared Rules have changed semantics), the
Session MUST be torn down and re-established. SCHC-TE does not provide a
mechanism to carry Datagrams across an incompatible update boundary.

# Integrity Protection

The I flag and CRC field provide optional integrity protection for the SCHC
Datagram.

## CRC Scope

The CRC covers the Session ID field through the end of the SCHC Datagram. This
ensures that Session misrouting (bit error in Session ID) is detected.

## CRC Algorithm

CRC-16/CCITT-FALSE (polynomial 0x1021, initial value 0xFFFF, no reflection,
no final XOR) is used. This is the same algorithm used in many constrained
network protocols (e.g., Bluetooth, CAN bus).

## Relationship to ULP Checksums

Some compression strategies elide Upper Layer Protocol (ULP) checksums (e.g.,
UDP checksum) to reduce residue size. When ULP checksums are elided, the
transport CRC is the only integrity mechanism for the Datagram. The tradeoff
is 2 bytes of overhead versus integrity protection. Profiles SHOULD specify
whether elision is permitted and whether the transport CRC is required in such
cases.

## Limitations

The CRC provides integrity (corruption detection) but NOT authentication. An
attacker can compute a valid CRC for a forged Datagram. Authentication must be
provided by the underlying transport or a higher-layer security mechanism.

# Next-Protocol Identification

The N flag and Reserved/NPI field provide optional next-protocol identification.
This is useful when the compressed residue elides protocol identification
information (e.g., the IP "Next Header" field or UDP "Protocol" field was
compressed away).

The 5-bit field supports protocol numbers 0-31. Common values include:
* 17 = UDP
* 41 = IPv6
* 50 = ESP
* 51 = AH

Protocol numbers above 31 (e.g., CoAP = 269) cannot be carried in this field.
Profiles that need to identify such protocols MUST use an alternative mechanism
(e.g., the Session ID implicitly identifies the protocol, or the protocol is
known from the deployment context).

# Interaction with Protocol Numbers

SCHC-TE operates in conjunction with the protocol numbers defined in
[DRAFT-PROTOCOL-NUMBERS]. The layering is:

~~~
  +------------------+
  | SCHC Datagram    |
  +------------------+
  | SCHC-TE Header   |  (optional)
  +------------------+
  | Transport        |  (Ethertype / IP Protocol / UDP)
  +------------------+
  | Link Layer       |
  +------------------+
~~~

**Figure 3: SCHC Layer Stack**

## Over Ethertype

When the SCHC Ethertype is used, SCHC-TE is carried directly in the Ethernet
payload following the Ethertype field.

## Over IP Protocol Number

When the SCHC IP Protocol Number is used, SCHC-TE is carried in the IPv6
extension header or directly following the IPv6 header.

## Over UDP

When the SCHC UDP port is used, SCHC-TE is carried in the UDP payload. The UDP
header provides its own checksum (integrity), which may make the SCHC-TE CRC
redundant. The UDP source port MAY carry the Session ID, in which case SCHC-TE
is not needed.

# Security Considerations

## Session Hijacking

If Session IDs are predictable, an attacker could inject Datagrams with a forged
Session ID to redirect traffic to a different Instance. Session IDs SHOULD be
randomly generated or derived from a secure key exchange.

## Integrity Limitations

The CRC provides corruption detection but not authentication. An attacker with
link access can forge Datagrams with valid CRCs. Authentication must be provided
by the underlying transport (e.g., IPsec, TLS) or a higher-layer mechanism
(e.g., OSCORE).

## Context Version Manipulation

A forged Context Version in the SCHC-TE header could cause a receiver to reject
valid Datagrams. Context Version values are authenticated by the Context
distribution channel (e.g., signed CORECONF operations), not by SCHC-TE itself.

## Denial of Service

An attacker could inject Datagrams with invalid Session IDs, causing the
receiver to waste resources on lookup failures. Implementations SHOULD rate-limit
Session ID lookup failures.

# IANA Considerations

## SCHC-TE Header Registry

This document requests the creation of an "SCHC Transport Encapsulation Header"
registry. The initial entries are:

| Field          | Size   | Description                    |
|----------------|--------|--------------------------------|
| V flag         | 1 bit  | Context Version presence       |
| I flag         | 1 bit  | CRC presence                   |
| N flag         | 1 bit  | Next-protocol presence         |
| Reserved/NPI   | 5 bits | Reserved / Next-protocol ID    |
| Session ID     | 16 bits| Session identifier             |
| Context Version| 16 bits| Context version (optional)     |
| CRC            | 16 bits| Integrity check (optional)     |

## Session ID Space

The Session ID space is locally significant to the link. No IANA assignment is
required.

# References

## Normative References

{::options tocrefs="RFC2119,RFC8174,RFC8724" /}

[RFC2119]
: Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, DOI 10.17487/RFC2119.

[RFC8174]
: Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words", BCP 14, RFC 8174, DOI 10.17487/RFC8174.

[RFC8724]
: Toutain, L., "Static Context Header Compression (SCHC): A Header Compression Scheme for Low-Power and Lossy Networks (LLNs)", RFC 8724, DOI 10.17487/RFC8724.

## Informative References

[DRAFT-SCHC-ARCH]
: Pelov, A. et al., "SCHC Architecture", Work in Progress.

[DRAFT-PROTOCOL-NUMBERS]
: Moskowitz, R., et al., "Protocol Numbers for SCHC", Work in Progress, Internet-Draft, draft-ietf-schc-protocol-numbers.
