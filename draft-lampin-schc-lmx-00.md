---
v: 3

title: SCHC Link Multiplexer
abbrev: SCHC LMX
docname: draft-lampin-schc-lmx-latest

stand_alone: true

ipr: trust200902
area: Internet
wg: schc Working Group
kw: Internet-Draft
cat: info
submissionType: IETF

coding: us-ascii
pi:
  toc: yes
  sortrefs: yes
  symrefs: yes

author:
  -
    ins: Q. Lampin
    name: Quentin Lampin
    org: Orange
    street: Orange 3 Massifs - 22 Chemin du Vieux Chene
    city: Meylan
    code: 38240
    country: France
    email: quentin.lampin@orange.com

normative:
  RFC8126:
  RFC8724: schc

informative:
  I-D.ietf-schc-architecture: arch
  I-D.toutain-schc-coreconf-management: coreconf
  I-D.ietf-schc-protocol-numbers: numbers 

  DWARF:
    title: "DWARF Debugging Information Format"
    author:
      - org: Dwarf Standards Committee
    date: false
    target: https://dwarfstd.org/documentation/
    seriesinfo:
      Web: https://dwarfstd.org/documentation/


--- abstract

The Static Context Header Compression (schc) framework identified the need for
a minimal transport encapsulation that provides Session multiplexing and
Context version awareness when extrinsic Discriminators are insufficient. This
document specifies a Link Multiplexer (LMX) that addresses those schc-driven
requirements while remaining general enough to accommodate other compression
mechanisms and uncompressed payloads.  The encapsulation is designed for
minimal overhead, reducing to 2 bytes in the common case, while supporting
optional integrity protection, content mechanism identification, and original
EtherType/port recovery.

--- middle

# Introduction

The SCHC framework {{RFC8724}} provides header compression and optional
fragmentation based on a static Context shared between Endpoint Instances. In
the common deployment -- a single Instance per Endpoint over a single link --
the mapping between the link and the Instance is trivial: all Datagrams on the
link belong to that one Instance, and no multiplexing mechanism is needed.

However, several deployment scenarios require a mechanism to distinguish
multiple Sessions over a shared link:

* An Endpoint hosts multiple Instances serving different Domains or tenants.
* Multiple Sessions share an Ethernet segment or IPv6 link.


These requirements were first identified by the SCHC architecture
{{I-D.ietf-schc-architecture}} for the case of SCHC-compressed Datagrams.
But the need is broader than SCHC alone. Operator and industrial deployments 
often carry a mix of traffic types on the same constrained link: SCHC-compressed
Datagrams from devices that use static Contexts; Datagrams from other mechanisms;
and uncompressed management or diagnostic traffic that bypasses compression. In
all of these cases, transport-level multiplexing, versioning, and optional
integrity are desirable.

This document specifies a Link Multiplexer (LMX) that satisfies the
requirements identified for SCHC while remaining general enough for other
compression mechanisms.  The LMX header carries a Session ID for
multiplexing, a Content Mechanism Identifier for dispatching the Datagram
to the correct handler, and an optional Reference field that a Content
Mechanism can use to carry stateful metadata (for example, a Context
Version).  Optional integrity protection is also provided. The encapsulation
is designed for minimal overhead, reducing to 2 bytes in the common case
(1-byte flag + 1-byte Session ID for values less than 128).

{::boilerplate bcp14-tagged-bcp14}

# Requirements

The requirements below are organized into two groups.  Requirements 1-4 were
first identified by the SCHC architecture {{I-D.ietf-schc-architecture}} for the
specific case of SCHC-compressed Datagrams.  Requirements 5-6 were added when 
the scope was broadened to encompass other compression mechanisms and uncompressed
payloads.

## Requirements driven by SCHC

{: group="1"}
1. **Session identification**:  A mechanism to distinguish Sessions and route
   Datagrams to the correct processing handler (for example, a SCHC Instance).  
   The identifier (Session ID) is locally significant to the link.
1. **Original EtherType/port recovery (optional)**:  A mechanism to carry the
   original EtherType or UDP port number when the carrier uses the SCHC
   EtherType or SCHC UDP port.  This is needed when the payload is
   decompressed so that the receiver can restore the original framing layer 
   after decompression.
2. **Context version awareness**:  A mechanism to carry the Context Version
   during coordinated Context updates, enabling receivers to detect Datagrams 
   sent with a mismatched Context version.
3. **Integrity protection (optional)**:  A mechanism to detect corruption of
   the Datagram, including the Session ID and the compressed residue.
