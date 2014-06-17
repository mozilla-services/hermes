===============================
Channel Service Connection Node
===============================

Mozilla Services Channel Service Connection Node handles holding open large
quantities of sockets, tracking clients connected, and routing messages
to/from clients and to/from other Mozilla Service services.

Overview: https://wiki.mozilla.org/CloudServices/FirefoxMobileServices/ChannelService#Cloud_Services_Channel_Service

Overview
========

The Connection Node (CN) performs four primary tasks:

1. Communicates with clients (FxOS, FF, etc) over a websocket
2. Pushes a list of connected clients to Outbound Relay's
3. Routes messages from clients over a TCP connection to the appropriate Inbound Relay
4. Accepts messages for delivery over a TCP connection from Outbound Routers

Secondary tasks:

* Indicate to connected clients when a message can't be delivered because the
  service its destined for is unavailable.
* Indicate to applications via the Inbound Relay messages that were delivered or not

When the CN starts up, before accepting clients it establishes a
connection to at least one Inbound Relay (IR) and one Outbound Router
(OR) (ideally 2 for redundancy). Once connections are established,
Clients are permitted to connect.

If the IR/OR connection is lost, clients connected may be retained, and
appropriate errors will be sent as documented.

The CN will always retry connection attempts to all the IR's/OR's it is configured
to connect to.

Guarantees
==========

To prevent difficult to diagnose error situations, a CN must:

* Deliver no messages for the Outbound Router if there is no Inbound Relay
  connected

Client API v1
=============

This is the client <-> connection node (CN) API that clients must implement for
delivery/reception of messages to Mozilla Services.

When a client disconnects intentionally or not, the CN must braodcast to all
Outbound Routers that the DeviceID is no longer connected.

The client must send/receive data or send a ping at least once ever PINGINTERVAL.

Helo
----

Upon initial connection, the client and server must agree on the supported version of
the protocol to speak.

**Client -> Server**

Example helo:

.. code-block:: txt

	HELO:v1:PINGINTERVAL

The PINGINTERVAL value is the longest without data sent/received the client can
handle before the connection should be considered invalid. This value must be
an integer.

**Server -> Client**

Example success:

.. code-block:: txt

	HELO:v1

If the server cannot speak the version requested, it will drop the connection.


Auth
----

**Client -> Server**

Upon opening a websocket to the CN, the client must authenticate itself with the DeviceID
and key that it was given on its first connection. If this is the first connection, then
no DeviceID/Key should be supplied.

Example first auth:

.. code-block:: txt

	AUTH:

Example later connects:

.. code-block:: txt

	AUTH:6ba7b810-9dad-11d1-80b4-00c04fd430c8:90761ab1794e4d68b83743c68c85acc8

**Server -> Client**

The server will respond with either:

* A success message
* An error if the Key is invalid
* A new DeviceID/Key if its a first-time auth
* A redirect indicating the client should change clusters its connecting to
* A retry if the DeviceID/Key is valid, but the server is full so the client should
  try a different IP for this cluster name

Example Success:

.. code-block:: txt

	AUTH:SUCCESS

Example key error:

.. code-block:: txt

	AUTH:INVALID

Example new device ID:

.. code-block:: txt

	AUTH:NEW:6ba7b810-9dad-11d1-80b4-00c04fd430c8:90761ab1794e4d68b83743c68c85acc8

Example redirect:

.. code-block:: txt

    AUTH:REDIRECT:cluster35.svcs.mozilla.com

Example retry:

    AUTH:RETRY

If the client receives an error, it must re-auth with no credentials to
get a new DeviceID and clear its old one out locally. It must not
broadcast a DeviceID change request as it did not properly auth.

If the client already has a DeviceID and has received a new one, it must alert
all client service code with the old DeviceID and new one so that the client
service code can ensure a transition occurs if needed. For example, with WebPush
a DeviceID change will require the client service code to update the WebPush
Endpoints for applications what use WebPush.

Should the remote service need to be informed of the client DeviceID change, the
client must send a DeviceIDChange message until its confirmed for the service(s)
that need to be aware of it.

