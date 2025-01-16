# Seed Queues Specification

Here is step-by-step guide on how you should interact with **Seed Queues**.

## Initialize connection

It's pretty simple, just open WebSocket (wss) connection to an endpoint.

Our pre-prod endpoint currently is: `wss://meetacy.app/seed-go`.

There will be multi-server support, so don't hardcode this string. 

> [!NOTE] 
> Client-server connection **must** be secured with **TLS**. The reason
for that is that traffic may contain some public-data like random chat identifiers,
if this can be spoofed, chats are easy to spam with gibberish data.

## Data Format

We stick to Json format ATM, since it's human-readable and easier to debug, 
though we do consider switching to some byte format in the future.

## Send Requests

Requests are pretty easy to send, you just specify arbitrary JSON to websocket.
Requests must be discriminated using `type` field. For example:

```json
{"type": "send", "content":  "..."}
```

## Receive Response

When receiving responses, you must remember that they are executed consequently.
That way, you are allowed to send multiple requests in parallel, but make a special
queue to receive responses. Here is the example of multiple requests sent in parallel
and their responses:

```diff
>>> {"type": "subscribe", "chatId": "..."}
>>> {"type": "subscribe", "chatId": "..."}
// Server is thinking...
<<< {"type": "response", "response": {"status": "true"}} // Response to the first request
<<< {"type": "response", "response": {"status": "true"}} // Response to the second request
```

Responses are required to conform to the following schema:

```json
{
  "type": "response",
  "response": { }
}
```

## Events

Even though requests are consequent, there are use-cases where you need to 
receive events in background, while executing some requests in parallel. Events
have no restriction on parallel execution and that may be utilized handy. 

Events are just JSON-objects with the following schema:

```json
{
  "type": "event",
  "event": { }
}
```

They can be sent at any moment by the server. It depends on which functions you invoke and their specifications.

An example of using events:

```diff
>>> {"type": "subscribe", "chatId": "..."}
<<< {"type": "response", "response": {"status": "true"}} // Server replies with success subscription
<<< {"type": "event", "event": {"chatId": "...", "message": "..."}} // First event received
>>> {"type": "send", "content": "..."} // You sent some request
<<< {"type": "event", "event": {"chatId": "...", "message": "..."}} // Events can be received in-between
<<< {"type": "response", "response": {"status": "true"}} // Response to the 'send' request
<<< {"type": "event", "event": {"chatId": "...", "message": "..."}} // And after response as well
```

## Ping

Ping is a part of **Seed Queues** specification. You can send ping with this request:

```json
{"type": "ping"}
```

If server is up and running, response will always be:

```json
{"type": "response", "response": {"status": true}}
```

## Private Key Exchange

Ironically, the main point of **Seed Queues** is not the part of this specification,
but rather a contract. You must not send **any** sensitive data to server. 
Every existing key-exchange algorithm requires server trust. Even Diffie-Hellman
is subject for MITM. That way, we just offloaded key-exchange, and it must be
executed through side-channels (using QR-code, link though different service, etc.)

Later we will provide a builtin way to exchange keys except that public keys will
still be distributed through side-channels, so Seed servers can't interfere with them.
