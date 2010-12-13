Overview
--------

Stomp is the Simple (or Streaming) Text Orientated Messaging Protocol.

Stomp provides an interoperable wire format so that Stomp clients can
communicate with any Stomp message broker to provide easy and widespread
messaging interoperability among many languages, platforms and brokers.

Specification
-------------

This git repository hosts the released and in progress STOMP specifications.  The
lasted released specification is located at:

[src/stomp-specification-1.0.md](src/stomp-specification-1.0.md)

Website Generation
------------------

This git repository generates the STOMP specification static website using the
[Maven](http://maven.apache.org/download.html) build tool.  Once you have maven
installed, you can generate the website by running:

    mvn package

It will generate the static site to the `target/sitegen` directory, just point 
your web browser at `target/sitegen/index.html`



