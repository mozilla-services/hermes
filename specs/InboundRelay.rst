=============================
Channel Service Inbound Relay
=============================

Mozilla Services Channel Service (CS) Inbound Relay (IR) accepts connections
from  CS Connection Nodes (CN) intended for Mozilla Services (such as
SimplePush). Connections are also accepted from the Services that messages are
intended to be delivered to.

Messages may be buffered in memory for temporary Service failures depending
on the configuration.

Overview
========

The Inbound Relay performs two primary tasks:

1. Accepts messages for delivery from Connection Nodes over a TCP connection
   intended for Services.
2. Delivers messages over a TCP connection to connected Services.

Secondary tasks:

* Spool messages for a Service if it loses its connection

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

CN's connect to the IR, say HELO with the version expected, and proceed to
send messages. Every message will be ACK'd if delivery was successful.

Helo (1)
--------

An initial Helo command is sent to verify the server supports the version
desired.

**CN -> IR**

Header: 1

Data:

    Version (Integer)

**IR -> CN**

If the version does not match what is sent, the connection will be dropped by
the server after returning this response. The maximum message batch size is
the amount of messages that can be processed in parallel. No more than the max
batch size of messages should be sent at once before replies should be
checked.

If sending/receiving in parallel, ensure no more than this batch size of
messages are outstanding.

Header: 1

Data:

    Version (Integer)

    Maximum Message Batch (Integer)


Deliver Message (2)
-------------------

Messages may be processed for delivery in parallel, it is up to the OR to
associate replies with the outbound message that was sent.

**CN -> IR**

Header: 2

Data:

    Message-ID (UUID)

    DeviceID (CSUUID)

    Service Name (String)

    Data (String)

**IR -> CN**

Header: 2

Data:

    Message-ID (UUID)

    Response (Integer)

Response values:

    0 - Delivered to the Service

    1 - Spooled

    2 - Service spool is full/unavailable

If the response is ``2``, then the CN should temporarily stop attempting to
deliver messages for that service.


DeviceID Change (3)
-------------------

When a DeviceID change occurs, the CN will send it to the IR and eventually
get a confirmation from the Service via the OR.

**CN -> IR**

Header: 3

Data:

    MessageID (UUID)

    DeviceID (CSUUID)

    Old DeviceID (CSUUID)

    Service Name (String)

**IR -> CN**

Header: 3

Data:

    MessageID (UUID)

    Response (Integer)

Response values:

    0 - Delivered to the Service

    1 - Spooled

    2 - Service spool is full/unavailable

If the response is ``2``, then the CN should temporarily stop attempting to
deliver messages for that service.


Service API v1
==============

A service can connect to the IR multiple times if needed, in which case
messages will be sent to a randomly chosen connection.

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

The Inbound Relay delivers messages to the Service, each message must be
acknowledged. Failure to acknowledge a message will result in repeat sends of
the message until it is acknowledged.

**IR -> Service**

Header: 2

Data:

    MessageID (UUID)

    DeviceID (CSUUID)

    Data (String)

**Service -> IR**

Header: 2

Data:

    MessageID (UUID)

    Success (Integer)

Success values:

    0 - Accepted


DeviceID Change (3)
-------------------

When a DeviceID change occurs, the client will send a DEVICECHANGE message to
the CN that the IR will relay to the server.

**IR -> Service**

Header: 3

Data:

    MessageID (UUID)

    DeviceID (CSUUID)

    Old DeviceID (CSUUID)

**Service -> IR**

Header: 3

Data:

    MessageID (UUID)

    Response (Integer)

Response values:

    0 - Accepted
