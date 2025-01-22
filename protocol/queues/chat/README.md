# Chat Queues Specification

The specification describes `content` field after it is decrypted.

For now there is only a single type of message content:

```typescript
{
  type: "regular",
  title: string,
  text: string,
}
```

Later we will add more fields inside `content` for images, videos, music, etc.
