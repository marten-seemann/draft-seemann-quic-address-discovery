---
title: "QUIC Address Discovery"
abbrev: "QUIC Address Discovery"
category: std

docname: draft-seemann-quic-address-discovery-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Transport"
workgroup: "QUIC"
keyword:
 - QUIC
 - STUN
 - Address Discovery
venue:
  group: "QUIC"
  type: "Working Group"
  mail: "quic@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/quic/"
  github: "marten-seemann/draft-seemann-quic-address-discovery"
  latest: "https://marten-seemann.github.io/draft-seemann-quic-address-discovery/draft-seemann-quic-address-discovery.html"
stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    fullname: "Marten Seemann"
    email: "martenseemann@gmail.com"
 -
    fullname: "Christian Huitema"
    org: "Private Octopus Inc."
    email: "huitema@huitema.net"


normative:

informative:


--- abstract

Unless they have out-of-band knowledge, QUIC endpoints have no information about
their network situation. They neither know their external IP address and port,
nor do they know if they are directly connected to the internet or if they are
behind a NAT. This QUIC extension allows nodes to determine their public IP
address and port for any QUIC path.


--- middle

# Introduction

STUN ({{!RFC8489}}) allows nodes to discover their reflexive transport address
by asking a remote server to report the observed source address. While the QUIC
({{!RFC9000}}) packet header was designed to allow demultiplexing from STUN
packets, moving address discovery into the QUIC layer has a number of
advantages:

1. STUN traffic is unencrypted, and can be observed and modified by on-path
   observers. By moving address discovery into QUIC's encrypted envelope it
   becomes invisible to observers.
2. When located behind a load balancer, QUIC packets may be routed based on the
   QUIC connection ID. Depending on the architecture, not using STUN might
   simplify the routing logic.
3. If QUIC traffic doesn't need to be demultiplexed from STUN traffic,
   implementations can enable QUIC bit greasing ({{!RFC9287}}).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Negotiating Extension Use {#negotiate-extension}

Endpoints advertise their support of the extension by sending the
address_discovery (0x9f81a175) transport parameter ({{Section 7.4 of RFC9000}})
with a variable-length integer value. The value determines the behavior with
respect to address discovery:

* 0: The node is willing to provide address observations to its peer, but is not
  interested in receiving address observations itself.
* 1: The node is interested in receiving address observations, but it is not
  willing to provide address observations.
* 2: The node is interested in receiving address observations, and it is willing
  to provide address observations.

Implementations that understand this transport parameter MUST treat the receipt
of any other value than these as a connection error of type
TRANSPORT_PARAMETER_ERROR.

When using 0-RTT, both endpoints MUST remember the value of this transport
parameter. This allows sending the frame defined by this extension in 0-RTT
packets. If 0-RTT data is accepted by the server, the server MUST NOT disable
this extension or change the value on the resumed connection.

# Frames

This extension defines the OBSERVED_ADDRESS frame.

## OBSERVED_ADDRESS

~~~
OBSERVED_ADDRESS Frame {
    Type (i) = 0x9f81a4..0x9f81a5,
    Sequence Number (i),
    [ IPv4 (32) ],
    [ IPv6 (128) ],
    Port (16),
}
~~~

The OBSERVED_ADDRESS frame contains the following fields:

Sequence Number:
: A variable-length integer specifying the sequence number assigned for
  this OBSERVED_ADDRESS frame. The sequence
  number MUST be monotonically increasing for OBSERVED_ADDRESS frame in the same connection.
  Frames may be received out of order. A peer SHOULD ignore an incoming
  OBSERVED_ADDRESS frame if it previously received another OBSERVED_ADDRESS frame
  for the same path with a Sequence Number equal to or higher than the
  sequence number of the incoming frame.

IPv4:

: The IPv4 address. Only present if the least significant bit of the frame type
  is 0.

IPv6:

: The IPv6 address. Only present if the least significant bit of the frame type
  is 1.

Port:

: The port number, in network byte order.

This frame MUST only appear in the application data packet
number space. It is a "probing frame" as defined in {{Section 9.1 of RFC9000}}.
OBSERVED_ADDRESS frames are ack-eliciting, and SHOULD be retransmitted if lost.
Retransmissions MUST happen on the same path as the original frame was sent on.

An endpoint MUST NOT send an OBSERVED_ADDRESS frame to a node that did not
request the receipt of address observations as described in
{{negotiate-extension}}. A node that did not request the receipt of address
observations MUST close the connection with a PROTOCOL_VIOLATION error if it
receives an OBSERVED_ADDRESS frame.

# Address Discovery

An endpoint that negotiated (see {{negotiate-extension}}) this extension and
offered to provide address observations to the peer MUST send an
OBSERVED_ADDRESS frame on every new path. This also applies to the path used for
the QUIC handshake.

The OBSERVED_ADDRESS frame SHOULD be sent as early as possible. However, during
the handshake an endpoint SHOULD prioritize frames that lead to handshake
progress (CRYPTO and ACK frames in particular) over sending of the
OBSERVED_ADDRESS frame.

For paths used after completion of the handshake, endpoints SHOULD bundle the
OBSERVED_ADDRESS frame with probing packets. This is possible, since the frame
is defined to be a probing frame ({{Section 8.2 of RFC9000}}).

Additionally, the sender SHOULD send an OBSERVED_ADDRESS frame when it detects a
change in the remote address on an existing path. This could be indicative of a
NAT rebindings.

# Security Considerations

## On the Requester Side

In general, nodes cannot be trusted to report the correct address in
OBSERVED_ADDRESS frames. If possible, endpoints might decide to only request
address observations when connecting to trusted peers, or if that is not
possible, define some validation logic (e.g. by asking multiple untrusted peers
and observing if the responses are consistent). This logic is out of scope for
this document.

## On the Responder Side

Depending on the routing setup, a node might not be able to observe the peer's
reflexive transport address, and attempts to do so might reveal details about
the internal network. In these cases, the node SHOULD NOT offer to provide
address observations.

# IANA Considerations

TODO: fill out registration request for the transport parameter and frame types

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
