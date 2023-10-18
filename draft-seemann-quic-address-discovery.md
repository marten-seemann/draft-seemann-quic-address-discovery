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
    organization: Protocol Labs
    email: "martenseemann@gmail.com"

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

1. STUN traffic is unencrypted, and can be observed by on-path observers. Moving
   address discovery into QUIC's encrypted envelope, it becomes invisible to
   observers.
2. When located behind a load balancer, QUIC packets may be routed based on the
   QUIC connection ID. Depending on the architecture, not using STUN might
   simplify the routing.
3. If QUIC traffic doesn't need to be demultiplexed from STUN traffic,
   implementations can enable QUIC bit greasing ({{!RFC9287}}).


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Negotiating Extension Use {#negotiate-extension}

Endpoints advertise their support of the extension by sending the
address_discovery (0x9f81a173) transport parameter ({{Section 7.4 of RFC9000}})
with an empty value. Implementations that understand this transport parameter
MUST treat the receipt of a non-empty value as a connection error of type
TRANSPORT_PARAMETER_ERROR.

The client MUST remember the value of this transport parameter. This allows
sending the frames defined by this extension in 0-RTT packets. If 0-RTT data is
accepted by the server, the server MUST NOT disable this extension on the
resumed connection.

# Frames

This extension defines three frames. These frames MUST only appear in the
application data packet number space. These frames are "probing frames" as
defined in {{Section 9.1 of RFC9000}}, which allows sending these frames when
probing a new path without migrating to that path.

## REQUEST_ADDRESS

~~~
REQUEST_ADDRESS Frame {
    Type (i) = 0x9f81a0,
    Sequence Number (i),
}
~~~

The REQUEST_ADDRESS frame contains the following fields:

Sequence Number:

: The sequence number of the request.

REQUEST_ADDRESS frames are ack-eliciting. If lost, it's the sender's decision if
it wants to retransmit the frame. A sender MAY send a new REQUEST_ADDRESS frame
with an incremented sequence number instead.

## REQUEST_DECLINED

~~~
REQUEST_DECLINED Frame {
    Type (i) = 0x9f81a0,
    Sequence Number (i),
}
~~~

The REQUEST_DECLINED frame contains the following fields:

Sequence Number:

: The sequence number of the request that is being declined.

REQUEST_DECLINED frames are ack-eliciting, and SHOULD be retransmitted if lost.

## OBSERVED_ADDRESS

~~~
OBSERVED_ADDRESS Frame {
    Type (i) = 0x9f81a2..0x9f81a3,
    Sequence Number (i),
    [ IPv4 (32) ],
    [ IPv6 (128) ],
    Port (16),
}
~~~

The OBSERVED_ADDRESS frame contains the following fields:

Sequence Number:

: The sequence number of the request for which this response is intended.

IPv4:

: The IPv4 address. Only present if the least significant bit of the frame type
  is 0.

IPv6:

: The IPv6 address. Only present if the least significant bit of the frame type
  is 1.

Port:

: The port number, in network byte order.

OBSERVED_ADDRESS frames are ack-eliciting, and SHOULD be retransmitted if lost.

# Address Discovery

An endpoint that wishes to determine its remote address of a path sends a
REQUEST_ADDRESS frame on that path. Since the frame is a probing frame, the
endpoint MAY bundle the frame with other probing frames during path validation
({{Section 8.2 of RFC9000}}). The receiver of the REQUEST_ADDRESS frame SHOULD
report the observed address using an OBSERVED_ADDRESS frame. Since this frame
doesn't need to be sent on the same path, since the sender can associate the
response with the corresponding request using the sequence number.

The receiver of a REQUEST_ADDRESS frame MAY decline to report the observed
address by sending a REQUEST_DECLINED frame on any path.

When receiving an OBSERVED_ADDRESS or a REQUEST_DECLINED frame with a sequence
number value that was not sent in a REQUEST_ADDRESS frame before, the receiver
SHOULD close the connection with a PROTOCOL_VIOLATION error code.

# Security Considerations

## On the Requester Side

In general, nodes cannot be trusted to report the correct address in
OBSERVED_ADDRESS frames. If possible, endpoints might decide to only trusted
peers, or if that is not possible, define some validation logic (e.g. by asking
multiple untrusted peers and observing if the responses are consistent).
This kind of logic is out of scope for this document.

## On the Responder Side

Depending on the routing setup, a node might not be able to observe the peer's
reflexive transport address, and attempts to do so might reveal details about
the internal network. In these cases, the node SHOULD NOT enable support the
extension, or decline reporting the address using REQUEST_DECLINED frames.

# IANA Considerations

TODO: fill out registration request for the transport parameter and frame types

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
