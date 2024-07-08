---
title: "SCONEPRO QUIC Protocol"
abbrev: "SCONEPRO QUIC Protocol"
category: info

docname: draft-joras-sconepro-quic-protocol-alt-latest
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

This document describes a wire format and a set of procedures used to
communicate network properties between a content endpoint and an on-path
device. The network properties are intended to enable self-adaptation of video
media rates by content endpoints.

The wire format in this document is defined as a new version of QUIC that
adheres to the version-independent properties of QUIC specified in RFC 8999.
Messages are sent adjacent to already established QUIC version 1 or version 2
connections on the same UDP 4-tuple. This version of QUIC uses long headers for
all its communication, and packet protection is achieved using publicly known
keys.


--- middle

# Introduction

The basic idea of SCONEPRO is to use an independent flow between a client
endpoint and devices in the network, in parallel to an end-to-end QUIC
connection [RFC 9000], to exchange network properties. This independent flow
uses a separate version of QUIC. This document will not describe what
information is exchanged for these properties, but rather the overall way in
which the communication functions.

The version of QUIC defined in this document protects packets using publicly
known keys, which means that confidentiality and integrity of protocol payload
cannot be guaranteed. The document describes how a connection can be upgraded
to a QUIC version that supports full TLS protection.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# New QUIC Version

RFC 8999 defines the version independent properties of QUIC to consist of
long and short header packets and version negotiation packets. This document
defines a new version of QUIC which exclusively uses Long header packets.

This version of QUIC has a single packet type, and a set of frame types
used to communicate network properties. Furthermore, packets are protected
using publicly-know keys, similar to the way Initial packets are protected
in QUIC version 1.

# SCONEPRO QUIC Packets

This version of QUIC only uses long header packets with the following format:

~~~~~
Long Header Packet {
  Header Form (1) = 1,
  Fixed Bit (1) = 0,
  Packet Type (2) = 0,
  Reserved Bits (4),
  Version (32),
  Destination Connection ID Length (8),
  Destination Connection ID (8..160),
  Source Connection ID Length (8),
  Source Connection ID (8..160),
  Payload (..),
}
~~~~~

Header Form:

: The most significant bit (0x80) of byte 0 (the first byte) is
set to 1 for long headers.

Fixed Bit:

: The next bit (0x40) of byte 0 is set to 1.

Packet Type:

: The next two bits contain a packet type. A single packet type
is defined for this protocol with the value 0b00. Future extensions MAY add
new packet types. A network device MUST ignore packets with unknown packet
types and SHOULD forward such packets.

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

# Packet Protection
This version of QUIC uses packet protection as defined for Initial packets in
section 5 of [QUIC-TLS].

This version of QUIC does not use packet numbers, therefore nonces are created
by combining the intial destination connection ID with the source connection ID
of the packet. Each SCONEPRO QUIC packet mus use a randomly generated source
connection ID with a high probability of being unique.

## Public Salt
A publicly known salt is used to derive the secrets, specifically
sconepro_salt=0x6784619005cadc9bb961ec4d31b76892eb1b567e.

## HKDF Labels
The labels used in [QUIC-TLS] to derive packet protection keys (Section 5.1),
header protection keys (Section 5.4) change from "quic key" to "quicscone key",
from "quic iv" to "quicscone iv", from "quic hp" to "quicscone hp", to meet the
guidance for new versions in Section 9.6 of that document.

# Frames
The payload of a SCONEPRO QUIC packet consists of a sequence of frames. Frames
consist of a type field optionally followed by type specific payload.

## Padding Frame

Padding frames are defined in section 19.1 of [RFC9000] and are used to
increase the size of a packet.

~~~
PADDING Frame {
  Type (i) = 0x00,
}
~~~

## Data Frame

~~~
DATA Frame {
  Type (i) = 0x01,
  Length (i),
  Payload (..),
}
~~~

Length:

:  The length of the DATA Frame payload in bytes.

Payload:

:  SCONEPRO specific payload such as media bitrate information.

## Alternative Hosts Frame

Used to communicate alternative endpoints. This can be used to
send a new request of QUIC version but to the network devices
IP address and port instead of using the same 4-tuple than the
corresponsing end-to-end QUIC connection. Alternative it can also
be used to setup a full QUIC version 1 connection from the client to the nextwork
device to obtain confidentiality and authentication of the communication. 

~~~
Alternative Hosts Frame {
  Type (i) = 0x02,
  Host (..)..,
}
~~~

Host:

: A tuple consisting of a host name, port and a QUIC protocol version. (TODO
define format)

# Communication Overview
The goal of SCONEPRO is to provide a way to communicate properties between an
on-path network device and a QUIC client endpoint, with the QUIC client
responsible for the initiation of that communication.

Before establishing the SCONEPRO communication, a QUIC client establishes its
normal end to end connection as per usual from RFC 9000. Once this is done, the
client opportunistically sends a SCONEPRO QUIC packet destined to the same
endpoint IP address and port. This packet can be be parsed by any
SCONEPRO-capable network element on the path. A SCONEPRO-capable elements
MAY forward these packets in the normal fashion, such that all
SCONEPRO-capable devices can see its contents. All SCONEPRO-capable elements
are able to respond in a similar fashion, by creating their own SCONEPRO QUIC
packets and sending it to the SCONEPRO QUIC client matching the IP/port tuple
being utilized by the end to end QUIC connection.

The SCONEPRO QUIC client MUST be able to distinguish the end to end QUIC
packets and the SCONEPRO QUIC packets. This can be done by looking for the
pattern of the SCONEPRO packet, combined with trial decryption.

## Use of Connection IDs
SCONEPRO QUIC packets contain both Source and Destination Connection IDs. A
client who initiates SCONEPRO communication sets both Source and Destination
Connection IDs to randomly generated values. A network device that 'responds'
to a SCONEPRO QUIC packet sets the Destination Connection ID to the value of
the Source Connection ID of the packet it responds to. The network device sets
the Source Connection ID to a randomly generated value.

# On Path Verification

SCONEPRO communication MUST only be done with network elements that can be
verified to be on the same network path as an end to end QUIC flow. This is
because SCONEPRO communication is only meant to be done with network elements
that have the ability to, for example, modify and drop packets relevant to an
end to end QUIC flow. As SCONEPRO QUIC packets are themselves carried in
separate UDP datagrams from the end to end QUIC flow, there is not an inherent
guarantee that they were generated by a network element.

A SCONEPRO network device MUST set the Destination Connection ID Length and
Destination Connection ID fields to the values received in the most recently
observed SCONEPRO QUIC packet sent by a client.


# Extensibility To Provide Confidentiality and Authenticity
The use of keys derived from a publicly known salt does not allow for
confidentiality or authenticity of the communication. The only manner of
authenticity defined in this document is verification of the on-path nature of
a network element. It may be desirable for SCONEPRO to be extended to allow for
confidentiality and authenticity.

Confidentiality with this protocol could be achieved by further leveraging the
provisions of QUIC version 1 to do a TLS handshake between the SCONEPRO QUIC
client and a SCONEPRO-capable network element. Authenticity could similarly
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

QUIC version.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
