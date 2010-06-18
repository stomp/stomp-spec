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

This git repository can also generate the STOMP specification static website using the 
[Webgen](http://webgen.rubyforge.org/) tool.  If you have Ruby and Ruby Gems installed,
you can install Webgen by running: `sudo gem install webgen haml`

Then generate the site by running `webgen` in this directory.  It will generate the 
static site to the `out` directory, just point your web browser at `out/index.html`



