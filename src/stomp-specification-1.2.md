# STOMP Protocol Specification, Version 1.2

{:toc:2-5}

## Abstract

STOMP is a simple interoperable protocol designed for asynchronous message
passing between clients via mediating servers. It defines a text based
wire-format for messages passed between these clients and servers.

STOMP has been in active use for several years and is supported by many
message brokers and client libraries. This specification defines the STOMP 1.2
protocol and is an update to [STOMP 1.1](stomp-specification-1.1.html).

Please send feedback to the stomp-spec@googlegroups.com mailing list.

## Overview

### Background

STOMP arose from a need to connect to enterprise message brokers from
scripting languages such as Ruby, Python and Perl. In such an
environment it is typically logically simple operations that are
carried out such as 'reliably send a single message and disconnect'
or 'consume all messages on a given destination'.

It is an alternative to other open messaging protocols such as AMQP
and implementation specific wire protocols used in JMS brokers such
as OpenWire. It distinguishes itself by covering a small subset of
commonly used messaging operations rather than providing a
comprehensive messaging API.

More recently STOMP has matured into a protocol which can be used past
these simple use cases in terms of the wire-level features it now
offers, but still maintains its core design principles of simplicity
and interoperability.

### Protocol Overview

STOMP is a frame based protocol, with frames modelled on HTTP. A frame
consists of a command, a set of optional headers and an optional body. STOMP
is text based but also allows for the transmission of binary messages. The
default encoding for STOMP is UTF-8, but it supports the specification of
alternative encodings for message bodies.

A STOMP server is modelled as a set of destinations to which messages can be
sent. The STOMP protocol treats destinations as opaque string and their syntax
is server implementation specific. Additionally STOMP does not define what the
delivery semantics of destinations should be. The delivery, or "message
exchange", semantics of destinations can vary from server to server and even
from destination to destination. This allows servers to be creative with the
semantics that they can support with STOMP.

A STOMP client is a user-agent which can act in two (possibly simultaneous)
modes:

* as a producer, sending messages to a destination on the server via a `SEND`
  frame

* as a consumer, sending a `SUBSCRIBE` frame for a given destination and
  receiving messages from the server as `MESSAGE` frames.

### Changes in the Protocol

STOMP 1.2 is mostly backwards compatible with STOMP 1.1. There are only two
incompatible changes:

* it is now possible to end frame lines with carriage return plus line feed
  instead of only line feed

* message acknowledgment has been simplified and now uses a dedicated header

Apart from these, STOMP 1.2 introduces no new features but focuses on clarifying
some areas of the specification such as:

* repeated frame header entries

* use of the `content-length` and `content-type` headers

* required support of the `STOMP` frame by servers

* connection lingering

* scope and uniqueness of subscription and transaction identifiers

* meaning of the `RECEIPT` frame with regard to previous frames

### Design Philosophy

The main philosophies driving the design of STOMP are simplicity and
interoperability.

STOMP is designed to be a lightweight protocol that is easy to implement both
on the client and server side in a wide range of languages. This implies, in
particular, that there are not many constraints on the architecture of servers
and many features such as destination naming and reliability semantics are
implementation specific.

In this specification we will note features of servers which are not
explicitly defined by STOMP 1.2. You should consult your STOMP server's
documentation for the implementation specific details of these features.

