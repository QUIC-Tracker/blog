---
layout: post
title:  "Creating a test scenario in QUIC-Tracker"
date:   2018-11-21 14:02:07 +0100
categories: Tutorial
---

In this blog post we detail all the steps needed to add a new test scenario to QUIC-Tracker.
We start by defining what QUIC functionality we want to test, implement it using QUIC-Tracker and finally gather results from the [existing QUIC server implementations][quic-implems].
We use the visualisation application to inspect the results we collected.

This tutorial assumes that the reader has already installed QUIC-Tracker, is familiar with the Go programming language and the [QUIC specification][quic-spec].

[quic-spec]: https://tools.ietf.org/html/draft-ietf-quic-transport-16
[quic-implems]: https://github.com/quicwg/base-drafts/wiki/Implementations#ietf-quic

Testing the PING Frame functionality
------------------------------------

The [QUIC specification][quic-spec-ping] states the following:

```
19.8.  PING Frame

   Endpoints can use PING frames (type=0x07) to verify that their peers
   are still alive or to check reachability to the peer.  The PING frame
   contains no additional fields.

   The receiver of a PING frame simply needs to acknowledge the packet
   containing this frame.

   ...
```

Our test will be very simplistic. 
It will complete a 1-RTT handshake and then send a Protected packet containing a single `PING` frame.
Then it will check if the server issues an `ACK` frame confirming the receipt of this packet.

We want the test to be resilient to packet loss, so the `PING` frame will be retransmitted as long as no corresponding `ACK` frame is received.
To ensure that the test is finite, we will set a timeout of 10 seconds for its completion.
The test will fail if the server does not send an `ACK` within this time window.
If the handshake does not complete, we will classify the test run as invalid, i.e. we were unable to observe the correctness of the `PING` mechanism, and not consider this as a test failure, i.e. the `PING` mechanism is not working.

[quic-spec-ping]: https://tools.ietf.org/html/draft-ietf-quic-transport-16#section-19.8

A brief overview of QUIC-Tracker's architecture
-----------------------------------------------

We will summarise here the architecture of the tool. More information is available in its [`godoc`][qt-doc].

QUIC-Tracker is composed of three parts.
The first is the main package, which contains types, methods and functions to parse and create QUIC packets that can be easily manipulated.

The second is the `agents` package, which implements all the features and behaviours of a QUIC client as asynchronous message-passing objects.
These agents exchange messages through the broadcasting channels defined in the [`Connection` structure][qt-doc-connection].
This allows additional behaviours to be hooked up and to respond to several events that occur when the connection is active.
We will see how we can leverage them to easily formalise our test.

The third is the `scenarii`package, which contains all the tests of the test suite.
They are run using the scripts in the package `bin/test_suite`.
The tests results are produced in a unified JSON format.
It is described in details in the [`Trace` structure documentation][qt-doc-trace], but we will summarise it in this tutorial.
We will add a new scenario to this package.


[qt-doc]: https://godoc.org/github.com/QUIC-Tracker/quic-tracker
[qt-doc-trace]: https://godoc.org/github.com/QUIC-Tracker/quic-tracker#Trace
[qt-doc-connection]: https://godoc.org/github.com/QUIC-Tracker/quic-tracker#Connection


Implementing the test
---------------------

Each scenario is a new type that implements the [`Scenario`][scenario-interface] interface.

```go
type Scenario interface {
	Name() string
	Version() int
	IPv6() bool
	Run(conn *qt.Connection, trace *qt.Trace, preferredUrl string, debug bool)
	Timeout() *time.Timer
}
```

First, let us create a new file named `scenarii/ping.go`.
It will define a new type for the scenario and the code that it will execute.
Each scenario must embed the `AbstractScenario` type. We thus create a new type and an associated constructor.

```go
import qt "github.com/QUIC-Tracker/quic-tracker"

type PingScenario struct {
	AbstractScenario
}

func NewPingScenario() *PingScenario {
	return &PingScenario{
		AbstractScenario{
			name: "ping",  // The name must match the scenario filename
			version: 1,    // This value is echoed in the test results traces
			ipv6: false,   // Forces the test to execute over IPv6
			timeout: nil,
		},
	}
}
```

