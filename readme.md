## Overview

STOMP is the Simple (or Streaming) Text Orientated Messaging Protocol.

STOMP provides an interoperable wire format so that STOMP clients can
communicate with any STOMP message broker to provide easy and widespread
messaging interoperability among many languages, platforms and brokers.

## Specification

This git repository hosts the released and in progress STOMP specifications.
The latest released specification is located at:

[src/stomp-specification-1.2.md](src/stomp-specification-1.2.md)

## Website Generation

This git repository generates the
[STOMP specification static website](http://stomp.github.com/)
using either [Maven](http://maven.apache.org/download.html) or the
[SBT](http://code.google.com/p/simple-build-tool/wiki/Setup) build tool.

### Building with Maven

Once you have [Maven](http://maven.apache.org/download.html) installed,
you can generate the website by running:

    mvn package

It will generate the static site to the `target/sitegen` directory, just
point your web browser at `target/sitegen/index.html`

### Building with SBT

Once you have
[SBT](http://code.google.com/p/simple-build-tool/wiki/Setup) installed,
you can generate the website by running:

    sbt update
    sbt package

It will generate the static site to the `target/scala_2.8.1/sitegen` directory, just
point your web browser at `target/scala_2.8.1/sitegen/index.html`

