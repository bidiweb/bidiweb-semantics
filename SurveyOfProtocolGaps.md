# Non Request-Response Communication over the Web, and What's Missing

- Authors: Takeshi Yoshino <tyoshino@google.com>, Wenbo Zhu <wenboz@google.com>
- First version published: Jan/16/2014

## Overview

This document provides a short survey on semantics typically expected by applications when doing client-server communication over the Web that does not follow the basic HTTP request-response pattern. Gaps of existing Web protocols, or their Web application APIs ([XHR] [WSAPI]), are identified with respect to those semantics. We hope that this survey provides useful inputs as new protocols or APIs are being proposed for the Web. 

We are only concerned about communication initiated explicitly by the client, e.g. from a browser, to a remote server over standard Web protocols, namely HTTP/* [HTTP] [HTTP2] and WebSocket [WS].

P2P communication (e.g. WebRTC) or any server-initiated communication (e.g. spontaneous server notification) are out of scope, although they do share many common properties.

## Introduction

In light of past and present discussion on how to extend HTTP/2 to support WebSocket (API or protocol), we find it useful to take a step back and look at the underlying problem first, i.e. the set of common semantics that applications may expect from their hosting environment for doing non-request-response communication. 

To the authors’ knowledge, use cases for non-request-response communication include:

1. Stateful, connection-oriented communication (i.e. sockets) between the client and server
1. Tunneling of TCP based L7 protocols (e.g. over HTTP or WebSocket)

The document is structured as following:

1. Survey of application-visible semantics
1. Gaps in HTTP and WebSocket, and their Web application APIs
1. The layering concern

To keep the doc short, we will not discuss semantics that also apply to the basic request-response communication, such as cookie (client-state), session stickiness and caching.  Certain advanced semantics, such as failure recovery and multi-homing, are skipped too.

Wire-level details, such as framing, compression, protocol negotiation, security-related functions are also skipped. E.g. for HTTP we are only concerned about its semantics that is visible to the application as opposed to the wire-level framing over TCP, SPDY or other “wire-level protocols”. 

## Core Semantics (Informal)

1. Concept of a stateful session
    - Being able to open/accept and close a stateful communication session by the client or server, conceptually at the process level.
1. Initial handshake
    - This allows the client to specify any handshake headers with the URL to open a new session, and then allows the server to inspect the URL and headers to accept or reject the newly initiated session.
1. Message boundary
    - Client or server application sends and receives data as atomic messages, e.g. JSON messages.
    - In cases where message boundary is insignificant, e.g. when a client or server application is producing or consuming a contiguous stream of bytes, then randomly generated “byte-chunks” may be produced or consumed by the application.  (Not to be confused with framing-level chunks such as chunked Transfer-Encoding in HTTP).
    - Batching is an optimization technique, which does not extend the basic message-boundary semantics.
1. Message-level metadata
    - In addition to session-level headers, per-message metadata allows the application to attach optional metadata to individual messages.
1. Ordered and reliable data delivery / Send anytime
    - Messages (or byte-chunks) are delivered in order (FIFO) and reliably over an open session, as data is being produced by the client or server application.
    - Once a session is opened, the client is free to send any amount of data or messages to the server spontaneously. For the purpose of this document, we call this client-streaming.
    - Likewise, server-streaming allows the server to send any amount of data to the client independently of client-streaming.
    - Unordered or unreliable data delivery is out of the scope of this discussion. 
1. Full-duplex and half-close
    - Full-duplex allows simultaneous client-streaming and server-streaming.
    - The concept of half-close allows the client to indicate to the server the completion of client-streaming, and vise versa.  
1. Abort signaling
    - A session may be aborted due to runtime failures, network failures, or peer failures of any sort, by either the client or the server. Explicit peer-initiated aborts may be distinguishable if supported by the underlying wire-level protocol. 
1. End-to-end flow-control
    - A way for the client or server application to indicate flow-control signals to the other end, e.g. to protect the application from being overloaded.
    - Transport-level flow-control may not be sufficient in the presence of proxies. E.g. a client may want to slow down the server during a download even when there is no buffer overflow at the network or proxy layer. 

## Networking Semantics

1. Keep-alive and failure detection
    - Some form of keep-alive is required to notify the disconnect of a peer due to network or machine failures. Without keep-alive support, idle sessions may start to consume too many resources on the server or client.
1. Non-buffering proxy
    - For non-request-response communication, buffering is often undesired for either messages or their byte chunks.  
1. Multiplexing and session priority
    - Multiplexing allows multiple sessions to be opened over a single TCP connection. Session priority is wire-level specific too, e.g. with multiplexing.  
1. Compression
    - Per-message compression or per-chunk compression as part of the wire-level protocol. 

## HTTP Gaps

The following semantics are missing from the standard HTTP protocol semantics (over TCP,  SPDY or other wire-level protocols):

1. Message boundary
1. Message-level metadata
1. Keep-alive
1. Non-buffering proxy

In addition, XMLHttpRequest Web application API [XHR] does not support any form of streaming (formally).

The new Fetch API [FETCH] + Streams API [STREAMS] will expose to the application the full HTTP semantics, including full-duplex communication [FULLDUPLEX]. In addition, the new API also supports message boundary and optionally message-level metadata with application supplied Transformer [STREAMS].  So this leaves only #3 and #4 to be desired. #3 can always be supported at the application level. With SPDY and the wide adoption of HTTPS, #4 is becoming less of a concern for many environments. 

We should note that semantics offered by the API layer won’t be visible to proxies. E.g. a proxy won’t be able to enable any message-level throttling without knowing the message boundary at the wire-level. 

## WebSocket Gaps

The following semantics are missing from the WS protocol or its Web application API:

1. Half-close
1. Flow-control
1. Message-level metadata
1. Keep-alive
1. Multiplexing and session priority.

WS over HTTP/2 will address #5, and then additional API changes are required to support #1 and #2. However, with HTTP/2, proxy buffering may become an issue again.

## The Layering Concern

For any missing semantics, it is always possible for such semantics to be provided above the protocol layer:

1. API offered as part of the client or server platform
1. Client or server libraries offered on top of the API
1. The application itself

This document will not discuss the benefits, or lack thereof, of offering any missing semantics above the protocol layer.

However, we like to note that the so-called end-to-end argument applies here too. Unless there is a significant gain in efficiency or simplicity, individual semantics likely should be supported either at the protocol layer or at the application layer. In other words, it is often desired to match the API with the exact protocol-level semantics.

## Acknowledgements

Thanks to Roberto Peon, Ilya Grigorik, Adam Rice, Yutaka Hirano, Domenic Denicola for their inputs on related work. This doesn't mean that they endorsed opinions in this survey.

## References

- [HTTP] https://www.ietf.org/rfc/rfc2616.txt
- [HTTP2] https://http2.github.io/http2-spec
- [WS] https://tools.ietf.org/html/rfc6455
- [XHR] https://xhr.spec.whatwg.org/
- [WSAPI] https://html.spec.whatwg.org/multipage/comms.html#network
- [FETCH] https://fetch.spec.whatwg.org/
- [STREAMS] https://streams.spec.whatwg.org/
- [FULLDUPLEX]	https://tools.ietf.org/html/draft-zhu-http-fullduplex
