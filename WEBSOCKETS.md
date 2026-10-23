# WebSockets — Revision Notes

## 1. What is a WebSocket?
**WebSocket** is a communication protocol that provides a persistent, bidirectional connection between a client and a server. After the connection is established, either side can send messages at any time without opening a new HTTP request for every message.

- `ws://` — WebSocket without TLS encryption.
- `wss://` — WebSocket secured with TLS.

## 2. Why use WebSockets?
With ordinary HTTP request-response, a client generally sends a request and receives a response. To learn about new information later, it needs another request or another update mechanism.

WebSockets let the server push an update over an existing connection without waiting for a new client request.

Example: a chat application can deliver a new message to a connected user as soon as the server receives it.

## 3. How the connection is established
For the traditional WebSocket handshake over HTTP/1.1:
1. The client initiates a WebSocket handshake request.
2. The server accepts the protocol upgrade with HTTP status `101 Switching Protocols`.
3. The connection then carries WebSocket frames in both directions.
4. Either side can initiate the closing handshake when finished.

WebSockets can also be used over other supported HTTP versions with different handshake details, so `101` specifically describes the traditional HTTP/1.1 upgrade.

## 4. HTTP vs. WebSockets

| Feature | Ordinary HTTP request-response | WebSocket |
|---|---|---|
| Communication model | Client requests, server responds | Bidirectional messages |
| Server-initiated updates | Requires another mechanism, such as SSE or long polling | Can be sent over the established connection |
| Repeated updates | May require repeated requests | Reuses the open connection |
| Common use | REST APIs, fetching resources | Chat, interactive live updates, multiplayer interactions |

HTTP connections can themselves be reused, so the distinction is not simply that HTTP always opens a new TCP connection. The key difference is the messaging model.

WebSockets are not the only real-time option. Server-Sent Events (SSE) and long polling can also support live updates in suitable use cases.

## 5. Common use cases
- Chat messages, typing indicators, and presence
- Multiplayer games and collaborative tools
- Live dashboards and monitoring
- Real-time notifications
- Interactive applications that need frequent two-way updates

WebSockets provide the communication channel; they do not automatically implement chat rooms, user identity, message persistence, delivery guarantees, or access control.

## 6. System-design challenges

### Connection scaling
Many clients may maintain open connections. Servers need enough memory, file descriptors, and network capacity to handle them.

### Multiple backend instances
If users connected to different servers need to exchange messages, the servers may need a shared message broker or another routing mechanism. Redis Pub/Sub, Kafka, or other systems may be appropriate depending on delivery and persistence requirements; they are not interchangeable in every scenario.

### Reconnection
Networks fail and devices sleep. Clients need a reconnection strategy, and applications may need to resume from a known message position.

### Authentication and authorization
Authenticate connections and verify which rooms, channels, or messages each user may access. Secure production connections with `wss://`.

### Heartbeats and timeouts
Ping/pong or application-level heartbeats can help detect connections that are no longer responsive.

### Backpressure
If a client consumes messages slowly, the server needs a strategy for limiting queued data or dropping, buffering, or throttling updates.

## 7. Delivery is not automatically guaranteed
A persistent WebSocket connection does not guarantee that every message is durably delivered, especially across disconnections or process crashes.

If messages must survive disconnects, the application may need:
- Durable message storage
- Message IDs or sequence numbers
- Acknowledgements
- Retry and deduplication logic
- Reconnection and replay support

## 8. Quick revision
**Remember:** WebSockets provide persistent, bidirectional communication. They are useful when either client or server needs to send frequent updates, but reliability, security, scaling, and message persistence remain application responsibilities.

## 9. Self-test
1. What does bidirectional communication mean?
2. What happens during a traditional HTTP/1.1 WebSocket upgrade?
3. Why might a multi-instance WebSocket system need a shared message broker?
4. Why doesn't an open WebSocket guarantee durable message delivery?
5. When might SSE or long polling be a suitable alternative?
