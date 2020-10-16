---
title: Tunneling Internet protocols inside QUIC
abbrev: QUIC Tunnel
docname: draft-piraux-quic-tunnel-04
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
    email: maxime.piraux@uclouvain.be

 -
    ins: "O. Bonaventure"
    name: "Olivier Bonaventure"
    organization: "UCLouvain"
    email: olivier.bonaventure@uclouvain.be

 -
    ins: "A. Masputra"
    name: "Adi Masputra"
    organization: "Apple Inc."
    email: adi@apple.com

normative:
  RFC2119:
  RFC1701:
  TS23501:
    title: >
      Technical Specification Group Services and System Aspects;
      System Architecture for the 5G System; Stage 2 (Release 16)
    author:
        org: 3GPP (3rd Generation Partnership Project)
    date: 2019
    seriesinfo:
      3GPP: TS23501

informative:
  I-D.pauly-quic-datagram:
  I-D.ietf-quic-transport:
  I-D.ietf-quic-tls:
  RFC1812:
  RFC2827:
  RFC6887:
  RFC7301:
  RFC8126:
  RFC6824:

--- abstract

This document specifies methods for tunneling Ethernet frames and
Internet protocols such as TCP, UDP, IP and QUIC inside a QUIC connection.


--- middle

# Introduction

Mobile devices such as laptops, smartphones or tablets have different
requirements than the traditional fixed devices. These mobile devices
often change their network attachment. They are often attached to
trusted networks, but sometimes they need to be connected to untrusted
networks where their communications can be eavesdropped, filtered or
modified. In these situations, the classical approach is to rely on VPN
protocols such as DTLS or IPSec. These VPN protocols provide the
encryption and authentication functions to protect those mobile clients
from malicious behaviors in untrusted networks.

However, some networks have deployed filters that block these VPN
protocols. When faced with such filters, users can either switch off
their connection or find alternatives, e.g. by using TLS to access
some services over TCP port 443. The planned deployment of QUIC
{{I-D.ietf-quic-transport}} {{I-D.ietf-quic-tls}} opens a new
opportunity for such users. Since QUIC will be used to access web
sites, it should be less affected by filters than VPN solutions such
as IPSec or DTLS. Furthermore, the flexibility of QUIC makes it
possible to easily extend the protocol to support VPN services.

This document explores how QUIC could be used to
enable devices to communicate securely in untrusted
networks. The QUIC protocol opens up a new way to find a clean solution to this
problem. First, QUIC includes the same encryption and authentication
techniques as deployed VPN protocols. Second, QUIC is intended to be
widely used to support web-based services, making it unlikely to be
filtered in many networks, in contrast with VPN protocols. Third, the
QUIC migration mechanism enables handovers between several network interfaces.

This document is organized as follows.
{{reference-environment}} describes our reference environment. Then, we propose
a first mode of operation, explained in {{the-tunnel-mode}}, that use the
recently proposed datagram extension
({{I-D.pauly-quic-datagram}}) for QUIC to transport plain packets over a
QUIC connection. {{connection-establishment}} specifies how a connection
is established in this document proposal. {{messages-format}} details the format
of the messages introduced by this document.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Reference environment

The reference scenario is a client that uses a QUIC tunnel to send all
its packets to a concentrator. The concentrator processes the
packets received from the client over the QUIC connection and
forwards them to their final destination.
It also receives the packets destined to the client and tunnels them through
the QUIC connection.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
                                                       +-------------+
+--------+              +--------------+               | Final       |
| Client |              | Concentrator |<===\ ... \===>| destination |
+--------+              +--------------+               | server      |
       ^    +---------+    ^                           +-------------+
       |    | Access  |    |            Legend:
       .----| network |----.              --- QUIC connection
            +---------+                   === TCP/UDP flow
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-simple-environment title="A client attached to a concentrator"}

