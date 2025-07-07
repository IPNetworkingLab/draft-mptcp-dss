---
title: "Multipath TCP with longer DSS mappings"
category: std

docname: draft-baerts-tcpm-mptcpdss-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Transport"
workgroup: "TCPM"
keyword:
 - mptcp
venue:
  group: "TCPM"
  type: "Individual"
  mail: "tcpm@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/tcpm/"
  github: "IPNetworkingLab/draft-mptcp-v2"
  latest: "https://ipnetworkinglab.github.io/draft-mptcp-dss/draft-baerts-tcpm-mptcpdss.html"

author:
 -
    fullname: Matthieu Baerts
    organization: UCLouvain & NGI Zero Core
    email: matttbe@kernel.org


normative:
  RFC8684:

informative:
 RFC2675:
 RFC8041:
 MPTCP-longitudinal: DOI.10.48550/arXiv.2205.12138


--- abstract

This document proposes an extension to improve Multipath TCP based on
operational experience by allowing Multipath TCP to use DSS mappings that are
longer than 64 KBytes.


--- middle

# Introduction

From a performance viewpoint, TCP stacks are optimised to leverage large
segments and use TCP Segment Offload / Generic Receive Offload (TSO/GRO). The
DSS option defined in Multipath TCP allows to map a series of bytes from the
bytestream on a specific subflow. Unfortunately, the length of this mapping is
encoded in a 16-bit field. Since each Multipath TCP segment must include a DSS
mapping before being sent to the network interface, this restricts the size of
the segments that Multipath TCP can use. In particular in IPv6, it is impossible
for Multipath TCP to leverage IPv6 jumbograms {{RFC2675}} in contrast to regular
TCP. This document proposes a modification of the DSS option to support longer
mappings.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Extending the DSS option

The extension proposed in this document is pretty simple. Given that the DSS
Checksum is rarely used in practice, we propose to reuse the space reserved for
this checksum in the DSS option to support 32-bit data-level mappings. This
enables Multipath TCP servers to send segments that are longer than 64 KBytes.
This extension is negotiated using the TBD flag in the MP_CAPABLE option during
the handshake.


~~~~~~~~~~~~~~~~~~~~~~~~~~~
   Host A                                  Host B
   ------                                  ------
   MP_CAPABLE                ->
   [flags (TBD is set)]
                             <-            MP_CAPABLE
                                           [B's key, flags (TBD is set)]
   ACK + MP_CAPABLE (+ data) ->
   [A's key, B's key, flags, (data-level details)]
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-dss-handshake title="Negotiation of the DSS option with longer mappings"}


The DSS option defined in {{RFC8684}} reserves 16 bits for the Checksum.
However, operational experience indicates that this checksum is almost never
used by Multipath TCP deployments. It was designed to detect middlebox
interference caused notably by Application Level Gateways that modify TCP
payloads {{RFC8041}}. Given the widespread adoption of TLS, such ALGs are rarely
used by applications using Multipath TCP {{MPTCP-longitudinal}}.


~~~~~~~~~~~~~~~~~~~~~~~~~~~
                          1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +---------------+---------------+-------+----------------------+
     |     Kind      |    Length     |Subtype| (reserved) |F|m|M|a|A|
     +---------------+---------------+-------+----------------------+
     |           Data ACK (4 or 8 octets, depending on flags)       |
     +--------------------------------------------------------------+
     |   Data Sequence Number (4 or 8 octets, depending on flags)   |
     +--------------------------------------------------------------+
     |              Subflow Sequence Number (4 octets)              |
     +-------------------------------+------------------------------+
     |  Data-Level Length (2 octets) |      Checksum (2 octets)     |
     +-------------------------------+------------------------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-dss title="The DSS option in RFC8684"}


This document proposes to use a 32-bit Data-Level Length to support large TCP
segments. The new DSS option is shown in {{fig-newdss}}. The other fields of
this option and the procedures defined in {{RFC8684}} are unchanged.


~~~~~~~~~~~~~~~~~~~~~~~~~~~
                          1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +---------------+---------------+-------+----------------------+
     |     Kind      |    Length     |Subtype| (reserved) |F|m|M|a|A|
     +---------------+---------------+-------+----------------------+
     |           Data ACK (4 or 8 octets, depending on flags)       |
     +--------------------------------------------------------------+
     |   Data Sequence Number (4 or 8 octets, depending on flags)   |
     +--------------------------------------------------------------+
     |              Subflow Sequence Number (4 octets)              |
     +-------------------------------+------------------------------+
     |                 Data-Level Length (4 octets)                 |
     +-------------------------------+------------------------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-newdss title="The new DSS option"}


{{RFC8684}} defines the MP_CAPABLE option as shown in {{fig-oldmpc}}. This
option contains several flags, A-H. Flags A, B, C, and H are specified in
{{RFC8684}}. This document uses Flag TBD to indicate in a SYN that the initiator
of a connection requests the utilization of 32-bit Data-Level Length. If this
Flag is set in a SYN, Flag A must also obviously be set to 0 to indicate that
the Checksum is not required on this connection. If both Flags A and TBD are set
in a SYN, the receiver MUST not continue the MPTCP connection, and SHOULD
fallback to TCP. A server that receives a SYN with the TBD Flag set can reply
with:

- a SYN+ACK with the TBD Flag set to 1 to confirm that it accepts to use 32-bit
Data-Level Length
- a SYN+ACK with the TBD Flag set to 0 to indicate that it prefers to use 16-bit
Data-Level Length

Even when the TBD Flag is set to 1, the MP_CAPABLE options continue to use a
16-bit Data-Level Length like before, to allow fallback if the receiver doesn't
support a 32-bit Data-Level Length.


~~~~~~~~~~~~~~~~~~~~~~~~~~~
                          1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +---------------+---------------+-------+-------+---------------+
     |     Kind      |    Length     |Subtype|Version|A|B|C|D|E|F|G|H|
     +---------------+---------------+-------+-------+---------------+
     |                   Option Sender's Key (64 bits)               |
     |                      (if option Length > 4)                   |
     |                                                               |
     +---------------------------------------------------------------+
     |                  Option Receiver's Key (64 bits)              |
     |                      (if option Length > 12)                  |
     |                                                               |
     +-------------------------------+-------------------------------+
     |  Data-Level Length (16 bits)  |  Checksum (16 bits, optional) |
     +-------------------------------+-------------------------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-oldmpc title="The MP_CAPABLE option"}


# Security Considerations

This document does not change the security considerations defined in
{{RFC8684}}.


# IANA Considerations

This document requests the IANA to reserve flag TBD of the MP_CAPABLE option as
defined in this document. It also proposes to change the format of the DSS
option. This document suggests using the D flag of the MP_CAPABLE option.


--- back

# Acknowledgments
{:numbered="false"}

This project is funded through NGI Zero Core, a fund established by NLnet with
financial support from the European Commission's Next Generation Internet
program.
