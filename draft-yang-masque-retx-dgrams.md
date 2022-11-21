---
title: "A Configurable Retransmission Extension for HTTP Datagrams"
abbrev: "TODO - Abbreviation"
category: info

docname: draft-yang-masque-retx-dgrams-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Transport"
workgroup: "Multiplexed Application Substrate over QUIC Encryption"
keyword:
 - Internet-Draft

author:
 -
    fullname: Furong Yang
    organization: Alibaba Group
    email: yfr256538@alibaba-inc.com

normative:

informative:
  MASQUE-EVALUATION: DOI.10.1145/3488660.3493806

--- abstract

TODO Abstract


--- middle

# Introduction {#intro}

HTTP Datagrams and the Capsule Protocol {{!HTTP-DATAGRAM=RFC9297}} defines how HTTP Datagrams can be sent either unreliably using the QUIC DATAGRAM extension {{!QUIC-DATAGRAM=RFC9221}} or reliably using the Capsule Protocol that encapsulates HTTP Datagrams into HTTP/2 {{?RFC7540}} streams, HTTP/3 {{?RFC9114}} streams or HTTP/1.x connections. The two modes, "reliable mode" and "unreliable mode", all have their pros and cons.

This document takes the scenario where HTTP Datagrams are leveraged to tunnel QUIC {{!QUIC=RFC9000}} connections from a QUIC client and a target QUIC server via an HTTP UDP proxy {{!CONNECT-UDP=RFC9298}} as a reference. However, the problems discussed below are not restricted to the reference scenario. Instead, the problems are general in other scenarios using HTTP Datagrams for traffic tunneling, e.g. {{?CONNECT-IP=I-D.ietf-masque-connect-ip-03}}.

 In the reference scenario, the reliable mode is usually worse than the unreliable mode in term of the transport performance of the end-to-end QUIC connection. The culprit is that the stream-based Capsule Protocol can lead the end-to-end QUIC connection to head-of-line blocking, which can inflate the RTT estimation of the end-to-end connection, make the connection perceive bursty losses, and prevent different  streams of the connection to be independently delivered. However, the reliable mode also has advantages sometimes. If the network path between the client and the UDP proxy is lossy and the end-to-end delay is a few times higher than the delay of the tunnel, the reliable mode can quickly recover the packet losses in the tunnel, hide the losses from the end-to-end connection, and avoid the reduction of the connection's congestion window. Some of the above behaviors were observed by a study {{MASQUE-EVALUATION}}.

This document defines an extension to the Capsule Protocol {{HTTP-DATAGRAM}}, which allows HTTP/3 Datagrams to be retransmitted according to the configuration of the HTTP/3 Datagram flow. In {{retx_signalling}}, a new Capsule Type is added to configure peers' retransmission limit of HTTP/3 Datagrams. Having the retransmission limit of HTTP/3 Datagrams, the handling of lost HTTP/3 Datagrams is discussed in {{retx_handling}}.  This extension brings the benefits of the reliable mode to the unreliable mode.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

The definition format of new capsules uses the notation from {{QUIC}}. Where fields within types are integers, they are encoded using the variable-length integer encoding from {{QUIC}}. Integer values do not need to be encoded on the minimum number of bytes necessary.
In this document, the term "UDP proxy" aligns with the definition in {{CONNECT-UDP}}, and the term "intermediary" refers to an HTTP intermediary as defined in {{Section 3.7 of !RFC9110}}.

The term "HTTP/3 Datagram flow" describes the HTTP/3 Datagrams associated with the same HTTP request, .e.g a Connect-UDP request {{CONNECT-UDP}} or a Connect-IP request {{CONNECT-IP}}.

# Signalling HTTP/3 Datagram Retransmission Limit {#retx_signalling}
This document defines a new Capsule Type SET_H3_DGRAM_RETX_LIMIT to communicate how many times an HTTP/3 Datagram can be retransmitted at most between peers. Note, the retransmission limit takes effect within the scope of an HTTP/3 Datagram flow.

