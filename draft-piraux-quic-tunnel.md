---
title: Tunneling Internet protocols inside QUIC
abbrev: QUIC Tunnel
docname: draft-piraux-quic-tunnel-01
date: 2020-02-05
category: exp

ipr: trust200902
area: Transport
workgroup: QUIC Working Group
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
  I-D.pauly-quic-datagram:
  I-D.schinazi-masque:
  RFC1812:
  RFC2827:
  RFC6887:
  RFC7301:
  RFC8126:
  IANA-ETHER-TYPES:
    title: IANA ETHER TYPES
    seriesinfo: https://www.iana.org/assignments/ieee-802-numbers/ieee-802-numbers.txt

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
supports those handovers and allows aggregating the bandwidth of
different access links. It could be combined with single-path VPN
protocols to support both seamless handovers and bandwidth aggregation
above VPN tunnels. Unfortunately, Multipath TCP is not yet deployed on most
Internet servers and thus few applications would benefit from such a use
case.

The QUIC protocol opens up a new way to find a clean solution to this
problem. First, QUIC includes the same encryption and authentication
techniques as deployed VPN protocols. Second, QUIC is intended to be
widely used to support web-based services, making it unlikely to be
filtered in many networks, in contrast with VPN protocols. Third, the
QUIC migration mechanism enables handovers between several network interfaces.

In this document, we explore how QUIC could be used to
enable multi-homed mobile devices to communicate securely in untrusted
networks. {{reference-environment}} describes the reference environment of this
document. Then, we propose a first mode of operation, explained in
{{the-datagram-mode}}, that uses the recently proposed datagram extension
({{I-D.pauly-quic-datagram}}) for QUIC to transport plain IP packets over a
QUIC connection. {{connection-establishment}} specifies how a connection is
established in this document proposal. {{messages-format}} details the format
of the messages introduced by this document.

This document shares some goals with the MASQUE framework
{{I-D.schinazi-masque}}. The proposed QUIC tunnel protocol contributes to the
effort of defining a signaling for conveying multiple proxied flows inside a
QUIC connection.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Reference environment

We consider a client that is attached to one or several
access networks. It can be multihomed, multistack or none of the previous
characteristics. It establishes one or several QUIC connections to a
concentrator, taking advantage of the several access networks available.
These QUIC connections are used to carry the packets sent by the
client. Thanks to the QUIC migration mechanism, the connection can be migrated
to another access network when needed. The client can group several QUIC
connections in a QUIC tunneling session to coordinate the conveying of packets.
Thanks to the security mechanisms used by QUIC,
the client data is protected against attacks in any of the access
networks. The client trusts the concentrator. The concentrator decrypts the QUIC
packets exchanged over the QUIC connections and interacts with the
remote hosts as a VPN concentrator would do.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
            +---------+
       .----| Access  |----.
       |    | network |    |
       |    |    A    |    |
       v    +----------    v                           +-------------+
+--------+              +--------------+               | Final       |
| Client |              | Concentrator |<===\ ... \===>| destination |
+--------+              +--------------+               | server      |
       ^    +---------+    ^                           +-------------+
       |    | Access  |    |
       |    | network |    |            Legend:
       .----|    B    |----.              --- QUIC connection
            +---------+                   === TCP/UDP flow
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-example-environment title="Example environment"}

{{fig-example-environment}} illustrates a client-initiated flow. We also
discuss inbound connections in this document in {{connection-establishment}}.

# The datagram mode

Our first mode of operation, called the datagram mode in this document,
enables the client and the concentrator to exchange raw packets through
the QUIC connection. This is done by using the recently
proposed QUIC datagram extension {{I-D.pauly-quic-datagram}}.
In a nutshell, to send a packet to a remote host, the client simply
passes the entire packet as a datagram to the QUIC connection
established with the concentrator.

This document specifies the following format for encoding packets in QUIC
DATAGRAM frame. It allows encoding packets from several protocols by
identifying the corresponding protocol of the packet in each QUIC DATAGRAM
frame. {{encoding-packets}} describes this encoding.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
                     1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|       Protocol Type (16)      |        Packet Tag (16)        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Packet (*)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #encoding-packets title="Encoding packets in QUIC DATAGRAM frame"}

This encoding defines three fields.

* Protocol Type: The Protocol Type field contains the protocol type of the
   payload packet. The values for the different protocols are defined as
   "ETHER TYPES" in {{IANA-ETHER-TYPES}}.

* Packet Tag: An opaque 16-bit value. The QUIC tunnel application is free to
   decide its semantic value. For instance, a QUIC tunnel endpoint MAY encode
   the sending order of packets in the Packet Tag, e.g. as a timestamp or a
   sequence number, to allow reordering on the receiver.

* Packet: The packet conveyed inside the QUIC tunnel connection.

