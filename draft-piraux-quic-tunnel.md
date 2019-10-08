---
title: Tunneling Internet protocols inside QUIC
abbrev: QUIC Tunnel
docname: draft-piraux-quic-tunnel-00
date: 2019-09-06
category: exp

ipr: trust200902
area: Transport
workgroup: TODO Working Group
keyword: Internet-Draft

coding: us-ascii
stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: "M. Piraux"
    name: "Maxime Piraux"
    organization: "UCLouvain"
    role: editor
    email: maxime.piraux@uclouvain.be

 -
    ins: "O. Bonaventure"
    name: "Olivier Bonaventure"
    organization: "UCLouvain"
    email: olivier.bonaventure@uclouvain.be

normative:
  RFC2119:

informative:
  I-D.ietf-lpwan-ipv6-static-context-hc:
  I-D.pauly-quic-datagram:
  I-D.deconinck-quic-multipath:
  I-D.ietf-tcpm-converters:
  RFC3095:
  RFC3843:
  RFC4019:
  RFC4815:
  RFC6846:
  CoNEXT:
    author:
      - ins: Q. De Coninck
      - ins: O. Bonaventure
    title: "Multipath QUIC: Design and Evaluation"
    seriesinfo: Proceedings of the 13th International Conference on emerging Networking EXperiments and Technologies (CoNEXT 2017)
    date: December 2017
  SIGCOMM19:
    author:
      - ins: Q. De Coninck
      - ins: F. Michel
      - ins: M. Piraux
      - ins: T. Given-Wilson
      - ins: A. Legay
      - ins: O. Pereira
      - ins: O. Bonaventure
    title: Pluginizing QUIC
    seriesinfo: Proceedings of the ACM Special Interest Group on Data Communication
    date: August 2019

--- abstract

This document specifies methods for tunneling Internet protocols inside
 a QUIC connection, including TCP, UDP, IP and QUIC flows.

--- middle

# Introduction

Mobile devices such as laptops, smartphones or tablets have different
requirements than the traditional fixed devices. These mobile devices
often change their network attachment. They are often attached to
trusted networks, but sometimes they need to be connected to untrusted
networks where their communications can be eavesdropped, filtered or
modified. In these situations, the classical approach is to rely on VPN
protocols such as DTLS, TLS or IPSec. These VPN protocols provide the
encryption and authentication functions to protect those mobile clients
from malicious behaviours in untrusted networks.

On the other hand, these devices are often multihomed and many expect to
be able to perform seamless handovers from one access network to another
without breaking the established VPN sessions. In some situations it can
also be beneficial to combine two or more access networks together to
increase the available host bandwidth. A protocol such as Multipath TCP
supports those handovers and allows to aggregate the bandwidth of
different access links. It could be combined with single-path VPN
protocols to support both seamless handovers and bandwidth aggregation
above VPN tunnels. Unfortunately, Multipath is not yet deployed on most
Intenet servers and thus few applications would benefit from such a use
case.

The QUIC protocol opens up a new way to find a clean solution to this
problem. First, QUIC includes the same encryption and authentication
techniques as deployed VPN protocols. Second, QUIC is intended to be
widely used to support web-based services, making it unlikely to be
filtered in many networks, in contrast with VPN protocols. Third, the
multipath extensions proposed for QUIC enable it to efficiently support
both seamless handovers and bandwidth aggregation.

In this document, we explore how (Multipath) QUIC could be used to
enable multipath mobile devices to communicate securely in untrusted
networks. More precisely, we explore and compare two different designs.
The first, section {{the-datagram-mode}}, uses the recently proposed datagram extension for
QUIC to transport plain IP packets over a Multipath QUIC connection. The
second, section {{the-stream-mode}}, uses the QUIC streams to transport TCP bytestreams over a
Multipath QUIC connection.

Our starting point for this work is Multipath QUIC that was initially
proposed in {{CoNEXT}}. A detailed specification of Multipath QUIC may be
found in {{I-D.deconinck-quic-multipath}}. Two implementations of different version of this
protocol are available {{CoNEXT}}, {{SIGCOMM19}}.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Reference environment

