---
title: "Multipath TCP with variable-length DSS mappings"
category: std

docname: draft-baerts-tcpm-mptcpdss-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "TCP Maintenance and Minor Extensions"
keyword:
 - mptcp
venue:
  group: "TCP Maintenance and Minor Extensions"
  type: "Working Group"
  mail: "tcpm@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/tcpm/"
  github: "IPNetworkingLab/draft-mptcp-dss"
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
operational experience by using variable-length encoding for the Subflow
Sequence Number and Data-Level Length fields of the DSS option. This allows
Multipath TCP to save option space for short connections while supporting
Data-Level Lengths larger than 64 KBytes.


--- middle

# Introduction

From a performance viewpoint, TCP stacks are optimised to leverage large
segments and use TCP Segment Offload / Generic Receive Offload (TSO/GRO). The
DSS option defined in Multipath TCP allows to map a series of bytes from the
bytestream on a specific subflow. Unfortunately, the length of this mapping is
encoded in a 16-bit field and the Subflow Sequence Number always uses 4 bytes.
Since each Multipath TCP segment must include a DSS mapping before being sent to
the network interface, this restricts the size of the segments that Multipath TCP
can use. In particular in IPv6, it is impossible for Multipath TCP to leverage
IPv6 jumbograms {{RFC2675}} in contrast to regular TCP. Conversely, short
connections waste option space by always using a 4-byte Subflow Sequence Number.
This document proposes a modification of the DSS option to use variable-length
encoding for both the Subflow Sequence Number and the Data-Level Length, allowing
shorter encodings when possible and longer mappings when needed.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Extending the DSS option

The extension proposed in this document modifies the DSS option to use
variable-length encoding for the Subflow Sequence Number (SSN) and the
Data-Level Length (DLL). Given that the DSS Checksum is rarely used in practice,
we propose to remove it and use flag bits in the reserved field to indicate the
length of the SSN and DLL fields. This extension is negotiated using the TBD
flag in the MP_CAPABLE option during the handshake.


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
{: #fig-dss-handshake title="Negotiation of the variable-length DSS option"}


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


## Variable-Length Encoding

This document introduces a variable-length encoding for the SSN and DLL fields
of the DSS option. Two bits from the reserved field are used to indicate the
length of each field. The encoding is as follows:

| Bits | Length   | Maximum Value |
|------|---------|---------------|
| 00   | 2 bytes | 65,535        |
| 01   | 3 bytes | 16,777,215    |
| 10   | 4 bytes | 4,294,967,295 |
| 11   | Reserved | -            |
{: #tab-varlen title="Variable-length encoding"}

The `ss` bits indicate the length of the Subflow Sequence Number and the `dd`
bits indicate the length of the Data-Level Length. This encoding allows the DSS
option to use as few as 4 bytes for these two fields combined (when both use
2-byte encoding), compared to the 6 bytes used in {{RFC8684}} (4-byte SSN +
2-byte DLL) or the 8 bytes that would be needed with two fixed 4-byte fields.

The Data-Level Length SHOULD NOT exceed 1,073,741,823 (about 1 GByte).

The new DSS option is shown in {{fig-newdss}}. The Data ACK and Data Sequence
Number fields continue to use the existing `a`/`A` and `m`/`M` flags as defined
in {{RFC8684}}.


~~~~~~~~~~~~~~~~~~~~~~~~~~~
                          1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +---------------+---------------+-------+---+----+----+-+-+-+-+-+
     |     Kind      |    Length     |Subtype|rsv| ss | dd |F|m|M|a|A|
     +---------------+---------------+-------+---+----+----+-+-+-+-+-+
     |           Data ACK (4 or 8 octets, depending on flags)       |
     +--------------------------------------------------------------+
     |   Data Sequence Number (4 or 8 octets, depending on flags)   |
     +--------------------------------------------------------------+
     |     Subflow Sequence Number (2, 3, or 4 octets, see ss)      |
     +--------------------------------------------------------------+
     |       Data-Level Length (2, 3, or 4 octets, see dd)          |
     +--------------------------------------------------------------+
~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-newdss title="The new DSS option with variable-length SSN and DLL"}


{{RFC8684}} defines the MP_CAPABLE option as shown in {{fig-oldmpc}}. This
option contains several flags, A-H. Flags A, B, C, and H are specified in
{{RFC8684}}. This document uses Flag TBD to indicate in a SYN that the initiator
of a connection requests the utilization of the variable-length DSS option. If
this Flag is set in a SYN, Flag A must also obviously be set to 0 to indicate
that the Checksum is not required on this connection. If both Flags A and TBD
are set in a SYN, the receiver MUST not continue the MPTCP connection, and
SHOULD fallback to TCP. A server that receives a SYN with the TBD Flag set can
reply with:

- a SYN+ACK with the TBD Flag set to 1 to confirm that it accepts to use the
variable-length DSS option
- a SYN+ACK with the TBD Flag set to 0 to indicate that it prefers to use the
standard DSS option as defined in {{RFC8684}}

Even when the TBD Flag is set to 1, the MP_CAPABLE options continue to use a
16-bit Data-Level Length like before, to allow fallback if the receiver doesn't
support the variable-length DSS option.


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
option by introducing variable-length encoding for the Subflow Sequence Number
and Data-Level Length fields using 4 bits (ss, dd) from the reserved field of
the DSS option. This document suggests using the D flag of the MP_CAPABLE
option.


--- back

# Acknowledgments
{:numbered="false"}

This project is funded through NGI Zero Core, a fund established by NLnet with
financial support from the European Commission's Next Generation Internet
program.