The format of the SET_H3_DGRAM_RETX_LIMIT capsule is shown in {{capsule-format}}. It has the following fields:

Context ID: It is the Context ID defined in {{CONNECT-UDP}} or {{CONNECT-IP}}. It describes the effect scope of the capsule. It is optional. If the Capsule Type is 0xbb (tentative), the capsule has no Context ID field, and the retransmission limit applies to all contexts.

Retransmission Limit: It is the maximum retransmission number of an HTTP/3 Datagram.

Hop Limit: It defines how many hops the capsule can be forwarded to intermediaries.

~~~
SET_H3_DGRAM_RETX_LIMIT {
    Capsule Type (i) = 0xba..0xbb,
	Capsule Length (i),
    [Context ID (i)],
    Retransmission Limit (i),
    Hop Limit (i),
}
~~~
{: #capsule-format title="SET_H3_DGRAM_RETX_LIMIT Format"}

When a peer that recognizes SET_H3_DGRAM_RETX_LIMIT capsules receives a SET_H3_DGRAM_RETX_LIMIT capsule, if it is using HTTP/3 Datagrams, it MUST start to retransmit lost HTTP/3 Datagrams until they are acknowledged or their retransmission limit specified in the capsule is reached. If the peer is an intermediary, it MUST decrease the Hop Limit of the capsule by one and forward the capsule to the next hop if the decreased Hop Limit is still large than zero. If an intermediary does not recognize SET_H3_DGRAM_RETX_LIMIT capsules, it SHOULD forward the capsules without any modification for the future extensibility as suggested by {{HTTP-DATAGRAM}}.

# Updating HTTP/3 Datagram Retransmission Limit

A peer can just send a new SET_H3_DGRAM_RETX_LIMIT capsule to update the retranmission limit of its downstream nodes. Note, the new limit will overwrite the old limit specified by a previous SET_H3_DGRAM_RETX_LIMIT capsule.

# Handling Lost HTTP/3 Datagrams {#retx_handling}

HTTP/3 Datagrams are encoded in QUIC DATAGRAM frames. As described in {{QUIC-DATAGRAM}}, QUIC MAY notify the sender upon a QUIC DATAGRAM frame is acknowledged or declared lost by the loss detection algorithm. This extension relies on the notifications of the acknowledgement and loss of QUIC DATAGRAM frames to handle the retransmission of lost HTTP/3 Datagrams.

A reference way of implementation is as follows. First, when the HTTP/3 Datagram layer calls the unreliable sending API of QUIC to send an HTTP/3 Datagram, it gets a connection-level unique ID (DATAGRAM_ID) from QUIC that corresponds to the underlying QUIC DATAGRAM frame. Then, if the retransmission limit is larger than zero, the HTTP/3 Datagram layer generates a record {id = DATAGRAM_ID, retx_times = 0} for the HTTP/3 Datagram. Afterwards, whether the HTTP/3 Datagram is acknowledged or declared lost, the HTTP/3 Datagram layer will get a corresponding notification. For the acknowledgement notification, the HTTP/3 Datagram layer just deletes the record. For the loss notification, the HTTP/3 Datagram layer retransmits the HTTP/3 Datagram and updates the id and retx_times of the record if the retransmission limit permits, otherwise, the record is deleted. Note, as QUIC holds the HTTP/3 Datagram as the payload of the QUIC DATAGRAM frame, the payload can be returned to the HTTP/3 Datagram layer for retranmission, which saves the HTTP/3 Datagram layer from buffering HTTP/3 Datagrams for retransmission.

# Security Considerations

This extension adds no additional considerations to those presented in {{HTTP-DATAGRAM}}.


# IANA Considerations

This document adds following entries to the "HTTP Capsule Types" registry:

|       Capsule Type      |    Value   | Specification |
|-------------------------|------------|-------------- |
| SET_H3_DGRAM_RETX_LIMIT | 0xba, 0xbb | This document |

{: #tab-capsule-registry title="SET_H3_DGRAM_RETX_LIMIT Capsule Type"}

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