If the client receives a redirect, it should drop the connection and connect to
the provided host-name. The client may be told to reconnect to the same cluster
which is used to indicate that a different IP for the host-name should be used.
For this reason, when looking up the IP for a host, the client should store
the response and cycle through the IP's as long as the redirect is the same
host-name.

The client must always store the DeviceID:Key and the cluster to
connect to, along with applications that have used it.

After authentication has completed, normal message delivery mode commences.

Ping
----

Sent on occasion to ensure the connection is still alive. The client gets to
choose the ping interval based on its own heuristics.

**Client -> Server**

Example:

.. code-block:: txt

    PING

**Server -> Client**

.. code-block:: txt

    PONG

DeviceID Change
---------------

**Client -> Server**

If a client has been issued a new DeviceID during AUTH that involved sending a
DeviceID, it must send a message indicating the change for each Service that
has used Channel Service with the old DeviceID that needs it. The client must
track which Services it has communicated with.

Format: ``DEVICECHANGE:SERVICE-NAME:OLD-ID:OLD-KEY:NEW-ID``

Example:

.. code-block:: txt

	DEVICECHANGE:WEBPUSH:0ce37cb2-d4fc-42d1-b0aa-5e6360c001c4:6ba7b810-9dad-11d1-80b4-00c04fd430c8:e343d79b-6380-451f-b549-16c8a7ee91bc

The CN **must verify the old DeviceID/Key and new DeviceID for accuracy before
sending it to the Inbound Relay**.


**Server -> Client**

When the connection node has successfully delivered the message to the
service, it will return a basic ACK:

.. code-block:: txt

	DEVICECHANGE:WEBPUSH:ACK

If the service is unable to verify the old DeviceID key supplied, an invalid will
be returned and the Service will not be notified:

.. code-block:: TXT

    DEVICECHANGE:WEBPUSH:INVALID

If the CN is unable to deliver the message to the service, it will be ``NACK``
instead of ``ACK`` and the client must try again later before it may resume
sending messages.

The client cannot deliver any messages to the Service until a further service
acknowledgment message is received. For services that have responded with an
the new DeviceID, the client may send messages and will receive them as normal.

If a new Device ID was not assigned to this client, DEVICECHANGE messages will
be discarded silently by the CN.

**Server -> Client**

When the service acknowledges the message, it will send a message with the
format: ``DEVICECHANGED:SERVICE-NAME:NEW-ID``

Example:

.. code-block:: txt

	DEVICECHANGED:WEBPUSH:e343d79b-6380-451f-b549-16c8a7ee91bc


Outgoing Messages
-----------------

**Client -> Server**

Delivering data to a Service is done via simple addressing of the data
including a message-id for tracking acknowledgment:
 ``OUT:SERVICE-NAME:MESSAGE-ID:BODY``

Example:

.. code-block:: txt

	OUT:WEBPUSH:20893117-7937-474b-909b-0c78ec03d0eb:{"messageType": "hello","uaid": "fd52438f-1c49-41e0-a2e4-98e49833cc9c","channelIDs":[]}

The client may make deliver as many messages at a time as desired and does not
need to wait for each reply individually before sending more.

**Server -> Client**

To save bandwidth, messages received by the server will not be ACK'd unless
the message spool for that Service is full (remote service is unavailable). In
that event a NACK will be sent for the messages that could not be spooled.

Example:

.. code-block:: txt

	OUT:WEBPUSH:20893117-7937-474b-909b-0c78ec03d0eb:NACK

The client should then assume that the Service is temporarily unavailable and
try again later.


Incoming Messages
-----------------

**Server -> Client**

Data sent to a client through CN includes the Service-Name it belongs to so
that the client may route it appropriately. The incoming data is in the format:
 ``INC:WEBPUSH:BODY``

Example:

.. code-block:: txt

	INC:WEBPUSH:{"messageType":"hello","status":200,"uaid":"fd52438f-1c49-41e0-a2e4-98e49833cc9c","connected":1399049780}

No response to the server is necessary.