In a nutshell, the solution proposed in this document works as
follows. The client opens a QUIC connection to a concentrator. The
concentrator authenticates the client through
means that are outside the scope of this document such as client
certificates, usernames/passwords, OAuth, ... If the authentication
succeeds, the client can use the tunnel to exchange any types of packets
with the concentrator over the QUIC session.

The client can send any packets via the concentrator by tunneling
them through the concentrator. The concentrator captures the
packets destined to the client and tunnels them over the QUIC connection.
Our solution is intended to provide a similar service as the one provided
by IPSec tunnels or DTLS. This document leaves address assignment mechanisms
out of scope, deployments can rely on out-of-band configurations for that
purpose.

# The tunnel mode

The tunnel mode of operation leverages the recently proposed
QUIC datagram extension {{I-D.pauly-quic-datagram}}. In a nutshell, to send a
packet to a remote host, the client simply encapsulates the entire packet inside
a QUIC DATAGRAM frame sent over the QUIC connection established with the
concentrator.

The frame transmission is subject to congestion
control, but the frame that contains the packet is not retransmitted in case
of loss as specified in {{I-D.pauly-quic-datagram}}.

This mode adds a minimal byte overhead for packet encapsulation in QUIC. It
does not define ways of indicating the protocol of the conveyed packets, which
can be useful in deployments for which out-of-band signaling may be used.

# Connection establishment

During QUIC connection establishment, the "tunnel mode" of operation
support is indicated by setting the ALPN token "qt" in the TLS
handshake. Draft-version implementations MAY specify a particular draft version
by suffixing the token, e.g. "qt-00" refers to the first
version of this document.

After the QUIC connection is established, the client can start using the
datagram or the stream mode. The client may use PCP {{RFC6887}} to request the
concentrator to accept inbound connections on their behalf. After
the negotiation of such port mappings, the concentrator can start sending
packets containing inbound connections in QUIC DATAGRAM frame.

# Reporting access networks availability

When several QUIC tunnels are used over different access networks, being able to
report their availability to the concentrator can help reduce the amount of
packets sent over unstable or unavailable paths. It can also resume quickly the
sending of packets over previously unavailable access networks.

To do so, we define in {{messages-format}} a message called Access Report TLV.
The message can be sent by the client to the concentrator. It identifies the
type of access network reported and its associated status. This message is sent
over the QUIC connection in a separate unidirectional stream.

# Messages format

In the following sections, we specify the format of each message introduced in
this document. The messages are encoded as TLVs, i.e. (Type, Length, Value) tuples,
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
Length field is encoded as a byte and indicate the length in bytes of the Value field.
A value of zero indicates that no Value field is present. The Value field is a
type-specific value whose length is determined by the Length field.

## QUIC tunnel control TLVs {#sec-session-format}

This document specifies the following QUIC tunnel control TLVs:

~~~~~~~~~~~~~~~~~~~~~~~~~~~
+------+----------+--------+------+-------------------+
| Type |     Size | Sender | Mode | Name              |
+------+----------+--------+------+-------------------+
| 0x00 |  4 bytes | Client |  all | Access Report TLV |
+------+----------+--------+------+-------------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #control-tlvs title="QUIC tunnel control TLVs"}

The Access Report TLV is sent by the client to periodically report on access
networks availability. Each Access Report TLV MUST be sent on a separate
unidirectional stream. The stream FIN bit MUST be set following the end of the
TLV.

### Access Report TLV

~~~~~~~~~~~~~~~~~~~~~~~~~~~
                     1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Type = 0x00  | Length = 0x02 | AI (4)| R (4) |   Signal (8)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #access-tlv title="Access Report TLV"}

The Access Report TLV contains the following:

* AI (Access ID) - a four-bit-long field that identifies the access network,
  e.g., 3GPP (Radio Access Technologies specified by 3GPP) or Non-3GPP
  (accesses that are not specified by 3GPP) {{TS23501}}.  The value is one
  of those listed below (all other values are invalid and the TLV that
  contains it MUST be discarded):

