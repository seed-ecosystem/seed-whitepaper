# Seed Queues

**Seed Queues** is the core part of our application. They provide
a secure way to send events between clients. However, those queues
are centralized (Not P2P, nor decentralized) and this is done for various of reasons:

- We need to work with queues even if no one is online
- TODO (I don't exactly remember all the reasons, but we considered 
  such option first, and it didn't fit well)

## Unidirectional

**Seed Queues** are unidirectional. Which means that by design there
is no way to retrieve updates before key-exchange point. This is powered by
changing keys every single message using [one-way KDF](https://en.wikipedia.org/wiki/Key_derivation_function).
So even having compromised server guarantees that no earlier events will be
retrievable.

Practically that means that you can share chat key with someone, and they won't
be able to scan all chat history since for them the chat history would be empty.

## More than just chats

\*Akshually\*, you can use **Seed Queues** for any kind of event-driven
systems and not just chats. We also incorporate this protocol for
accountless synchronization between multiple devices.

## Specification

Specification for queues is described [here](SPECIFICATION.md).
