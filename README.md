# Introduction
LamaMQ protocol V.1
- [Types](#types)
- [Errors](#errors)
- [Handshake](#handshake)
- [Messages](#messages)

# Types
All types related in the message protocol are explained here ;

## Basic

Name |  Description
-----|---------------
UINT8 | Unsigned 8 bits
UINT16 | Unsigned 16 bits
UINT32 | Unsigned 32 bits
ARRAY | UINT16 defining the size of the array (array copy the following type)
STRING | UINT32 defining the size of the string, following string.
BUFFER | UINT32 defining the size of the buffer (in byte) following the buffer.

## Structures

Name | Description
-------|--------------
[Message header](#structure-message-header) | Message header is used to communicate between a client a server.
[Message](#structure-message) | A message is the basic data transmited on the server.

### Structure - Message header
Message header is used to communicate between a client a server.
```
Message header{
  Message id => UINT32
  Message type => UINT8
}
```
Field | Description
------|-------------
Message id | Contain an id, the id need to be uniq at a given time, only between the client and the server. The server could use same id at a given to communicate with another client. And in one way, the server and the client can use both the same id at a given time (for standard message). The message id is used to match the answer with the request.
Message type | A number to know which [message](#messages) the client or the server have received.

### Structure - Message
A message is the basic data transmited on the server.
```
Message{
  created at => UINT32
  routing key => STRING
  headers => ARRAY (header)
  header{
    key => STRING
    value => STRING
  }
  payload type => UINT8
    - [payload type = 0] => payload_empty
    - [payload type = 1] => payload_string
    - [payload type = 2] => payload_buffer

  payload_empty{}

  payload_string{
    payload => STRING
  }

  payload_buffer{
    payload => BUFFER
  }
}
```
Field | Description
------|-------------
created at | A timestamp set by the message emitter to track when the message was created
routing key | The routing key used to know where the message need to be forwarded
headers | You can set multiple headers, headers could be processed by the message receiver or a middleware.
header.key | The key of the header (like a variable name).
header.value | The value of the header (like a variable value).
payload type | Define the content of the message.
payload_empty | If payload type=0, the message contains nothing more.
payload_string | If payload type=1, the message contains a string. Could be use to share data in JSON, YAML, XML...
payload_buffer | If payload type=2, The message contains a buffer.

# Errors
Error code help create easy exception matching the case of the errors.

Error code  | Error Name | Description 
-------------|--------------|-------------------------
 1   | Server Exception | Unknown error from the server 
 2   | Bad request | The request make do not follow expected payload 

# Handshake
When a client connect to a Lama node. It need to init a handshake, the sequence of messages does not evolve.

## Sequence
- Client => Server : Authentification
- Server => Client : Supported protocol version
- Client => Server : Choose a protocol version to use

## Authentification

Client call the server, for authentification.
```
auth_method => UINT8
identity_name => STRING
identity_key => [BUFFER]
```

Field | Description
------|--------------
auth_method | Version of the auth method (always 0 currently).
identity_name | The name of the client
identity_key | Optional buffer containing a key for identifily uniquely a client

## Supported protocol version
Server answer (if success) with the protocol version supported

```
max_version_major_supported=UINT8
max_version_minor_supported=UNIT8
min_version_major_supported=UINT8
min_version_minor_supported=UINT8
```

Field | Description
-------|-----------
max_version_major_supported | Max version supported (major.minor.patch) of the protocol
max_version_minor_suppored | Max version supported (major.minor.patch) of the protocol
min_version_major_supported | Min version supported (major.minor.patch) of the protocol
min_version_minor_supported | Min version supported (major.minor.patch) of the protocol

## Choose the protocol version to use
Client answer
```
version_used_major=UINT8
version_used_minor=UINT8
```

# Messages

Message Code | Message Name | Description 
-------------|---------------|--------------
0 | [Basic ACK](#basic-ack) | Standard answer for many message
1 | [Push message](#push-message) | Push a message on a given server
2 | [Forward message](#forward-message) | Transmit a message from the server to a client or another server
32 | [Topic subscribe](#topic-subscribe) | Register a client to a specific topic
33 | [Topic subscription ACK](#topic-subscription-ack) | ACK to a topic subscription
34 | [Topic unsubscribe](#topic-unsubscribe) | Unregister to a topic
64 | [Middleware register](#middleware-register) | Register a new middleware
65 | [Middleware registration ack](#middleware-registration-ack) | ACK to a middleware subscription
66 | [Middleware unregister](#middleware-unregister) | Remove a middleware

## Basic ACK

```
message header => message header (with message type = 0)
error code => INT8
error message => STRING
```
Field | Description
------|-------------
[message header](#structure-message-header) | The message header with, message type = 0
error code | If error code = 0, no error, the message was processed successfuly. [Error code will match the errors listed here.](#errors)
error message | Optional message if error code > 0

## Push message

Push a message on a server
```
message header => message header (with message type = 1)
mesage => message
```
Field | Description
------|-------------
[message header](#structure-message-header) | The message header with, message type = 1
[message](#structure-message) | The message content

Server will answer with [Basic ack](#basic-ack) when message will be processed.

## Forward message
Forward a message on a client or a server
```
message header => message header (with message type = 2)
mesage => message

```
Field | Description
------|-------------
[message header](#structure-message-header) | The message header with, message type = 2
[message](#structure-message) | The message content

Client will answer with [Basic ack](#basic-ack) when message will be processed.
## Topic subscribe
Subscribe to a topic. All message mathing the topic will be forwarded on the client
```
message header => message header (with message type = 32)
routing key => STRING
```
Field | Description
------|-------------
[message header](#structure-message-header) | The message header with, message type = 32
routing key | The routing key used to know when message need to be routed to this suscribtion

The server response with [a topic subscribtion ACK](#topic-subscription-ack).

## Topic subscription ack
```
message header => message header (with message type = 33)
suscribtion id => UINT32
```
Field | Description
------|-------------
[message header](#structure-message-header) | The message header with, message type = 33
subscription id | A unique id, use to refer this suscribtion (used during message forwarding).

## Topic unsubscribe
Unsubscribe to a topic.
```
message header => message header (with message type = 34)

```
Field | Description
------|-------------
[message header](#structure-message-header) | The message header with, message type = 34
subscription id | The unique id used to refer this suscribtion.

## Middleware register
```
message header => message header (with message type = 64)

```
Field | Description
------|-------------
[message header](#structure-message-header) | The message header with, message type = 64

The server will answer with [a middleware registration ack](#middleware-registration-ack).

## Middleware registration ack
```
message header => message header (with message type = 65)

```
Field | Description
------|-------------
[message header](#structure-message-header) | The message header with, message type = 65

## Middleware unregister
```
message header => message header (with message type = 66)

```
Field | Description
------|-------------
[message header](#structure-message-header) | The message header with, message type = 66