The Go typing system does not allow dynamic imports.
We have to add this constructor to the test suite manually.
To do this, locate the [`GetAllScenarii` function][get-all-scenarii] and add a mapping with the name and constructor of the scenario to its return value.

We are now missing the last important bit, the scenario code itself.
We will complete the `Run` function step by step to achieve our goal. 

```go
func (s *PingScenario) Run(conn *qt.Connection, trace *qt.Trace, preferredUrl string, debug bool) {

}
```

This function is called each time the test is run against a particular host. It has four parameters.
* `conn` is a pointer to a `Connection` initialised for a particular host, i.e. a UDP socket has been established and the initial QUIC context is ready to be used but no handshake or any particular packets were already exchanged.
* `trace` is a pointer to a `Trace` used to collect the test results.
A test execution must set its `ErrorCode` field according to its outcome. The meaning of its value is test-specific, but we generally choose 0 to represent a test success.
The `Results` field is a map that will be serialised as a JSON object, allowing a test to output test-specific results. For example, in this test, we could measure the time elapsed between the sending of a `PING` and the receipt of an acknowledgement. We could also report the number of `PING`s that were sent before the test completed
* `preferredURL` is a string indicating which URL is valid for generating traffic in response to an HTTP request. We will not use it in this test, because we do not want to generate data in order to isolate the `PING` mechanism.
* `debug` is a flag that allows putting conditional debug statements inside scenarii.

Let us define error codes constants before filling this method body.

```go
const (
	P_TLSHandshakeFailed = 1  // The handshake did not complete
	P_NoACKReceived      = 2  // No ACK was received
)
```

### Establishing a QUIC connection

For the purpose of our test, we want to establish a QUIC connection.
If the tested server fails to succeed it, the test is aborted, otherwise it can proceed.
The `AbstractScenario` type provides a helper method that achieve all of this.

```go
//func (s *PingScenario) Run(conn *qt.Connection, trace *qt.Trace, preferredUrl string, debug bool) {
s.timeout = time.NewTimer(10 * time.Second)

connAgents := s.CompleteHandshake(conn, trace, P_TLSHandshakeFailed)
if connAgents == nil {
	return
}
defer connAgents.CloseConnection(false, 0, "")  // Sends an APPLICATION_CLOSE with a zero error code and no reason phrase.
```

We first arm a timer for the test completion. This should be the first statement of `Run`.
Then we call the blocking method `CompleteHandshake` which will handle the handshake until completion or failure. The method will return a common set of agents needed for the handshake to complete.
These agents are:

* The `SocketAgent`, responsible for reading from the UDP socket.
* The `ParsingAgent`, responsible for decrypting and parsing UDP payloads into QUIC packets.
* The `BufferAgent`, responsible for keeping UDP payloads that cannot be decrypted until the required decryption level is available.
* The `TLSAgent`, responsible for reading the data from `CRYPTO` frames, interacting with the TLS stack and sending its responses.
* The `AckAgent`, responsible for issuing `ACK` frames in response to received packets.
* The `SendingAgent`, responsible for scheduling the sending of frames into packets.
* The `RecoveryAgent`, responsible for retransmitting frames that are part of packets considered as lost.

If the handshake fails, the method will set the error code in the test trace to the given value.
A `nil` value will be returned in this case.
Otherwise, we defer the closing of the connection at the end of the test.

### Sending the PING frame and listening for incoming packets 

At this point, the connection is established and we are ready to send a `PING` frame.

```go
pp := qt.NewProtectedPacket(conn)           // We create a short header packet and reserve a packet number
pp.AddFrame(new(qt.PingFrame))              // We add a PING frame to the payload of the packet
conn.SendPacket(pp, qt.EncryptionLevel1RTT) // We send this packet encrypted using the 1-RTT keys
```

The next step is to listen to incoming packets and verify the `ACK` frames they may contain.
Incoming packets are broadcast using the broadcast channel `IncomingPackets` of the connection.
We have to register our own channel to receive these packets.

```go
incomingPackets := make(chan interface{}, 1000)  // We create a new channel with sufficient buffer space
conn.IncomingPackets.Register(incomingPackets)   // We register the channel
```

The logic of our test is the following. For every packet, if it is a protected packet, we iterate over the `ACK` frames it may contain.
For each of the `ACK` frame, we compute the packets acknowledged. If the packet containing the `PING` frame is acknowledged, the test succeeds.
Otherwise, we wait until the test timer fires and declares the test as failed. Go allows us to express this logic as follows.

