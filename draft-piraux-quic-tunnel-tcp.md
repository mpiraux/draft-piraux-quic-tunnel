---
title: Tunneling TCP inside QUIC
abbrev: QUIC Tunnel for TCP
docname: draft-piraux-quic-tunnel-tcp-00
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
  RFC4291:
  RFC6890:
  I-D.piraux-quic-tunnel:

informative:
  I-D.ietf-tcpm-converters:
  I-D.ietf-quic-transport:
  RFC7301:
  RFC8126:

--- abstract

This document specifies a new operating mode for a QUIC tunnel to convey TCP
bytestreams.

--- middle

# Introduction

The recently proposed QUIC tunnel protocol ({{I-D.piraux-quic-tunnel}}) allows
conveying several Internet protocols inside a QUIC connection. Its first
operating mode, the datagram mode, proposes to transport plain IP packets
inside QUIC packets. Its main advantage is that it supports IP and any protocol
above the network layer. However, this advantage comes with a large per-packet
overhead since each packet contains both a network and a transport header. All
these headers must be transmitted in addition with the IP/UDP/QUIC headers of
the QUIC connection. For TCP connections for instance, the per-packet overhead
can be large.

In this document, we propose a new operating mode for the QUIC tunnel protocol,
called the stream mode. It takes advantage of the QUIC streams to transport TCP
bytestreams over a QUIC connection. Furthermore, we define methods for grouping
QUIC tunnel connections and steering TCP flows from one to another.

{{the-stream-mode}} describes this new mode.  {{messages-format}} specifies the
format of the messages introduced by this document. {{example-flows}} contains
example flows.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# The stream mode

Since QUIC supports multiple streams, another possibility to
carry the data exchanged over TCP connections between the client and the concentrator is to
transport the bytestream of each TCP connection as one of the bidirectional streams of the
 QUIC connection. For this, we base our approach on the 0-RTT Converter
protocol {{I-D.ietf-tcpm-converters}} that was proposed to ease the
deployment of TCP extensions. In a nutshell, it is an application proxy that
converts TCP connections, allowing the use of new TCP extensions
through an intermediate relay.

We use a similar approach in our stream mode. When a client opens a stream, it
sends at the beginning of the bytestream one or more TLV messages indicating the
IP address and port number of the remote destination of the bytestream.
Their format is detailed in section {{sec-stream-format}}. Upon reception of such a
TLV message, the concentrator opens a TCP connection towards the specified
destination and connects the incoming bytestream of the QUIC connection to the
bytestream of the new TCP connection (and similarly in the opposite direction).

{{tcp-proxy-stream}} summarizes how the new TCP connection is mapped to the
QUIC stream. Actions and events of a TCP connection are translated to action and
events of a QUIC stream, so that a state transition of one is translated to
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

When sending STOP_SENDING or RESET_STREAM frames in response to the receipt of a
TCP RST, QUIC tunnel peers MUST use the application protocol error code 0x00
(TCP_CONNECTION_RESET).

The QUIC stream-level flow control can be tuned to match the receive
window size of the corresponding TCP, so that no excessive
data needs to be buffered.

In order to take advantage of the several access networks to which the client is
connected, we define a way of grouping the QUIC connections in a single
tunneling session. This allows steering the TCP flows mapped to a QUIC stream of
a given connection to another QUIC stream of another QUIC connection in the
tunneling session. For that purpose, the concentrator sends a token that
identifies the tunneling session after the QUIC connection has been established.
The client has then the opportunity of opening new QUIC connections and join
them to the tunneling session. The messages exchanged for this mechanism are
described in {{sec-session-format}}.

# Connection establishment

During the connection establishment, the concentrator can control the number of
connections bytestreams that can be opened initially by setting the
initial_max_streams_bidi QUIC transport parameter as defined in
{{I-D.ietf-quic-transport}}.

# Joining a tunneling session {#sec-joining}

Joining a tunneling session allows pausing and resuming tunneled bytestreams from
one QUIC connection to the other. The messages used for this purpose are
described in {{sec-session-format}}. A dedicated unidirectional stream is used to convey
these messages and establish the negotiation of a tunneling session. This
negotiation MUST NOT take place more than once per QUIC connection.

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

The Type field is encoded as a byte and identifies the type of the TLV. The Length
field is encoded as a byte and indicate the length of the Value field. A value
of zero indicates that no Value field is present. The Value field is a
type-specific value whose length is determined by the Length field.

## QUIC tunnel stream TLVs {#sec-stream-format}

When using the stream mode, a one or more messages are used to trigger
and confirm the establishment of a connection towards the
final destination for a given stream. Those messages are exchanged on this given
QUIC stream before the TCP connection bytestream. This section describes the
format of these messages.

