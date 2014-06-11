===============================
Channel Service Outbound Router
===============================

Mozilla Services Channel Service (CS) Outbound Router (OR) accepts connections
from Mozilla Services (such as SimplePush) intended for delivery to a device.
The OR connects to the Connection Nodes (CN) to deliver messages and provides
immediate replies to the Service if delivery succeeded.

Overview
========

The Outbound Router performs two primary tasks:

1. Accepts messages for delivery from Services over a connection
   intended for devices.
2. Delivers messages over a connection to CN's for delivery.

Secondary tasks:

* Retain a DeviceID list indicating which DeviceID is connected to each
  CN.

TCP Connection Protocol
=======================

All byte order is network (big-endian), characters are expected to be
standard char (1-byte) in length. Integers are expected to be unsigned
short (2-bytes). Strings are prefixed by a 4-byte ``int`` indicating
the length of characters being sent.

UUID's are encoded as binary data, 16 bytes in length. CSUUID is a
cluster UUID which should be treated as a cluster identifier (first 2
bytes, int) followed by the 16 byte UUID.

All commands begin with a header packet containing an unsigned short (2
bytes) that corresponds to the type of the command being sent/received.
The commands integer type is shown in parenthesis.

There is no authentication in this protocol so firewall settings should
be set appropriately.

Service API v1
==============

A service connects to the OR to send messages intended for device's in that
cluster.


Helo (1)
--------

An initial Helo command is sent to verify the server supports the
version desired.


**Service -> IR**

Header: 1

Data:

    Version (Integer)

    Service Name (String, assumed to be ascii)

**IR -> Service**

If the version does not match what is sent, the connection will be dropped by
the Service after returning this response. The maximum message batch size is the
amount of messages that can be processed in parallel. No more than the max
batch size of messages should be sent at once before replies should be checked.

If sending/receiving in parallel, the IR will ensure no more than this
batch size of messages are outstanding.

Header: 1

Data:

    Version (Integer)

    Maximum Message Batch (Integer)


Deliver Message (2)
-------------------

The Outbound Router delivers messages to the appropriate CN for the device,
each message must be acknowledged.

**Service -> OR**

Header: 2

Data:

    DeviceID (CSUUID)

    MessageID (UUID)

    Data (String)

**OR -> Service**

Header: 2

Data:

    MessageID (UUID)

    Response (Integer)

Response values:

    0 - Delivered to device

    1 - Unable to deliver to device (not connected)

    2 - Invalid cluster for this DeviceID

    3 - Inbound Relay is not available

    4 - Error delivering

If the CN is unable to talk to an IR, then the message will be rejected with
response 3. The Service should retry again later when the IR is connected to
the CN.

Note that in the event of an *Error delivering*, it is possible that the
message was actually delivered to the device but verification was not possible
because of any of several reasons (connection drop to the CN, connection drop
to the Device from the CN, etc).
