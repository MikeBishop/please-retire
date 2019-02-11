---
title: PLEASE_RETIRE_CONNECTION_ID Extension for QUIC
abbrev: PLEASE_RETIRE_CONNECTION_ID
docname: draft-bishop-quic-please-retire-latest
date: {DATE}
category: std

ipr: trust200902
area: Transport
workgroup: QUIC
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: M. Bishop
    name: Mike Bishop
    organization: Akamai Technologies
    email: mbishop@evequefou.be

normative:

informative:

--- abstract

QUIC version 1 divides control over the lifecycle of Connection IDs between the issuer
(the recipient of packets) and the consumer (the sender of packets).  Once a CID has been
issued, only the consumer is able to retire them from use.  This document describes an
extension whereby the issuer can signal that a particular CID needs to be retired.

--- middle

# Introduction

In QUIC version 1 {{!QUIC=I-D.ietf-quic-transport}}, Connection IDs are issued
by each endpoint.  These connection IDs are used in the Destination Connection
ID field of future QUIC packets directed toward the issuer of the CID.  An
issuer is permitted to issue as many CIDs as they wish, with guidance from the
consumer around the desired number of simultaneous CIDs in the form of the
`max_connection_ids` transport parameter (see PR#1998 in the QUIC repo).

The structure of a Connection ID and any semantics tied to it are private
arrangements between the issuer and its local routing infrastructure, if any.
QUIC does not attempt to dictate how the Connection ID is formed or interpreted,
though {{?QUIC-LB=I-D.duke-quic-load-balancers}} offers some options for
servers and load balancers to consider.

The consumer then uses the CIDs and retires them once they will no longer be used.
The issuer is expected to replace retired CIDs promptly, since if the consumer
runs out of valid CIDs, the connection will close.  However, the issuer is not
obligated to do so as a safeguard against consumers who retire too aggressively.

However, this leaves the only perpetual obligation in the hands of the issuer:

> When an endpoint issues a connection ID, it MUST accept packets that carry
> this connection ID for the duration of the connection or until its peer
> invalidates the connection ID via a RETIRE_CONNECTION_ID frame.

Under certain designs, it might become necessary for an issuer to discontinue
use of a set of Connection IDs.  For example, suppose the Connection IDs are
encrypted and have an embedded key identifier.  If the key referenced by this
identifier is about to be replaced, the issuer will need to close any
connections on which Connection IDs bound to the old key are still active.

In other scenarios, an endpoint might wish to provoke a change of Connection ID
from its peer in order to reduce linkability.  While implementations are required
to change Connection IDs when the local or remote address of the connection changes,
there is no way to force a rotation without performing migration and triggering
associated activities (Section 9.3 of {{!QUIC}}).

This document defines a simple extension to QUIC by which an issuer can prompt
the retirement of a previously-issued Connection ID.

# Extension Definition

## please_retire_cid_supported Transport Parameter

This document defines one new tranport parameter:

please_retire_cid_supported (0xTBD1):
: This extension is supported and the frame type 0xTBD2 can be understood.  This
  transport parameter MUST have a length of zero.

If the transport parameter is present, this extension is supported.  If the peer
does not supply this transport parameter, the PLEASE_RETIRE_CONNECTION_ID frame
MUST NOT be sent during the connection.

## PLEASE_RETIRE_CONNECTION_ID Frame

An endpoint sends a PLEASE_RETIRE_CONNECTION_ID frame (type=0xTBD2) to indicate
that it no longer wishes its peer to use a connection ID that it previously
issued. This may include the connection ID provided during the handshake.

The PLEASE_RETIRE_CONNECTION_ID frame is as follows:

~~~ drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Sequence Number (i)                    ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

PLEASE_RETIRE_CONNECTION_ID frames contain the following fields:

Sequence Number:
: The sequence number of the connection ID to be retired. See Section 5.1.2 of
  {{!QUIC}}.

An endpoint cannot send this frame if it provided its peer with a zero-length
connection ID. An endpoint that is using a zero-length connection ID MUST treat
receipt of a PLEASE_RETIRE_CONNECTION_ID frame as a connection error of type
PROTOCOL_VIOLATION.

The PLEASE_RETIRE_CONNECTION_ID frames are retransmitted if the packet
containing them is declared lost.  The frames MAY also be retransmitted
if the packet containing them arrives successfully, but the Connection ID
in question is not retired after a reasonable length of time.

### Frame Processing

Upon receipt of a PLEASE_RETIRE_CONNECTION_ID frame, an endpoint SHOULD
immediately cease using the indicated Connection ID and send a
RETIRE_CONNECTION_ID frame containing the same Sequence Number.  If the
indicated connection ID is the only currently active connection ID, the
recipient MAY defer this action until an alternate connection ID has been
received.

Receipt of a PLEASE_RETIRE_CONNECTION_ID frame containing an unknown sequence
number MUST NOT be treated as an error. If the sequence number is less than the
greatest sequence number previously received from the peer, the recipient SHOULD
immediately send a RETIRE_CONNECTION_ID frame for the indicated sequence number.
The NEW_CONNECTION_ID frame can be ignored on receipt.

If the indicated sequence number is greater than any previously received from
the peer, the recipient SHOULD remember that it has been asked to retire the
Connection ID in question and retire it immediately upon receipt, but MAY
discard the frame.

This specification does not modify the requirement that implementations not send
a RETIRE_CONNECTION_ID frame containing a sequence number greater than any
previously received.

# Security Considerations {#security}

This extension helps to mitigate some attacks on Connection ID issuers by
recipients who fail to retire old connection IDs.  While the issuer is in full
control of the number of connection IDs issued, the consumer can force the
maintenance of state from the beginning of a long-lived connection by using only
newer connection IDs.

However, if a consumer assiduously tracks future connection IDs it has been
asked to retire upon receipt, this is also a potential security threat.  The
rules for retransmission and processing are intended to ensure that consumers
can safely drop frames which reference unknown sequence numbers and still reach
a consistent state with a well-behaved peer.

# IANA Considerations {#iana}

## Transport Parameter Registration

This document registers one entry in the "QUIC Transport Parameters" registry
established by {{!QUIC}}.

Value:
: 0xTBD1

Parameter Name:
: please_retire_cid_supported

Specification:
: This document

## Frame Type Registration

This document registers one entry in the "QUIC Frame Type" registry established by {{!QUIC}}.

Value:
: 0xTBD2

Frame Name:
: PLEASE_RETIRE_CONNECTION_ID

Specification:
: This document

--- back

