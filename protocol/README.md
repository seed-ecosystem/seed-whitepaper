# Protocol

Our protocol is divided into multiple separate parts. The core part is
called **Seed Queues**. This is what powers everything in our ecosystem.
They help securely distribute events between multiple clients without
server being able to read contents of the events.

**Seed Queues** is what server needs to implement. You can read more [here](queues/README.md).

There are, however, a bunch of **protocol extensions** which are 
implemented client-side (since server can't access required information):

- [Chat Queues](chat/README.md)
- [Sync Queues](sync/README.md)