We consider a multihomed client that establishes a Multipath QUIC
connection to a concentrator. Once the connection has been established,
it is used to carry all the data traffic sent by the client. The
Multipath QUIC connection ensures that the client data is protected
against attacks in one or both of the access networks. The client trusts
the concentrator. The concentrator decrypts the frames exchanged over
the Multipath QUIC connection and interacts with the remote hosts as a
VPN concentrator would do.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
            +---------+
       .----| Access  |----.
       |    | network |    |
       |    |    A    |    |
+--------+  +----------    v                          +-------------+
| Multi  |              +--------------+              | Final       |
| homed  |              | Concentrator |===\ ... \===>| destination |
| client |              +--------------+              | server      |
+--------+  +---------+    ^                          +-------------+
       |    | Access  |    |
       |    | network |    |            Legend:
       .----|    B    |----.              --- Multipath QUIC subflow
            +---------+                   === Original tunneled flow
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-reference-environment title="Reference environment"}

TODO(mp): The figure illustrate the client to server connection example, but not
the reverse

# The datagram mode

Our first mode of operation, called the datagram mode in this document,
enables the client and the concentrator exchange raw IP packets through
the Multipath QUIC connection. This is done by using the recently
proposed QUIC datagram extension {{I-D.pauly-quic-datagram}}.
In a nutshell, to send an IP packet to a remote host, the client simply
passes the entire packet as a datagram to the Multipath QUIC connection
established with the concentrator. The packet is encoded in a QUIC frame,
then encrypted and authenticated in a QUIC packet. This transmission is
subject to congestion control, but the datagram that contains the packet is
not retransmitted in case of losses as specified in {{I-D.pauly-quic-datagram}}.

The datagram mode is intended to provide a similar service as the one
provided by IPSec tunnels or DTLS.

TODO(ob): Maybe look at MTU issues, because those will appear

# The stream mode

The main advantage of the datagram mode is that it supports IP and any
protocol above the network layer. Any IP packet can be transported using
the datagram extension over a Multipath QUIC connection. However, this
advantage comes with a large per-packet overhead since each packet
contains a network and a transport header. All these headers must be
transmitted in addition with the IP/UDP/QUIC headers of the Multipath
QUIC connection. For TCP connections, the overhead can be large.

Since QUIC support multiple parallel streams, another possibility to
carry TCP connections between the client and the concentrator is to
transport the bytestream of each connection as one of the streams of the
Multipath QUIC connection. For this, we base our approach on the 0-RTT Converter
protocol ({{I-D.ietf-tcpm-converters}}) that was proposed to each the
deployment of TCP extensions. In a nutshell, it is an application proxy that
converts TCP connections, allowing the use of new TCP extensions while avoiding
inducing extra delay in the process.

We use a similar approach in our stream mode. When a client (or the
concentrator) opens a stream, it sends at the beginning of the
bytestream a series of TLV messages that indicate the IP address and
port number of the remote destination of the bytestream. Upon reception
of these TLV messages, the concentrator (or the client) opens a TCP
connection towards the specified destination and connects the incoming
bytestream of the Multipath QUIC connection to the bytestream of the new
TCP connection (and similarly in the opposite direction).
{{tcp-proxy-stream}} summarises how the new TCP connection mapped to the
QUIC stream.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
+------------------+-------------------------+
|        TCP       |      QUIC Stream        |
+------------------+-------------------------+
| SYN received     | Open Stream, send TLVs  |
| FIN received     | Send Stream FIN         |
| RST received     | Send STOP_SENDING       |
|                  | Send RESET_STREAM       |
| Data received    | Send Stream data        |
+------------------+-------------------------+

+-------------------------------+------------+
|         QUIC Stream           |    TCP     |
+-------------------------------+------------+
| Stream opened, TLVs received  | Send SYN   |
| Stream FIN received           | Send FIN   |
| STOP_SENDING received         | Send RST   |
| RESET_STREAM received         | Send RST   |
| Stream data received          | Send data  |
+-------------------------------+------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #tcp-proxy-stream title="TCP connection to QUIC stream mapping"}

