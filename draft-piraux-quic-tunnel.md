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
  RFC4291:
  RFC6890:

informative:
  I-D.ietf-lpwan-ipv6-static-context-hc:
  I-D.pauly-quic-datagram:
  I-D.deconinck-quic-multipath:
  I-D.ietf-tcpm-converters:
  I-D.ietf-quic-transport:
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

This document specifies methods for tunneling Internet protocols such
 as TCP, UDP, IP and QUIC inside a QUIC connection.

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
from malicious behaviors in untrusted networks.

On the other hand, these devices are often multihomed and many expect to
be able to perform seamless handovers from one access network to another
without breaking the established VPN sessions. In some situations it can
also be beneficial to combine two or more access networks together to
increase the available host bandwidth. A protocol such as Multipath TCP
supports those handovers and allows to aggregate the bandwidth of
different access links. It could be combined with single-path VPN
protocols to support both seamless handovers and bandwidth aggregation
above VPN tunnels. Unfortunately, Multipath is not yet deployed on most
Internet servers and thus few applications would benefit from such a use
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
found in {{I-D.deconinck-quic-multipath}}. Two implementations of different versions of this
protocol are available {{CoNEXT}}, {{SIGCOMM19}}.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Reference environment

We consider a multihomed client that is attached to two different
access networks. It establishes a Multipath QUIC
connection to a concentrator. This MPQUIC connection is used to carry
all the UDP and TCP packets sent by the client. Thanks to the security
mechanisms used by the Multipath QUIC connection, all the client data
is protected against attacks in one or both of the access networks. The client trusts
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
            +---------+                   === TCP/UDP flow
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-reference-environment title="Reference environment"}

In this version of the document, we focus on client initiated flows. A
subsequent version will discuss how the client can accept incoming UDP
flows and TCP connections.

# The datagram mode

Our first mode of operation, called the datagram mode in this document,
enables the client and the concentrator exchange raw IP packets through
the Multipath QUIC connection. This is done by using the recently
proposed QUIC datagram extension {{I-D.pauly-quic-datagram}}.
In a nutshell, to send an IP packet to a remote host, the client simply
passes the entire packet as a datagram to the Multipath QUIC connection
established with the concentrator. The IP packet is encoded in a QUIC frame,
encrypted and authenticated in a QUIC packet. This transmission is
subject to congestion control, but the datagram that contains the packet is
not retransmitted in case of losses as specified in {{I-D.pauly-quic-datagram}}.

The datagram mode is intended to provide a similar service as the one
provided by IPSec tunnels or DTLS. As IP packets are encoded in QUIC frames.

TODO(ob): Maybe look at MTU issues, because those will appear
-> move in another part of the document that discusses more details

# The stream mode

The main advantage of the datagram mode is that it supports IP and any
protocol above the network layer. Any IP packet can be transported
using the datagram extension over a Multipath QUIC connection. However, this
advantage comes with a large per-packet overhead since each packet
contains both a network and a transport header. All these headers must be
transmitted in addition with the IP/UDP/QUIC headers of the Multipath
QUIC connection. For TCP connections, the per-packet overhead can be large.

Since QUIC support multiple streams, another possibility to
carry the data exchanged over TCP connections between the client and the concentrator is to
transport the bytestream of each TCP connection as one of the streams of the
Multipath QUIC connection. For this, we base our approach on the 0-RTT Converter
protocol ({{I-D.ietf-tcpm-converters}}) that was proposed to ease the
deployment of TCP extensions. In a nutshell, it is an application proxy that
converts TCP connections, allowing the use of new TCP extensions
through an intermediate relay.

We use a similar approach in our stream mode. When a client opens a stream, it sends at the beginning of the
bytestream one or more TLV messages that indicate the IP address and
port number of the remote destination of the bytestream. Their format is
detailed in section {{sec-format}}. Upon reception of such a TLV message, the concentrator opens a TCP connection towards the specified destination and
connects the incoming bytestream of the Multipath QUIC connection to the
bytestream of the new TCP connection (and similarly in the opposite direction).

{{tcp-proxy-stream}} summarizes how the new TCP connection is mapped to the
QUIC stream. Actions and events of a TCP connection are translated to action and
events of a QUIC stream, so that the state transition of one is translated to
the other.

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

The QUIC stream-level flow control can be tuned to match the receive
window size of the corresponding TCP, so that no excessive
data needs to be buffered.

A timeout can be associated with each mapped QUIC stream for its associated
state to expire when the TCP connection is inactive for a long period.

TODO: Why adding this ? Is this like NAT timeout ?
# Connection establishment

The client MUST establish a connection using the Multipath Extensions defined
in {{I-D.deconinck-quic-multipath}}.