This encoding is sent inside a QUIC DATAGRAM frame, which is then encrypted and
authenticated in a QUIC packet. This transmission is subject to congestion
control, but the frame that contains the packet is not retransmitted in case
of losses as specified in {{I-D.pauly-quic-datagram}}. The datagram mode is
intended to provide a similar service as the one provided by IPSec tunnels or
DTLS.


~~~~~~~~~~~~~~~~~~~~~~~~~~~
             ,->+----------+
             |  |    IP    |
 QUIC packet |  +----------+
 containing  |  |    UDP   |
 a DATAGRAM  |  +----------+
 frame       |  |   QUIC   |
             |  |..........|
             |  | DATAGRAM |
             |  |  P. Type |
             |  |  P. Tag  |
             |  |+--------+|<-.
             |  ||   IP   ||  |
             |  |+--------+|  | Tunneled
             |  ||   UDP  ||  | UDP packet
             |  |+--------+|  |
             |  |   ....   |<-.
             `->+----------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #datagram-example title="QUIC packet sent by the client when
tunneling a UDP packet"}


{{datagram-example}} illustrates how a UDP packet is tunneled using the datagram
mode.
The main advantage of the datagram mode is that it supports IP and any
protocol above the network layer. Any IP packet can be transported
using the datagram extension over a QUIC connection. However, this
advantage comes with a large per-packet overhead since each packet
contains both a network and a transport header. All these headers must be
transmitted in addition with the IP/UDP/QUIC headers of the QUIC connection.
For TCP connections for instance, the per-packet overhead can be large.

# Connection establishment

During connection establishment, the QUIC tunnel support is indicated by
setting the ALPN token "qt" in the TLS handshake. Draft-version implementations
MAY specify a particular draft version by suffixing the token, e.g. "qt-00"
refers to the first version of this document.

After the QUIC connection is established, the client can start using the
datagram or the stream mode. The client may use PCP {{RFC6887}} to request the
concentrator to accept inbound connections on their behalf. After the negotiation
of such port mappings, the concentrator can start sending packets containing inbound
connections in QUIC DATAGRAM frame.

# Joining a tunneling session {#sec-joining}

Joining a tunneling session allows grouping several QUIC connections to the
concentrator. Each endpoint can then coordinate the use of the Packet Tag across
the tunneling session. The messages used for this purpose are
described in {{messages-format}}. A dedicated unidirectional stream is used
to convey these messages and establish the negotiation of a tunneling session.
This negotiation MUST NOT take place more than once per QUIC connection.

# Messages format

In the following sections, we specify the format of each message introduced in
this document. They are encoded as TLVs, i.e. (Type, Length, Value) tuples,
as illustrated in {{tlv}}. All TLV fields are encoded in network-byte order.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
                     1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Type (8)   |   Length (8)  |          [Value (*)]        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #tlv title="QUIC tunnel TLV Format"}

The Type field is encoded as a byte and identifies the type of the TLV. The
Length field is encoded as a byte and indicate the length of the Value field.
A value of zero indicates that no Value field is present. The Value field is a
type-specific value whose length is determined by the Length field.

## QUIC tunnel control TLVs {#sec-session-format}

In order to negotiate the tunneling session used with the concentrator, the
client and the concentrator open their first unidirectional stream (i.e. stream
2 and 3), named QUIC tunnel control stream. The client MAY either start a new
session or join an existing session.

This document specifies the following QUIC tunnel control TLVs:

~~~~~~~~~~~~~~~~~~~~~~~~~~~
+------+----------+--------------+------------------+
| Type |     Size |       Sender | Name             |
+------+----------+--------------+------------------+
| 0x00 |  2 bytes |       Client | New Session TLV  |
| 0x01 | Variable | Concentrator | Session ID TLV   |
| 0x02 | Variable |       Client | Join Session TLV |
+------+----------+--------------+------------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #control-tlvs title="QUIC tunnel control TLVs"}

The New Session TLV is used by the client to initiate a new tunneling session.
The Session ID TLV is used by the concentrator to communicate to the client the
Session ID identifying this tunneling session. The Join Session TLV is used to
join a given tunneling session, identified by a Session ID. All QUIC tunnel
control TLVs MUST NOT be sent on other streams than the QUIC tunnel control
streams.

### New Session TLV

The New Session TLV does not contain a value. It initiates a new tunneling
session at the concentrator. The concentrator MUST send a Session ID TLV in
response, with the Session ID corresponding to the tunneling session created.
After sending a New Session TLV, the client MUST close the QUIC tunnel control
stream.

The concentrator MUST NOT send New Session TLVs.

### Session ID TLV

~~~~~~~~~~~~~~~~~~~~~~~~~~~
                     1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Type (8)   |   Length (8)  |        Session ID (*)       ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #session-tlv title="Session ID TLV"}

The Session ID TLV contains an opaque value that identifies the current
tunneling session. It can be used by the client in subsequent QUIC connections
to join them to this tunneling session. The concentrator MUST send a Session ID
TLV in response of a New Session TLV, with the Session ID corresponding to the
tunneling session created.

The client MUST NOT send a Session ID TLV. The concentrator MUST close the QUIC
tunnel control stream after sending a Session ID TLV.

### Join Session TLV

~~~~~~~~~~~~~~~~~~~~~~~~~~~
                     1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Type (8)   |   Length (8)  |        Session ID (*)       ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #join-tlv title="Join Session TLV"}

The Join Session TLV contains an opaque value that identifies a tunneling
session to join. The client can send a Join Session TLV to join the QUIC
connection to a particular tunneling session. The tunneling session is
identified by the Session ID.
After sending a Join Session TLV, the client MUST close the QUIC tunnel control
stream.

The concentrator MUST NOT send Join Session TLVs. After receiving a Join Session
TLV, the concentrator MUST use the Session ID to join this QUIC connection to
the tunneling session. Joining the tunneling session implies merging the state
of this QUIC tunnel connection to the session.
A successful joining of connection is indicated by the
closure of the QUIC tunnel control stream of the concentrator.

In cases of failure when joining a tunneling session, the concentrator MUST send
a RESET_STREAM with an application error code discerning the cause of the
failure. The possible codes are listed below:

* UNKNOWN_ERROR (0x0): An unknown error occurred when joining the tunneling
  session. QUIC tunnel peers SHOULD use more specific error codes when
  applicable.
* UNKNOWN_SESSION_ID (0x1): The Session ID used in the Join Session TLV is not a
  valid ID. It was not issued in a Session ID TLV or refers to an expired
  tunneling session.
* CONFLICTING_STATE (0x2): The current state of the QUIC tunnel connection
  could not be merged with the tunneling session.

# Security Considerations

## Privacy

The Concentrator has access to all the packets it processes. It MUST be
protected as a core IP router, e.g. as specified in {{RFC1812}}.

## Ingress Filtering

Ingress filtering policies MUST be enforced at the network boundaries, i.e. as
specified in {{RFC2827}}.

# IANA Considerations

## Registration of QUIC tunnel Identification String

This document creates a new registration for the identification of the QUIC
tunnel protocol in the "Application Layer Protocol Negotiation (ALPN) Protocol
IDs" registry established in {{RFC7301}}.

The "qt" string identifies the QUIC tunnel protocol.

   Protocol: QUIC tunnel

   Identification Sequence: 0x71 0x74 ("qt")

   Specification: This document


## QUIC tunnel control TLVs

IANA is requested to create a new "QUIC tunnel control Parameters" registry.

The following subsections detail new registries within "QUIC tunnel control
Parameters" registry.

### QUIC tunnel control TLVs Types

IANA is request to create the "QUIC tunnel control TLVs Types" sub-registry. New
values are assigned via IETF Review (Section 4.8 of {{RFC8126}}).

The initial values to be assigned at the creation of the registry are as
follows:

~~~~~~~~~~~~~~~~~~~~~~~~~~~
+------+-----------------------+------------+
| Code | Name                  | Reference  |
+------+-----------------------+------------+
|    0 | New Session TLV       | [This-Doc] |
|    1 | Session ID TLV        | [This-Doc] |
|    2 | Join Session TLV      | [This-Doc] |
+------+-----------------------+------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~

## QUIC tunnel control Error Codes

This document establishes a registry for QUIC tunnel control stream error
codes. The "QUIC tunnel control Error Code" registry manages a 62-bit space. New
values are assigned via IETF Review (Section 4.8 of {{RFC8126}}).

The initial values to be assigned at the creation of the registry are as
follows:

~~~~~~~~~~~~~~~~~~~~~~~~~~~
+------+-----------------------+------------+
| Code | Name                  | Reference  |
+------+-----------------------+------------+
|    0 | UNKNOWN_ERROR         | [This-Doc] |
|    1 | UNKNOWN_SESSION_ID    | [This-Doc] |
|    2 | CONFLICTING_STATE     | [This-Doc] |
+------+-----------------------+------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~

--- back

# Change Log

## Since draft-piraux-quic-tunnel-00

* Separate the document in two and put the stream mode in another document
* Remove TCP Extended TLV
* Add a mechanism for joining QUIC connections in a QUIC tunneling session
* Add a format for encoding any network-layer protocol packets in DATAGRAM
  frames

# Acknowledgments
{:numbered="false"}

Thanks to Quentin De Coninck and François Michel for their comments and
the proofreading of the first version of this document.
Thanks to Gregory Vander Schueren for his comments on the first version of
this document.

