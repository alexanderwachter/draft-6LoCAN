%%%
title = "IPv6 over Controller Area Network"
abbrev = "6LoCAN"
docName = "draft-ietf-6lo-can-00"
ipr= "trust200902"
area = "Internet"
workgroup = "6Lo Working Group"
submissiontype = "IETF"
keyword = ["Internet-Draft"]
date = 2019-08-01T00:00:00Z

[seriesInfo]
status = "standard"
name = "Internet-Draft"
value = "draft-ietf-6lo-can-00"
stream = "IETF"

[pi]
toc = "yes"

[[author]]
initials="A."
surname="Wachter"
fullname="Alexander Wachter"
organization = "University of Technology Graz"
  [author.address]
  email = "alexander@wachter.cloud"
%%%

.# Abstract

This is the abstract

.# Status of This Memo

This Internet-Draft is submitted in full conformance with the provisions of BCP 78 and BCP 79. Internet-Drafts are working documents of the Internet Engineering Task Force (IETF). Note that other groups may also distribute working documents as Internet-Drafts. The list of current Internet-Drafts is at http://datatracker.ietf.org/drafts/drafts/current.

Internet-Drafts are draft documents valid for a maximum of six months and may be updated, replaced, or obsoleted by other documents at any time. It is inappropriate to use Internet- Drafts as reference material or to cite them other than as "work in progress.

.# Copyright Notice
Copyright (c) 2019 IETF Trust and the persons identified as the document authors. All rights reserved.

This document is subject to BCP 78 and the IETF Trust's Legal Provisions Relating to IETF Documents http://trustee.ietf.org/license-info in effect on the date of publication of this document. Please review these documents carefully, as they describe your rights and restrictions with respect to this document. Code Components extracted from this document must include Simplified BSD License text as described in Section 4.e of the Trust Legal Provisions and are provided without warranty as described in the Simplified BSD License.

This Internet-Draft will expire on February 2, 2020.

{mainmatter}

# Introduction

Controller Area Network (CAN) is mostly known for its use in the automotive domain.
However, it is also used in building automation, for example, heating control.
It is a two-wire wired-AND multi-master bus that uses CSMA/CR in its arbitration field.
CAN use 11 bit (standard ID) and 29 bit (extended ID) identifiers to identify frames instead of addressing nodes.
The maximum payload data size is 8 octets for classical and 64 octets for CAN-FD.
Therefore a mechanism to support larger MTU is needed.
This document uses a slightly modified version of the ISO-TP protocol to transfer data up to 65KB per packet.
Mapping addresses to identifier uses an addressing schema with 14-bit source, a 14-bit destination address, and a multicast bit.
This schema utilizes the whole 29-bit extended identifier.
To make data transfer more efficient IPHC [@!RFC6282] is used.
Due to the limited address space of 14 bit, random address generation would fail with an unacceptably high probability.
For this reason, a link-layer duplicate address detection to avoid overlapping addresses is introduced.
 An Ethernet border translator is designed to connect a 6LoCAN bus segment to other networks.

## Terminology

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**", "**SHALL NOT**", "**SHOULD**",
"**SHOULD NOT**", "**RECOMMENDED**", "**NOT RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this
document are to be interpreted as described in BCP 14 [@!RFC2119] [@!RFC8174] when, and only when,
they appear in all capitals, as shown here.

## Controller Area Network Overview

This section provides a brief overview of Controller Area Network (CAN), as specified in [ISO 11898-1:2015].
CAN use two wires, CAN High and CAN Low, where CAN High is tight to 5V and CAN Low to 0V when transmitting a dominant (0) bit.
Both wires are at the same level (approximately 2,5V) when transmitting a recessive (1) bit.
Because of the wired-AND structure, a dominant bit overrides a recessive bit.
To resolve collisions in the arbitration field, a can controller checks for overridden recessive bits.
The sender that was sending the recessive bit then stops the transmission.
Therefor an identifier with all zeros has the highest priority.
CAN controllers usually can install identifier filtering.
Messages are received only when the dedicated filter matches. The identifiers usually can be masked where the mask defines which bits of the identifier must match and which bits do not care.

## ISO-TP Overview

A subset ISO-TP (ISO 15765-2) is used to fragment and reassemble the packets.
This subset of ISO-TP can send packets with a payload size up to 4095 octets, enough for IPv6 minimum packet size of 1280 octets.
ISO-TP is designed for CAN, and the small frame size of CAN and therefore preferred over [@!RFC4944] fragmentation.
The 6LoWPAN fragmentation would use more than the halve of the available payload for the fragmentation headers.

# Addressing

