Surveyed different flow-control API styles and their tradeoffs. 

Originally posted for JSR-369, which adopted the Flow (RS) API for Servlet 4.0.

# Reads

## 1. Push delivery + explicit flow-control signals

API pattern:  

* `callback#onMessage(msgs)`    // messages are delivered to applications
* `stream#startFlowControl()`   // signal to the other end to stop writing 
* `stream#stopFlowControl()`    // signal to the other end to resume writing

Examples: Node streams, WebSocket

## 2. Push delivery + poll based flow-control

API pattern:  

* `stream#readMore()`
* `callback#onMessage(msgs)`    // only to be invoked following a readMore()

Examples: window-based (credit, token) flow control, RS

Notes:
* assuming `onMessage()` is completely non-blocking
* `readMore()` may specify maximum data to read
* `readMore()` may be implicitly called when `onMessage()` returns

## 3. Pull delivery (epoll style) 

API pattern:  

* `stream#notifyWhenReadable(callback)`
* `stream#read(...)`    // may specify the size to read

Examples: Java/Servlet async I/O, .... 

# Writes 

## 1. 

* `stream#write(msgs)`
* `callback#onFlowControl(state)`    // OK to write more or not 

## 2.  

* `stream#writeMore(callback)`     //  intend to write, may also specify the size to write
* `stream#write(msgs)`    // only to be invoked from the callback

## 3.  

* `stream#notifyWhenWritable(callback)`
* `stream#write(msgs)`    


# Trade-offs (reads)

## 1. 

* no transport coupling
* timed delivery is guaranteed as flow control is out of the critical path
* timed flow-control is not guaranteed, as decided by transport and run-time implementation details

## 2. 

* it’s possible to guarantee both timed delivery and flow control with explicit `readMore()` controls
* when the underlying transport is TCP (e.g. HTTP/1.1), explicit `readMore()` may not be supported
* complicated

## 3.

* default behavior will protect the app in a very straightforward way, esp. by leveraging the buffer owned by the transport
* default behavior hurts throughput, esp. if the link is slow
* sync API often shares the same properties