During connection establishment, the QUIC tunnel support is indicated by
setting the ALPN token "qt" in the TLS handshake. Draft-version implementations
MAY specify a particular draft version by suffixing the token, e.g. "qt-00"
refers to the first version of this document.

The concentrator can control the amount of connections bytestreams that can be
opened at first by setting the initial_max_streams_bidi QUIC transport parameter
as defined in {{I-D.ietf-quic-transport}}.

The client and concentrator MUST open each a unidirectional stream for port
mapping as soon as the QUIC connection is established. The first byte of each
stream MUST be 'P', signalling the stream is used for port mapping.

After the QUIC connection is established, the client can start using the
datagram or the stream mode. The client can also request the concentrator to
accept inbound connections on their behalf. First the client has to send a
message on the port mapping stream to request a new port to be opened by the
concentrator. After a port mapping response has been received, the concentrator
can start opening bidirectional streams to forward inbound connections.

# Messages format

In the following sections, we specify the format of each message introduced in
this document.

## QUIC tunnel stream TLVs {#sec-format}

When using the stream mode, a series of messages are used to trigger
and confirm the establishment of a connection towards the
final destination for a given stream. Those messages are exchanged on a QUIC
stream before the connection bytestream. This section describes the format of
these messages. They are encoded as TLVs, i.e. (Type, Length, Value) tuples,
as illustrated in {{tlv}}. All TLV fields are encoded in network-byte order.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
                     1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Type (8)   |   Length (8)  |          [Value (*)]        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #tlv title="QUIC tunnel stream TLV Format"}

The Type field is encoded as a byte and identifies the type of the TLV. The Length
field is encoded as a byte and indicate the length of the Value field. A value
of zero indicates that no Value field is present. The Value field is a
type-specific value whose length is determined by the Length field.

This document specifies the following QUIC tunnel stream TLVs:

~~~~~~~~~~~~~~~~~~~~~~~~~~~
+------+----------+--------------------+
| Type |     Size | Name               |
+------+----------+--------------------+
| 0x00 | 20 bytes | TCP Connect TLV    |
| 0x01 |  2 bytes | TCP Connect OK TLV |
| 0x02 | Variable | Error TLV          |
| 0xff |  2 bytes | End TLV            |
+------+----------+--------------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #stream-tlvs title="QUIC tunnel stream TLVs"}

The TCP Connect TLV is used to establish a TCP connection through the
tunnel towards the final destination. The TCP Connect OK TLV confirms
the establishment of this TCP connection. The Error TLV is used to
indicate any out-of-band error that occurred on the TCP connection
associated to the QUIC stream. Finally, the End TLV marks the end of the
series of TLVs and the start of the bytestream on a given QUIC stream. These
TLVs are detailed in the following sections.

TODO: role of EndTLV unclear at this stage
MP: Is it clearer now ?

### TCP Connect TLV {#sec-connect-tlv}

The TCP Connect TLV indicates the final destination of the TCP
connection associated to a given QUIC
stream. The fields Remote Peer Port and Remote Peer IP Address contain the
destination port number and IP address of the final destination.

The Remote Peer IP Address MUST be encoded as an IPv6 address. IPv4 addresses
MUST be encoded using the IPv4-Mapped IPv6 Address format defined in
{{RFC4291}}.
Further, the Remote Peer IP address field MUST NOT include multicast,
broadcast, and host loopback addresses {{RFC6890}}.

The TCP Connect TLV is always sent at the beginning of a QUIC
stream. It is followed by the End TLV.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
                     1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Type (8)   |   Length (8)  |     Remote Peer Port (16)     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                  Remote Peer IP Address (128)                 |
|                                                               |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #connect-tlv title="Connect TLV"}

### TCP Connect OK TLV

The TCP Connect OK TLV does not contain a value. Its presence confirms
the successful establishment of connection to the final destination.

### Error TLV {#sec-error-tlv}

The Error TLV indicates out-of-band errors that occurred during the
establishment of the connection to the final destination. These errors can be
ICMP Destination Unreachable messages for instance. In this case the
ICMP packet received by the concentrator is
copied inside the Error Payload field.

TODO: la longueur limite l'extrait de l'ICMP que l'on peut fournir. Il
faudrait regarder la taille des IPv6 d'erreur en IPv6. En IPv4, il y a
une limite sur l'info qui est placée dans le paquet, pas toujours en IPv6

~~~~~~~~~~~~~~~~~~~~~~~~~~~
                     1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Type (8)   |   Length (8)  |        Error Code (16)        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     [Error Payload (*)]                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #error-tlv title="Error TLV"}

The following bytestream-level error codes are defined in this document:

