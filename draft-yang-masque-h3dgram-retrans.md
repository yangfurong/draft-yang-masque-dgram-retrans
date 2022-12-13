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
title: "A Configurable Retransmission Extension for HTTP/3 Datagrams"
abbrev: "h3-dgram-retrans"
category: exp

docname: draft-yang-masque-h3dgram-retrans-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Transport"
workgroup: "MASQUE"
keyword:
 - Internet-Draft

author:
 -
   fullname: Furong Yang
   organization: Alibaba Inc.
   email: yfr256538@alibaba-inc.com
 -
   fullname: Yanmei Liu
   organization: Alibaba Inc.
   email: miaoji.lym@alibaba-inc.com
 -
   fullname: Yunfei Ma
   organization: Alibaba Inc.
   email: yunfei.ma@alibaba-inc.com
 -
   fullname: Qinghua Wu
   organization: ICT CAS
   email: wuqinghua@ict.ac.cn

normative:

informative:
   MASQUE-EVALUATION: DOI.10.1145/3488660.3493806
   PR:
       title: iCloud Private Relay Overview
       author:
          org: Apple Inc.
       date: 2021

--- abstract

When using HTTP/3 Datagrams for traffic tunneling, it is desirable to retransmit HTTP/3 Datagrams in some scenarios where the retransmission is beneficial for the tunneled end-to-end connection. This document defines an extension to the HTTP Datagrams and the Capsule Protocol, which allows HTTP/3 Datagrams to be retransmitted according to the configuration of the HTTP/3 Datagram flow.

--- middle

# Introduction {#intro}

HTTP Datagrams and the Capsule Protocol {{!HTTP-DATAGRAM=RFC9297}} defines how HTTP Datagrams can be sent either unreliably using the QUIC DATAGRAM extension {{!QUIC-DATAGRAM=RFC9221}} or reliably using the Capsule Protocol that encapsulates HTTP Datagrams into HTTP/2 {{?RFC7540}} streams, HTTP/3 {{?RFC9114}} streams or HTTP/1.x connections. The two modes, "reliable mode" and "unreliable mode", all have their pros and cons.

This document takes the scenario where HTTP Datagrams are leveraged to tunnel QUIC {{!QUIC=RFC9000}} connections from a QUIC client and a target QUIC server via an HTTP UDP proxy {{!CONNECT-UDP=RFC9298}} as a reference. However, the problems discussed below are not restricted to the reference scenario. Instead, the problems are general in other scenarios using HTTP Datagrams for traffic tunneling, e.g. {{!CONNECT-IP=I-D.ietf-masque-connect-ip-03}}.

In the reference scenario, the reliable mode is usually worse than the unreliable mode in terms of the transport performance of the end-to-end QUIC connection (i.e. the connection tunneled by the proxy). The culprit is that the stream-based Capsule Protocol can stall the end-to-end QUIC connection due to head-of-line blocking, which can inflate the RTT estimation of the end-to-end connection, make the connection perceive bursty losses, and hinder different streams of the connection from independent delivery. However, the reliable mode also has advantages sometimes. If the network path between the client and the UDP proxy is lossy and the end-to-end delay is a few times higher than the delay of the tunnel, the reliable mode can quickly recover the lost packets in the tunnel, hide the losses from the end-to-end connection, and avoid the reduction of the connection's congestion window. Some of the above behaviors were observed by a study {{MASQUE-EVALUATION}}.

This document defines an extension to the Capsule Protocol {{HTTP-DATAGRAM}}, which allows HTTP/3 Datagrams to be retransmitted according to the configuration of the HTTP/3 Datagram flow. In {{retx_signaling}}, a new Capsule Type is added to configure peers' retransmission limit of HTTP/3 Datagrams. Having such a signaling mechanism instead of just locally configuring the retransmission capability at endpoints (i.e. the client and the proxy) is necessary for enforcing retransmission policies in both upstream and downstream directions. As the proxy does not know the end-to-end connection's preference for retransmission, the client needs to inform the proxy what is the retransmission preference. Depending on the retransmission limit of HTTP/3 Datagrams, the handling of lost HTTP/3 Datagrams is discussed in {{retx_handling}}.