```go
for {
	select {  // We wait repeatedly until one of two events occurs
	case i := <-incomingPackets:  // We either receive a packet
		switch p := i.(type) {
		case *qt.ProtectedPacket:
			for _, f := range p.GetAll(qt.AckType) {
				// We iterate over the packets acknowledged
				for _, pn := range f.(*qt.AckFrame).GetAckedPackets() {
					if pp.Header().PacketNumber() == pn {
						trace.ErrorCode = 0
						return
					}
				}
			}
		}
	case <-s.Timeout().C:  // Or the timeout of this test expires
		trace.ErrorCode = P_NoACKReceived
		return
	}
}
```

This is a good first step, but we are missing a requirement as the `PING` frame will not be retransmitted in case of losses.
The `RecoveryAgent` does not retransmit `PING` frame because it is not enforced by the specification.
We will supplement the select statement with another linear timer of half a second that will retransmit the frame inside a new packet.
Because we set a timer for the test duration, we are sure that we will not storm the tested server with `PING` frames indefinitely.
Setting an exponential backoff instead of a linear timer is left as an exercise for the reader.

We first modify the code before the select loop.

```go
pp := qt.NewProtectedPacket(conn)
pp.AddFrame(new(qt.PingFrame))
conn.SendPacket(pp, qt.EncryptionLevel1RTT)

// We keep track of the packets containing the `PING` frames
pings := []qt.PacketNumber{pp.Header().PacketNumber()}
// We arm a timer for retransmitting the frame
pingRetransmit := time.NewTicker(500 * time.Millisecond)
```

Then we thus have to modify the `ACK` processing to check whether a packet containing the `PING` was acknowledged. 



```go
for {
	select {
	case i := <-incomingPackets:
		switch p := i.(type) {
		case *qt.ProtectedPacket:
			for _, f := range p.GetAll(qt.AckType) {
				for _, pn := range f.(*qt.AckFrame).GetAckedPackets() {
					for _, ping := range pings {
						if pn == ping {
							trace.ErrorCode = 0
							return
						}
					}
				}
			}
		}
	case <-pingRetransmit.C: // The timer ticks every half second
		pp := qt.NewProtectedPacket(conn)
		pp.AddFrame(new(qt.PingFrame))
		conn.SendPacket(pp, qt.EncryptionLevel1RTT)
		// We add the packet we just sent to the list
		pings = append(pings, pp.Header().PacketNumber())
	case <-s.Timeout().C:
		trace.ErrorCode = P_NoACKReceived
		return
	}
}
```

[get-all-scenarii]: https://godoc.org/github.com/QUIC-Tracker/quic-tracker/scenarii#GetAllScenarii
[scenario-interface]: https://godoc.org/github.com/QUIC-Tracker/quic-tracker/scenarii#Scenario

Collecting test results
-----------------------

The test is now implemented, we are thus ready to run it against QUIC servers and collect its results.
Several QUIC servers are publicly available in the case that you are not running your own server to test.
Consulting [the daily results grid of QUIC-Tracker][qt-grid] is the easiest way of determining which server is currently available.

There exists two scripts, located in the `bin/test_suite` package, to run the scenario.
The first is `scenario_runner.go` which runs a single test against a single server.
For example, starting the scenario against the [picoquic endpoint][picoquic] is the following: 

```sh
go run bin/test_suite/scenario_runner.go -scenario ping \ 
	-host test.privateoctopus.com:4433 \
	-output /tmp/qt_ping_picoquic.json
```

The test trace is written to the given `-output` parameter value at the end of the test run. We detail in the next section how to use this trace.
Additional parameters allows to specify a network interface to capture the test traffic into a pcap file, and to separate the 
Use the `-h` parameter to get the full usage of the script.

The `test_suite.go` script runs all or one scenario against several servers. A list of servers is already bundled with QUIC-Tracker.
It can run the test runs in parallel, randomise their order and output all the results in a single trace file.

```sh
go run bin/test_suite/test_suite.go -scenario ping \
	-hosts ietf_quic_hosts.txt \
	-parallel -max-instances 4 \
	-output /tmp/qt_ping_all.json
```

Visualising QUIC-Tracker traces
-------------------------------