This document specifies the following QUIC tunnel stream TLVs:

~~~~~~~~~~~~~~~~~~~~~~~~~~~
+------+----------+-----------------------------+
| Type |     Size | Name                        |
+------+----------+-----------------------------+
| 0x00 | 20 bytes | TCP Connect TLV             |
| 0x01 | 38 bytes | TCP Extended Connect TLV    |
| 0x02 |  2 bytes | TCP Connect OK TLV          |
| 0x03 | Variable | TCP Resume Token TLV        |
| 0x04 | Variable | TCP Resume TLV              |
| 0x05 | Variable | Error TLV                   |
| 0xff |  2 bytes | End TLV                     |
+------+----------+-----------------------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #stream-tlvs title="QUIC tunnel stream TLVs"}

The TCP Connect TLV is used to establish a TCP connection through the
tunnel towards the final destination. The TCP Extended Connect TLV allows
indicating more information in the establishment request. The TCP Connect OK TLV
confirms the establishment of this TCP connection. The TCP Resume Token TLV is
used to associate the TCP connection with a particular token. This token can be
used to pause and resume its associated TCP connection over another QUIC
connection part of the tunneling session using the TCP Resume TLV. The Error TLV is
used to indicate any error that occurred during the TCP connection
establishment associated to the QUIC stream. Finally, the End TLV marks the end
of the series of TLVs and the start of the bytestream on a given QUIC stream.
These TLVs are detailed in the following sections.


~~~~~~~~~~~~~~~~~~~~~~~~~~~
      Offset 0         Offset 20               Offset 24  Offset 26
         |                 |                      |         |
         v                 v                      v         v
         +-----------------+----------------------+---------+----------------
Stream 0 | TCP Connect TLV | TCP Resume Token TLV | End TLV | TCP bytestream ...
         +-----------------+----------------------+---------+----------------

         +----------------+---------+----------------
Stream 4 | TCP Resume TLV | End TLV | TCP bytestream ...
         +----------------+---------+----------------
         ^                ^         ^
         |                |         |
      Offset 0         Offset 4   Offset 6
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #tlvs-in-stream title="Examples of use of QUIC tunnel stream TLVs"}

In {{tlvs-in-stream}}, two examples of use of QUIC tunnel streams TLVs are
given. In the first one, the client opens Stream 0 and sends three TLVs. The
first one will establish a new TCP connection through the tunnel. This
TCP connection will be associated with the Resume Token contained in the second
TLV. The third TLV marks the end of the series of TLV and the start of the TCP
bytestream.

The second example illustrates how a Resume Token can be used using a TCP Resume
TLV to resume a TCP connection that was established through another QUIC
connection part of the tunneling session.

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