4. **Minimal overhead**:  The common case (Session identification only, no
   optional fields) MUST minimize header bytes. The target is 2 bytes.

## Requirements driven by multi-mechanism and uncompressed payloads

{: group="1"}
5. **Content mechanism identification**:  A mechanism to identify how the 
   Datagram payload is encoded when the link carries Datagrams from
   multiple mechanisms (for example, SCHC, uncompressed).  This allows the
   receiver to dispatch the Datagram to the correct decompressor without
   inspecting its contents.
6. **Layer independence**:  The encapsulation MUST operate over any link
   layer that carries compressed traffic, whether identified by an Ethertype,
   IP Protocol Number, or UDP port {{I-D.ietf-schc-protocol-numbers}}.

# Gap Analysis

Several existing mechanisms can provide multiplexing or labeling. This
section analyzes their suitability for SCHC and identifies the gap that LMX
fills.

## MPLS

MPLS labels provide efficient multiplexing (20-byte label stack) and are
widely deployed in operator networks.  However:

* MPLS is not available on all link types relevant to SCHC (LPWAN, PPP,
  low-speed serial links).
* MPLS provides no version awareness mechanism.
* MPLS provides no integrity protection.
* MPLS adds 4 bytes per label (plus optional 4-byte padding), which may
  be excessive for highly constrained deployments.

## UDP Encapsulation

UDP is commonly used for Internet traversal and NAT traversal.  The UDP
source port can carry a Session ID:

* The UDP header is 8 bytes, which is larger than the 2-byte LMX minimal
  header.
* UDP provides no version awareness mechanism.
* Using the UDP source port as Session ID is fragile in the presence of NAT
  (port remapping) and port exhaustion (65535 limit shared with other
  applications).
* UDP checksum is optional and not designed for SCHC Datagram integrity.

UDP remains a valid transport option when NAT traversal is required, but LMX
provides a more efficient and purpose-built alternative when NAT is not a
concern.

## IP Protocol Number and SCHC Ethertype

The SCHC IP Protocol Number and Ethertype {{I-D.ietf-schc-protocol-numbers}} 
identify SCHC traffic at the respective layers but do not provide:

* Session multiplexing (one protocol number or Ethertype per link, not per
  Session).
* Context version awareness.
* Integrity protection.

They are necessary to identify SCHC traffic but insufficient for
multiplexing.

## Summary

