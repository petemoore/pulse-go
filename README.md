# pulse-go

[![logo](https://tools.taskcluster.net/b2d854df0391f8b777f39a486ebbc868.png)](https://tools.taskcluster.net/b2d854df0391f8b777f39a486ebbc868.png)

[![Build Status](https://secure.travis-ci.org/taskcluster/pulse-go.png)](http://travis-ci.org/taskcluster/pulse-go)
[![GoDoc](https://godoc.org/github.com/taskcluster/pulse-go?status.png)](https://godoc.org/github.com/taskcluster/pulse-go)

A go (golang) library for consuming mozilla pulse messages (http://pulse.mozilla.org/).

## This project contains three go packages:

# Package 1: "github.com/taskcluster/pulse-go"

This is a command line interface for consuming mozilla pulse messages, written in go.

The best way to see how to use this command line tool, is to run it with the --help option:

```
pmoore@home:~ $ pulse-go --help
pulse-go
pulse-go is a very simple command line utility that allows you to specify a list of Pulse
exchanges/routing keys that you wish to bind to, and prints the body of the Pulse messages
to standard out.

Derivation of username, password and AMQP server url
====================================================
If no AMQP server is specified, production will be used (amqps://pulse.mozilla.org:5671).

If a pulse username is specified on the command line, it will be used.
Otherwise, if the AMQP server url is provided and contains a username, it will be used.
Otherwise, if a value is set in the environment variable PULSE_USERNAME, it will be used.
Otherwise the value 'guest' will be used.

If a pulse password is specified on the command line, it will be used.
Otherwise, if the AMQP server url is provided and contains a password, it will be used.
Otherwise, if a value is set in the environment variable PULSE_PASSWORD, it will be used.
Otherwise the value 'guest' will be used.

  Usage:
      pulse-go [-u <pulse_user>] [-p <pulse_password>] [-s <amqp_server_url>] (<exchange> <routing_key>)...
      pulse-go -h | --help

  Options:
    -h, --help            Display this help text.
    -u <pulse_user>       The pulse user to connect with (see http://pulse.mozilla.org/).
    -p <pulse_password>   The password to use for connecting to pulse.
    -s <amqp_server_url>  The full amqp/amqps url to use for connecting to the pulse server.

  Examples:
    1)  pulse-go -u pmoore_test1 -p potato123 \
        exchange/build/ '#' \
        exchange/taskcluster-queue/v1/task-defined '*.*.*.*.*.null-provisioner.buildbot-try.#'

    This would display all messages from exchange exchange/build/ and only messages from
    exchange/taskcluster-queue/v1/task-defined with provisionerId = "null-provisioner" and
    workerType = "buildbot-try" (see
    https://docs.taskcluster.net/reference/platform/queue/exchanges#taskDefined for more
    information).

    Remember to quote your routing key strings on the command line, so they are not
    interpreted by your shell!

    Please note if you are interacting with taskcluster exchanges, please consider using one
    of the following libraries, for better handling:

      * https://github.com/taskcluster/taskcluster-client-go
      * https://github.com/taskcluster/taskcluster-client

    2) pulse-go -s amqps://admin:peanuts@localhost:5671 exchange/treeherder/v2/new-result-set '#'

    This would match all messages on the given exchange, published to the local AMQP service
    running on localhost. Notice that the user and password are given as part of the url.
pmoore@home:~ $ 
```

# Package 2: "github.com/taskcluster/pulse-go/pulse"

This is a pulse client library, written in go.

The full API documentation is available at http://godoc.org/github.com/taskcluster/pulse-go/pulse

# Package 3: "github.com/taskcluster/pulse-go/pulsesniffer"

This package contains the following *example* program, which uses the pulse library (package 2 above).

```go
// Package pulsesniffer provides a simple example program that listens to some
// real world pulse messages.
package main

import (
	"fmt"
	"github.com/taskcluster/pulse-go/pulse"
	"github.com/streadway/amqp"
)

func main() {
	// Passing all empty strings:
	// empty user => use PULSE_USERNAME env var
	// empty password => use PULSE_PASSWORD env var
	// empty url => connect to production
	conn := pulse.NewConnection("", "", "")
	conn.Consume(
		"taskprocessing", // queue name
		func(message interface{}, delivery amqp.Delivery) { // callback function to pass messages to
			fmt.Println("Received from exchange " + delivery.Exchange + ":")
			fmt.Println(string(delivery.Body))
			fmt.Println("")
			delivery.Ack(false) // acknowledge message *after* processing
		},
		1,     // prefetch 1 message at a time
		false, // don't auto-acknowledge messages
		pulse.Bind( // routing key and exchange to get messages from
			"*.*.*.*.*.*.gaia.#",
			"exchange/taskcluster-queue/v1/task-defined"),
		pulse.Bind( // another routing key and exchange to get messages from
			"*.*.*.*.*.aws-provisioner.#",
			"exchange/taskcluster-queue/v1/task-running"))
	conn.Consume( // a second workflow to manage concurrently
		"", // empty name implies anonymous queue
		func(message interface{}, delivery amqp.Delivery) { // simpler callback than before
			fmt.Println("Buildbot message received")
			fmt.Println("")
		},
		1,    // prefetch
		true, // auto acknowledge, so no need to call delivery.Ack
		pulse.Bind( // routing key and exchange to get messages from
			"#", // get *all* normalized buildbot messages
			"exchange/build/normalized"))
	// wait forever
	forever := make(chan bool)
	<-forever
}
```

Travis build success/failure messages are posted to irc channel #tcclient-go on irc.mozilla.org:6697.
