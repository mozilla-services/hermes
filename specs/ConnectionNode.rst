===============================
Channel Service Connection Node
===============================

Mozilla Services Channel Service Connection Node handles holding open large
quantities of HTTP2 connections, tracking open URI's, and routing
requests/responses to/from clients and to/from other Mozilla Service services.

Overview: https://wiki.mozilla.org/CloudServices/FirefoxMobileServices/ChannelService#Cloud_Services_Channel_Service

In effect, the Connection Node serializes HTTP requests to a message stream
and deserializes responses to reply with from a returning message stream.

Overview
========

The Connection Node (CN) performs four primary tasks:

1. Communicates with clients (FxOS, FF, etc) over a HTTP/2 connection
2. Pushes a list of open URI's to Outbound Relay's
3. Routes HTTP requests from clients over a TCP connection to the appropriate
   Inbound Relay
4. Accepts messages containing HTTP responses for delivery over a TCP connection
   from Outbound Routers

Secondary tasks:

* Return a HTTP Error 503 to connected clients when a service for a request is
  unavailable to answer it.
* Return a HTTP Error 504 to connected clients when a service fails to respond to
  a client in a timely manner.

If the IR/OR connection is lost, clients connected may be retained, and
appropriate errors will be sent as documented.

The CN will always retry connection attempts to all the IR's/OR's it is
configured to connect to.

Configuration
=============

The domains and URL valid URL spaces for each domain are specified in a INI-
style configuration format. The ConnectionNode must be supplied with a valid
SSL certificate for all domains that it is expected to answer for.

URL spaces configured with no TTL are considered 'open' URI's that will result
in the Outbound Router getting notified of.

Example:

.. code-block:: ini

	[webpush]

	domain = c92.webpush.svcs.mozilla.org
	urls =
		POST /v1/webpush 5
		GET /v1/channel/ 5
		POST /v1/channel/ 5
		DELETE /v1/channel/* 5
		GET /v1/monitor/*

Each block is identified by the service name that all requests will be
addressed under. The domain, request method, and URL must match the configured
settings for a request to not immediately 404, at which point it will be
serialized to a message and sent out. The last number after the method/URL is
the TTL for how long a request may be outstanding before it will be 504'd.
