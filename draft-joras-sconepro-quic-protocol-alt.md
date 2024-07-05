---

title: "SCONEPRO QUIC Protocol"
abbrev: "SCONEPRO QUIC Protocol"
category: info

docname: draft-joras-sconepro-quic-protocol-latest
submissiontype: IETF # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
keyword:

- next generation
- unicorn
- sparkling distributed ledger
  author:
- fullname: Matt Joras
  organization: Meta Platforms, Inc.
  email: matt.joras@gmail.com
- fullname: Marcus Ihlar
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

# Version Field

The version field of Long Headers in this QUIC version is 0x5509c337, which was
a value chosen at random.

# Long Header Packet Format

This version of QUIC only uses long header packets with the following format:

Long Header Packet {
Header Form (1) = 1,
Fixed Bit (1) = 0,
Packet Type (2),
Reserved (4),
Version (32),
Destination Connection ID Length (8),
Destination Connection ID (0..160),
Source Connection ID Length (8),
Source Connection ID (0..160),
Payload (..),
}

**Header Form:** The most significant bit (0x80) of byte 0 (the first byte) is
set to 1 for long headers.
**Fixed Bit:** The next bit (0x40) of byte 0 is set to 1.
**Packet Type** The next two bits contain a packet type. A single packet type
is defined for this protocol with the value 0b00. Future extensions MAY add
new packet types. A network device MUST ignore packets with unknown packet
types and SHOULD forward such packets.

# Initial Salt

This version of QUIC uses an altered initial salt to derive the Initial Keys,
specifically initial_salt=0x6784619005cadc9bb961ec4d31b76892eb1b567e

# HKDF Labels

The labels used in [QUIC-TLS] to derive packet protection keys (Section 5.1),
header protection keys (Section 5.4) change from "quic key" to "quicscone key",
from "quic iv" to "quicscone iv", from "quic hp" to "quicscone hp", to meet the
guidance for new versions in Section 9.6 of that document.

# Frames

This version of QUIC defines a set of frames that carry information related to
network properties.

## Padding

## Data

## Upgrade

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
SCONEPRO-capable devices on the path can see its contents. All SCONEPRO-capable elements
are able to respond in a similar fashion, by creating their own SCONEPRO QUIC
packets and sending it to the SCONEPRO QUIC client matching the IP/port tuple
being utilized by the end to end QUIC connection.

The SCONEPRO QUIC client MUST be able to distinguish the end to end QUIC
packets and the SCONEPRO QUIC packets. This can be done by looking for the
pattern of the SCONEPRO packet, combined with trial decryption.

# On Path Verification

TODO: describe connection ID use

# Extensibility To Provide Confidentiality and Authenticity

TODO describe QUIC V1 alternative.

# Security Considerations

TODO Security

# IANA Considerations

QUIC version.

--- back

# Acknowledgments

{:numbered="false"}

TODO acknowledge.
