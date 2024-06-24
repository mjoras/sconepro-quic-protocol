---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "SCONEPRO QUIC Protocol"
abbrev: "SCONEPRO QUIC Protocol"
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

This document describes the wire protocol and procedures that can be used to achieve endpoint to network communication of properties, for the purposes of video self adaptation by a content endpoint.


--- middle

# Introduction

The basic idea of SCONEPRO is to utilize similar technology to QUIC [RFC 9000], employing a second independent flow between a client endpoint and devices in the network. This independent flow uses a separate version of QUIC. This document will not describe what information is exchanged for these properties, but rather the overall way in which the communication functions.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# New QUIC Version

In QUIC version 1 defined in RFC 9000, there are different packet types for different phases of the connection. “Long headers” are used for sending messages that are part of the TLS handshake. These packets are encrypted but not private and use publicly-known keys. They also have an explicit packet length. After the handshake is completed endpoints switch to using “short header” packets which are encrypted privately. For SCONEPRO we define a new version of QUIC which exclusively uses Long header packets. For the purposes of this document these packets will also exclusively be encrypted using public keys. Note that a further extension allowing communication with private keys and short header packets is feasible.

# Version Field
The version field of Long Headers in this QUIC version is 0x5509c337, which was a value chosen at random.

# Long Header Packet Types
This version of QUIC for now only specifies the long header packet type of Initial, unchanged from QUIC version 1 as 0b00. Long Header packet types for Handshake may be added if a full TLS handshake mechanism is added to this version of QUIC.

# Initial Salt
This version of QUIC uses an altered initial salt to derive the Initial Keys, specifically initial_salt=0x6784619005cadc9bb961ec4d31b76892eb1b567e

# HKDF Labels
The labels used in [QUIC-TLS] to derive packet protection keys (Section 5.1), header protection keys (Section 5.4) change from "quic key" to "quicscone key", from "quic iv" to "quicscone iv", from "quic hp" to "quicscone hp", to meet the guidance for new versions in Section 9.6 of that document.

# Stream and Frame Usage
This version of QUIC does not utilize the vast majority of frames used in QUIC version one. The main one that is needed are STREAM frames as defined in section 19.8 of RFC 9000. Unlike RFC 9000, there is only a single stream that is valid in this version of QUIC, with stream ID of 0, initiated by the client. There is no flow control associated with this stream. This document does not define the contents of the stream.

# Communication Overview
The goal of SCONEPRO is to provide a way to communicate properties between an on-path network device and a QUIC client endpoint, with the QUIC client responsible for the initiation of that communication.

Before establishing the SCONEPRO communication, a QUIC client establishes its normal end to end connection as per usual from RFC 9000. Once this is done, the client opportunistically sends a SCONEPRO QUIC Initial destined to the same endpoint IP address and port. This Initial can be be parsed by any SCONEPRO-capable network element on the path. A SCONEPRO-capable elements SHOULD forward these packets in the normal fashion, such that all SCONEPRO-capable devices can see its contents. All SCONEPRO-capable elements are able to respond in a similar fashion, by creating their own SCONEPRO QUIC packets and sending it to the SCONEPRO QUIC client matching the IP/port tuple being utilized by the end to end QUIC connection.

The SCONEPRO QUIC client MUST be able to distinguish the end to end QUIC packets and the SCONEPRO QUIC packets. This can be done by looking for the pattern of the SCONEPRO packet, combined with trial decryption.

# On Path Verification

SCONEPRO communication MUST only be done with network elements that can be verified to be on the same network path as an end to end QUIC flow. This is because SCONEPRO communication is only meant to be done with network elements that have the ability to, for example, modify and drop packets relevant to an end to end QUIC flow. As SCONEPRO QUIC packets are themselves carried in separate UDP datagrams from the end to end QUIC flow, there is not an inherent guarantee that they were generated by a network element. To combat this without requiring a full TLS handshake, SCONEPRO network element MUST include a sample of a SCONEPRO initial from the QUIC client. This gives the SCONEPRO QUIC client credible reason that the information received was generated by an element on the forwarding path. This is accomplished through SAMPLE frames.

# Sample Frame
A sample frame is a very simple construct meant to contain the first N bytes of the entropy (i.e. the encrypted portion) of a SCONEPRO initial. A SCONEPRO-capable network element SHOULD send at least 32 bytes from one of these packets.

SAMPLE Frame {
  Type (i) = 0xe3754d5b,
  Length (i),
  Sample Data (..),
}

To verify the data in a SAMPLE frame, a SCONEPRO QUIC client must track the first 32 bytes of the SCONEPRO initials it sent to initiate the communication, to compare against the SAMPLE frames received from the network element.



# Extensibility To Provide Confidentiality and Authenticity
The use of QUIC initials explicitly does not allow for confidentiality or authenticity of the communication. The only manner of authenticity defined in this document is verification of the on-path nature of a network element. It may be desirable for SCONEPRO to be extended to allow for confidentiality and authenticity.

Confidentiality with this protocol could be achieved by further leveraging the provisions of QUIC version 1 to do a TLS handshake between the SCONEPRO QUIC client and a SCONEPRO-capable network element. Authenticity could similarly leverage the provisions of TLS. However, this comes with significant complications. TLS achieves authenticity by using Public Key Infrastructure (PKI), where each participant can choose to trust the certificate offered by their peer. While this PKI exists today for Internet endpoints, there is no such existing PKI for network elements. It is important to note that conducting a TLS handshake would restrict the communication between the QUIC client and exactly one on-path network element.

# Security Considerations

TODO Security


# IANA Considerations

QUIC version.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
