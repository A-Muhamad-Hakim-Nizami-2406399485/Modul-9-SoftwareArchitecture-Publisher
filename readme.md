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