## Conformance

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](http://tools.ietf.org/html/rfc2119).

Implementations may impose implementation-specific limits on unconstrained
inputs, e.g. to prevent denial of service attacks, to guard against running
out of memory, or to work around platform-specific limitations.

The conformance classes defined by this specification are STOMP clients and
STOMP servers.

## STOMP Frames

STOMP is a frame based protocol which assumes a reliable 2-way streaming
network protocol (such as TCP) underneath. The client and server will
communicate using STOMP frames sent over the stream. A frame's structure
looks like:

    COMMAND
    header1:value1
    header2:value2

    Body^@

The frame starts with a command string terminated by an end-of-line (EOL),
which consists of an OPTIONAL carriage return (octet 13) followed by a
REQUIRED line feed (octet 10). Following the command are zero or more header
entries in `<key>:<value>` format. Each header entry is terminated by an EOL.
A blank line (i.e. an extra EOL) indicates the end of the headers and the
beginning of the body. The body is then followed by the NULL octet. The
examples in this document will use `^@`, control-@ in ASCII, to represent
the NULL octet. The NULL octet can be optionally followed by multiple EOLs.
For more details, on how to parse STOMP frames, see the
[Augmented BNF](#Augmented_BNF) section of this document.

All commands and header names referenced in this document are case sensitive.

### Value Encoding

The commands and headers are encoded in UTF-8. All frames except the `CONNECT`
and `CONNECTED` frames will also escape any carriage return, line feed or colon
found in the resulting UTF-8 encoded headers.

Escaping is needed to allow header keys and values to contain those frame
header delimiting octets as values. The `CONNECT` and `CONNECTED` frames do not
escape the carriage return, line feed or colon octets in order to remain backward
compatible with STOMP 1.0.

C style string literal escapes are used to encode any carriage return, line feed
or colon that are found within the UTF-8 encoded headers. When decoding frame headers,
the following transformations MUST be applied:

* `\r` (octet 92 and 114) translates to carriage return (octet 13)
* `\n` (octet 92 and 110) translates to line feed (octet 10)
* `\c` (octet 92 and 99) translates to `:` (octet 58)
* `\\` (octet 92 and 92) translates to `\` (octet 92)

Undefined escape sequences such as `\t` (octet 92 and 116) MUST be treated as
a fatal protocol error. Conversely when encoding frame headers, the reverse
transformation MUST be applied.

The STOMP 1.0 specification included many example frames with padding in the
headers and many servers and clients were implemented to trim or pad header
values. This causes problems if applications want to send headers that SHOULD
not get trimmed. In STOMP 1.2, clients and servers MUST never trim or pad
headers with spaces.

### Body

Only the `SEND`, `MESSAGE`, and `ERROR` frames MAY have a body. All other
frames MUST NOT have a body.

### Standard Headers

Some headers MAY be used, and have special meaning, with most frames.

#### Header content-length

All frames MAY include a `content-length` header. This header is an octet
count for the length of the message body. If a `content-length` header is
included, this number of octets MUST be read, regardless of whether or not
there are NULL octets in the body. The frame still needs to be terminated
with a NULL octet.

If a frame body is present, the `SEND`, `MESSAGE` and `ERROR` frames SHOULD
include a `content-length` header to ease frame parsing. If the frame body
contains NULL octets, the frame MUST include a `content-length` header.

#### Header content-type

If a frame body is present, the `SEND`, `MESSAGE` and `ERROR` frames SHOULD
include a `content-type` header to help the receiver of the frame interpret
its body. If the `content-type` header is set, its value MUST be a MIME type
which describes the format of the body. Otherwise, the receiver SHOULD
consider the body to be a binary blob.

The implied text encoding for MIME types starting with `text/` is UTF-8. If
you are using a text based MIME type with a different encoding then you
SHOULD append `;charset=<encoding>` to the MIME type. For example,
`text/html;charset=utf-16` SHOULD be used if you're sending an HTML body in
UTF-16 encoding. The `;charset=<encoding>` SHOULD also get appended to any
non `text/` MIME types which can be interpreted as text. A good example of
this would be a UTF-8 encoded XML. Its `content-type` SHOULD get set to
`application/xml;charset=utf-8`

All STOMP clients and servers MUST support UTF-8 encoding and decoding. Therefore,
for maximum interoperability in a heterogeneous computing environment, it is
RECOMMENDED that text based content be encoded with UTF-8.

#### Header receipt

Any client frame other than `CONNECT` MAY specify a `receipt` header with an
arbitrary value. This will cause the server to acknowledge the processing of
the client frame with a `RECEIPT` frame (see the [RECEIPT](#RECEIPT) frame
for more details).

    SEND
    destination:/queue/a
    receipt:message-12345

    hello queue a^@

### Repeated Header Entries

Since messaging systems can be organized in store and forward topologies,
similar to SMTP, a message may traverse several messaging servers before
reaching a consumer. A STOMP server MAY 'update' header values by either
prepending headers to the message or modifying a header in-place in the
message.

If a client or a server receives repeated frame header entries, only the
first header entry SHOULD be used as the value of header entry. Subsequent
values are only used to maintain a history of state changes of the header
and MAY be ignored.

For example, if the client receives:

    MESSAGE
    foo:World
    foo:Hello

    ^@

The value of the `foo` header is just `World`.

### Size Limits

To prevent malicious clients from exploiting memory allocation in a
server, servers MAY place maximum limits on:

* the number of frame headers allowed in a single frame
* the maximum length of header lines
* the maximum size of a frame body

If these limits are exceeded the server SHOULD send the client an `ERROR`
frame and then close the connection.

### Connection Lingering

STOMP servers must be able to support clients which rapidly connect and
disconnect.

This implies a server will likely only allow closed connections to linger
for short time before the connection is reset.

As a consequence, a client may not receive the last frame sent by the server
(for instance an `ERROR` frame or the `RECEIPT` frame in reply to a
`DISCONNECT` frame) before the socket is reset.

## Connecting

A STOMP client initiates the stream or TCP connection to the server by sending
the `CONNECT` frame:

    CONNECT
    accept-version:1.2
    host:stomp.github.org

    ^@

If the server accepts the connection attempt it will respond with a
`CONNECTED` frame:

    CONNECTED
    version:1.2

    ^@

The server can reject any connection attempt. The server SHOULD respond back
with an `ERROR` frame explaining why the connection was rejected and then close
the connection.

### CONNECT or STOMP Frame

STOMP servers MUST handle a `STOMP` frame in the same manner as a `CONNECT`
frame. STOMP 1.2 clients SHOULD continue to use the `CONNECT` command to
remain backward compatible with STOMP 1.0 servers.

Clients that use the `STOMP` frame instead of the `CONNECT` frame will only
be able to connect to STOMP 1.2 servers (as well as some STOMP 1.1 servers)
but the advantage is that a protocol sniffer/discriminator will be able to
differentiate the STOMP connection from an HTTP connection.

STOMP 1.2 clients MUST set the following headers:

* `accept-version` : The versions of the STOMP protocol the client supports.
  See [Protocol Negotiation](#Protocol_Negotiation) for more details.

* `host` : The name of a virtual host that the client wishes to connect to.
  It is recommended clients set this to the host name that the socket
  was established against, or to any name of their choosing. If this
  header does not match a known virtual host, servers supporting virtual
  hosting MAY select a default virtual host or reject the connection.

STOMP 1.2 clients MAY set the following headers:

* `login` : The user identifier used to authenticate against a secured STOMP server.

* `passcode` : The password used to authenticate against a secured STOMP
  server.

* `heart-beat` : The [Heart-beating](#Heart-beating) settings.

### CONNECTED Frame

STOMP 1.2 servers MUST set the following headers:

* `version` : The version of the STOMP protocol the session will be using.
  See [Protocol Negotiation](#Protocol_Negotiation) for more details.

STOMP 1.2 servers MAY set the following headers:

* `heart-beat` : The [Heart-beating](#Heart-beating) settings.

* `session` : A session identifier that uniquely identifies the session.

* `server` : A field that contains information about the STOMP server.
  The field MUST contain a server-name field and MAY be followed by optional 
  comment fields delimited by a space character.

  The server-name field consists of a name token followed by an optional version
  number token.

    `server      = name ["/" version] *(comment)`

  Example:

    `server:Apache/1.3.9`

### Protocol Negotiation

From STOMP 1.1 and onwards, the `CONNECT` frame MUST include the
`accept-version` header. It SHOULD be set to a comma separated list of
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
sever MUST respond with an `ERROR` frame similar to the following and then
close the connection:

    ERROR
    version:1.2,2.1
    content-type:text/plain

    Supported protocol versions are 1.2 2.1^@

### Heart-beating

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

The `heart-beat` header is OPTIONAL. A missing `heart-beat` header MUST be
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

* if the sender has no real STOMP frame to send, it MUST send an end-of-line (EOL)

* if, inside a time window of at least `<n>` milliseconds, the receiver did
  not receive any new data, it MAY consider the connection as dead

* because of timing inaccuracies, the receiver SHOULD be tolerant and take
  into account an error margin

## Client Frames

A client MAY send a frame not in this list, but for such a frame a STOMP 1.2
server MAY respond with an `ERROR` frame and then close the connection.

* [`SEND`](#SEND)
* [`SUBSCRIBE`](#SUBSCRIBE)
* [`UNSUBSCRIBE`](#UNSUBSCRIBE)
* [`BEGIN`](#BEGIN)
* [`COMMIT`](#COMMIT)
* [`ABORT`](#ABORT)
* [`ACK`](#ACK)
* [`NACK`](#NACK)
* [`DISCONNECT`](#DISCONNECT)

### SEND

The `SEND` frame sends a message to a destination in the messaging system. It
has one REQUIRED header, `destination`, which indicates where to send the
message. The body of the `SEND` frame is the message to be sent. For example:

    SEND
    destination:/queue/a
    content-type:text/plain

    hello queue a
    ^@

This sends a message to a destination named `/queue/a`. Note that STOMP treats
this destination as an opaque string and no delivery semantics are assumed by
the name of a destination. You should consult your STOMP server's
documentation to find out how to construct a destination name which gives you
the delivery semantics that your application needs.

The reliability semantics of the message are also server specific and will
depend on the destination value being used and the other message headers
such as the `transaction` header or other server specific message headers.

`SEND` supports a `transaction` header which allows for transactional sends.

`SEND` frames SHOULD include a
[`content-length`](#Header_content-length) header and a
[`content-type`](#Header_content-type) header if a body is present.

An application MAY add any arbitrary user defined headers to the `SEND` frame.
User defined headers are typically used to allow consumers to filter
messages based on the application defined headers using a selector
on a `SUBSCRIBE` frame. The user defined headers MUST be passed through
in the `MESSAGE` frame.

If the sever cannot successfully process the `SEND` frame for any reason,
the server MUST send the client an `ERROR` frame and then close the connection.

### SUBSCRIBE

The `SUBSCRIBE` frame is used to register to listen to a given destination.
Like the `SEND` frame, the `SUBSCRIBE` frame requires a `destination` header
indicating the destination to which the client wants to subscribe. Any
messages received on the subscribed destination will henceforth be delivered
as `MESSAGE` frames from the server to the client. The `ack` header controls
the message acknowledgment mode.

Example:

    SUBSCRIBE
    id:0
    destination:/queue/foo
    ack:client

    ^@

If the sever cannot successfully create the subscription,
the server MUST send the client an `ERROR` frame and then close the connection.

STOMP servers MAY support additional server specific headers to customize the
delivery semantics of the subscription. Consult your server's documentation for
details.

#### SUBSCRIBE id Header

Since a single connection can have multiple open subscriptions with a
server, an `id` header MUST be included in the frame to uniquely identify
the subscription. The `id` header allows the client and server to relate
subsequent `MESSAGE` or `UNSUBSCRIBE` frames to the original subscription.

Within the same connection, different subscriptions MUST use different
subscription identifiers.

#### SUBSCRIBE ack Header

The valid values for the `ack` header are `auto`, `client`, or
`client-individual`. If the header is not set, it defaults to `auto`.

When the `ack` mode is `auto`, then the client does not need to send the
server `ACK` frames for the messages it receives. The server will assume the
client has received the message as soon as it sends it to the client.
This acknowledgment mode can cause messages being transmitted to the client
to get dropped.

When the `ack` mode is `client`, then the client MUST send the server
`ACK` frames for the messages it processes. If the connection fails before a
client sends an `ACK` frame for the message the server will assume the message
has not been processed and MAY redeliver the message to another client. The
`ACK` frames sent by the client will be treated as a cumulative acknowledgment.
This means the acknowledgment operates on the message specified in the `ACK`
frame and all messages sent to the subscription before the `ACK`'ed message.

In case the client did not process some messages, it SHOULD send `NACK` frames
to tell the server it did not consume these messages.

When the `ack` mode is `client-individual`, the acknowledgment operates just
like the `client` acknowledgment mode except that the `ACK` or `NACK` frames
sent by the client are not cumulative. This means that an `ACK` or `NACK`
frame for a subsequent message MUST NOT cause a previous message to get
acknowledged.

### UNSUBSCRIBE

The `UNSUBSCRIBE` frame is used to remove an existing subscription. Once the
subscription is removed the STOMP connections will no longer receive messages
from that subscription.

Since a single connection can have multiple open subscriptions with a
server, an `id` header MUST be included in the frame to uniquely identify
the subscription to remove. This header MUST match the subscription
identifier of an existing subscription.

Example:

    UNSUBSCRIBE
    id:0

    ^@

### ACK

`ACK` is used to acknowledge consumption of a message from a subscription
using `client` or `client-individual` acknowledgment. Any messages received
from such a subscription will not be considered to have been consumed until
the message has been acknowledged via an `ACK`.

The `ACK` frame MUST include an `id` header matching the `ack` header of the
`MESSAGE` being acknowledged. Optionally, a `transaction` header MAY be
specified, indicating that the message acknowledgment SHOULD be part of the
named transaction.

    ACK
    id:12345
    transaction:tx1

    ^@

### NACK

`NACK` is the opposite of `ACK`. It is used to tell the server that the
client did not consume the message. The server can then either send the
message to a different client, discard it, or put it in a dead letter queue.
The exact behavior is server specific.

`NACK` takes the same headers as `ACK`: `id` (REQUIRED) and `transaction`
(OPTIONAL).

`NACK` applies either to one single message (if the subscription's `ack`
mode is `client-individual`) or to all messages sent before and not yet
`ACK`'ed or `NACK`'ed (if the subscription's `ack` mode is `client`).

### BEGIN

`BEGIN` is used to start a transaction. Transactions in this case apply to
sending and acknowledging - any messages sent or acknowledged during a
transaction will be processed atomically based on the transaction.

    BEGIN
    transaction:tx1

    ^@

The `transaction` header is REQUIRED, and the transaction identifier will be
used for `SEND`, `COMMIT`, `ABORT`, `ACK`, and `NACK` frames to bind them to
the named transaction. Within the same connection, different transactions MUST
use different transaction identifiers.

Any started transactions which have not been committed will be implicitly
aborted if the client sends a `DISCONNECT` frame or if the TCP connection
fails for any reason.

### COMMIT

`COMMIT` is used to commit a transaction in progress.

    COMMIT
    transaction:tx1

    ^@

The `transaction` header is REQUIRED and MUST specify the identifier of the
transaction to commit.

### ABORT

`ABORT` is used to roll back a transaction in progress.

    ABORT
    transaction:tx1

    ^@

The `transaction` header is REQUIRED and MUST specify the identifier of the
transaction to abort.

### DISCONNECT

A client can disconnect from the server at anytime by closing the socket but
there is no guarantee that the previously sent frames have been received by
the server. To do a graceful shutdown, where the client is assured that all
previous frames have been received by the server, the client SHOULD:

1. send a `DISCONNECT` frame with a `receipt` header set. Example:

        DISCONNECT
        receipt:77
        ^@

2. wait for the `RECEIPT` frame response to the `DISCONNECT`. Example:

        RECEIPT
        receipt-id:77
        ^@

3. close the socket.

Note that, if the server closes its end of the socket too quickly, the
client might never receive the expected `RECEIPT` frame. See the
[Connection Lingering](#Connection_Lingering) section for more information.

Clients MUST NOT send any more frames after the `DISCONNECT` frame is sent.

## Server Frames

The server will, on occasion, send frames to the client (in addition to the
initial `CONNECTED` frame). These frames MAY be one of:

* [`MESSAGE`](#MESSAGE)
* [`RECEIPT`](#RECEIPT)
* [`ERROR`](#ERROR)

### MESSAGE

`MESSAGE` frames are used to convey messages from subscriptions to the client.

The `MESSAGE` frame MUST include a `destination` header indicating the
destination the message was sent to. If the message has been sent using
STOMP, this `destination` header SHOULD be identical to the one used in the
corresponding `SEND` frame.

The `MESSAGE` frame MUST also contain a `message-id` header with a unique
identifier for that message and a `subscription` header matching the
identifier of the subscription that is receiving the message.

If the message is received from a subscription that requires explicit
acknowledgment (either `client` or `client-individual` mode) then the
`MESSAGE` frame MUST also contain an `ack` header with an arbitrary
value. This header will be used to relate the message to a subsequent
`ACK` or `NACK` frame.

The frame body contains the contents of the message:

    MESSAGE
    subscription:0
    message-id:007
    destination:/queue/a
    content-type:text/plain

    hello queue a^@

`MESSAGE` frames SHOULD include a
[`content-length`](#Header_content-length) header and a
[`content-type`](#Header_content-type) header if a body is present.

`MESSAGE` frames will also include all user defined headers that were present
when the message was sent to the destination in addition to the server
specific headers that MAY get added to the frame. Consult your server's
documentation to find out the server specific headers that it adds to
messages.

### RECEIPT

A `RECEIPT` frame is sent from the server to the client once a server has
successfully processed a client frame that requests a receipt. A `RECEIPT`
frame MUST include the header `receipt-id`, where the value is the value of
the `receipt` header in the frame which this is a receipt for.

    RECEIPT
    receipt-id:message-12345

    ^@

A `RECEIPT` frame is an acknowledgment that the corresponding client frame
has been _processed_ by the server. Since STOMP is stream based, the receipt
is also a cumulative acknowledgment that all the previous frames have been
_received_ by the server. However, these previous frames may not yet be
fully _processed_. If the client disconnects, previously received frames
SHOULD continue to get processed by the server.

### ERROR

The server MAY send `ERROR` frames if something goes wrong. In this case, it
MUST then close the connection just after sending the `ERROR` frame. See the
next section about [connection lingering](#Connection_Lingering).

The `ERROR` frame SHOULD contain a `message` header with a short description
of the error, and the body MAY contain more detailed information (or MAY be
empty).

    ERROR
    receipt-id:message-12345
    content-type:text/plain
    content-length:171
    message: malformed frame received

    The message:
    -----
    MESSAGE
    destined:/queue/a
    receipt:message-12345

    Hello queue a!
    -----
    Did not contain a destination header, which is REQUIRED
    for message propagation.
    ^@

If the error is related to a specific frame sent from the client, the server
SHOULD add additional headers to help identify the original frame that caused
the error. For example, if the frame included a receipt header, the `ERROR`
frame SHOULD set the `receipt-id` header to match the value of the `receipt`
header of the frame which the error is related to.

`ERROR` frames SHOULD include a
[`content-length`](#Header_content-length) header and a
[`content-type`](#Header_content-type) header if a body is present.

## Frames and Headers

In addition to the [standard headers](#Standard_Headers) described above
(`content-length`, `content-type` and `receipt`), here are all the headers
defined in this specification that each frame MUST or MAY use:

* `CONNECT` or `STOMP`
    * REQUIRED: `accept-version`, `host`
    * OPTIONAL: `login`, `passcode`, `heart-beat`
* `CONNECTED`
    * REQUIRED: `version`
    * OPTIONAL: `session`, `server`, `heart-beat`
* `SEND`
    * REQUIRED: `destination`
    * OPTIONAL: `transaction`
* `SUBSCRIBE`
    * REQUIRED: `destination`, `id`
    * OPTIONAL: `ack`
* `UNSUBSCRIBE`
    * REQUIRED: `id`
    * OPTIONAL: none
* `ACK` or `NACK`
    * REQUIRED: `id`
    * OPTIONAL: `transaction`
* `BEGIN` or `COMMIT` or `ABORT`
    * REQUIRED: `transaction`
    * OPTIONAL: none
* `DISCONNECT`
    * REQUIRED: none
    * OPTIONAL: `receipt`
* `MESSAGE`
    * REQUIRED: `destination`, `message-id`, `subscription`
    * OPTIONAL: `ack`
* `RECEIPT`
    * REQUIRED: `receipt-id`
    * OPTIONAL: none
* `ERROR`
    * REQUIRED: none
    * OPTIONAL: `message`

In addition, the `SEND` and `MESSAGE` frames MAY include arbitrary user
defined headers that SHOULD be considered as being part of the carried
message. Also, the `ERROR` frame SHOULD include additional headers to help
identify the original frame that caused the error.

Finally, STOMP servers MAY use additional headers to give access to features
like persistency or expiration. Consult your server's documentation for
details.

## Augmented BNF

A STOMP session can be more formally described using the
Backus-Naur Form (BNF) grammar used in HTTP/1.1
[RFC 2616](http://tools.ietf.org/html/rfc2616#section-2.1).

    NULL                = <US-ASCII null (octet 0)>
    LF                  = <US-ASCII line feed (aka newline) (octet 10)>
    CR                  = <US-ASCII carriage return (octet 13)>
    EOL                 = [CR] LF 
    OCTET               = <any 8-bit sequence of data>

    frame-stream        = 1*frame

    frame               = command EOL
                          *( header EOL )
                          EOL
                          *OCTET
                          NULL
                          *( EOL )

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
    header-name         = 1*<any OCTET except CR or LF or ":">
    header-value        = *<any OCTET except CR or LF or ":">

## License

This specification is licensed under the
[Creative Commons Attribution v3.0](http://creativecommons.org/licenses/by/3.0/)
license.