QUIC-Tracker outputs its results as a JSON object. `scenario_runner.go` outputs a trace as a single JSON object. `test_suite.go` outputs the traces as a list of JSON objects. 
The trace format is [formalised as a Go structure][qt-doc-trace] which is then serialised into JSON.
The simplest way of extracting information from the traces is to retrieve the `error_code` field.

There exists [a web application][qt-web-app] that allows to visualise the traces, i.e. consult the decrypted packets exchanged during the test run as well as more information.
To use this application with new scenarios, some metadata have to be added to the [`scenarii.yaml` file][scenarii-yaml] to allow the application to annotate the test results in a human-readable manner.

We add the following block to describe this new test:

```yaml
ping:  # This corresponds to the test filename
  name: PING
  description: |
    This test verifies that packets containing only <code>PING</code> frames elicit an acknowledgement from the server.
    The test send <code>PING</code> frames regurlarly until an acknowledgement is received.
  error_codes:  # This converts the error code into a sentence
    1: The TLS handshake failed
    2: No acknowledgement was received
  error_types:  # This classifies the error codes
    error: # The test encountered an error and could not be conducted
      - 1
    failure: # The test has been conducted and failed
      - 2
```

We can now give the set of traces collected by `test_suite.go` to the application.
The JSON trace file should be placed in the `traces` directory, [at the root of the application package][webapp-root].
The filename must follow the format `%Y%m%d.json`.
This is a design choice we took because the test suite was originally running daily, it is not ideal and we are working toward a more flexible solution.

Consulting the application will present the latest trace in the directory, as indicated by its filename. 
Showing a particular trace is achieved by navigating to `/traces/%Y%m%d`.

![Trace list](/blog/assets/trace_list.png)

This page shows the list of traces collected. One can click on a particular trace to get more information. 
Clicking on the fifth trace present the following page.

![Trace detail 1](/blog/assets/trace_detail_1.png)

A list of the packets sent and received is displayed in the top-left corner. 
A summary of the trace, its description, a link to the test source code and download links are displayed in the top-right corner.
The cleartext content of the highlighted packet is displayed in the bottom-left corner.
The bottom-right corner offers a dissection of the highlighted packet.

![Trace detail 2](/blog/assets/trace_detail_2.png)

One can highlight a dissected value on the right side and its counterpart in the packet payload will be highlighted as well. 
In this test results, the 13th packet is the one acknowledging the packet containing the `PING` frame.

[qt-web-app]: https://github.com/QUIC-Tracker/web-app
[qt-grid]: https://quic-tracker.info.ucl.ac.be/grid
[picoquic]: https://github.com/private-octopus/picoquic
[scenarii-yaml]: https://github.com/QUIC-Tracker/web-app/blob/master/quic_tracker/scenarii.yaml
[webapp-root]: https://github.com/QUIC-Tracker/web-app/tree/master/quic_tracker

Going further
-------------

The test scenario we present and implement in this tutorial has been kept quite simple in the interest of space.
But it can be improved in several manners. 
For example, the `ACK_ECN` frame also carries acknowledgements and [is sent by the server][quic-ack-ecn] whenever [the ECN bits in the IP header of the packets it receives are set][ip-ecn].
Therefore, the test should also process these frames.
The test could also be modified to always wait for the timer to expire before finishing.
The receipt of an acknowledgement would only stop the transmission of `PING`s in that case.
It has been shown that some of our tests triggered abnormal behaviours on several implementations.

[quic-ack-ecn]: https://tools.ietf.org/html/draft-ietf-quic-transport-16#section-13.3
[ip-ecn]: https://tools.ietf.org/html/rfc3168

Conclusion
----------

In this tutorial we explain the architecture of QUIC-Tracker and build a new scenario to test a particular feature of QUIC.
We detail every step of the implementation of this test. We gather results from existing QUIC servers and inspect them using the visualisation tool.
The tutorial code can be found [on Github][tutorial-code] for reference.

Do not hesitate to share the experiments and new tests you developed using QUIC-Tracker.
You are also welcomed to report bugs and suggest improvements [on Github][qt-github].

[tutorial-code]: https://github.com/QUIC-Tracker/quic-tracker/compare/master...quic-tracker:tutorial_ping
[qt-github]: https://github.com/QUIC-Tracker
