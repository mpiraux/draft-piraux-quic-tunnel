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

--- abstract

This document specifies methods for tunneling Internet protocols inside
 a QUIC connection, including TCP, UDP, IP and QUIC flows.

--- middle

# Introduction

Access networks evolve rapidly. 5G access networks and hybrid access networks
are two examples of their constant evolution. These networks claim to offer
more performance to the users, but they first require the transport layer to
 adapt. On the one hand, experience with the deployment of TCP extensions
showed that new extensions can take years to get wide adoption. On the other
hand, experience with SCTP showed that deploying a new transport protocol is
hard (rephrase).

As a result, small and medium size network application developers teams rarely
revisit their choice in terms of transport layer uses. Moreover, transport
protocols are usually implemented in the kernel and are used through API
abstractions, which slows down the innovation in this layer. To benefit from
those new access networks, applications developers have to wait for kernel
developers to implement the required extensions and then they have to update
their own applications.

QUIC has been designed as an userspace transport protocol atop UDP to address
some of these issues. It also brings new features that can leverage these new access
networks, such as connection migration. Its built-in and extensive encryption
combined with its clean extensibility model brings back innovation in the
transport layer. But application developers still have to update their
application to benefit from QUIC.

Using a QUIC tunnel to transparently convey Internet services accross those
new access networks provides a smooth upgrade-path for legacy application. In
this document, we specifies methods for tunneling Internet protocols inside a
QUIC connection, including TCP, UDP, IP and QUIC flows.

TODO Introduction

TODO Motivate the need of a proxy (e.g. in 4G/5G, in hybrid network access)
to provide a smooth upgrade-path for legacy applications

TODO Introduce QUIC

TODO Introduce Multipath QUIC

TODO Layout the document organisation

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Use cases

## Multiple access networks in 5G networks

## Hybrid access networks for residential gateways

# Benefits of using QUIC

QUIC introduces several new features to the transport layer. Being able to
tunnel Internet services into QUIC offers a smooth upgrade path for legacy
applications.
In the context of a QUIC tunnel operating in the use cases presented in
{{use-cases}}, those applications can benefit from several features.

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

## Transparent proxying {#transparent-proxy}

As QUIC offers a reliable and in-order bytestream service, one could proxy and
terminate a TCP connection. Its data can then be sent over a QUIC stream.
The QUIC stream must start with a special header indicating the TCP original
destination.

This approach has advantages and drawbacks. On the one hand, by terminating the
connection closer the user, it can be established rapidly and the losses in the
access networks can be quickly detected and recovered from. It also lowers the
encapsulation overhead by only including a small header before the bytestream.

On the other hand, terminating the TCP connection breaks the end-to-end
principle. It introduces ossification in the network and slows down the
deployment of new TCP extensions.

## Packet tunneling {#packet-tunneling}

Packet tunneling is the traditional approach used by many VPNs and tunnels.
In this approach, each Internet protocol is only considered on a per-packet
basis. The packets are fully preserved when sent and received through the
tunnel.

This approach respects the end-to-end principle and thus does not introduce
ossification. New transport protocols extensions can be negotiated in an
end-to-end manner.

This approach results in a higher encapsulation overhead, but it can be mitigated
using advanced header compression techniques such as
{{I-D.ietf-lpwan-ipv6-static-context-hc}}.

Reliable Internet services require the QUIC connection to adapt, i.e. to
transport their packets in an unreliable manner. Layering reliable transport
protocols leads to the well-known meltdown problem in which the retransmissions
mechanism of the different layers can severerly interfere with each other.

## One-to-One QUIC tunnel connection {#one-to-one}

The simplest approach when conveying an Internet service over QUIC would be to map a
single connection to a single QUIC connection. This approach does not take
advantage of the multiples streams feature of QUIC.

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
methods to tunnel the most ubiquitous ones.

QUIC version 1 provides reliable and in-order bytestreams in the form of QUIC streams,
which can be either unidirectional or bidirectional. An alternative view of
QUIC unidirectional streams is an elastic message. The loss of a QUIC packet
containing QUIC streams triggers a retransmission. This added complexity is
un-needed in the case of a QUIC tunnel.

