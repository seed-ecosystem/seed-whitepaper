# Protocol Specification

Here is a step-by-step guide on how you should interact with **Seed Protocol**.

## Initialize connection

It's pretty simple, just open a WebSocket (wss) connection to the endpoint.

We have 2 equal pre-prod servers for now:

- `wss://meetacy.app/seed-go?queues=1`
- `wss://meetacy.app/seed-kt?queues=1&forwarding=1`

> [!NOTE]
> Client-server connection **must** be secured with **TLS**. The reason
for that is that traffic contain public queue identifiers, byt
if this can be spoofed, queues are easy to spam with gibberish data.

## Data Format

We stick to JSON format ATM, since it's human-readable and easier to debug,
though we do consider switching to some byte format in the future.

## Send Requests

Requests are pretty easy to send, you just specify arbitrary JSON to websocket.
Requests must be discriminated using `type` field. For example:

```
{"type": "send", "content":  "..."}
```

## Receive Response

When receiving responses, you must remember that they are executed consequently.
That way, you are allowed to send multiple requests in parallel, but make a special
queue to receive responses. Here is the example of multiple requests sent in parallel
and their responses:

```diff
>>> {"type": "subscribe", "queueId": "..."}
>>> {"type": "subscribe", "queueId": "..."}
// Server is thinking...
<<< {"type": "response", "response": {"status": "true"}} // Response to the first request
<<< {"type": "response", "response": {"status": "true"}} // Response to the second request
```

Responses are required to conform to the following schema:

```
{
  "type": "response",
  "response": { }
}
```

They do not contain "type" inside "response", since it is guaranteed that
the response would be of the type that is known from the request you sent earlier.

## Events

Even though requests are consequent, there are use-cases where you need to
receive events in background, while executing some requests in parallel. Events
have no restriction on parallel execution and that may be utilized handy.

Events are just JSON-objects with the following schema:

```
{
  "type": "event",
  "event": { }
}
```

They can be sent at any moment by the server. It depends on which functions you invoke and their specifications.

An example of using events:

```
>>> {"type": "subscribe", "queueId": "..."}
<<< {"type": "response", "response": {"status": "true"}} // Server replies with success subscription
<<< {"type": "event", "event": {"queueId": "...", "message": "..."}} // First event received
>>> {"type": "send", "content": "..."} // You sent some request
<<< {"type": "event", "event": {"queueId": "...", "message": "..."}} // Events can be received in-between
<<< {"type": "response", "response": {"status": "true"}} // Response to the 'send' request
<<< {"type": "event", "event": {"queueId": "...", "message": "..."}} // And after response as well
```

## Ping

The simplest request you can invoke is ping. You can send it with this request:

```
{"type": "ping"}
```

If server is up and running, response will always be:

```
{"type": "response", "response": {"status": true}}
```

## Features

Other features are server-dependent and not all features are required to be supported.
Every feature has `feature-key` and every supporting feature must be denoted in `serverUrl`
like this: `wss://example.com/ws?feature-key=1`.

Here is the set of supported features:

- [Queues](queues/README.md)
- [Forwarding](forwarding/README.md)
