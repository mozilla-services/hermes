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

On startup, the OR will listen for connections from CN's and Mozilla Services.

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

Connection Node API v1
======================

CN's connect to the OR, say HELO with the version they expect, send the
current list of clients connected, then proceed to accept messages and
send client connection updates.

Helo (1)
--------

An initial Helo command is sent to verify the server supports the
version desired. A reply will be sent echo'ing the Helo with the
version if its supported, otherwise the highest version supported will
be returned and the connection closed by the server.

The maximum message batch size is the amount of messages that can be
processed in parallel. No more than the max batch size of messages
should be sent to the CN at once before replies should be checked.

If sending/receiving in parallel, the OR must ensure no more than this
batch size of messages are outstanding.

**CN -> OR**

Header: 1

Data:

    Version (Integer)

    Maximum Message Batch (Integer)

**OR -> CN**

If the version does not match what is sent, the connection will be dropped by
the server after returning this response.

Header: 1

Data:

    Version (Integer)


Device Listing (2)
------------------

When a CN finishes saying HELO, it send a full list of all DeviceID's
connected.

**CN -> OR**

Header: 2

Data:

    Device count (Integer - N)

    List of devices (CSUUID * N)


Device Connection Change (3)
----------------------------

Anytime clients connect and complete authentication, or disconnect, a Device
Connection Change message will be sent to the OR.

**CN -> OR**

Header: 3

Data:

    DeviceID (CSUUID)

    Change (Integer)

Integer values:

    0 - Disconnected
    1 - Connected


DeviceID Change (4)
-------------------

When a DeviceID change occurs, the CN will send it to the IR and eventually
get a confirmation from the Service via the OR.

**OR -> CN**

Header: 3

Data:

    MessageID (UUID)

    DeviceID (CSUUID)

    Old DeviceID (CSUUID)

**CN -> OR**

Header: 3

Data:

    MessageID (UUID)

    Response (Integer)

Response values:

    0 - Delivered to the device

    1 - Error delivering to the device

In the event that there was an error, the OR may safely discard the message as
the client will try again before resuming its message flow for the Service.


Deliver Message (5)
-------------------

Messages may be processed for delivery in parallel, it is up to the OR
to associate replies with the outbound message that was sent.

**OR -> CN**

Header: 2

Data:

    MessageID (UUID)

    DeviceID (CSUUID)

    Data (String)

**CN -> OR**

Header: 2

Data:

    MessageID (UUID)

    Response (Integer)

Response values:

    0 - Delivered to the device

    1 - Error delivering to the device

    2 - Device is not connected to this node

    3 - Inbound Relay is not available


Service API v1
==============

A service connects to the OR to send messages intended for device's in that
cluster.


Helo (1)
--------

An initial Helo command is sent to verify the server supports the
version desired.


**Service -> OR**

Header: 1

Data:

    Version (Integer)

    Service Name (String, assumed to be ascii)

**OR -> Service**

If the version does not match what is sent, the connection will be dropped by
the Service after returning this response. The maximum message batch size is the
amount of messages that can be processed in parallel. No more than the max
batch size of messages should be sent at once before replies should be checked.

If sending/receiving in parallel, the OR will ensure no more than this
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

    MessageID (UUID)

    DeviceID (CSUUID)

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


DeviceID Change (3)
-------------------

When a DeviceID change occurs, the CN will send it to the IR and eventually the
server should send this message to the OR to be relayed back to the device.

**Service -> OR**

Header: 3

Data:

    MessageID (UUID)

    DeviceID (CSUUID)

    Old DeviceID (CSUUID)

**OR -> Service**

Header: 3

Data:

    MessageID (UUID)

    Response (Integer)

Response values:

    0 - Delivered to the device

    1 - Error delivering to the device

In the event that there was an error, the OR may safely discard the message as
the client will try again before resuming its message flow for the Service.
