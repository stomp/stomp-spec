# Stomp Protocol Specification, Version 1.1

* Table of contents
{:toc}

## DRAFT STATUS

Version 1.1 of the specification is still being developed. This is only
a draft document.

## Stomp Frames

Stomp is designed to work best over a stream based communications transport
like TCP. The client and server will communicate using Stomp frames sent over
the stream. A frame's structure looks like:

    command
    header1:value1
    header2:value2
    
    Body^@

The frame starts with a command string, followed by a newline, followed by
header entries in `<key>`:`<value>` format. Each header entry is followed by
a newline. A blank line indicates the end of the headers and beginning of the
body. The body is then followed by the null byte (0x00). The examples in
document will use `^@`, control-@ in ASCII, to represent the null byte. The
null byte can be optionally be followed by multiple newlines. For more
details, on how to parse Stomp frames, see the [Augmented
BNF](#augmented_bnf) section of this document.

It is important to note that in Stomp 1.1, clients and servers should not trim the 
white space in header values.  Many Stomp 1.0 brokers did trim the white space.  Stomp 
1.0 servers would treat a header line like `destination: /queue/alpha` the same as
`destination:/queue/alpha`.  In Stomp 1.1, clients MUST never pad the header with 
a space.

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

A Stomp client initiates the stream or TCP connection to the server. The
client must then send the `CONNECT` frame.

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
with a `ERROR` frame listing why the connection was rejected and then the sever
will close the connection. Since Stomp servers must support clients which
rapidly connect and disconnect, a server will likely only allow closed
connections to linger for short time before the connection is reset. This means
that a client may not fully receive the `ERROR` frame before the socket is
reset.

### CONNECT Frame

Stomp 1.1 clients MUST set the following headers:

* `accept-version` : The versions of the Stomp protocol the client supports.
  See [Protocol Negotiation](#protocol_negotiation) for more details.

* `host` : The host name that the socket was established against. This allows
  the server to implement virtual hosts.

Stomp 1.1 clients MAY set the following headers

* `login` : The user id used to authenticate against a secured Stomp server.

* `passcode` : The password used to authenticate against a secured Stomp
  server.

#### Future Compatibility    

In future versions of the specification, the `CONNECT` frame will be renamed
to `STOMP`. Stomp 1.1 servers should handle a `STOMP` frame the same way as
the `CONNECT` frame. Stomp 1.1 clients should continue to use the `CONNECT`
command to remain backward compatible with Stomp 1.0 servers.

The reason to frame is being renamed is so that the protocol can more easily
be differentiated from the HTTP protocol by a protocol sniffer/discriminator.

### CONNECTED Frame

Stomp 1.1 servers MUST set the following headers:

* `version` : The version of the Stomp protocol the session will be using.
  See [Protocol Negotiation](#protocol_negotiation) for more details.

Stomp 1.1 servers MAY set the following headers

* `session` : A session id that uniquely identifies the session.  

<!--
TODO: is this the reasoning for the session id? The client can use 
the session id as the base to generate globally unique identifies 
by appending a incrementing counter.
-->

## Protocol Negotiation

From Stomp 1.1 and onwards, the `CONNECT` frame MUST include the
`accept-version` header. It should be set to a comma separated list of
incrementing Stomp protocol versions that the client supports. If the
`accept-version` header is missing, it means that the client only supports
version 1.0 of the protocol.

The protocol that will be used for the reset of the session will be the
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
sever should respond with an `ERROR` frame that looks like:

    ERROR
    version:1.2,2.1
              
    Supported protocol versions are 1.2 2.1^@

## Once Connected

Once the client is connected it may send any of the following frames:

* [SEND](#send)
* [SUBSCRIBE](#subscribe)
* [UNSUBSCRIBE](#unsubscribe)
* [BEGIN](#begin)
* [COMMIT](#commit)
* [ABORT](#abort)
* [ACK](#ack)
* [DISCONNECT](#disconnect)

## Client Frames

### SEND

The `SEND` frame sends a message to a destination in the messaging system. It
has one required header, `destination`, which indicates where to send the
message. The body of the `SEND` frame is the message to be sent. For example:

    SEND
    destination:/queue/a
    
    hello queue a
    ^@

This sends a message to a destination named `/queue/a`. Even though queue and
topic delivery semantics are the most popular in messing servers, Stomp does
not define what the delivery semantics of destinations should be. The
delivery, or "message exchange", semantics of destinations can vary from
server to server and even from destination to destination. This allows
servers to be even more creative with the semantics that they can support
with Stomp. You should consult your Stomp server's documentation to find out
how to construct a destination name which gives you the delivery semantics
that your application needs.

The reliability semantics of the message will also be server specific and 
typically will depend on the destination value being used and other headers
on the message such as the `transaction` header and other server specific headers
on the message.

`SEND` supports a `transaction` header which allows for transaction sends.

It is recommended that `SEND` frames include a `content-length` header which is a
byte count for the length of the message body. If a `content-length` header is
included, this number of bytes should be read, regardless of whether or not
there are null characters in the body. The frame still needs to be terminated
with a null byte and if a `content-length` is not specified, the first null
byte encountered signals the end of the frame.

An application may add any arbitrary user defined headers to the SEND frame.
User defined headers are typically used to allow consumers to filter
messages based on the application defined headers using a selector 
on a SUBSCRIBE frame.  The user defined headers should be passed through
in the MESSAGE frame.

### SUBSCRIBE

The `SUBSCRIBE` frame is used to register to listen to a given destination. Like
the `SEND` frame, the `SUBSCRIBE` frame requires a `destination` header indicating
which destination to subscribe to. Any messages received on the subscription
will henceforth be delivered as `MESSAGE` frames from the server to the client.
The `ack` header is to control the message acknowledgement mode. The valid
values for `ack` are `auto`, `client`, or `client-individual`. If the header
is not set, it defaults to `auto`.

Example:

    SUBSCRIBE
    id:0
    destination:/queue/foo
    ack:client
    
    ^@

The body of the `SUBSCRIBE` frame is ignored.

#### SUBSCRIBE `id` Header

You MUST specify an `id` header to uniquely identify the subscription within
the stomp connection session.  Since a single connection can have multiple open
subscriptions with a broker, the `id` header allows the client and broker to 
relate subsequent `ACK` and `UNSUBSCRIBE` frames to the original subscription.

#### SUBSCRIBE `ack` Header

When the the `ack` mode is `auto`, then the client does not need to send the
server `ACK` frames for the messages it receives. The server will assume the
client has received the message as soon as it sends it to the the client. This
acknowledgment mode cause cause messages being transmitted to the client to
get dropped.

When the the `ack` mode is `client`, then the client must send the server `ACK`
frames for the messages it processes. If the connection fails before a client
sends an `ACK` for the message the server will assume the message has not been
processed and may redeliver the message to another client. The `ACK` frames sent
by the client will be treated as a cumulative `ACK`. This means the `ACK` operates
on the message specified in the `ACK` frame and all messages sent before the
messages to the subscription.

When the the `ack` mode is `client-individual`, the ack mode operates just
like the `client` ack mode except that the ACK frames sent by the client are
not cumulative. This means that an `ACK` for a subsequent message should
not cause a previous message to get acknowledged.

#### SUBSCRIBE `selector` Header

Stomp brokers may support the `selector` header which allows you to specify a
[SQL 92 style WHERE clause](http://download.oracle.com/docs/cd/E17477_01/javaee/1.4/api/javax/jms/Message.html)
which operates against the message headers.  This allows you to apply a filter 
on the server side which can be used to do content based routing.

### UNSUBSCRIBE

The `UNSUBSCRIBE` frame is used to remove an existing subscription.  Once the subscription is
removed the stomp connections will no longer receive messages from that destination. 
It requires the `id` header matches the `id` value of previous `SUBSCRIBE` operation. 
Example:

    UNSUBSCRIBE
    id:0
    
    ^@

### ACK

`ACK` is used to acknowledge consumption of a message from a subscription using
client acknowledgment. When a client has issued a `SUBSCRIBE` frame with the
`ack` header set to `client` or `client-individual`  any messages received from 
that destination will not be considered to have been consumed (by the server) 
until the message has been acknowledged via an `ACK`.

`ACK` has two required header, `message-id`, which must contain a value
matching the `message-id` for the MESSAGE being acknowledged and `subscription`,
which must be set to match the value of the subscription`s `id` header. 
Optionally, a `transaction` header may be specified, indicating that the message
acknowledgment should be part of the named transaction.

    ACK
    subscription:0
    message-id:007
    transaction:tx1

    ^@

### BEGIN

`BEGIN` is used to start a transaction. Transactions in this case apply to
sending and acknowledging - any messages sent or acknowledged during a
transaction will be handled atomically based on the transaction.

    BEGIN
    transaction:tx1

    ^@

The `transaction` header is required, and the transaction identifier will be
used for `SEND`, `COMMIT`, `ABORT`, and `ACK` frames to bind them to the named
transaction.

Any started transactions which have not been committed will be implicitly aborted
if the client sends a `DISCONNECT` frame or if the TCP connection fails for
any reason.

### COMMIT

`COMMIT` is used to commit a transaction in progress.

    COMMIT
    transaction:tx1

    ^@

The `transaction` header is required, you must specify which transaction to
commit\!

### ABORT

`ABORT` is used to roll back a transaction in progress.

    ABORT
    transaction:tx1

    ^@


The `transaction` header is required, you must specify which transaction to
abort\!

### DISCONNECT

`DISCONNECT` does a graceful disconnect from the server. It is quite polite to
use this before closing the socket.

    DISCONNECT

    ^@


## Standard Headers

Some headers may be used, and have special meaning, with most packets

### Receipt

Any client frame other than `CONNECT` may specify a `receipt` header with an
arbitrary value. This will cause the server to acknowledge receipt of the
frame with a `RECEIPT` frame which contains the value of this header as the
value of the `receipt-id` header in the `RECEIPT` packet.

    SEND
    destination:/queue/a
    receipt:message-12345

    hello queue a^@


## Server Frames 

The server will, on occasion, send frames to the client (in additional to the
initial `CONNECTED` frame). These frames may be one of:

* [MESSAGE](#message)
* [RECEIPT](#receipt)
* [ERROR](#error)

### MESSAGE

`MESSAGE` frames are used to convey messages from subscriptions to the
client. The `MESSAGE` frame will include a header, `destination`, indicating
the destination the message was sent to. It will also contain a `message-id`
header with a unique identifier for that message. The `subscription` header
will be set to match the `id` header of the subscription that is receiving
the message. The frame body contains the contents of the message:

    MESSAGE
    subscription:0
    destination:/queue/a
    message-id:007
    
    hello queue a^@

It is recommended that `MESSAGE` frames include a `content-length` header which
is a byte count for the length of the message body. If a `content-length`
header is included, this number of bytes should be read, regardless of
whether or not there are null characters in the body. The frame still needs
to be terminated with a null byte, and if a `content-length` is not specified
the first null byte encountered signals the end of the frame.

### RECEIPT

Receipts are issued from the server when the client has requested a receipt
for a given frame. A `RECEIPT` frame will include the header `receipt-id`,
where the value is the value of the `receipt` header in the frame which this
is a receipt for.

    RECEIPT
    receipt-id:message-12345

    ^@


The receipt body will be empty.

### ERROR

The server may send `ERROR` frames if something goes wrong. The error frame
should contain a `message` header with a short description of the error, and
the body may contain more detailed information (or may be empty).

    ERROR
    message: malformed packet received
    
    The message:
    -----
    MESSAGE
    destined:/queue/a
    
    Hello queue a!
    -----
    Did not contain a destination header, which is required 
    for message propagation.
    ^@


It is recommended that `ERROR` frames include a `content-length` header which
is a byte count for the length of the message body. If a `content-length`
header is included, this number of bytes should be read, regardless of
whether or not there are null characters in the body. The frame still needs
to be terminated with a null byte, and if a `content-length` is not specified
the first null byte encountered signals the end of the frame.

## Heart-beating

Heart-beating can optionally be used to test the healthiness of the
underlying TCP connection and to make sure that the remote end is alive and
kicking.

In order to enable heart-beating, each party has to declare what it can do
and what it would like the other party to do. This happens at the very
beginning of the Stomp session, by adding a `heart-beat` header to the
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
find out if heart-beats can be used, in which direction and with which
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

A Stomp session can be more formally described using the 
Backus-Naur Form (BNF) grammar used in the HTTP/1.1
[rfc2616](http://www.w3.org/Protocols/rfc2616/rfc2616-sec2.html#sec2.1).

    NL                  = <US-ASCII new line (line feed) (octect 10)>
    CHAR                = <any US-ASCII character (octets 0 - 127)>
    OCTET               = <any 8-bit sequence of data>
    DIGIT               = <any US-ASCII digit "0".."9">
    NULL                = <octect 0>
    
    frame-stream        = 1*frame
    
    frame               = command NL
                          *( header NL )
                          NL
                          [ content ]
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
                          | "DISCONNECT"
                          | "STOMP"
    
    server-command      = "CONNECTED"
                          | "MESSAGE"
                          | "RECEIPT"
                          | "ERROR"
    
    header              = header-name ":" header-value
    header-name         = 1*<any CHAR except NL or ":">
    header-value        = 1*<any CHAR except NL>
    
    content             = text-content | binary-content
    text-content        = 1*<any OCTET except NULL>
    binary-content      = 1*OCTECT

## License

This specification is licensed under the 
[Creative Commons Attribution v2.5](http://creativecommons.org/licenses/by/2.5/) 
license.



