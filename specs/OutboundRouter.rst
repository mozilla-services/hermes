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
bytes, int) followed by the 16 byte UUID. A bool is a single byte of
0 for False, 1 for True.

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


URI Listing (2)
---------------

When a CN finishes saying HELO, it send a full list of all open URI's
available for PUSH.

**CN -> OR**

Header: 2

Data:

    URI count (Integer - N)

    List of devices (String * N)


URI Change (3)
--------------

When a client connected to a CN drops a URI that was left open, or a new
client connects and opens a URI with no TTL (to be left open), then a URI
Change message will be sent.

**CN -> OR**

Header: 3

Data:

    URI (String)

    Change (Int)

Change values:

    0 - URI is no longer present

    1 - New URI that is now available


Deliver Response Message (4)
----------------------------

Messages may be processed for delivery in parallel, it is up to the OR
to associate replies with the outbound message that was sent.

This API is for sending messages in response to a request that was made.

**OR -> CN**

Header: 4

Data:

    MessageID (UUID)

    Headers (Hash String/String)

    Body (String)

**CN -> OR**

Header: 4

Data:

    MessageID (UUID)

    Response (Integer)

Response values:

    0 - Delivered to the device

    1 - Error delivering to the device

    2 - Device is not connected to this node


Deliver URI Message (5)
-----------------------

Messages may be processed for delivery in parallel, it is up to the OR
to associate replies with the outbound message that was sent.

This API is for sending messages to an open URI, setting Push to 1 will
result in the open URI being used to deliver Push Promises + Bodies, while
setting Push to 0 will result in the request being answered with this 
response and the URI will then be closed.

MessageID should be fabricated to track the response.

**OR -> CN**

Header: 5

Data:

    URI (String)

    MessageID (UUID)

    Headers (Hash String/String)

    Body (String)

    Push (Bool)


**CN -> OR**

Header: 5

Data:

    MessageID (UUID)

    Response (Integer)

Response values:

    0 - Accepted for delivery

    1 - Invalid URI


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


Deliver Response Message (2)
----------------------------

The Outbound Router delivers messages to the appropriate CN for the device,
each message must be acknowledged.

**Service -> OR**

Header: 2

Data:

    ConnectionNode (UUID)

    MessageID (UUID)

    TTL (long long)

    Headers (Hash String/String)

    Body (String)


Deliver URI Message (3)
-----------------------

Messages may be processed for delivery in parallel, it is up to the OR
to associate replies with the outbound message that was sent.

This API is for sending messages to an open URI, setting Push to 1 will
result in the open URI being used to deliver Push Promises + Bodies, while
setting Push to 0 will result in the request being answered with this 
response and the URI will then be closed.

MessageID should be fabricated to track the response.

**OR -> CN**

Header: 3

Data:

    URI (String)

    MessageID (UUID)

    Headers (Hash String/String)

    Body (String)

    Push (Bool)


**CN -> OR**

Header: 5

Data:

    MessageID (UUID)

    Response (Integer)

Response values:

    0 - Accepted for delivery

    1 - Invalid URI
