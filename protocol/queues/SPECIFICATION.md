# Seed Queues Specification

Here is step-by-step guide on how you should interact with **Seed Queues**.

## Initialize connection

It's pretty simple, just open WebSocket (wss) connection to an endpoint.

Our pre-prod endpoint currently is: `wss://meetacy.app/seed-go`.

There will be multi-server support, so don't hardcode this string. 

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

## Send

Send request makes it possible to send an event for everyone listening for the
queue. 

You can do it by sending the following JSON:

```
{
  "type": "send",
  "message": {
    "queueId": string, // Queue where you want send messages to
    "nonce": number,
    "signature": string, // base64-encoded 32 bytes 
    "content": string, // base64-encoded content bytes
    "contentIV": string //  base64-encoded 12 initialization vector bytes
  }
}
```

`nonce` is a little bit tricky. It starts from zero and client should increment it.
To get next value for `nonce`, you just take last known event from this queue
and add `1` to it.

Response would be:

```
{"type": "response", "response": {"status": boolean}}
```

`status` is either true or false and it reflects if the provided nonce was valid

### What if multiple client will send message with the same nonce?

We will talk about encryption later, but this is something we want to
avoid, because encryption basically implies that you know the last 
event before encrypting the next one.

So, if such occurrence happens, you just resend it. Here is the example:

We start from subscribing to queue.

```
>>> {"type": "subscribe", "queueId": "...", "nonce": 1}
<<< {"type": "response", "response": {"status": "true"}} 
<<< {"type": "event", "event": {"type": "new", "message": {"nonce": 1, ...}}
```

We subscribed to queue with some id, provided that we need to receive messages with `nonce >= 1`.
Server then sent us a new event with `nonce = 1`. Next nonce is clearly `2`. So we try to send message with
that `2` nonce.

```
>>> {"type": "send", "message": {"nonce": 2, ...}
<<< {"type": "response", "response": {"status": "false"}} 
<<< {"type": "event", "event": {"type": "new", "message": {"nonce": 2, ...}}
```

Apparently, we got `status = false` and a new event with `nonce = 2`. That way server says to us,
that while we were trying to send our message, some client was faster and already got his message
in database before ours. So, new nonce is now `3` and we resend our message with that nonce.

We do that, until we can finally insert our message. In official apps there is a limit on 20 attempts
to prevent bugs causing infinite attempts and server spam.

## Subscribe

To receive updates from queue, you send the following JSON:

```
{
  "type": "subscribe", 
  "queueId": string, // queueId that was shared with you
  "nonce": number // first message that you want to receive (it is usually shared with you with privateKey and queueId)
}
```

First what server will do is respond to your with the following response:

```
{"type": "response", "response": {"status": true}}
```

That means that server successfully offloaded a job that will send events from this queue.
The job itself will send you events in `2` phases:

- **First phase**
  
  If you had some events that you didn't receive while you were offline, server first will
  send them to you. And then you will see a message like this:
  ```
  {"type": "event", "event": {"type": "wait", "queueId": "..."}}
  ```
  Basically before you have received `wait` event, you may show a loader to user, so
  he knows that history isn't loaded yet. `wait` event is sent even if there is no events that
  you missed.

- **Second phase**

  This is where you just wait for new messages from queue. They have the following format:
  ```
  {"type": "event", "event": {"type": "new", "message": {  }}}
  ```
  `message` is basically the same you sent using `send` request.

## Encryption

TODO: Encryption part is not ready yet.

## Private Key Exchange

Ironically, the main point of **Seed Queues** is not the part of this specification,
but rather a contract. You must not send **any** sensitive data to server. 
Every existing key-exchange algorithm requires server trust. Even Diffie-Hellman
is subject for MITM. That way, we just offloaded key-exchange, and it must be
executed through side-channels (using QR-code, link though different service, etc.)

Later we will provide a builtin way to exchange keys except that public keys will
still be distributed through side-channels, so Seed servers can't interfere with them.
