# carmine-streams

Utility functions for working with [Redis streams](https://redis.io/topics/streams-intro) in [carmine](https://github.com/ptaoussanis/carmine).

Redis does a brilliant job of being fast with loads of features and Carmine does a brilliant job of exposing all the low-level Redis commands
in Clojure. Working with Redis' streams API requires quite a lot of interaction to produce desirable high-level behaviour, and that is what this
library provides.

**carmine-streams** allows you to create streams and consumer groups, consume streams reliably, deal with failed consumers and unprocessable messages
and gain visibility on the state of it all with a few simple functions.

[![Clojars Project](https://img.shields.io/clojars/v/carmine-streams.svg)](https://clojars.org/carmine-streams)

## Usage

- [Creating consumer groups and consumers](#creating-consumer-groups-and-consumers)
- [Visibility](#visibility)
- [Recovering from failures](#recovering-from-failures)
- [Utilities](#utilities)

### Creating consumer groups and consumers

#### Naming things

Consistent naming conventions for streams, groups and consumers:

```clj
(require '[carmine-streams.core :as cs])
(def conn-opts {})

(def stream (cs/stream-name "sensor-readings"))        ;; -> stream/sensor-readings
(def group (cs/group-name "persist-readings"))         ;; -> group/persist-readings
(def consumer (cs/consumer-name "persist-readings" 0)) ;; -> consumer/persist-readings/0
```

#### Consumer group creation

Idempotent consumer group creation:

```clj
(cs/create-consumer-group! conn-opts stream group)
```

#### Consumer creation

Start an infinite loop that consumes from the group:

```clj
(def opts {:block 5000})

(future
 (cs/start-consumer! conn-opts
                     stream
                     group
                     consumer
                     #(println "Yum yum, tasty message" %)
                     opts))
```

Consumer behaviour is as follows:

 - Calls the callback for every message received, with the message
   coerced into a keywordized map, and acks the message.
   If the callback throws an exception the message will not be acked
 - Processes all pending messages on startup before processing new ones
 - Processes new messages until either:
   - The consumer is stopped (see `stop-consumers!`)
   - There are no messages delivered during the time it was blocked waiting
     for a new message, upon which it will check for pending messages and
     begin processing the backlog if any are found, returning to wait for
     new messages when the backlog is cleared

 Options to the consumer consist of:

 - `:block` ms to block waiting for a new message before checking the backlog

#### Stop consumers

Send an unblock message to blocked consumers letting them exit gracefully:

```clj
;; stop all consumers matching consumer/*
(cs/stop-consumers! conn-opts)

;; stop only consumers matching consumer/persist-readings/*
(cs/stop-consumers! conn-opts (cs/consumer-name "persist-readings"))

;; stop all consumers of group
(cs/stop-consumers! conn-opts stream group)
```

### Visibility

#### All stream keys

```clj
;; all stream keys matching stream/*
(cs/all-stream-keys conn-opts) ;; -> #{"stream/sensor-readings"}

;; all stream keys matching persist-*
(cs/all-stream-keys conn-opts "persist-*")
```

#### All group names for a stream

```clj
(cs/group-names conn-opts stream) ;; -> #{"group/persist-readings"}
```

#### Stats for a consumer group

```clj
(cs/group-stats conn-opts stream group)

{:name "group/my-group",
 :consumers ({:name "consumer/my-consumer/0", :pending 1, :idle 102}
             {:name "consumer/my-consumer/1", :pending 0, :idle 208}
             {:name "consumer/my-consumer/2", :pending 0, :idle 311}),
 :pending 1,
 :last-delivered-id "0-2",
 :unconsumed 0}
```

### Recovering from failures

Garbage collect consumer groups to reallocate pending messages from dead consumers to live ones
and send undeliverable messages to a Dead Letter Queue (DLQ).

When a message is not acknowledged by the consumer (i.e. your consumer died halfway through,
or the callback threw an exception) it remains pending and its idle time is how long it has been
since it was first read.

The two possibilities are handled differently:

- If your consumer died and remains dead
  - The delivery count will remain at 1 and the idle time will increase
  - When the idle time has increased enough that it's obvious the consumer can't still be processing it
    we want to send it to another consumer that is alive - this is called rebalancing
  - The `:rebalance` option specifies
    - The `:idle` time necessary for a consumer/message to be considered dead before its messages are sent to another consumer
    - The `:siblings` option, when `:active` will apply the same test of idleness to sibling workers before claiming messages for them
    - The `:distribution` option decides how to distribute work to the siblings, the choices are:
      - `:random` random
      - `:lra` least-recently-active (with the highest idle time)
      - `:mra` most-recently-active (with the lowest idle time)
- If the message was bad and the worker throws an exception trying to process it
  - It will remain in the backlog which the worker will attempt to process during quiet times
  - The delivery count will increase on each attempt
  - When it reaches a particular value we will decide it cannot be processed and send it to a DLQ for later inspection
  - The `:dlq` option specifies
    - The number of `:deliveries` required before the message is considered unprocessable
    - The name of the `:stream` to write the message metadata to


```clj
(cs/gc-consumer-group! conn-opts stream group {:rebalance {:idle 60000
                                                           :siblings :active
                                                           :distribution :random}
                                               :dlq {:deliveries 5
                                                     :stream "dlq"}})
```

GC behaviour is as follows:

- Checks the pending messages for every consumer in the group
- Any message exceeding the threshold for the DLQ is sent to the DLQ
- Any remaining messages exceeding the threshold for rebalancing is rebalanced to other consumers based on the options
- Any remaining messages remain pending for their original consumer

Note that both `rebalance` and `dlq` criteria can specify `:idle` and `:deliveries` and that a message said to be exceeding the
criteria must have values exceeding one OR the other of the thresholds. By not specifying the threshold the criteria will not be compared.

You should run this function periodically, choosing values which trade off the following characteristics:
- What the maximum latency for a single message should be before it either fails or succeeds
- How many times you should attempt to rebalance a message before considering that it is killing consumers or is unprocessable

### Utilities

#### Message ids

Get the next smallest message id (useful for iterating through ranges as per `xrange` or `xpending`:

```clj
(cs/next-id "0-1") ;; -> 0-2
```

## Development

Start a normal REPL. You will need redis-server v5+ running on the default port to run the tests.

## License

Copyright © 2020 oliyh

This program and the accompanying materials are made available under the
terms of the Eclipse Public License 2.0 which is available at
http://www.eclipse.org/legal/epl-2.0.

This Source Code may also be made available under the following Secondary
Licenses when the conditions for such availability set forth in the Eclipse
Public License, v. 2.0 are satisfied: GNU General Public License as published by
the Free Software Foundation, either version 2 of the License, or (at your
option) any later version, with the GNU Classpath Exception which is available
at https://www.gnu.org/software/classpath/license.html.