We propose to use a lighter abstraction to convey Internet packets. The DATAGRAM
extensions {{I-D.pauly-quic-datagram}} provide a datagram mode for QUIC.
DATAGRAM frames are simple messages that must fit the size of a QUIC packet.

## Transport layer tunneling

We choose to encapsulate the Internet services at the transport layer and not at
the network layer. IP Options are seldom used today and thus maintaining
end-to-end IP connectivity is not addressed by this document.

IP datagrams can be created by the QUIC tunnel when receiving traffic from its
peer. We propose to use the following flow header to exchange the required
information to create the correct IP datagrams, and in turn to forward the
received traffic to the correct peer, we propose to use the following flow
header.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Flow ID (i)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                  Destination IP Address (128)                 |
|                                                               |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Destination Port (16)     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #quic-tunnel-flow-header title="QUIC Tunnel Flow Header"}

The Flow Header associates a given ID with a destination address and port.
Mechanisms for exchanging and using the Flow Header are detailed in the
following sub-sections on a per-protocol basis.

The address is an IPv6 address. IPv4 addresses can be encoded as defined in
TODO.

TODO(mp): What is the implication of not maintaining IP connectivity, with for
instance FTP or others ?

## Tunneling TCP connections

TCP is a connection-oriented transport protocol. We consider two approaches for
tunneling TCP connections. The first is to terminate and map a given TCP
connection to a QUIC stream. The second is to convey the TCP segments as is and
keep its connection end-to-end.

### TCP to QUIC stream mapping

We propose a TCP connection to QUIC stream mapping when considering the
tunneling of TCP at the connection level.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
Client          QUIC Tunnel          QUIC Tunnel          Server
     ----------->
         SYN
     <-----------
        SYN+ACK
     ----------->
       ACK, Req           ===========>
     <-----------           Initial            ----------->
         ACK            0-RTT[F.Hdr,Req]           SYN
                                               <-----------
                                                  SYN+ACK
                                               ----------->
                                                 ACK, Req
                                               <-----------
                          <===========             Resp
                           1-RTT[Resp]
     <---------
        Resp

------- TCP                                        ======== QUIC
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #tcp-proxy-conn title="Establishing a new QUIC tunnel connection when
proxying a request/response TCP connection"}

{{tcp-proxy-conn}} illustrates how a new TCP connection is conveyed inside a
QUIC tunnel. The TCP connection is terminated at the entrance of the tunnel. A
new TCP connection is created at its exit. A Flow Header is placed at the start
of the stream. Then the TCP connection bytestream is carried over a QUIC stream.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
+------------------+-------------------------+
|        TCP       |      QUIC Stream        |
+------------------+-------------------------+
| SYN received     | Open Stream, send F.Hdr |
| FIN received     | Send Stream FIN         |
| RST received     | Send STOP_SENDING       |
|                  | Send RESET_STREAM       |
| Data received    | Send Stream data        |
+------------------+-------------------------+

+-------------------------------+------------+
|         QUIC Stream           |    TCP     |
+-------------------------------+------------+
| Stream opened, F.Hdr received | Send SYN   |
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
state to expire when the TCP connection is inactive for a long period of time.

### TCP segments tunneling

We propose to tunnel TCP segments inside DATAGRAM frames.
When segments of a new TCP connection are conveyed, the QUIC tunnel
must send a Flow Header to its peer for it to correctly handle the segments.

Two approaches can be taken. In the first one, the header can be placed before the DATAGRAM
payload. Then, when the DATAGRAM is acknowledged, subsequent DATAGRAMs from
this flow can replace the Flow Header used with its Flow ID solely.
The disadvantage is that the MTU available is lowered first, then raised upon
Flow Header acknowledgment.

In the second approach, TCP segments that are sent before acknowledgement of a
Flow Header are conveyed in unidirectional streams. The Flow Header is placed at
the beginning of the stream. Each stream can be reset as soon as its content as
been transmitted in order to avoid retransmissions. Once a stream has been
fully acknowledged, the QUIC tunnel can switch to transmitting DATAGRAMs with
the corresponding Flow ID. This does not introduce MTU fluctuations but is more
costly for the QUIC tunnel.

## Tunneling UDP flows

## Tunneling QUIC connections

## Tunneling IP flows

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