~~~~~~~~~~~~~~~~~~~~~~~~~~~
+-----------+-----------------------+
| Access ID | Description           |
+-----------+-----------------------+
|      1    | 3GPP Network          |
|      2    | Non-3GPP Network      |
+-----------+-----------------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~

* R (Reserved) - a four-bit-long field that MUST be zeroed on transmission and
  ignored on receipt.

* Signal - a one-octet-long field that identifies the report signal, e.g.,
  available or unavailable.  The value is supplied to the QUIC tunnel through
  some mechanism that is outside the scope of this document.  The value is one
  of those listed in {{quic-tunnel-access-report-signal-codes}}.

The client that includes the Access Report TLV sets the value of the Access ID
field according to the type of access network it reports on.  Also, the client
sets the value of the Signal field to reflect the operational state of the access
network.  The mechanism to determine the state of the access network is outside
the scope of this specification.

The client MUST be able to cancel the sending of an Access Report TLV that is
pending delivery, i.e. by resetting its corresponding unidirectional stream.
This can be used when the information contained in the TLV is no longer
relevant, e.g. the access network availability has changed. The time of
canceling is based on local policies and network environment.

Reporting the unavailability of an access network to the concentrator can serve as
an indication to stop sending packets over this network while
maintaining the QUIC tunnel connection. Upon reporting of the availability of
this network, the concentrator can quickly resume sending packets over this
network.

# Security Considerations

## Privacy

The Concentrator has access to all the packets it processes. It MUST be
protected as a core IP router, e.g. as specified in {{RFC1812}}.

## Ingress Filtering

Ingress filtering policies MUST be enforced at the network boundaries, i.e. as
specified in {{RFC2827}}.

# IANA Considerations

## Registration of QUIC tunnel Identification String

This document creates two new registrations for the identification of the QUIC
tunnel protocol in the "Application Layer Protocol Negotiation (ALPN) Protocol
IDs" registry established in {{RFC7301}}.

The "qt" string identifies the QUIC tunnel protocol datagram mode.

  Protocol:
  : QUIC Tunnel

  Identification Sequence:
  : 0x71 0x74 ("qt")

  Specification:
  : This document

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
|    0 | Access Report TLV     | [This-Doc] |
+------+-----------------------+------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~

## QUIC tunnel Access Report Signal Codes

This document establishes a registry for QUIC tunnel Access Report Signal codes.
The "QUIC tunnel Access Report Signal Code" registry manages a 62-bit space.
New values are assigned via IETF Review (Section 4.8 of {{RFC8126}}).

The initial values to be assigned at the creation of the registry are as
follows:

~~~~~~~~~~~~~~~~~~~~~~~~~~~
+------+-----------------------+------------+
| Code | Name                  | Reference  |
+------+-----------------------+------------+
|    1 | Access Available      | [This-Doc] |
|    2 | Access Unavailable    | [This-Doc] |
+------+-----------------------+------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~

--- back

# Change Log

## Since draft-piraux-quic-tunnel-03

* Make the lightweight mode the default "tunnel mode"
* Rename the datagram mode as tunnel session mode in
  draft-piraux-quic-tunnel-session

## Since draft-piraux-quic-tunnel-02

* Add the lightweight mode

## Since draft-piraux-quic-tunnel-01

* Add the Access Report TLV for reporting access networks availability
* Add a section with examples of use of the Packet Tag

## Since draft-piraux-quic-tunnel-00

* Separate the document in two and put the stream mode in another document
* Remove TCP Extended TLV
* Add a mechanism for joining QUIC connections in a QUIC tunneling session
* Add a format for encoding any network-layer protocol packets and Ethernet
 frames in QUIC DATAGRAM frames

# Acknowledgments
{:numbered="false"}

Thanks to Quentin De Coninck and François Michel for their comments and
the proofreading of the first version of this document.
Thanks to Gregory Vander Schueren for his comments on the first version of
this document.
Thanks to Florin Baboescu for his comments on the fifth version of this document.

