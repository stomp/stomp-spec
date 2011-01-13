# STOMP Protocol Specification, Version 1.1

{:toc:2-5}

## DRAFT STATUS

Version 1.1 of the specification is still being developed. This is only
a draft document.

## Overview

STOMP is a simple interoperable wire format designed for asynchronous
message passing between clients via mediating servers.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119.

## STOMP Frames

The client and server will communicate using STOMP frames sent over
the stream. A frame's structure looks like:

    command
    header1:value1
    header2:value2
    
    Body^@

The frame starts with a command string terminated by a newline. Following the 
command are one or more header entries in `<key>:<value>` format. Each header 
entry is terminated by a newline. A blank line indicates the end of the headers 
and the beginning of the body. The body is then followed by the null byte (0x00). 
The examples in this document will use `^@`, control-@ in ASCII, to represent the 
null byte. The null byte can be optionally followed by multiple newlines. For more
details, on how to parse STOMP frames, see the [Augmented
BNF](#augmented_bnf) section of this document.

The headers are encoded in UTF-8. Since the colon and newline characters are
used to delimit the keys and values, c style string literal escapes are used
to encode any colons and newlines that are included within the headers. When decoding 
frame headers, the following transformations should get applied:

* `\n` translates to newline (octect 10)
* `\c` translates to `:`
* `\\` translates to `\`

Conversely when encoding frame headers, the reverse transformation should be
applied.

Only the `SEND`, `MESSAGE`, and `ERROR` frames can have a body. All other
frames MUST not have a body.

The STOMP 1.0 specification included many example frames with padding in the
headers and many servers and clients were implemented to trim or pad header
values. This causes problems if applications want to send headers that should
not get trimmed. In STOMP 1.1, clients and servers MUST never trim or pad or
headers with spaces.

To prevent malicious clients from exploiting memory allocation in a 
server, servers may place maximum limits on:

* the number of frame headers allowed in a single frame
* the maximum length of header lines
* the maximum size of a frame body

If these limits are exceeded the server should send the client an `ERROR`
frame and disconnect the client.

### Repeated Header Entries

Since messaging systems can be organized in store and forward topologies,
similar to SMTP, a message may traverse several messaging servers before
reaching a consumer. The intermediate server may 'update' header values in the
message by prepending headers to the message.

If the client receives repeated frame header entries, only the first header
entry should be used as the value of header entry. Subsequent values are only
use to maintain a history of state changes of the header. For example, if the
client receives:

    MESSAGE
    foo:World
    foo:Hello
    
    ^@

The value of the `foo` header is just `World`.

## Connecting

A STOMP client initiates the stream or TCP connection to the server by sending 
the `CONNECT` frame:

    CONNECT
    accept-version:1.1
    host:stomp.github.org
              
    ^@

If the server accepts the connection attempt it will respond with a
`CONNECTED` frame:

    CONNECTED
    version:1.1
    
    ^@

The sever may reject any connection attempt. The server SHOULD respond back
with an `ERROR` frame listing why the connection was rejected and then close the 
connection. Since STOMP servers must support clients which rapidly connect and 
disconnect, a server will likely only allow closed
connections to linger for short time before the connection is reset. This means
that a client may not fully receive the `ERROR` frame before the socket is
reset.

### CONNECT or STOMP Frame

STOMP servers should handle a `STOMP` frame in the same manner as a `CONNECT` frame.
STOMP 1.1 clients should continue to use the `CONNECT` command to remain
backward compatible with STOMP 1.0 servers.

Clients that use the `STOMP` frame instead of the `CONNECT` frame will only be
able to connect to STOMP 1.1 servers but the advantage is that a protocol
sniffer/discriminator will be able to differentiate the STOMP connection from
an HTTP connection.

STOMP 1.1 clients MUST set the following headers:

* `accept-version` : The versions of the STOMP protocol the client supports.
  See [Protocol Negotiation](#protocol_negotiation) for more details.

* `host` : The host name that the socket was established against. This allows
  the server to implement virtual hosts.

STOMP 1.1 clients MAY set the following headers:

* `login` : The user id used to authenticate against a secured STOMP server.

* `passcode` : The password used to authenticate against a secured STOMP
  server.

### CONNECTED Frame

STOMP 1.1 servers MUST set the following headers:

* `version` : The version of the STOMP protocol the session will be using.
  See [Protocol Negotiation](#protocol_negotiation) for more details.

STOMP 1.1 servers MAY set the following headers:

* `session` : A session id that uniquely identifies the session.  

## Protocol Negotiation

From STOMP 1.1 and onwards, the `CONNECT` frame MUST include the
`accept-version` header. It should be set to a comma separated list of
incrementing STOMP protocol versions that the client supports. If the
`accept-version` header is missing, it means that the client only supports
version 1.0 of the protocol.

The protocol that will be used for the rest of the session will be the
highest protocol version that both the client and server have in common.

For example, if the client sends:

    CONNECT
    accept-version:1.0,1.1,2.0
    host:stomp.github.org
              
    ^@

The server will respond back with the highest version of the protocol that
it has in common with the client:

    CONNECTED
    version:1.1
    
    ^@

If the client and server do not share any common protocol versions, then the
sever should respond with an `ERROR` frame similar to:

    ERROR
    version:1.2,2.1
    content-type:text/plain
              
    Supported protocol versions are 1.2 2.1^@

## Once Connected

Once the client is connected it may send any of the following frames:

* [`SEND`](#SEND)
* [`SUBSCRIBE`](#SUBSCRIBE)
* [`UNSUBSCRIBE`](#UNSUBSCRIBE)
* [`BEGIN`](#BEGIN)
* [`COMMIT`](#COMMIT)
* [`ABORT`](#ABORT)
* [`ACK`](#ACK)
* [`NACK`](#NACK)
* [`DISCONNECT`](#DISCONNECT)

## Client Frames

### SEND

The `SEND` frame sends a message to a destination in the messaging system. It
has one required header, `destination`, which indicates where to send the
message. The body of the `SEND` frame is the message to be sent. For example:

    SEND
    destination:/queue/a
    content-type:text/plain
    
    hello queue a
    ^@

This sends a message to a destination named `/queue/a`. Even though queue and
topic delivery semantics are the most popular in messing servers, STOMP does
not define what the delivery semantics of destinations should be. The
delivery, or "message exchange", semantics of destinations can vary from
server to server and even from destination to destination. This allows
servers to be creative with the semantics that they can support
with STOMP. You should consult your STOMP server's documentation to find out
how to construct a destination name which gives you the delivery semantics
that your application needs.

The reliability semantics of the message are also server specific and will 
depend on the destination value being used and the other message headers 
such as the `transaction` header or other server specific message headers.

`SEND` supports a `transaction` header which allows for transactional sends.

`SEND` frames should include a 
[`content-length`](#Header_content-length) header and a 
[`content-type`](#Header_content-type) header if a body is present.

An application may add any arbitrary user defined headers to the `SEND` frame.
User defined headers are typically used to allow consumers to filter
messages based on the application defined headers using a selector 
on a `SUBSCRIBE` frame. The user defined headers MUST be passed through
in the `MESSAGE` frame.

If the sever cannot successfully process the `SEND` frame frame for any reason,
the server MUST send the client an `ERROR` frame and disconnect the client.

### SUBSCRIBE

The `SUBSCRIBE` frame is used to register to listen to a given destination. Like
the `SEND` frame, the `SUBSCRIBE` frame requires a `destination` header indicating
the destination to which the client wants to subscribe. Any messages received on the 
subscribed destination will henceforth be delivered as `MESSAGE` frames from the 
server to the client.
The `ack` header controls the message acknowledgement mode.

Example:

    SUBSCRIBE
    id:0
    destination:/queue/foo
    ack:client
    
    ^@

If the sever cannot successfully create the subscription, 
the server MUST send the client an `ERROR` frame and disconnect the client.

STOMP servers may support additional server specific headers to customize the
delivery semantics of the subscription. Consult your server's documentation for
details.

#### SUBSCRIBE id Header

An `id` header must be included in the frame to uniquely identify the subscription within the
STOMP connection session. Since a single connection can have multiple open
subscriptions with a broker, the `id` header allows the client and broker to
relate subsequent `ACK`, `NACK` or `UNSUBSCRIBE` frames to the original
subscription.

#### SUBSCRIBE ack Header

The valid values for the `ack` header are `auto`, `client`, or
`client-individual`. If the header is not set, it defaults to `auto`.

When the the `ack` mode is `auto`, then the client does not need to send the
server `ACK` frames for the messages it receives. The server will assume the
client has received the message as soon as it sends it to the the client. This
acknowledgment mode can cause messages being transmitted to the client to get
dropped.

When the the `ack` mode is `client`, then the client must send the server `ACK`
frames for the messages it processes. If the connection fails before a client
sends an `ACK` for the message the server will assume the message has not been
processed and may redeliver the message to another client. The `ACK` frames sent
by the client will be treated as a cumulative `ACK`. This means the `ACK` operates
on the message specified in the `ACK` frame and all messages sent before the
messages to the subscription.

When the the `ack` mode is `client-individual`, the ack mode operates just like
the `client` ack mode except that the `ACK` or `NACK` frames sent by the client
are not cumulative. This means that an `ACK` or `NACK` for a subsequent message
should not cause a previous message to get acknowledged.

### UNSUBSCRIBE

The `UNSUBSCRIBE` frame is used to remove an existing subscription.  Once the subscription is
removed the STOMP connections will no longer receive messages from that destination. 
It requires that the `id` header matches the `id` value of previous `SUBSCRIBE` operation. 
Example:

    UNSUBSCRIBE
    id:0
    
    ^@

### ACK

`ACK` is used to acknowledge consumption of a message from a subscription using
`client` or `client-individual` acknowledgment. Any messages received from such
a subscription will not be considered to have been consumed until the message
has been acknowledged via an `ACK` or a `NACK`.

`ACK` has two required headers: `message-id`, which must contain a value
matching the `message-id` for the `MESSAGE` being acknowledged and `subscription`,
which must be set to match the value of the subscription's `id` header. 
Optionally, a `transaction` header may be specified, indicating that the message
acknowledgment should be part of the named transaction.

    ACK
    subscription:0
    message-id:007
    transaction:tx1

    ^@

### NACK

`NACK` is the opposite of `ACK`. It is used to tell the server that the
client did not consume the message. The server can then either send the message to a
different client, discard it, or put it in a dead letter queue. The exact
behavior is server specific.

`NACK` takes the same headers as `ACK`: `message-id` (mandatory),
`subscription` (mandatory) and `transaction` (optional).

`NACK` applies either to one single message (if the subscription's ack
mode is `client-individual`) or to all messages sent before and not yet
`ACK`'ed or `NACK`'ed.

### BEGIN

`BEGIN` is used to start a transaction. Transactions in this case apply to
sending and acknowledging - any messages sent or acknowledged during a
transaction will be handled atomically based on the transaction.

    BEGIN
    transaction:tx1

    ^@

The `transaction` header is required, and the transaction identifier will be
used for `SEND`, `COMMIT`, `ABORT`, `ACK`, and `NACK` frames to bind them to the
named transaction.

Any started transactions which have not been committed will be implicitly aborted
if the client sends a `DISCONNECT` frame or if the TCP connection fails for
any reason.

### COMMIT

`COMMIT` is used to commit a transaction in progress.

    COMMIT
    transaction:tx1

    ^@

The `transaction` header is required and must specify the id of the transaction to
commit\!

### ABORT

`ABORT` is used to roll back a transaction in progress.

    ABORT
    transaction:tx1

    ^@


The `transaction` header is required and must specify the id of the transaction to
abort\!

### DISCONNECT

A client can disconnect from the server at anytime by closing his socket but
but there is no guarantee that the previously sent frames have been received
by the server. To do a graceful shutdown, where the client is assured that all
previous frames have been received by the server, the client should:

1. send a `DISCONNECT` frame with a `receipt` header set.  Example:

        DISCONNECT
        receipt:77
        ^@

2. wait for the `RECEIPT` frame response to the `DISCONNECT`. Example:

        RECEIPT
        receipt-id:77
        ^@
        
3. close the socket.

Clients MUST not send any more frames after the `DISCONNECT` frame is sent.

## Standard Headers

Some headers may be used, and have special meaning, with most frames.

### Header content-length

The `SEND`, `MESSAGE` and `ERROR` frames should include a `content-length`
header if a frame body is present. The header is a byte count for the length
of the message body. If a `content-length` header is included, this number of
bytes must be read, regardless of whether or not there are null characters in
the body. The frame still needs to be terminated with a null byte. If a
`content-length` is not specified, the first null byte encountered signals
the end of the frame.

### Header content-type

The `SEND`, `MESSAGE` and `ERROR` frames should include a `content-type`
header if a frame body is present. It should be set to a mime type which
describes the format of the body to help the receiver of the frame interpret
it's contents. If the `content-type` header is not set, the receiver should
consider the body to be a binary blob.

The implied text encoding for mime types starting with `text/` is UTF-8. If
you are using a text based mime type with a different encoding then you
should append `;charset=<encoding>` to the mime type. For example,
`text/html;charset=utf-16` should be used if your sending an html body in
UTF-16 encoding. The `;charset=<encoding>` should also get appended to any
non `text/` mime types which can be interpreted as text. A good example of
this would be a UTF-8 encoded XML. It's `content-type` should get set to
`application/xml;charset=utf-8`

### Header receipt

Any client frame other than `CONNECT` may specify a `receipt`
header with an arbitrary value. This will cause the server to acknowledge
receipt of the frame with a `RECEIPT` frame which contains the value of this
header as the value of the `receipt-id` header in the `RECEIPT` frame.

    SEND
    destination:/queue/a
    receipt:message-12345

    hello queue a^@

## Server Frames 

The server will, on occasion, send frames to the client (in addition to the
initial `CONNECTED` frame). These frames may be one of:

* [`MESSAGE`](#MESSAGE)
* [`RECEIPT`](#RECEIPT)
* [`ERROR`](#ERROR)

### MESSAGE

`MESSAGE` frames are used to convey messages from subscriptions to the
client. The `MESSAGE` frame will include a `destination` header indicating
the destination the message was sent to. It will also contain a `message-id`
header with a unique identifier for that message. The `subscription` header
will be set to match the `id` header of the subscription that is receiving
the message. The frame body contains the contents of the message:

    MESSAGE
    subscription:0
    message-id:007
    destination:/queue/a
    content-type:text/plain
    
    hello queue a^@

`MESSAGE` frames should include a 
[`content-length`](#Header_content-length) header and a 
[`content-type`](#Header_content-type) header if a body is present.

`MESSAGE` frames will also include all user defined headers that were present
when the message was sent to the destination in addition to the server specific headers 
that may get added to the frame.  Consult your server's documentation to find out the 
server specific headers that it adds to messages.

### RECEIPT

A `RECEIPT` frame is sent from the server to the client once a server has has
successfully processed a client frame that requests a receipt. A `RECEIPT`
frame will include the header `receipt-id`, where the value is the value of
the `receipt` header in the frame which this is a receipt for.

    RECEIPT
    receipt-id:message-12345

    ^@


The receipt body will be empty.

### ERROR

The server may send `ERROR` frames if something goes wrong. The error frame
should contain a `message` header with a short description of the error, and
the body may contain more detailed information (or may be empty).

    ERROR
    receipt-id:message-12345
    content-type:text/plain
    message: malformed frame received
    
    The message:
    -----
    MESSAGE
    destined:/queue/a
    receipt:message-12345
    
    
    Hello queue a!
    -----
    Did not contain a destination header, which is required 
    for message propagation.
    ^@


If the error is related to specific frame sent from the client, the server
should add additional headers to help identify the original frame that caused
the error. For example, if the frame included a receipt header, the `ERROR`
frame SHOULD set the `receipt-id` header to match the value of the `receipt`
header of the frame which the error is related to.

`ERROR` frames should include a 
[`content-length`](#Header_content-length) head and a
[`content-type`](#Header_content-type) header if a body is present.

## Heart-beating

Heart-beating can optionally be used to test the healthiness of the
underlying TCP connection and to make sure that the remote end is alive and
kicking.

In order to enable heart-beating, each party has to declare what it can do
and what it would like the other party to do. This happens at the very
beginning of the STOMP session, by adding a `heart-beat` header to the
`CONNECT` and `CONNECTED` frames.

When used, the `heart-beat` header MUST contain two positive integers
separated by a comma.

The first number represents what the sender of the frame can do (outgoing
heart-beats): 
 
* 0 means it cannot send heart-beats 

* otherwise it is the smallest number of milliseconds between heart-beats
  that it can guarantee

The second number represents what the sender of the frame would like
to get (incoming heart-beats):

* 0 means it does not want to receive heart-beats

* otherwise it is the desired number of milliseconds between heart-beats

The `heart-beat` header is optional. A missing `heart-beat` header MUST be
treated the same way as a "heart-beat:0,0" header, that is: the party cannot
send and does not want to receive heart-beats.

The `heart-beat` header provides enough information so that each party can
find out if heart-beats can be used, in which direction, and with which
frequency.

More formally, the initial frames look like:

    CONNECT
    heart-beat:<cx>,<cy>

    CONNECTED:
    heart-beat:<sx>,<sy>

For heart-beats from the client to the server:

* if `<cx>` is 0 (the client cannot send heart-beats) or `<sy>` is 0 (the
  server does not want to receive heart-beats) then there will be none

* otherwise, there will be heart-beats every MAX(`<cx>`,`<sy>`) milliseconds

In the other direction, `<sx>` and `<cy>` are used the same way.

Regarding the heart-beats themselves, any new data received over the network
connection is an indication that the remote end is alive. In a given
direction, if heart-beats are expected every `<n>` milliseconds:

* the sender MUST send new data over the network connection at least every
  `<n>` milliseconds

* if the sender has no real STOMP frame to send, it MUST send a single
  newline byte (0x0A)

* if, inside a time window of at least `<n>` milliseconds, the receiver did
  not receive any new data, it CAN consider the connection as dead

* because of timing inaccuracies, the receiver SHOULD be tolerant and take
  into account an error margin


## Augmented BNF

A STOMP session can be more formally described using the 
Backus-Naur Form (BNF) grammar used in HTTP/1.1
[rfc2616](http://www.w3.org/Protocols/rfc2616/rfc2616-sec2.html#sec2.1).

    NL                  = <US-ASCII new line (line feed) (octect 10)>
    OCTET               = <any 8-bit sequence of data>
    NULL                = <octect 0>
    
    frame-stream        = 1*frame
    
    frame               = command NL
                          *( header NL )
                          NL
                          *OCTECT
                          NULL
                          *( NL )
    
    command             = client-command | server-command
    
    client-command      = "SEND"
                          | "SUBSCRIBE"
                          | "UNSUBSCRIBE"
                          | "BEGIN"
                          | "COMMIT"
                          | "ABORT"
                          | "ACK"
                          | "NACK"
                          | "DISCONNECT"
                          | "CONNECT"
                          | "STOMP"
    
    server-command      = "CONNECTED"
                          | "MESSAGE"
                          | "RECEIPT"
                          | "ERROR"
    
    header              = header-name ":" header-value
    header-name         = 1*<any OCTET except NL or ":">
    header-value        = *<any OCTET except NL>
    
## License

This specification is licensed under the 
[Creative Commons Attribution v2.5](http://creativecommons.org/licenses/by/2.5/) 
license.