| Mechanism       | Multiplexing | Version Awareness | Integrity | Overhead | Link Coverage |
|-----------------|--------------|-------------------|-----------|----------|---------------|
| MPLS            | Yes          | No                | No        | 4+ bytes | Limited       |
| UDP (src port)  | Yes          | No                | No        | 8 bytes  | IP only       |
| IP Protocol Num | No           | No                | No        | 0 bytes  | IP only       |
| Ethertype       | No           | No                | No        | 0 bytes  | IEEE 802 only |
| **LMX**         | **Yes**      | **Yes**           | **Opt.**  | **2 B**  | **Any**       |
{: title="LMX compared" #tab-gap-summary}

LMX fills the gap by providing all five required capabilities (multiplexing,
version awareness, integrity, content mechanism identification, and original
EtherType/port recovery) with minimal overhead.  The comparison is summarized
in {{tab-gap-summary}}.

## Encoding Within SCHC Datagram

Encoding session or version information inside the SCHC rules or rule results
would couple transport-layer concerns (multiplexing, version negotiation) to
compression-layer concerns (what to compress, how to parse the residue).  A
separate encapsulation keeps the SCHC datagram focused on compression results
and allows the transport header to be added or removed without modifying the
compression strategy or the Context/Rules.

Furthermore, when multiple compression mechanisms share the same link, an
inner-field approach would require every mechanism to reserve space for the
same routing and version metadata, reducing compression efficiency.  LMX
places this metadata in a single, mechanism-agnostic header.

# Header Format

~~~
0                   1                   2
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3   bits
+-+-+-+-+-------+---------------- - - - - - - - +
|V|O|I|R|  CMI  |         Session ID            | (1B if <128 else 2)
+-+-+-+-+-------+---------------- - - - - - - - +
+- - - - - - - - - - - - - - - -+
|              CRC              | (optional, present if I=1)
+- - - - - - - - - - - - - - - -+
+- - - - - - - - - - - - - - - -+ (optional, present if O=1,
|    Original EtherType/Port    |  2B for Ethertype or UDP port,
+- - - - - - - - - - - - - - - -+  1B for IPv6 Next Header)
+- - - - - - - - - - - - - - - -+
|           Reference           | (optional, present if R=1,
+- - - - - - - - - - - - - - - -+  per-mechanism/profile 
                                   defined length)
~~~
{: #fig-lmx-header title="LMX Header (2-3 bytes)}


The flags (4 bits), CMI field (4 bits), and the Session ID (1-2 bytes) are
always present.  The CRC (2 bytes) is present when I=1.  The Original
EtherType/Port (1-2 bytes) is present when O=1.  The Reference field
(variable length) is present when R=1.

The Datagram payload follows immediately after the last header field.

**Parsing order:**

1. Read byte 0; extract V, O, I, R, CMI.
2. Read Session ID (LEB128, 1-2 bytes).
3. If I=1, read 2-byte CRC, check CRC against reconstructed frame, drop
   frame if CRC is invalid.
4. If O=1, read 2-byte Ethertype if above Ethernet or UDP, or 1-byte Next
   Header if above IPv6.
5. If R=1, read n-bytes Reference value and provide it to the identified
   Content Mechanism.
6. Pass remaining buffer to the identified Content Mechanism and recover
   original/uncompressed content.
7. If 0-1, restore original Ethertype or Port number.
8. Return processed frame.

## Fields

* **V (1 bit):**  LMX header format version.  V=0 for this draft.
  V=1 for future LMX revisions.  Implementations that encounter V=1 MUST
  handle the version update gracefully (for example, by ignoring
  unrecognized flag combinations).

* **O (1 bit):**  Original EtherType/Port present.  When set, the Original
  EtherType/Port field is present, carrying the EtherType, IP Next Header,
  or UDP port number that was replaced by the LMX EtherType, LMX IP
  Protocol Number, or LMX UDP port.  The field is interpreted as an
  EtherType when LMX is carried over a link-layer transport (for example,
  IEEE 802 Ethertype), as a Next Header if carried over IP, and as a UDP
  port when LMX is carried in a UDP payload.  This restoration is an LMX
  responsibility; the Content Mechanism does not need to manage framing
  recovery.

* **I (1 bit):**  Integrity flag.  When set, a CRC-16 field is present and
  covers the Session ID through the end of the datagram.  When clear, no
  integrity check is carried.

* **R (1 bit):**  Reference field present.  When set, a Reference field
  follows the fixed optional fields.  The Reference field's size, format,
  and interpretation are defined by the Content Mechanism (CMI) and its
  associated profile.  The mechanism profile MUST specify whether the
  Reference is used and how it is encoded.  When the mechanism does not
  use References, R MUST NOT be set.

* **CMI (4 bits):**  Content Mechanism Identifier.  Identifies the
  compression or encoding mechanism used for the datagram payload.  The
  mechanism profile defines the interpretation of each CMI value.  LMX
  profiles register new CMI values as needed.

**Initial CMI assignments:**

| CMI  | Content Mechanism                                                 |
|------|-------------------------------------------------------------------|
| 0    | schc -- standard schc compressed residue                          |
| 1    | Uncompressed / raw -- Datagram passed through without compression |
| 2-15 | Reserved for future mechanisms                                    |
{: #tab-cmi-initial}

Profiles that register a new CMI value MUST specify the compression
mechanism, its parameters, and the interpretation of the Reference field
(if used).

* **Session ID (variable length, LEB128):**  Identifies the logical session
  that owns this Datagram.  When a Content Mechanism is registered with
  LMX, the mechanism profile assigns Session IDs and registers them with
  the LMX instance.  The receiver LMX uses the Session ID to dispatch the
  Datagram to the correct handler -- for schc (CMI=0), the handler is an
  schc Instance; for other mechanisms, the handler is defined by the
  mechanism profile.  The Session ID space (0-65535) is local to the link
  over which LMX is carried.

  Values up to 127 fit in 1 byte; larger values use 2 bytes.  The Session
  ID is encoded as a LEB128 variable-length integer {{DWARF}}:

  * If the value is less than 128, a single byte is used (MSB = 0).
  * If the value is 128 or greater, two bytes are used (first byte MSB = 1).
  * No values larger than 16 bits (65535) are supported.

  The receiver reads the Session ID by inspecting the most significant bit
  of each byte: if the MSB is 1, the next byte is part of the value; if 0,
  the byte is the last.

* **Original EtherType/Port (1-2Bytes, optional):**  Present when O=1.
  Carries the EtherType or UDP port number that was replaced by the LMX
  carrier.  The field is interpreted as an EtherType when LMX is carried
  over a link-layer transport (for example, IEEE 802 Ethertype) and as a
  UDP port when LMX is carried in a UDP payload.

* **CRC (16 bits, optional):**  Present when I=1.  CRC-16/CCITT-FALSE over
  the Session ID field through the end of the Datagram.

* **Reference (variable length, optional):**  Present when R=1. The size,
  format, and interpretation are defined by the Content Mechanism (CMI)
  profile. The mechanism parser consumes the Reference bytes and returns
  the remainder as the datagram payload.  For SCHC (CMI=0), the Reference
  is the Context Version (profile-defined size, for example, 16 bits).
  For mechanisms that maintain session state, the Reference encodes
  whatever state identifier the mechanism requires (for example, dictionary
  revision, configuration revision).  When the mechanism does not use
  References, R MUST NOT be set.

## Minimal Header

When no optional fields are needed (V=0, O=0, I=0, R=0), the LMX header
reduces to 2-3 bytes (flag byte + 1-2 byte Session ID):

~~~
  0                   1
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5   bits
 +-+-+-+-+-------+---------------+
 |V|O|I|R|  CMI  |  Session ID   | (SID < 128: 2 B)
 +-+-+-+-+-------+---------------+
~~~
{: #fig-lmx-minimal title="Minimal LMX Header (2-3 bytes)"}

## Header Size Summary

Minimum header size with all optional fields present and Session ID less
than 128.  The Reference field (if R=1) adds mechanism-defined bytes
beyond the listed sizes.

| Configuration           | V | O | I | R | SID < 128 | SID >= 128 |
|-------------------------|---|---|---|---|-----------|------------|
| Session ID only         | 0 | 0 | 0 | 0 | 2 B       | 3 B        |
| + CRC                   | 0 | 0 | 1 | 0 | 4 B       | 5 B        |
| + Orig. EtherType/Port  | 0 | 1 | 0 | 0 | 4 B       | 5 B        |
| + Reference             | 0 | 0 | 0 | 1 | 2 B*      | 3 B*       |
| All fields              | 0 | 1 | 1 | 1 | 6 B*      | 7 B*       |
{: #tab-header-size title="LMX header size summary (sizes without Reference field)"}

\\* Plus mechanism-defined Reference length when R=1.

## Header Field Reference

| Field             | Size   | Description                                   |
|-------------------|--------|-----------------------------------------------|
| V                 | 1 bit  | LMX header version                            |
| O                 | 1 bit  | Original EtherType/Port presence              |
| I                 | 1 bit  | CRC presence                                  |
| R                 | 1 bit  | Reference field presence                      |
| CMI               | 4 bits | Content Mechanism Identifier                  |
| Session ID        | 1-2 B  | Session identifier (LEB128)                   |
| Original ET/Port  | 1-2 B  | EtherType, Next Header, or UDP port (if O=1)  |
| CRC               | 2 B    | Integrity check (if I=1)                      |
| Reference         | var    | Mechanism-defined state (if R=1)              |
{: #tab-header-fields title="LMX header field summary"}

# Session ID Allocation

The Session ID is locally significant to the link.  Allocation strategies
depend on the deployment topology:

## P2P Deployments

Session IDs MAY be negotiated between peers during Session establishment,
or assigned by the Domain Manager during provisioning.

## Star Topologies

The Network Gateway assigns Session IDs and communicates them to Devices
during provisioning.  The Gateway maintains the Session ID to handler
mapping.

## Mesh and Other Topologies

Session IDs MAY be assigned by a Network or Domain Manager, or negotiated
between peers.

## Relay Remapping

A relay or gateway translating between links MAY remap Session IDs.  The
Session ID space is local to each link segment; there is no requirement
for global uniqueness.

# Reference Field

The R flag and Reference field allow a Content Mechanism to pass state
information to the receiver.  The Reference field's size, format, and
interpretation is specifc to each Content Mechanism (CMI) and its
associated profile.

The Reference field is used, for example, to carry the SCHC Context Version
during coordinated Context updates, or to pass dictionary revisions for
generic compressors. The mechanism profile specifies whether the
Reference is used and how it is encoded.

## SCHC Context Version

{:aside}
> NOTE: Section content should go to a separate Profile document but is
> included here for illustration of the Reference field purpose.

For schc (CMI=0), the Reference field carries the Context Version as
defined by the schc architecture {{I-D.ietf-schc-architecture}}. 
The Context Version is 16 bits, monotonically increasing. Version 0 is a valid
value; the R flag distinguishes "Reference absent" from "Reference present, 
value 0". The 16-bit space provides 65,535 distinct values, which is sufficient
for the expected lifetime of any practical Session (devices with 10-year
operational lifetimes and update intervals of minutes to hours).  Wrap behavior 
is deferred to a future revision.

The three-state update model (stable, transition, stable) operates as
follows:

**Stable State:**  All Instances in the Session operate on Context Version
vN.  The R flag is clear (R=0).  No Reference field is present.

**Transition State:**  At least one Instance has moved to vN+1 while at
least one remains on vN.  Datagrams carry R=1 with the Reference field
containing the Context Version.  A receiver that detects a version
mismatch MAY drop the Datagram and notify the Domain Manager.

**Stable State:**  Both sides confirm the peer has updated.  The R flag
returns to R=0.

**Incompatible Updates:**  For incompatible Context updates (shared Rules
have changed semantics), the Session MUST be torn down and re-established.
LMX does not provide a mechanism to carry Datagrams across an incompatible
update boundary.  Incompatibility is determined by the Domain Manager and
communicated out of band (for example, via a signed CORECONF operation
{{I-D.toutain-schc-coreconf-management}}).  A receiver that cannot decompress a 
Datagram with either the old or the new Context reports the failure to the 
Domain Manager. This is distinct from a version mismatch during the transition
state, which is resolved by the receiver dropping the Datagram and waiting
for the peer to catch up.

## Mechanism-Defined Reference

For mechanisms that maintain session state, the Reference might carry an
opaque identifier that the receiver uses to locate the appropriate
compressor state.  When the receiver encounters an unknown identifier, it
requests re-synchronization through a mechanism-specific exchange.  For
these mechanisms, the Reference field is optional and may be omitted
during stable operation (R=0) or present transiently after a state update
(R=1).

# Integrity Protection

The I flag and CRC field provide optional integrity protection for the
Datagram.

## CRC Scope

The CRC covers the Session ID field through the end of the Datagram.  This
ensures that Session misrouting (bit error in Session ID) is detected.

## CRC Algorithm

CRC-16/CCITT-FALSE (polynomial 0x1021, initial value 0xFFFF, no
reflection, no final XOR) is used.  This is the same algorithm used in
many constrained network protocols (for example, Bluetooth, CAN bus).

## Relationship to ULP Checksums

Some compression strategies elide Upper Layer Protocol (ULP) checksums
(for example, UDP checksum) to reduce residue size.  When ULP checksums
are elided, the transport CRC is the only integrity mechanism for the
Datagram.  The tradeoff is 2 bytes of overhead versus integrity
protection.  Profiles SHOULD specify whether elision is permitted and
whether the transport CRC is required in such cases.

## Relationship to Codec-Level Integrity

Datagrams from compression mechanisms that maintain session state may
carry their own integrity mechanisms inside the Datagram payload.  The
LMX CRC operates at the transport envelope -- covering the Session ID and
the entire Datagram -- while codec-level integrity operates at the
decompression level, verifying decompression correctness.  The two layers
are complementary and serve different purposes.  A sender MAY enable
both, neither, or either of them depending on profile requirements and
link reliability.

## Limitations

The CRC provides integrity (corruption detection) but NOT authentication.
An attacker can compute a valid CRC for a forged Datagram.  Authentication
must be provided by the underlying transport or a higher-layer security
mechanism.

# Content Mechanism Identification

The CMI field provides content mechanism identification.  The receiver
uses the CMI value to dispatch the Datagram to the correct decompressor
without inspecting the Datagram contents.

This is needed when a link carries Datagrams from multiple mechanisms
simultaneously.  Common scenarios include:

* A gateway that receives both standard schc-compressed Datagrams from
  legacy devices and uncompressed Datagrams from devices that do not
  support schc.
* Management and diagnostic traffic that bypasses compression entirely.
* Future registrations of additional compression mechanisms via new CMI
  values.

## Registration of New Mechanisms

Profiles that register a new CMI value MUST specify:

* The Content Mechanism handler (for example, Endpoint in the case of schc).
* The format, size, or encoding required to delineate the Reference field.

# Interaction with Protocol Numbers

The protocol numbers defined in {{I-D.ietf-schc-protocol-numbers}} are used to 
identify traffic on a link.  Two operational cases are possible:

**Case 1: Single schc Session, no multiplexing, no integrity, no versioning.**
The protocol number routes the frame directly to the sole schc Instance.
No LMX header is present.  Compressed residue starts immediately after the
carrier header.  Overhead beyond the carrier header is zero.

**Case 2: Multiplexing, integrity, versioning, or non-schc mechanisms are needed.**
The protocol number identifies LMX on the wire.  The LMX header follows
the carrier header and provides Session multiplexing, Content Mechanism
dispatch, and optional integrity and reference data.

~~~
   +------------------+
   |  Payload         |  (content mechanism determined by CMI)
   +------------------+
   |  LMX Header      |  (present in case 2; absent in case 1)
   +------------------+
   |  Carrier Header  |  (Ethertype / IP Protocol / UDP)
   +------------------+
   |  ...             |
   +------------------+
~~~
{: #fig-lmx-stack title="LMX Layer Stack"}

When LMX is not used (Case 1), the Payload is bare schc compressed residue
and the CMI is implicitly 0.  When LMX is used (Case 2), the CMI field
identifies the mechanism (CMI=0: schc; CMI=1: uncompressed; other values:
future registrations).

## Over Ethertype

The Ethertype identifies either bare schc (Case 1) or LMX (Case 2).
When LMX is present, the LMX header follows the Ethertype field.  When
O=1, the Original EtherType/Port field carries the replaced Ethertype
value.

## Over IP Protocol Number

The IP Protocol Number identifies either bare schc (Case 1) or LMX (Case
2).  When LMX is present, the LMX header follows the IPv6 header (or the
IPv6 extension containing the protocol number).  When O=1, the Original
EtherType/Port field carries the replaced value -- a UDP port number if
the stratum was L4, or an Ethertype if the stratum was L2.

## Over UDP

When the LMX UDP port is used, LMX or bare schc residue is carried in the
UDP payload.  The UDP header provides its own checksum, which may make the
LMX CRC redundant.  The UDP source port MAY carry the Session ID, in
which case LMX is not needed.  When LMX is present and O=1, the Original
EtherType/Port field carries the replaced UDP port number.

# Security Considerations

## Session Hijacking

If Session IDs are predictable, an attacker could inject Datagrams with a
forged Session ID to redirect traffic to a different handler.  Session IDs
SHOULD be randomly generated or derived from a secure key exchange.  In
star topologies where the Domain Manager assigns Session IDs, the assigned
values SHOULD be cryptographically random rather than sequential or
otherwise predictable.

## Integrity Limitations

The CRC provides corruption detection but not authentication.  An attacker
with link access can forge Datagrams with valid CRCs.  Authentication must
be provided by the underlying transport (for example, IPsec, TLS) or a
higher-layer mechanism (for example, OSCORE).

## Context Version Manipulation

A forged Context Version in the LMX header could cause a receiver to
reject valid Datagrams.  Context Version values are authenticated by the
Context distribution channel (for example, signed CORECONF operations),
not by LMX itself.  Deployments using unauthenticated Context distribution
(for example, CORECONF without TLS) should be aware that an attacker could
manipulate Context Version values in transit.

## Denial of Service

An attacker could inject Datagrams with invalid Session IDs, causing the
receiver to waste resources on lookup failures.  Implementations SHOULD
rate-limit Session ID lookup failures.

## Replay Attacks

LMX carries no sequence number or timestamp.  An attacker with link access
could replay previously captured Datagrams.  For schc's primary use cases
(sensor telemetry, periodic reporting), replayed Datagrams carry stale data
that is not harmful.  Deployments requiring replay protection SHOULD use a
higher-layer mechanism (for example, OSCORE, DTLS) or the underlying
transport.

# IANA Considerations

## Content Mechanism Identifier Registry

This document requests the creation of a "Content Mechanism Identifier (CMI)"
registry.  The initial entries are:

| Value | Content Mechanism                   | Reference     |
|-------|-------------------------------------|---------------|
| 0     | schc {{-schc}}                       | {{-schc}}    |
| 1     | Uncompressed / raw                  | This document |
| 2-15  | Reserved                            | --            |
{: #tab-iana-cmi title="Initial CMI registry entries"}

New CMI values are assigned per {{RFC8126}} "Specification Required"
policy.

## Session ID Space

The Session ID space is locally significant to the link.  No IANA assignment
is required.

## Future Extensions

The LMX header allows for future extensions.  New flags or fields would be
introduced through a subsequent revision of this document, with IANA
registry updates.  Existing implementations that encounter unrecognized
flag combinations MUST treat the unrecognized flags as zero and process
the header according to their supported flags.

--- back


