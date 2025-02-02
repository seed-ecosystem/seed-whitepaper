# Forwarding

feature-key: `forwarding`

---

Server also may be used as a proxy-server to access other servers. This helps us reduce
WebSocket connections amount, and makes chatting more efficient.

## Connect

Before sending forward request to any server, you 
must connect this server with `connect` request:

```
{
  "type": "connect",
  "url": "https://meetacy.app/seed-go"
}
```

Response (even if connection was not successful) should always be:

```
{
  "type": "response",
  "response": {
    "status": true
  }
}
```

If connection was not successful or if it was closed later server will 
send you the following event:

```
{
  "type": "event",
  "event": {
    "type": "disconnected",
    "url": "https://meetacy.app/seed-go"
  }
}
```

Calling `connect` if endpoint is already connected will do
nothing and server will respond with:

```
{
  "type": "response",
  "response": {
    "status": false
  }
}
```

Reconnecting and resending failed requests is client's responsibility.

## Forward

Forward request looks like this:

```
{
  "type": "forward", 
  "url": "https://meetacy.app/seed-kt", 
  "request": {...}
}
```

Server then responds with:

```
{
  "type": "response",
  "status": boolean
}
```

Where `status` shows whether connection to `url` was opened or not.
If `status` is false, it means that server was not connected to `url`.

After some time, when forwarding server received response from `url`,
forwarding server sends you the following message:

```
{
  "type": "forward", 
  "url": "https://meetacy.app/seed-kt", 
  "forward": {
    "type": "response", 
    "response": {...}
  }
}
```

Note that `forward` object contains the exact message from the forwarded 
server including `"type": "response"`. This is because you also may also 
receive forwarded events like this:

```
{
  "type": "forward", 
  "url": "https://meetacy.app/seed-kt", 
  "forward": {
    "type": "event", 
    "event": {...}
  }
}
```

If `url` was not connected, forwarding server will just ignore
those requests.

> [!WARNING]
> API like this makes it possible for forwarding server to read all the traffic.
> It's not that critical since we anyway don't pass any sensitive data, yet
> this makes it possible to track people IP-addresses and mapping them to sent requests 
> just by controlling this single server.
> 
> In the future we are considering to implement onion-routing which will makes it so
> server that knows IP-address doesn't know what is inside the request and vice versa
> server that knows what's inside request doesn't know what is the IP-address.

WebSocket connection to other server is opened upon the first request
and is reused for every subsequent request.

`forward` requests unlike other requests must run in background, so specific poor server
will not affect user in a way that every just becomes slow.

> [!WARNING]
> You can't pass 'forward' request inside 'forward' request (yet).

## Self-Forwarding

Server **must** support this optimization since clients with
multi-server connections will use this feature. If `url` field is set
to the url of the current server, you don't need to `forward` request,
but rather it should be executed in place.

Imagine running at `https://example.com/` and you get the following request:

```
{
  "type": "forward",
  "url": "https://example.com",
  "request": {...}
}
```

You don't need to make a recursive request since you are already at
the domain. So, just execute it in-place, but make it look for the 
client like you did the actual forwarding.

If client sent `connect` request with the url of forwarding server,
server should ignore it.

## Ping-Forwarding

When receiving `ping` request, forward it to all the underlying opened connections
to other servers to make sure connections with them are kept alive.