~~~~~~~~~~~~~~~~~~~~~~~~~~~
+------+---------------------------+
| Code | Name                      |
+------+---------------------------+
|  0x0 | Unknown TLV               |
|  0x1 | ICMP packet received      |
|  0x2 | Malformed TLV             |
|  0x3 | Network Failure           |
+------+---------------------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #error-tlv-codes title="Bytestream-level Error Codes"}

- Unknown TLV (0x0): This code indicates that a TLV of unknown type
  was received. The Error Payload contains the received type value.
- ICMP packet (0x1): This code indicates that the concentrator
  received an ICMP packet while trying to create the associated TCP
  connection. The Error Payload contains the packet.
- Malformed TLV (0x2): This code indicates that a received TLV was not
  successfully parsed or formed. A peer receiving a Connect TLV with
  an invalid IP address MUST send an Error TLV with this error code.
TODO: Quel exemple de malforme ?
- Network Failure (0x3): This codes indicates that a network failure
  prevented the establishment of the connection.

After sending one or more Error TLVs, the sender MUST send an End TLV and
terminate the stream, i.e. set the FIN bit after the End TLV.

TODO: est-ce nécessaire d'en envoyer plusieurs ? un seul error tlv ne
pourrait-il pas suffire ?

### End TLV

The End TLV does not contain a value. Its existence signals the end of
the series of TLVs. The next byte after this TLV is the start of the
tunneled bytestream.

TODO: après un error TLV, il n'y a pas de stream comme il n'y a pas de
connexion TCP. Peut-on réutiliser un stream après la fin d'une
connexion TCP?

## QUIC tunnel port mapping TLVs

This document also specifies TLVs to request port mapping at the concentrator.
A client can request ports to be opened at the concentrator. Inbound connections
for these ports are accepted by the concentrator and forwarded to the clients.

These TLVs MUST be exchanged on the QUIC tunnel control stream. TODO(mp): Define
this stream.

TODO(mp): Introduce what for and how they are used somewhere before

~~~~~~~~~~~~~~~~~~~~~~~~~~~
+-------------+----------+---------------------------+
|        Type |     Size | Name                      |
+-------------+----------+---------------------------+
| 0x00 - 0x01 |  4 bytes | Port Mapping Request TLV  |
| 0x10 - 0x11 | 20 bytes | Port Mapping Response TLV |
| 0x20 - 0x21 |  5 bytes | Port Mapping Error TLV    |
+-------------+----------+---------------------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #port-mapping-tlvs title="Port Mapping TLVs"}

### Port Mapping Request TLV {#sec-port-mapping-request-tlv}

The Port Mapping Request TLV is sent by clients to request the concentrator to
accept inbound connections from remote peers on a particular port. The port is
encoded into 16 bits. This document only specifies methods for requesting ports
for TCP and UDP.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
                     1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Type (7)  |P|   Length (8)  |   Suggested Remote Port (16)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #port-mapping-request-tlv title="Port Mapping Request TLV"}

The P flag indicates whether the port is requested for TCP or UDP connections.
A value of 0 is TCP, a value of 1 is UDP

### Port Mapping Response TLV {#sec-port-mapping-response-tlv}

The Port Mapping Response TLV is sent by the concentrator in response of a Port
Mapping Request TLV. It indicates the port opened to accept connections on
behalf of the client.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
                     1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Type (7)  |P|   Length (8)  |    Actual Remote Port (16)    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                  Remote Peer IP Address (128)                 |
|                                                               |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #port-mapping-response-tlv title="Port Mapping Response TLV"}

The P flag indicates whether the port is requested for TCP or UDP connections.
A value of 0 is TCP, a value of 1 is UDP.

The Remote Peer IP Address MUST be encoded as an IPv6 address. IPv4 addresses
MUST be encoded using the IPv4-Mapped IPv6 Address format defined in
{{RFC4291}}. Further, Remote Peer IP address field MUST NOT include multicast,
broadcast, and host loopback addresses {{RFC6890}}.

### Port Mapping Error TLV {#sec-port-mapping-error-tlv}

The Port Mapping Error TLV is sent by the concentrator in response of a Port
Mapping Request TLV. It indicates that the concentrator was unable to open the
suggested port and to provide an alternative port for the given protocol.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
                     1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Type (7)  |P|   Length (8)  |   Suggested Remote Port (16)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Error Code (8)|
+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #port-mapping-error-tlv title="Port Mapping Response TLV"}

The P flag indicates whether the port is requested for TCP or UDP connections.
A value of 0 is TCP, a value of 1 is UDP. This document specifies the following
error codes:

- Protocol Not Supported (0x00): The protocol requested is not supported.

- No Port Available (0x01): The concentrator has no more ports to open for the
  client.

TODO(mp): Specify a mechanism to release ports, or at least set a timeout

# Security Considerations

TODO Security

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
