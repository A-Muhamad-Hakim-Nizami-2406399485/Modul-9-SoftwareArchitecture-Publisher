# Publisher — Tutorial Modul 9 (Event-Driven Architecture)

**Name:** Muhamad Hakim Nizami
**NPM:** 2406399485

This is the **publisher** side of the event-driven tutorial. On every run, it
opens an AMQP connection to RabbitMQ at `amqp://guest:guest@localhost:5672`
and publishes **five** `UserCreatedEventMessage` events to the `user_created`
queue. The broker forwards them to whatever subscribers are listening.

---

## Understanding publisher and message broker

### a. How much data does the publisher send in one run?

In one run the publisher emits **five** events to the message broker. Each
event is a `UserCreatedEventMessage`, which is just two `String` fields:

| Field        | Example value         |
| ------------ | --------------------- |
| `user_id`    | `"1"` .. `"5"`        |
| `user_name`  | `"2406399485-Amir"` … |

The messages are serialized with **Borsh** (a compact binary format). For
each event, Borsh encodes each `String` as a 4-byte little-endian length
prefix followed by the UTF-8 bytes. For event #1 (`user_id = "1"`,
`user_name = "2406399485-Amir"`) the payload is:

```
4 + 1   (user_id  "1")           =  5 bytes
4 + 15  (user_name "2406399485-Amir") = 19 bytes
                                ------
                                  24 bytes payload
```

The other four events follow the same pattern (the name is always 15 bytes:
`2406399485-` + 4-letter name), so each event payload is **24 bytes**, and
all five events together carry **120 bytes** of application payload.

On the wire it is a bit more than that, because every published message is
wrapped in AMQP frames (a method frame, a content-header frame with the
properties, and one or more content-body frames). With AMQP framing overhead
of roughly 20–30 bytes per message on top of the payload, the publisher
sends on the order of **~250 bytes** of AMQP traffic per run for the five
events combined — small, but enough to clearly show up on the RabbitMQ
"Message rates" chart.

### b. What does `amqp://guest:guest@localhost:5672` mean? (same URL as the subscriber)

The publisher and the subscriber use the **same broker URL** on purpose —
that is *the entire point* of a message broker: both sides agree on where to
meet (the broker), and then they no longer have to know about each other.

Reading the URL piece-by-piece:

- **`amqp://`** — the **scheme**, telling the client to speak the Advanced
  Message Queuing Protocol (the standard wire protocol that RabbitMQ
  implements).
- **`guest:guest`** — `username:password`. The first `guest` is the default
  user that ships with RabbitMQ; the second `guest` is its default password.
  (This account is only allowed to connect from `localhost` for security.)
- **`localhost`** — the **host** where the broker is running. In this
  tutorial it is the same machine that runs the publisher (and the
  subscriber).
- **`5672`** — the **default AMQP port**. RabbitMQ listens on 5672 for AMQP
  client traffic; its web management UI lives on the separate port `15672`.

So both publisher and subscriber are saying: "*Connect to the AMQP broker on
this machine, port 5672, and log in as `guest`/`guest`.*" The publisher uses
that connection to **push** events into the `user_created` queue; the
subscriber uses the same connection details to **pull** events out of it.
The broker is the rendezvous point.

---

## Running RabbitMQ as message broker

The broker is started locally as a Docker container with the standard
RabbitMQ management image:

```bash
docker run -d --rm --name rabbitmq \
    -p 5672:5672 -p 15672:15672 \
    rabbitmq:3.13-management
```

- Port **5672** is the AMQP port (the one the publisher and subscriber
  connect to).
- Port **15672** is the management UI port — browsing to
  `http://localhost:15672` and logging in with `guest` / `guest` shows the
  Overview dashboard with queues, exchanges, connections, and the live
  message-rate charts.

Screenshot of the running RabbitMQ dashboard (Overview tab, port 15672):

![RabbitMQ Overview](screenshots/rabbitmq-overview.png)

---

## Sending and processing event

With RabbitMQ running and the subscriber connected to the `user_created`
queue, running `cargo run` in this publisher directory pushes **five
`UserCreatedEventMessage`s** to the broker. The subscriber console then
prints one line per received event:

```
In Hakim's Computer [2406399485]. Message received: UserCreatedEventMessage { user_id: "1", user_name: "2406399485-Amir" }
In Hakim's Computer [2406399485]. Message received: UserCreatedEventMessage { user_id: "2", user_name: "2406399485-Budi" }
In Hakim's Computer [2406399485]. Message received: UserCreatedEventMessage { user_id: "3", user_name: "2406399485-Cica" }
In Hakim's Computer [2406399485]. Message received: UserCreatedEventMessage { user_id: "4", user_name: "2406399485-Dira" }
In Hakim's Computer [2406399485]. Message received: UserCreatedEventMessage { user_id: "5", user_name: "2406399485-Emir" }
```

Screenshot of the two consoles side-by-side (publisher on the left having
finished, subscriber on the right printing the five received events):

![Publisher and Subscriber consoles](screenshots/publisher-subscriber-consoles.png)

What is happening here is the canonical event-driven flow:

1. The publisher process opens an AMQP connection to RabbitMQ and calls
   `publish_event("user_created", …)` five times in quick succession,
   then exits — its job is done the moment the broker has acknowledged the
   message. It does not know who (if anyone) will consume the events.
2. The broker stores each event in the `user_created` queue and immediately
   delivers it to any subscriber that has registered as a consumer of that
   queue. With one subscriber running, it sees all five events.
3. The subscriber's `UserCreatedHandler::handle` method runs once per
   delivered event and prints the line above.

This *decoupling in time and address space* is the whole point of using a
broker: even if the subscriber were restarted, slow, or temporarily
disconnected, RabbitMQ would still accept the publisher's events and hold
them in the queue until a consumer is ready.

---

## Monitoring chart based on publisher

If `cargo run` is executed several times in a row from this directory while
the management UI at `http://localhost:15672` is open on the **Overview**
tab, the **Message rates** chart (second chart from the top) shows clear
**spikes** — one per publisher run — on the *Publish*, *Deliver* and
*Consumer ack* lines. Each spike is 5 messages tall, matching the five
`publish_event` calls in `main.rs`.

Screenshot of the management UI showing the spikes:

![RabbitMQ message-rates spikes](screenshots/rabbitmq-message-spikes.png)

### Why does the chart spike like that?

Each `cargo run` is one *burst* of activity:

1. The publisher process starts, opens a TCP/AMQP connection to the
   broker, declares the `user_created` queue, and sends five
   `basic.publish` frames back-to-back.
2. The broker accepts those five messages, increments its **Publish**
   counter for the current second, and almost immediately dispatches them
   to the connected subscriber, which increments **Deliver (manual ack)**
   and then **Consumer ack** as each message's handler returns `Ok(())`.
3. The publisher process exits, closing its connection.

Because all five messages travel through the broker within the same
1-second sampling window, the chart records them as a single spike of
~5 msg/s. Between runs the rates fall back to zero — no traffic, no work
to do. The more publisher runs are fired in quick succession, the more
spikes appear, and the spike heights grow if multiple runs land in the
same one-second window. The chart is, in effect, a histogram of how
"busy" the publisher has been per second.