This section provides information about node address to identifier mapping.

Because CAN uses identifiers instead of addresses to identify the frame's content, an addressing schema is introduced to map addresses to identifiers.
Every node has a unique 14-bit address.
The address schema uses the 29-bit extended identifier only. It is a combination of source address, a destination address, and a multicast bit.
Every identifier MUST be unique, and therefore a source and destination combination MUST NOT send more than one packet concurrently.
The address 0x3DFE is reserved for link-layer duplicate address detection, and address 0x3DF0 is reserved for Ethernet border translator.
Addresses from 0x0100 to 0x3DEF are used as node address.
Other addresses (0x0000 to 0x00FF and 0x0x3DF0 to 0x3FFF) are reserved or used for special purposes.
Note that a lower address number has a higher priority on the bus.
6LoCAN does not use the 11-bit standard identifier. They may be used for other purposes.

{#tab-address-layout}
Address         | Description
----------------|----------------
0x3DFE - 0x3FF  | Reserved
0x3DFE          | LLDAD
0x3DF1 - 0x3DFD | Reserved
0x3DF0          | Ethernet Translator
0x0100 - 0x3DEF | Node addresses
0x0000 - 0x00FF | Reserved
Table: Address layout

{#fig-identifier-layout}
~~~
 0|0            1|1            2|
 0|1            4|5            8|
+-+--------------+--------------+
|M|     DEST     |     SRC      |
+-+--------------+--------------+
~~~
Figure: Addressing Schema

M
 : Multicast.

DEST
 : Destination Address (14 bit).

SRC
 : Source Address (14 bit).

For example, a destination of 0x3055 and source address of 0x3AAF result in the following identifier:

{#fig-unicast-id}
~~~
 0|0            1|1            2|
 0|1            4|5            8|
+-+--------------+--------------+
|0|11000001010101|11101010101111|
+-+--------------+--------------+
~~~
Figure: Unicast identifier example

A multicast group of 1 and a source address of 0x3AAF result in the following identifier:

{#fig-bulticast-id}
~~~
 0|0            1|1            2|
 0|1            4|5            8|
+-+--------------+--------------+
|1|00000000000001|11101010101111|
+-+--------------+--------------+
~~~
Figure: Multicast identifier example

## Unicast

For unicast packets, the multicast bit is set to zero and the 14-bit source address is the address of the sender.
The 14-bit destination address of the receiver is discovered by IPv6 NDP defined in [@!RFC4861].
Every node MUST set a CAN filter that matches its address as a destination address.

## Multicast

For multicast packets, the multicast bit is set to one and the 14-bit source address is the address of the sender.
The 14-bit destination address is the last 14 bits of the multicast group.
Every node MUST set CAN filters to match the last 14 bit of the joined multicast groups as the destination address.

## Address Generation

Every node has a 14-bit address. This address MUST be unique within the CAN bus segment.
The address can either be statically defined or assigned randomly.
For the random address assignment, the node tries randomly chosen addresses until the link-layer duplicate address detection succeeds.
The link-layer duplicate address detection prevents nodes from assigning an address already in use.

# Stateless Address Autoconfiguration

This section defines how to obtain an IPv6 Interface Identifier.

It is RECOMMENDED to form an IID derived from the node's address. IID's generated from the node address result in most efficient IPHC header compression.
However, IID's MAY also be generated from other sources.
The general procedure for creating an IID is described in Appendix A of [@!RFC4291], "Creating Modified EUI-64 Format Interface Identifiers", as updated by [@!RFC7136].

The Interface Identifier for link-local addresses SHOULD be formed by concatenating the node's 14-bit address to the six octets 0x00, 0x00, 0x00, 0xFF, 0xFE, 0x00 and two bits 0b00.
For example, an address of hexadecimal value 0x3AAF results in the following IID:

{#fig-generate-iid}
~~~
|0              1|1              3|3              4|4              6|
|0              5|6              1|2              7|8              3|
+----------------+----------------+----------------+----------------+
|0000000000000000|0000000011111111|1111111000000000|0011101010101111|
+----------------+----------------+----------------+----------------+
~~~
Figure: IID from Address 0x3AAF

# IPv6 Link Local Address

The IPv6 link-local address [@!RFC4291] for a 6LoCAN interface is formed by appending the Interface Identifier, as defined above, to the prefix FE80::/64.

{#fig-link-loca-address}
~~~
  10 bits            54 bits                  64 bits
+----------+-----------------------+----------------------------+
|1111111010|         (zeros)       |    Interface Identifier    |
+----------+-----------------------+----------------------------+
~~~
Figure: Link Local address from IID

# Frame Format {#sec-frame-format}

This section provides information about data arrangement in the frame data field.

{#fig-frame-format}
~~~
+----------------------------+-------------------------------------+
| ISO-TP Header (1-3 octets) | Dispatch + LOWPAN_IPHC (2-3 octets) | 
+----------------------------+-------------+-----------------------+
| In-line IPv6 Header Fields (0-38 octets) | Payload 
+------------------------------------------+----------
~~~
Figure: 6LoCAN Frame Format

Packets with a destination or source address of the 0x3DF0 (Translator address) carry the Ethernet MAC address inline directly after the ISO-TP Header.
For packets going to the translator, it is the destination MAC address, and for packets coming from the translator, it is the source MAC address.

{#fig-frame-format-translator}
~~~
+----------------------------+--------------------------------+
| ISO-TP Header (1-3 octets) | Ethernet MAC Address (48 bits) |
+----------------------------+-------+------------------------+
|Dispatch + LOWPAN_IPHC (2-3 octets) |
+------------------------------------+-----+----------
| In-line IPv6 Header Fields (0-38 octets) | Payload 
+------------------------------------------+----------
~~~
Figure: 6LoCAN Translator Frame Format

# ISO-TP

This section provides information about the use of ISO-TP in this document.
Parts of ISO-TP are used to provide a reliable way for sending up to 4095 octets as a packet.
It includes a flow-control mechanism for unicast-packets and timeouts.
Multicast packets do not use any flow-control mechanism and are therefore not covered by the ISO-TP standard.
Network address extension and packet size larger than 4095 octets defined by ISO-TP MUST NOT be used for 6LoCAN.
Single Frame packets are only useful for CAN-FD because the eight octets of classical CAN are too small for any IPv6 header.

## Multicast

Multicast packets MUST be transferred in a single frame when the packet fits in a single frame.
Multicast packets that are too big for single frames start with a First Frame (FF) followed by a break of 1 millisecond and as many Consecutive Frames (CF) as needed.

{#fig-multicast-sequence}
~~~
Sender   Multicast Listener
  |-----FF---->|
  |  I 1 ms I  |
  |----CF 1--->|
  | I ST min I |
  |----CF 2--->|
  |     .      |
  |     .      |
  |----CF n--->|
  |            |
~~~
Figure: Multicast packet sequence.

## Unicast

The flow-control mechanism uses Flow Control (FC) frames to steer the sender's behavior. The receiver can choose a block size and a minimum separation time (ST min).

The block size (BS) defines how many frames are transmitted before the sender MUST wait for another FC Frame. A zero BS is allowed and denotes that the sender MUST NOT wait for another FC Frame.
ST min defines a pause between the end of the previous frame and the start of the next frame.
The receiver MAY change BS and ST min for following FC Frames.

The receiver MUST answer a FF within 1 second. After this timeout a the sender SHOULD abort and stop waiting for an FC frame.
CF frames MUST have a separation time smaller or equal to one second. After this timeout, a receiver SHOULD abort and stop waiting for CF.
Receivers and sender SHOULD handle more than one packet reception from different peers at the same time.

{#fig-unicast-sequence}
~~~
Sender      Receiver
  |-----FF---->|
  |            |
  |<----FC-----|
  |            |
  |----CF 1--->|
  | I ST min I |
  |----CF 2--->|
  |      .     |
  |      .     |
  |---CF BS--->|
  |            |
  |<----FC-----|
  |            |
  |--CF BS+1-->|
  | I ST min I |
  |--CF BS+2-->|
  |      .     |
  |      .     |
~~~
Figure: Unicast packet sequence.

## Frame Format

The first 4 bits denote the Protocol Control Information (PCI).

{#fig-iso-tp-frame-format}
~~~
|0  0|0
|0  3|4
+----+-----
|PCI |
+----+-----
~~~
Figure: ISO-TP Frame format

{#tab-pci-numbers}
Number | Description
:-----:|:-----------
   0   | Single Frame
   1   | First Frame
   2   | Consecutive Frame
   3   | Flow-Control Frame
  4-15 | Reserved
Table: PCI Numbers

## Single Frame

The single-frame PCI is 0 and the rest of the octet is padded with 0.
This format is compatible with ISO-TP with data size greater than 16 octets.

{#fig-single-frame}
~~~
|0  0|0  0|       1|1
|0  3|4  7|       5|6
+----+----+--------+--------
|0000|0000|  Size  | Data
+----+----+--------+--------
~~~
Figure: Single Frame Format

Size
 : Number of payload data octets.

## First Frame

The first frame PCI is 1, and the remaining 4-bit nibble of the first byte carries the upper 4-bit nibble of the payload data length.
The second byte contains the lower byte of the payload data length. The rest of the frame is filled with payload data.
The first fame MUST have a data length of the maximum CAN data length. For example, classic can has a maximum data length of 8 octets, and therefore six payload bytes are included in the FF.

{#fig-first-frame}
~~~
|0  0|0           1|1
|0  3|4           5|6
+----+-------------+-------
|0001|    Size     | Data
+----+-------------+-------
~~~
Figure: First Frame Format

Size
 : Number of payload data octets.

## Consecutive Frame

The first frame PCI is two, and the remaining 4-bit nibble of the first byte carries an index.
This index starts with one for the first CF and wraps around at 16. Then it starts at 0 again.
The index is used to check for lost or out of order frames. When the index is not sequential, the reception MUST be aborted.
The last consecutive frame MAY include padding at the end to obtain a valid data length for CAN-FD frames.
The RECOMMENDED padding value is 0xCC.

{#fig-consecutive-frame}
~~~
|0  0|0  0|0
|0  3|4  7|8
+----+----+---------
|0010|Idx | Data
+----+----+---------
~~~
Figure: Consecutive Frame Format

## Flow-Control Frame

The first frame PCI is three, and the remaining 4-bit nibble of the first byte carries a Flow State (FS). The second byte is the block size, and the third byte is the ST min.
The flow states are:

{#tab-flow-state}
 Number| Description
:-----:|:----------------
  0    | CTS (Continue To Send)
  1    | WAIT
  2    | OVFLW (Overflow)
Table: Flow State

CTS advises the sender to continue sending CF frames.

WAIT resets the timeout for receiving an FC frame on the sender side. The sender SHOULD only accept a limited number of wait states and silently abort when reaching the limit.

OVFLW is sent when the receiver is running out of recourses and can't handle the packet. The sender MUST abort when getting an OVFLW state.

{#fig-flow-control-frame}
~~~
|0  0|0  0|       1|1      2|
|0  3|4  7|       5|6      3|
+----+----+--------+--------+
|0011| FS |   BS   | ST min |
+----+----+--------+--------+
~~~
Figure: Flow-Control Frame Format

FS
 : Frame State.

BS
 : Block Size.

ST min
 : Minimal Separation Time

# Link Layer Duplicate Address Detection

This section provides information about how to perform link-layer duplicate address detection (LLDAD).

LLDAD is introduced to prevent collisions of CAN identifiers and make it possible to use random address assignment with only 14-bit address space.
To perform an LLDAD, a DAD-request is sent. If there is no DAD-response sent back within 100 milliseconds, the DAD is considered successful.
DAD-requests are remote transition request frames with the desired address as the destination and 14 bits entropy as the source address. The entropy prevents identifier collisions when nodes are tying to get the same address at the same time.
DAD-responses are data-frames sent to the LLDAD address (0x3DFE) with the responder's address as the source address.
Both DAD-request and DAD-response have a data length of zero.
A node MUST attach one filter for the LLDAD-response address, and another filter for DAD-requests with the desired address.
The DAD-response-filter MUST be applied, and the node MUST be ready to answer DAD-request on the desired address before sending the DAD-request.
The DAD-response-filter MAY be removed after the DAD is successful. The DAD-request filter MUST stay applied and the node MUST be able to respond within 90 milliseconds as long an it uses the address.

(#fig-dad-fail) shows a DAD Fail example where node A performs a DAD-request on address 0x3055 where this address is already in use by node B.

{#fig-dad-fail}
~~~
Node A          Node B
  |--DAD-request->|
  |               |
  |<-DAD-response-|

DAD-request identifier: This frame is a Remote Transmission Request (RTR)
|0|0            1|1            2|
|0|1            4|5            8|
+-+--------------+--------------+
|0|11000001010101|   entropy    |
+-+--------------+--------------+

DAD-response identifier:
|0|0            1|1            2|
|0|1            4|5            8|
+-+--------------+--------------+
|0|11110111111110|11000001010101|
+-+--------------+--------------+
~~~
Figure: DAD Fail example

# Ethernet Border Translator

This section provides information about translating 6LoCAN packets to Ethernet frames.

The Ethernet Border Translator connects 6LoCAN bus-segments to other 6LoCAN bus-segments or other technologies.
Ethernet is a widely used technology that provides enough bandwidth to connect several 6LoCAN segments.
A 6LoBR is not necessary because there is no routing on 6LoCAN segments. To provide routing or switch capabilities, the Ethernet Border Translator connects a 6LoCAN network to such devices via Ethernet.

Bus segments MUST NOT have more than one translator. The translator has a fixed node address (0x3DF0) and a range of Ethernet MAC addresses.
Every packet sent to this node address or any multicast address is forwarded to Ethernet. Every Ethernet frame matching the MAC address range and every multicast Ethernet frame is forwarded to the 6LoCAN bus-segment.

For translating a 6LoCAN packet to an Ethernet frame, the source address is extended with the first 34 bits of the translator MAC address and the IPHC compressed headers are decompressed. The destination MAC is carried in-line before the compressed IPv6 header (see (#sec-frame-format), (#fig-frame-format-translator)).
For translating Ethernet frames to 6LoCAN packets, the source MAC address is carried in-line, the destination node address is the last 14 bits of the MAC address, and the IPv6 headers are compressed using IPHC.

For multicast Ethernet frames, the last 14 bits of the multicast group is the destination address, and the multicast bit is set.

If the translator includes a network stack, the last 14 bits of it's MAC address MUST be the translator node address (0x3DF0) to avoid address collisions.

(#fig-translator-setup) shows an example setup of a 6LoCAN segment connected to an Ethernet network.

This example shows a translation from Ethernet MAC to CAN identifier. The source (src) MAC address is carried in-line in the CAN frame data.
The translator MAC address for this example is 02:00:5E:10:3D:F0.

{#fig-unicast-mac-translation}
~~~
|0                            4|4                              9
|0                            7|8                              5
+------------------------------+-------------------------------+
| dest MAC (02:00:5E:10:30:55) |  src MAC (02:00:5E:10:00:FF)  |
+------------------------------+-------------------------------+
                          Ethernet MAC

|0|0            1|1            2|
|0|1            4|5            8|
+-+--------------+--------------+
|0|dest (0x3055) | src (0x3DF0) |
+-+--------------+--------------+
            CAN identifier
~~~
Figure: Example address translation from Ethernet MAC to CAN identifier.

This example shows a translation from multicast Ethernet MAC to CAN identifier. The source MAC address is carried in-line in the CAN frame data.

{#fig-multicast-mac-translation}
~~~
|0                            4|4                             9|
|0                            7|8                             5|
+------------------------------+-------------------------------+
| dest MAC (33:33:00:01:00:01) |  src MAC (02:00:5E:10:00:FF)  |
+------------------------------+-------------------------------+
                          Ethernet MAC

|0|0            1|1            2|
|0|1            4|5            8|
+-+--------------+--------------+
|1|dest (0x0001) | src (0x3DF0) |
+-+--------------+--------------+
            CAN identifier
~~~
Figure: Example address translation from Ethernet to CAN for multicast Frames.

This example shows a translation CAN identifier to Ethernet MAC. The destination (dest) MAC address is carried inline in the CAN frame data.
The translator MAC address for this example is 02:00:5E:10:3D:F0.

{#fig-unicast-id-translation}
~~~
|0|0            1|1            2|
|0|1            4|5            8|
+-+--------------+--------------+
|0|dest (0x3DF0) | src (0x3055) |
+-+--------------+--------------+
            CAN identifier
|0                            4|4                             9|
|0                            7|8                             5|
+------------------------------+-------------------------------+
| dest MAC (02:00:5E:10:00:FF) |  src MAC (02:00:5E:10:30:55)  |
+------------------------------+-------------------------------+
                          Ethernet MAC
~~~

This example shows a translation form multicast CAN frame to Ethernet MAC.
The translator MAC address for this example is 02:00:5E:10:3D:F0.

{#fig-multicast-id-translation}
~~~
|0|0            1|1            2|
|0|1            4|5            8|
+-+--------------+--------------+
|1|dest (0x0001) | src (0x3055) |
+-+--------------+--------------+
            CAN identifier
|0                            4|4                             9|
|0                            7|8                             5|
+------------------------------+-------------------------------+
| dest MAC (33:33:00:00:00:01) |  src MAC (02:00:5E:10:30:55)  |
+------------------------------+-------------------------------+
                          Ethernet MAC
~~~

{#fig-translator-setup}
~~~
                                          +-----+  +-----+
                                          |CAN  |  |CAN  |  ...
                                          |node |  |node |
                                          +--+--+  +--+--+
+--------+---+     +---+----------+---+      |        |
|        |   |     |   |Ethernet  |   |      |        |
| Switch |ETH|<--->|ETH|Border    |CAN|------+--------+---- ...
|        |   |     |   |Translator|   |
+--------+---+     +---+----------+---+
~~~
Figure: Example setup with Ethernet Border Translator

{backmatter}