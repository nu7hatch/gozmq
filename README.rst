Go Bindings for 0mq (zeromq)
############################
This package implements `Go <http://golang.org>`_ bindings for the `0mq
<http://zeromq.org>`_ C API.

Note that this is *not* the same as `this implementation
<http://github.com/boggle/gozero>`_.

Installing
==========
Install gozmq with::

  goinstall github.com/alecthomas/gozmq

If that doesn't work you might need to checkout the source and play with the
CGO_LDFLAGS and CGO_CFLAGS in the Makefile::

  git clone git://github.com/alecthomas/gozmq.git
  cd gozmq
  gomake install
  popd

Differences from the C API
==========================
The API implemented by this package does not attempt to expose ``zmq_msg_t`` at
all. Instead, ``Recv()`` and ``Send()`` both operate on byte slices, allocating
and freeing the memory automatically. Currently this requires copying to/from C
malloced memory, but a future implementation may be able to avoid this to a
certain extent.

All major features are supported: contexts, sockets, devices, and polls.

Example
-------
Here are direct translations of some of the examples from `this blog post
<http://nichol.as/zeromq-an-introduction>`_.

A simple echo server::

  package main

  import zmq "github.com/alecthomas/gozmq"

  func main() {
    context, _ := zmq.NewContext()
    socket, _ := context.NewSocket(zmq.REP)
    socket.Bind("tcp://127.0.0.1:5000")
    socket.Bind("tcp://127.0.0.1:6000")

    for {
      msg, _ := socket.Recv(0)
      println("Got", string(msg))
      socket.Send(msg, 0)
    }
  }

A simple client for the above server::

  package main

  import "fmt"
  import zmq "github.com/alecthomas/gozmq"

  func main() {
    context, _ := zmq.NewContext()
    socket, _ := context.NewSocket(zmq.REQ)
    socket.Connect("tcp://127.0.0.1:5000")
    socket.Connect("tcp://127.0.0.1:6000")

    for i := 0; i < 10; i++ {
      msg := fmt.Sprintf("msg %d", i)
      socket.Send([]byte(msg), 0)
      println("Sending", msg)
      socket.Recv(0)
    }
  }

Caveats
=======

Memory management
-----------------
It's not entirely clear from the 0mq documentation how memory for ``zmq_msg_t``
and packet data is managed once 0mq takes ownership. After digging into the
source a little, this package operates under the following (educated)
assumptions:

- References to ``zmq_msg_t`` structures are not held by the C API beyond the
  duration of any function call.
- Packet data is reference counted internally by the C API. The count is
  incremented when a packet is queued for delivery to a destination (the
  inference being that for delivery to N destinations, the reference count will
  be incremented N times) and decremented once the packet has either been
  delivered or errored.