This extension brings the benefits of the reliable mode to the unreliable mode. It is beneficial for traffic tunneling scenarios where the last-mile link could be very lossy (e.g. Apple's iCloud Private Relay scenario {{PR}} where the last-mile link is usually wireless).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the notation from {{QUIC}} for the format of the new capsule definition. Where fields are encoded using the variable-length integer, they need not be encoded on the minimum number of bytes.

In this document, the term "UDP proxy" aligns with the definition in {{CONNECT-UDP}}, and the term "intermediary" refers to an HTTP intermediary as defined in {{Section 3.7 of !RFC9110}}.

The term "HTTP/3 Datagram flow" describes the HTTP/3 Datagrams associated with the same HTTP request, .e.g a Connect-UDP request {{CONNECT-UDP}} or a Connect-IP request {{CONNECT-IP}}.

# Negotiating The Extension Between Peers

Peers indicate support for this extension by including the boolean-valued Item Structured Field "DG-Retrans: ?1" in the HTTP Request and Response headers (See {{Section 3.3.6 of ?RFC8941}} for information about the boolean format.). Peers MUST NOT use any following mechanisms described by this extension unless the support is explicitly expressed.

# Signaling HTTP/3 Datagram Retransmission Limit {#retx_signaling}
This document defines a new Capsule Type SET_H3_DGRAM_RETX_LIMIT to communicate how many times an HTTP/3 Datagram can be retransmitted at most between peers. Note, the retransmission limit takes effect within the scope of an HTTP/3 Datagram flow.

The format of the SET_H3_DGRAM_RETX_LIMIT capsule is shown in {{capsule-format}}. It has the following fields:

Context ID: It is the Context ID defined in {{CONNECT-UDP}} or {{CONNECT-IP}}. It describes the effect scope of the capsule. It is optional. If the Capsule Type is 0xbb (tentative), the capsule has no Context ID field, and the retransmission limit applies to all contexts.

Retransmission Limit: It is the maximum retransmission number of an HTTP/3 Datagram.

~~~
SET_H3_DGRAM_RETX_LIMIT {
    Capsule Type (i) = 0xba..0xbb,
    Capsule Length (i),
    [Context ID (i)],
    Retransmission Limit (i),
}
~~~
{: #capsule-format title="SET_H3_DGRAM_RETX_LIMIT Format"}

When a peer that recognizes SET_H3_DGRAM_RETX_LIMIT capsules receives a SET_H3_DGRAM_RETX_LIMIT capsule, if it is using HTTP/3 Datagrams, it MUST start to retransmit lost HTTP/3 Datagrams until they are acknowledged or their retransmission limit specified in the capsule is reached. If the peer is an intermediary, it SHOULD NOT forward the capsule to the next hop, as the aim of retransmissions is to recover the lost packets at the probably lossy last-mile link between the client and the first hop proxy. If an intermediary does not recognize SET_H3_DGRAM_RETX_LIMIT capsules, it SHOULD forward the capsules without any modification for the future extensibility as suggested by {{HTTP-DATAGRAM}}.

Finding the best way to set the limit of retransmission is out of this document's scope. Nonetheless, a possible way to calculate the retransmission limit is as follows. Considering the reference scenario of this document (shown in {{fig-scenario}}), the client can set its local retransmission limit to floor(RTT2 / RTT1) and use the SET_H3_DGRAM_RETX_LIMIT capsule to set the proxy's retransmission limit to floor(RTT2 / RTT1). As the loss detection algorithm takes at least one RTT to detect a packet loss, this setting intends to only allow a lost packet to be retransmitted by the tunnel before it is retransmitted by the end-to-end QUIC connection. Note, the client can subtract RTT1 from the RTT of the end-to-end QUIC connection to get RTT2.

~~~
┌───────────────┐              ┌─────────┐            ┌───────────┐
│  QUIC Client  │    RTT1      │UDP Proxy│    RTT2    │QUIC Server│
├───────────────┼──────────────┤         ├────────────┤           │
│ MASQUE Client │              │         │            │           │
└───────────────┘              └─────────┘            └───────────┘
~~~
{: #fig-scenario title="The reference scenario"}

# Updating HTTP/3 Datagram Retransmission Limit

A peer can just send a new SET_H3_DGRAM_RETX_LIMIT capsule to update the retransmission limit of its peer if necessary. Note, the new limit will overwrite the old limit specified by a previous SET_H3_DGRAM_RETX_LIMIT capsule.

# Handling Lost HTTP/3 Datagrams {#retx_handling}

HTTP/3 Datagrams are encoded in QUIC DATAGRAM frames. As described in {{QUIC-DATAGRAM}}, QUIC MAY notify the sender upon a QUIC DATAGRAM frame is acknowledged or declared lost by the loss detection algorithm. This extension relies on the notifications of the acknowledgement and loss of QUIC DATAGRAM frames to handle the retransmission of lost HTTP/3 Datagrams.

A reference way of implementation is as follows. First, when the HTTP/3 Datagram layer calls the unreliable sending API of QUIC to send an HTTP/3 Datagram, it gets a connection-level unique ID (DATAGRAM_ID) from QUIC that corresponds to the underlying QUIC DATAGRAM frame. Then, if the retransmission limit is larger than zero, the HTTP/3 Datagram layer generates a record {id = DATAGRAM_ID, retx_times = 0} for the HTTP/3 Datagram. Afterwards, whether the HTTP/3 Datagram is acknowledged or declared lost, the HTTP/3 Datagram layer will get a corresponding notification. For the acknowledgement notification, the HTTP/3 Datagram layer just deletes the record. For the loss notification, the HTTP/3 Datagram layer retransmits the HTTP/3 Datagram and updates the id and retx_times of the record if the retransmission limit permits, otherwise, the record is deleted. Note, as QUIC holds the HTTP/3 Datagram as the payload of the QUIC DATAGRAM frame, the payload can be returned to the HTTP/3 Datagram layer for retransmission, which saves the HTTP/3 Datagram layer from buffering HTTP/3 Datagrams for retransmission.

# Security Considerations

This extension adds no additional considerations to those presented in {{HTTP-DATAGRAM}}.


# IANA Considerations

This document adds following entry to the "Hypertext Transfer Protocol (HTTP) Field Name Registry":

|       Header Field      |    Status  |   Reference   |
|-------------------------|------------|---------------|
|        DG-Retrans       |     Exp    | This document |
{: #tab-http-registry title="New HTTP Header Field"}

This document adds following entries to the "HTTP Capsule Types" registry:

|       Capsule Type      |    Value   | Specification |
|-------------------------|------------|-------------- |
| SET_H3_DGRAM_RETX_LIMIT | 0xba, 0xbb | This document |
{: #tab-capsule-registry title="New Capsule Type"}

--- back


# Contributors
{:numbered="false"}

TBD.

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