Actions and events of the TCP state machine are mapped to the QUIC stream state
 machine as illustrated in {{tcp-proxy-stream}}.

The QUIC stream-level flow control can be tuned to match the terminated TCP
connection receive window size, so that no excessive amount of data is buffered
inside the tunnel.

A timeout can be associated with a given mapped QUIC stream for its associated
state to expire when the TCP connection is inactive for a long period.

# TODO(mp): Integrate the following text if needed

TODO(mp): The 1-to-1 vs N-to-1 paragraphs are interesting imo.

# Benefits of using QUIC

QUIC introduces several new features to the transport layer. Being able to
tunnel Internet services into QUIC offers a smooth upgrade path for legacy
applications.
In the context of a QUIC tunnel operating in the use cases presented in ..,
those applications can benefit from several features.

The additional QUIC encryption layer masks the conveyed protocols and prevent,
 up to a certain degree, their pervasive monitoring. Zero RTT QUIC connection
establishment allows the tunnel to be established without impacting the
conveyed service latency. QUIC connection migration allows
decoupling the conveyed Internet service from the network layer. For instance,
tunneled TCP flows can be migrated from a 5G base station to another by
migrating the QUIC tunnel connection.

# Approaches for conveying Internet services

Several directions can be taken when tunneling Internet services. First, we
consider the scope of the QUIC tunnel, i.e. at the service connection level
({{transparent-proxy}}) or at the service packet level ({{packet-tunneling}}).
Second, we consider whether only a single Internet service can be conveyed in a
QUIC tunnel or whether multiple services can share the same QUIC tunnel
connection ({{one-to-one}} and {{n-to-one}}).

## Packet tunneling {#packet-tunneling}

Packet tunneling is the traditional approach used by many VPNs and tunnels.
In this approach, each Internet protocol is considered on a per-packet
basis. The packets are fully preserved when sent and received through the
tunnel.

This approach respects the end-to-end principle and thus does not introduce
ossification. New transport protocols extensions can be negotiated in an
end-to-end manner.

This approach results in a higher encapsulation overhead, but it can be mitigated
using advanced header compression techniques such as
ROHC ({{RFC3095}}) and SHCH ({{I-D.ietf-lpwan-ipv6-static-context-hc}}).

Reliable Internet services require the QUIC connection to adapt, i.e. to
transport their packets unreliably. Layering reliable transport
protocols leads to the well-known meltdown problem in which the retransmissions
mechanism of the different layers can severly interfere with each other.

## Transparent proxying {#transparent-proxy}

As QUIC offers a reliable and in-order bytestream service, a
connection-oriented, non-authenticated, transport protocol could be terminated
by the tunnel and proxied over QUIC. Its bytestream can then be sent over a QUIC
 stream. The QUIC stream must start with a special header indicating the
original destination and could contain other protocol-specific control data.
For instance, proxying a TCP connection requires to indicate the destination
address and port.

This approach has advantages and drawbacks. On the one hand, by terminating the
connection closer the user, it can be established rapidly and losses in the
access networks can be quickly detected and recovered from. It also lowers the
encapsulation overhead by only including a small header before the bytestream.

On the other hand, terminating a transport procotol connection breaks the
end-to-end principle. It introduces ossification in the network and slows down
the deployment of new transport protocols and extensions.

## One-to-One QUIC tunnel connection {#one-to-one}

The simplest approach when conveying an Internet service over QUIC would be to
map a single connection to a single QUIC connection. This approach does not take
advantage of the multiples streams and out-of-order packet processing features
of QUIC.

## N-to-One QUIC tunnel connection {#n-to-one}

When a client is using many Internet services through the QUIC tunnel, one could
convey several of them inside a single QUIC connection. By leveraging the
multiple streams feature of QUIC, this approach reduces the fixed cost of
establishing and using a QUIC connection.

Moreover, bundling several Internet connections inside a QUIC connection
provides a more global point of view on the client network performances. It
allows to take more informed decisions during packet scheduling or congestion
control.

