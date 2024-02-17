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
address_discovery (0x9f81a173) transport parameter ({{Section 7.4 of RFC9000}})
with an empty value. Implementations that understand this transport parameter
MUST treat the receipt of a non-empty value as a connection error of type
TRANSPORT_PARAMETER_ERROR.

When using 0-RTT, both endpoints MUST remember the value of this transport
parameter. This allows sending the frames defined by this extension in 0-RTT
packets. If 0-RTT data is accepted by the server, the server MUST NOT disable
this extension on the resumed connection.

# Frames

This extension defines three frames. These frames MUST only appear in the
application data packet number space. These frames are "probing frames" as
defined in {{Section 9.1 of RFC9000}}.

## REQUEST_ADDRESS

~~~
REQUEST_ADDRESS Frame {
    Type (i) = 0x9f81a0,
    Request ID (i),
}
~~~

The REQUEST_ADDRESS frame contains the following fields:

Request ID:

: A variable-length integer encoding the request identifier of the request.

REQUEST_ADDRESS frames are ack-eliciting. If lost, it's the sender's decision if
it wants to retransmit the frame. A sender MAY send a new REQUEST_ADDRESS frame
instead.

## REQUEST_DECLINED

~~~
REQUEST_DECLINED Frame {
    Type (i) = 0x9f81a1,
    Request ID (i),
}
~~~

The REQUEST_DECLINED frame contains the following fields:

Request ID:

: The request ID of the request that is being declined.

REQUEST_DECLINED frames are ack-eliciting, and SHOULD be retransmitted if lost.

## OBSERVED_ADDRESS

~~~
OBSERVED_ADDRESS Frame {
    Type (i) = 0x9f81a2..0x9f81a3,
    Request ID (i),
    [ IPv4 (32) ],
    [ IPv6 (128) ],
    Port (16),
}
~~~

The OBSERVED_ADDRESS frame contains the following fields:

Request ID:

: The request identifier of the request for which this response is intended.

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

An endpoint that wishes to determine the remote address of a path sends a
REQUEST_ADDRESS frame on that path. Since the REQUEST_ADDRESS frame is a
probing frame, the endpoint MAY bundle it with other probing frames during path
validation ({{Section 8.2 of RFC9000}}).

The receiver of the REQUEST_ADDRESS frame SHOULD report the observed address
using an OBSERVED_ADDRESS frame. The OBSERVED_ADDRESS frame does not need to be
sent on the same path, since the requester can associate the response with the
corresponding request using the request ID.

The receiver of a REQUEST_ADDRESS frame MAY decline to report the observed
address by sending a REQUEST_DECLINED frame. The REQUEST_DECLINED frame also
contains a request ID, and therefore may be sent on any path.

The sender MAY send a REQUEST_ADDRESS frame for the same path after a some time
has elapsed. This allows it to detect when a NAT rebinding has happened. To
speed up the discovery, it MAY also send another REQUEST_ADDRESS frame when the
peer changes the connection ID used on the path.

When receiving an OBSERVED_ADDRESS or a REQUEST_DECLINED frame with a request
ID value that was not previously sent in a REQUEST_ADDRESS frame, the receiver
MUST close the connection with a PROTOCOL_VIOLATION error code if it can
detect this condition.

# Security Considerations

## On the Requester Side

In general, nodes cannot be trusted to report the correct address in
OBSERVED_ADDRESS frames. If possible, endpoints might decide to only use this
extension when connecting to trusted peers, or if that is not possible, define
some validation logic (e.g. by asking multiple untrusted peers and observing if
the responses are consistent). This logic is out of scope for this document.

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
