---
title: Tunneling transport protocols inside QUIC
abbrev: QUIC Tunnel
docname: draft-piraux-quic-tunnel-00
date: 2019-09-06
category: exp

ipr: trust200902
area: General
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
    ins: O. Bonaventure
    name: Olivier Bonaventure
    organization: Tessares
    email: Olivier.Bonaventure@tessares.net

normative:
  RFC2119:

informative:
  I-D.ietf-lpwan-ipv6-static-context-hc:


--- abstract

This document specifies methods for tunneling transport protocols inside
 a QUIC connection, including TCP, UDP, IP and QUIC flows.

--- middle

# Introduction

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

## Mutiple access networks in 5G networks

## Hybrid access networks for residential gateways

# Benefits of using QUIC

QUIC introduces several new features to the transport layer. In addition to
being a secure transport protocol with authenticated and encrypted header and
payload, QUIC brings several features of interest.

# Approaches for conveying Internet services

## Transparent proxying

As QUIC offers a reliable and in-order bytestream service, one could proxy and
terminate a TCP connection. Its data can then be sent over a QUIC stream.
The QUIC stream must start with a special header indicating the TCP original
destination.

This approach has advantages and drawbacks. On the one hand, by terminating the
connection closer the user, the losses in the access networks can be quickly
detected and recovered from. The tunnel has to deliver the TCP bytestream
It also lowers the encapsulation overhead by only including a small header
before the bytestream.

On the other hand, terminating the TCP connection breaks the end-to-end
principle. It introduces ossification in the network and slows down the
deployment of new TCP extensions.

## Packet tunneling

Packet tunneling is the traditional approach used by many VPNs and tunnels.
In this approach, each network protocol is only considered on a per-packet
basis. The packets are fully preserved when sent and received through the
tunnel.

This approach respects the end-to-end principle and thus does not introduce
ossification. New transport protocols extensions can be negotiated in an
end-to-end manner.

This approach results in a higher encapsulation overhead, but it can be mitigated
using header compression techniques such as {{I-D.ietf-lpwan-ipv6-static-context-hc}}.

Reliable Internet services require the QUIC connection to adapt, i.e. to
transport the packets in an unreliable manner. Layering reliable transport
protocols leads to the well-known meltdown problem in which the retransmissions
mechanism of the different layers can severerly interfere with each other.

## One-to-One QUIC tunnel connection

The simplest approach when conveying an Internet service over QUIC would be to map a
single connection to a single QUIC connection. This approach does not take
advantage of the multiples streams feature of QUIC.

## N-to-One QUIC tunnel connection

When a client is using many Internet services through the QUIC tunnel, one could
convey several of them inside a single QUIC connection. By leveraging the
multiple streams feature of QUIC, this approach reduces the fixed cost of
establishing a QUIC connection.

Moreover, bundling several Internet connections inside a QUIC connection
provides a more global point of view on the client network performances. It
allows to take more informed decisions during packet scheduling or congestion
control.

TODO(mp): The preceding paragraph is not clear

# Tunneling methods

Internet services rely on several network protocols. In this section we present
methods to tunnel the most ubiquitous ones.

TODO: Introduce DATAGRAM frames

## Tunneling TCP connections

### TCP to QUIC stream mapping

~~~~~~~~~~~~~~~~~~~~~~~~~~~
Client          QUIC Tunnel          QUIC Tunnel          Server
     ----------->
         SYN
     <-----------
        SYN+ACK
     ----------->
       ACK, Req           ===========>
     <-----------           Initial            ----------->
         ACK             0-RTT[Hdr,Req]            SYN
                                               <-----------
                                                  SYN+ACK
                                               ----------->
                                                 ACK, Req
                                               <-----------
                          <===========             Resp
                           1-RTT[Resp]
     <---------
        Resp

~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #tcp-proxy-conn title="Establishing a new QUIC tunnel connection when
proxying a TCP connection"}


~~~~~~~~~~~~~~~~~~~~~~~~~~~
+------------------+------------------------|
|        TCP       |      QUIC Stream       |
+------------------+------------------------+
| SYN received     | Open Stream, send Hdr  |
| FIN received     | Send Stream FIN        |
| RST received     | Send STOP_SENDING      |
|                  | Send RESET_STREAM      |
| Data received    | Send Stream data       |
+------------------+------------------------+

+-------------------------------+-----------+
|     QUIC Stream               |    TCP    |
+-------------------------------+-----------+
| Stream opened, Hdr received   | Send SYN  |
| Stream FIN received           | Send FIN  |
| STOP_SENDING received         | Send RST  |
| RESET_STREAM received         | Send RST  |
| Stream data received          | Send data |
+-------------------------------+-----------+

~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #tcp-proxy-stream title="TCP connection to QUIC stream mapping"}


### TCP segments tunneling

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