TODO(mp): The preceding paragraph is not clear

# Tunneling methods

Internet services rely on several protocols. In this section we present
a flexible method to tunnel the most ubiquitous ones.

QUIC version 1 provides reliable and in-order bytestreams in the form of QUIC streams,
which can be either unidirectional or bidirectional. An alternative view of
QUIC unidirectional streams is an elastic message.
In addition, the DATAGRAM extension {{I-D.pauly-quic-datagram}} provide a
datagram mode for QUIC in the form of DATAGRAM frames. They are simple
unreliable messages that must fit the size of a QUIC packet.

In the following sections, we explain how connection-oriented protocols can
benefit from QUIC streams and how other protocols can be conveyed using DATAGRAM
frames.

## Tunneling connection-oriented protocols over QUIC streams

In this sub-section, we explain a method for proxying and conveying
non-authenticated, connection-oriented protocols such as TCP over a QUIC tunnel.
 When terminating a connection, additional informations must be provided in
order for the other peer to create a new connection to the final destination.
For instance, proxying a TCP connection requires to communicate the destination
address and port to use.

To this effect, we propose a flexible format to encode protocol header fields.
This protocol can be used specify header fields for IPv4 and IPv6 as well as any
 IP protocols.

TODO(mp): Rephrase so that it's clear we mean protocols running above IP, not
new IP protocols

~~~~~~~~~~~~~~~~~~~~~~~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Fields (8)  |              [Header Fields (*)]            ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #quic-tunnel-headers-block title="QUIC Tunnel Headers Block"}

A number of header fields can be sent using an Headers Block as illustrated in
{{quic-tunnel-headers-block}}. Those header fields allow exchanging fields of
the network and transport header used by the terminated connection. A Headers
Block can be placed at the beginning of the QUIC stream and followed by the
connection bytestream.

It consists of the following fields:

Fields:

: A 8-bit value indicating the number of Header Fields in the Headers
Block.

Header Fields:

: A sequence of Header Field.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| IP Next Hdr(8)|                  Field ID (i)               ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Length (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            Value (*)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #quic-tunnel-header-block title="QUIC Tunnel Header Field"}

An Header Field consists of four fields:

IP Next Hdr:

: A Internet Protocol number designating the protocol which this header field
 is part of.

TODO(mp): How to represent IPv4 and v6 ? 0x04 (IP in IP) and 0x29 (IPv6
Encapsulation) ?

Field ID:

: A varint-encoded number designating the particular field in the protocol. The
field is identified by its relative position in the header. For instance, The first field
is represented by the value 0, the second by the value 1.

Length:

: The length of the Value field is varint-encoded in the Length field.

Value:

: The field value bytes are placed at the end of the Header Field.

TODO(mp): What is the implication of not maintaining IP connectivity, with for
instance FTP or others ?

## Tunneling other protocols using DATAGRAM frames

The unreliable message service provided by DATAGRAM frames allows tunneling
other protocols over QUIC. It can also benefit to connection-oriented protocols
that can be conveyed on a per-packet basis in an end-to-end manner and thus
without introducing ossification. We discuss both cases in this subsection.

We propose to convey unreliable protocols, e.g. IP and UDP, over a flow of
DATAGRAM frames. Each tunneled packet could be conveyed over QUIC unidirectional
streams but this adds un-necessary complexity in the form of retransmissions
and stream state handling.

Similarly, we propose to convey reliable transport protocols on a per-packet
basis over DATAGRAM frames. Using QUIC unidirectional streams has the same
disadvantages. Moreover, the layering of retransmissions mechanisms could lead
to the *TCP meltdown* problem.

Each packet of a tunneled flow is encapsulated inside the payload of a DATAGRAM
frame. We consider each packet as starting from the network layer, such that the
packet can then be forwarded at the exit of the QUIC Tunnel. The QUIC Tunnel
must be able to advertise the correct MTU for the flows entering the tunnel to
adapt their packet sizes.

TODO(mp): This implies several assumptions on the client address and the tunnel
placement in the network.

# Security Considerations

TODO Security

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