A QUIC tunnel peer MUST NOT send more than one TCP Connect TLV per QUIC stream.
A QUIC tunnel peer MUST NOT send a TCP Connect TLV if a TCP Extended Connect
TLV or a TCP Resume TLV was previously sent on a given stream. A QUIC tunnel
peer MUST NOT send a TCP Connect TLV on non-self initiated streams.


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
{: #connect-tlv title="TCP Connect TLV"}

### TCP Extended Connect TLV {#sec-extended-connect-tlv}

The TCP Extended Connect TLV is an extended version of the TCP Connect TLV.
It also indicates the source of the TCP connection.
The fields Remote Peer Port and Remote Peer IP Address contain the
destination port number and IP address of the final destination.
The fields Local Peer Port and Local Peer IP Address contain the source port
number and IP address of the source of the TCP connection.

The Remote (resp. Local) Peer IP Address MUST be encoded as an IPv6 address.
IPv4 addresses MUST be encoded using the IPv4-Mapped IPv6 Address format defined
in {{RFC4291}}.
Further, the Remote (resp. Local) Peer IP address field MUST NOT include multicast,
broadcast, and host loopback addresses {{RFC6890}}.

A QUIC tunnel peer MUST NOT send more than one TCP Extended Connect TLV per QUIC
stream. A QUIC tunnel peer MUST NOT send a TCP Extended Connect TLV if a TCP
Connect TLV or a TCP Resume TLV was previously sent on a given stream. A QUIC
tunnel peer MUST NOT send a TCP Extended Connect TLV on non-self initiated
streams.

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
|      Local Peer Port (16)     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                   Local Peer IP Address (128)                 |
|                                                               |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #extended-connect-tlv title="TCP Extended Connect TLV"}

### TCP Connect OK TLV

The TCP Connect OK TLV does not contain a value. Its presence confirms
the successful establishment of connection to the final destination. This
message is sent both for new connection establishment, as result of the
receipt of a TCP Connect (Extended) TLV, and for connection
resumption, as a result of the receipt of a TCP Resume TLV.
A QUIC peer MUST NOT send a TCP Connect OK TLV on self-initiated streams.

### TCP Resume Token TLV

The TCP Resume Token TLV contains an opaque value that identifies this
bytestream across the tunneling session. The semantic scope of this value is
limited by the peer that sent it.
As a result, both peers can use the same value to identify two different
bytestreams. Each TCP Resume Token TLV sent MUST
contain a value that is unique in that scope.

A QUIC tunnel peer MUST NOT send more than one TCP Resume Token TLV per QUIC
stream. A QUIC tunnel peer MUST NOT send a TCP Resume Token TLV if a TCP Resume
TLV was previously sent on a given stream. A QUIC tunnel peer MUST NOT send a
TCP Resume Token TLV on non-self initiated streams.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
                     1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Type (8)   |   Length (8)  |       Resume Token (*)      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #resume-token-tlv title="TCP Resume Token TLV"}

### TCP Resume TLV

The TCP Resume TLV contains two values. The Resume Token identifies a TCP
connection previously established in the tunneling session. The Bytestream
Offset indicates the offset in the TCP bytestream at which this QUIC stream will
resume. Thus, the offset in the TCP bytestream of the first byte after the End
TLV is indicated by this value.

When pausing and resuming a TCP connection, a QUIC tunnel peer MUST resume its
bytestream at an offset that does not introduce a gap in the bytestream. The
peer SHOULD track the parts of the bytestream that were successfully received to
resume it at an efficient offset.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
                     1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Type (8)   |   Length (8)  |       Resume Token (*)      ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Bytestream Offset                       |
|                              (64)                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #resume-tlv title="TCP Resume TLV"}

A QUIC tunnel peer MUST NOT send more than one TCP Resume TLV per QUIC
stream. A QUIC tunnel peer MUST NOT send a TCP Resume TLV if a TCP
Connect TLV or a TCP Connect Extended TLV was previously sent on a given stream.
A QUIC tunnel peer MUST NOT send a TCP Resume TLV on non-self initiated
streams.

A QUIC tunnel peer receiving a TCP Resume TLV with an unknown Resume Token MUST send an
Error TLV with the code 0x5 (Unknown Token) and close the QUIC stream.

### Error TLV {#sec-error-tlv}

The Error TLV indicates out-of-band errors that occurred during the
establishment of the connection to the final destination. These errors can be
ICMP Destination Unreachable messages for instance. In this case the
ICMP packet received by the concentrator is copied inside the Error Payload
field.

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
|  0x0 | Protocol Violation        |
|  0x1 | ICMP Packet Received      |
|  0x2 | Malformed TLV             |
|  0x3 | Network Failure           |
|  0x4 | Token Already Used        |
|  0x5 | Unknown Token             |
+------+---------------------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #error-tlv-codes title="Bytestream-level Error Codes"}

- Protocol Violation (0x0): A general error code for all non-conforming
  behaviors encountered. A QUIC tunnel peer SHOULD use a more specific error
  code when possible.
- ICMP Packet Received (0x1): This code indicates that the concentrator
  received an ICMP packet while trying to create the associated TCP
  connection. The Error Payload contains the packet.
- Malformed TLV (0x2): This code indicates that a received TLV was not
  successfully parsed or formed. A peer receiving a Connect TLV with
  an invalid IP address MUST send an Error TLV with this error code.
- Network Failure (0x3): This codes indicates that a network failure
  prevented the establishment of the connection.
- Token Already Used (0x4): A TCP Resume Token TLV was received with a token
  that has already been used.
- Unknown Token (0x5): A TCP Resume TLV was received with an unknown token.

After sending one or more Error TLVs, the sender MUST send an End TLV and
terminate the stream, i.e. set the FIN bit after the End TLV.

### End TLV

The End TLV does not contain a value. Its existence signals the end of
the series of TLVs. The next byte in the QUIC stream after this TLV is the start
of the tunneled bytestream.

## QUIC tunnel control TLVs {#sec-session-format}

In order to negotiate the tunneling session used with the concentrator, the
client and the concentrator open their first unidirectional stream (i.e. stream
2 and 3), named afterwards as QUIC tunnel control stream. The client MAY either
start a new session or join an existing session.

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
of this QUIC tunnel connection to the session, e.g. the Resume Tokens exchanged.
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
  could not be merged with the tunneling session. For instance, Resume Tokens
  with identical values have already been exchanged.

# Example flows

This section illustrates the different messages described previously and how
they are used in a QUIC tunnel connection. For QUIC STREAM frames, we use the
following syntax: STREAM\[ID, Stream Data \[, FIN\]\]. The first element is the
Stream ID, the second is the Stream Data contained in the frame and the last one
is optional and indicates that the FIN bit is set.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
Client                      Concentrator           Final Destination
 | STREAM[0, "TCP Connect, End"] ||                               |
 |------------------------------>||              SYN              |
 |                               ||==============================>|
 |                               ||            SYN+ACK            |
 |STREAM[0,"TCP Connect OK, End"]||<==============================|
 |<------------------------------||                               |
 | STREAM[0, "bytestream data"]  ||                               |
 |------------------------------>||     bytestream data, ACK      |
 |                               ||==============================>|
 |                               ||     bytestream data, ACK      |
 |  STREAM[0, "bytestream data"] ||<==============================|
 |<------------------------------||              FIN              |
 |      STREAM[0, "", FIN]       ||<==============================|
 |<------------------------------||              ACK              |
 |      STREAM[0, "", FIN]       ||==============================>|
 |------------------------------>||              FIN              |
 |                               ||==============================>|
 |                               ||              ACK              |
 |                               ||<==============================|

Legend:
   --- QUIC connection
   === TCP connection
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #example-stream-mode title="Example flow for the stream mode"}

On {{example-stream-mode}}, the Client is initiating a TCP connection in
stream mode to the Final Destination. A request and a response are exchanged,
then the connection is torn down gracefully.
A remote-initiated connection accepted by the concentrator on behalf of the
client would have the order and the direction of all messages reversed.

# Security Considerations

## Denial of Service

There is a risk of an amplification attack when the Concentrator sends a TCP SYN
in response of a TCP Connect TLV. When a TCP SYN is larger than the client
request, the Concentrator amplifies the client traffic. To mitigate such attacks,
the Concentrator SHOULD rate limit the number of pending TCP Connect from a
given client.

# IANA Considerations

## Registration of QUIC tunnel Identification String

This document creates a new registration for the identification of the QUIC
tunnel protocol in the "Application Layer Protocol Negotiation (ALPN) Protocol
IDs" registry established in {{RFC7301}}.

The "qt" string identifies the QUIC tunnel protocol.

   Protocol: QUIC tunnel

   Identification Sequence: 0x71 0x74 ("qt")

   Specification: This document

## QUIC tunnel stream TLVs

IANA is requested to create a new "QUIC tunnel stream Parameters" registry.

The following subsections detail new registries within "QUIC tunnel stream
Parameters" registry.

### QUIC tunnel stream TLVs Types

IANA is request to create the "QUIC tunnel stream TLVs Types" sub-registry. New
values are assigned via IETF Review (Section 4.8 of {{RFC8126}}).

The initial values to be assigned at the creation of the registry are as
follows:

~~~~~~~~~~~~~~~~~~~~~~~~~~~
+------+-----------------------------+------------+
| Code | Name                        | Reference  |
+------+-----------------------------+------------+
|    0 | TCP Connect TLV             | [This-Doc] |
|    1 | TCP Extended Connect TLV    | [This-Doc] |
|    2 | TCP Connect OK TLV          | [This-Doc] |
|    3 | TCP Resume Token TLV        | [This-Doc] |
|    4 | TCP Resume TLV              | [This-Doc] |
|    5 | Error TLV                   | [This-Doc] |
|  255 | End TLV                     | [This-Doc] |
+------+-----------------------------+------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~

### QUIC tunnel streams TLVs Error Types

IANA is request to create the "QUIC tunnel stream TLVs Error Types" sub-registry.
New values are assigned via IETF Review (Section 4.8 of {{RFC8126}}).

The initial values to be assigned at the creation of the registry are as
follows:

~~~~~~~~~~~~~~~~~~~~~~~~~~~
+------+---------------------------+------------+
| Code | Name                      | Reference  |
+------+---------------------------+------------+
|    0 | Protocol Violation        | [This-Doc] |
|    1 | ICMP packet received      | [This-Doc] |
|    2 | Malformed TLV             | [This-Doc] |
|    3 | Network Failure           | [This-Doc] |
|    4 | Token Already Used        | [This-Doc] |
|    5 | Unknown Token             | [This-Doc] |
+------+---------------------------+------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~

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

The initial values to be assigned at the creation of ther registry are as
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

# Acknowledgments
{:numbered="false"}

This documents draws heavily on the initial version of {{I-D.piraux-quic-tunnel}}.
As such, their contributors are thanked again here.

