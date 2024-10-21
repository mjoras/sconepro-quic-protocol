---
title: "A new QUIC version for network property communication"
abbrev: "QUIC for network property communication"
category: info

docname: draft-joras-sconepro-quic-protocol-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
author:
 -
    fullname: Matt Joras
    organization: Meta Platforms, Inc.
    email: matt.joras@gmail.com
 -
    fullname: Marcus Ihlar
    organization: Ericsson
    email: marcus.ihlar@ericsson.com

normative:

informative:

--- abstract

This document describes a new QUIC version. The proposed wire format and a set
of procedures can be used to communicate throughput advice between an endpoint
and an on-path network element. Throughput advice are sent in QUIC packets of
a new QUIC version. These QUIC packets are sent adjecent to established QUIC
version 1 and 2 connections, within the same UDP 4-tuple.


--- middle

# Introduction

This document describes SCONE, a protocol for the purpose of communicating
throughput advice from network elements to endpoints. SCONE uses QUIC
long header packets to send throughput advice in parallel to established
end-to-end QUIC connections.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# SCONE Packet Format

A SCONE packet consists of a QUIC long header optionally followed by
throughput advice fields:

~~~~~
SCONE Packet {
  Header Form (1) = 1,
  Fixed Bit (1) = 0,
  Throughput Advice Flag (1),
  Average Window Flag (1),
  Reserved Bits (4),
  Version (32),
  Destination Connection ID Length (8),
  Destination Connection ID (8..160),
  Source Connection ID Length (8),
  Source Connection ID (8..160),
  [Throughput Advice (32)],
  [Average Window (32)]
}
~~~~~

Header Form:

: The most significant bit (0x80) of byte 0 (the first byte) is
set to 1 to indicate a QUIC long header.

Fixed Bit:

: The next bit (0x40) of byte 0 is set to 1.

Throughput Advice Flag:

: The next bit (0x20) indicates the precense of a throughput advice field
in this packet.

Average Window Flag:

: The next bit (0x10) indicates the precense of average window field in
this packet.

Reserved Bits:

: These bits are reserved for future use and SHOULD be set to 0.

Version:

: This QUIC version uses the value 0x5509c337, which was chosen at random.

Destination Connection ID Length:

: The byte following the version contains the length in bytes of the Destination
  Connection ID field that follows it.  This length is encoded as an 8-bit
  unsigned integer.

Destination Connection ID:

: The Destination Connection ID field follows the Destination Connection ID
  Length field, which indicates the length of this field. A Destination
  Connection ID MUST be at least 8 bytes long.

Source Connection ID Length:

: The byte following the Destination Connection ID contains the length in bytes
  of the Source Connection ID field that follows it.  This length is encoded as
  an 8-bit unsigned integer.

Source Connection ID:

: The Source Connection ID field follows the Source Connection ID Length field,
  which indicates the length of this field. A Source Connection ID MUST be at
  least 8 bytes long.

Throughput Advice:

: The throughput advice is a 32bit unsigned integer representing maximum
sustainable throughput through the network element. Expressed in Kb/s.

Average Window:

: Indicates the duration over which the bitrate is enforced. Expressed in
milliseconds.

# Packet Protection
SCONE uses packet protection as defined for Initial packets in section 5 of
[QUIC-TLS].

SCONE packets do not have packet numbers, therefore nonces are created
by combining the initial Destination Connection ID with the Source Connection
ID of the packet. A sender MUST generate a Source Connection ID with a high
probability of being unique for each packet.

## Public Salt
A publicly known salt is used to derive the secrets, specifically
sconepro_salt=0x6784619005cadc9bb961ec4d31b76892eb1b567e.

## HKDF Labels
The labels used in [QUIC-TLS] to derive packet protection keys (Section 5.1),
header protection keys (Section 5.4) change from "quic key" to "quicscone key",
from "quic iv" to "quicscone iv", from "quic hp" to "quicscone hp", to meet the
guidance for new versions in Section 9.6 of that document.

# Communication Overview
The goal of SCONE is to provide a way to communicate throughput advice
between an on-path network device and a QUIC client endpoint, with the QUIC
client responsible for the initiation of that communication.

Before establishing the communication, a QUIC client usually establishes a
QUIC version 1 or 2 end-to-end connection as per RFC 9000. Once this is done,
the client opportunistically sends a SCONE packet destined to the same
endpoint IP address and port. This packet can be parsed by any capable network
element on the path. All capable elements are able to respond to the initial packet in a
similar fashion, by creating their own SCONE packets and sending them to the
QUIC client matching the IP/port tuple being utilized by the end-to-end QUIC
connection.

~~~
+--------+      +---------+       +--------+
|  QUIC  |      | Network |       |  QUIC  |
| Client |      | Element |       | Server |
+---+----+      +----+----+       +---+----+
    |                |                |
    +----------- QUICv1/v2 ---------->|
    |                |                |
    |---- SCONE ---->|---- SCONE ---->|
    |<--- SCONE -----|                |
    |                |                |
~~~

The QUIC client must be able to distinguish the end-to-end QUIC version 1 or 2
packets and SCONE packets. The QUIC server does not need to be SCONE-aware as
it will ignore the packet based on the (unknown) version number.

## Use of Connection IDs
SCONE packets contain both Source and Destination Connection IDs. A
client who initiates SCONE communication sets both Source and Destination
Connection IDs to randomly generated values. A network device that 'responds'
to a SCONE packet sets the Destination Connection ID to the value of
the Source Connection ID of the packet it responds to. The network device sets
the Source Connection ID to a randomly generated value.

# On Path Verification
Communication using this new QUIC version MUST only be done with network elements that can be
verified to be on the same network path as an end to end QUIC flow. This is
because this communication is only meant to be done with network elements
that have the ability to, for example, modify and drop packets relevant to an
end-to-end QUIC flow. As QUIC packets for this new version are themselves carried in
separate UDP datagrams from the end to end QUIC flow, there is not an inherent
guarantee that they were generated by a network element.

A capable network device MUST set the Destination Connection ID Length and
Destination Connection ID fields to the values received in the most recently
observed new version QUIC packet sent by a client.


# Extensibility To Provide Confidentiality and Authenticity
The use of keys derived from a publicly known salt does not allow for
confidentiality or authenticity of the communication. The only manner of
authenticity defined in this document is verification of the on-path nature of
a network element. It may be desirable for this version of QUIC to be extended to allow for
confidentiality and authenticity.

Confidentiality with this protocol could be achieved by further leveraging the
provisions of QUIC version 1 to do a TLS handshake between the QUIC
client and a capable network element. Authenticity could similarly
leverage the provisions of TLS. However, this comes with significant
complications. TLS achieves authenticity by using Public Key Infrastructure
(PKI), where each participant can choose to trust the certificate offered by
their peer. While this PKI exists today for Internet endpoints, there is no
such existing PKI for network elements. It is important to note that conducting
a TLS handshake would restrict the communication between the QUIC client and
exactly one on-path network element.

Alternatively, a network element could advertise a set of hosts to which the
client can connect using QUIC version 1.

# Security Considerations

TODO Security


# IANA Considerations

TBD - QUIC version.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
