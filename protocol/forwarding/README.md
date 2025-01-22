# Forwarding

feature-key: `forwarding`

---

Server also may be used as a proxy-server to access other servers. This helps us reduce
WebSocket connections amount, and makes chatting more efficient.

Forward request looks like this:

```
{
  "type": "forward", 
  "url": "https://meetacy.app/seed-kt", 
  "request": {...}
}
```

Server response would be something like this:

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

> [!WARNING]
> API like this makes it possible for forward server to read all the traffic.
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
