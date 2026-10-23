# Polling, Long Polling, and Server-Sent Events (SSE)
## Beginner-Friendly Revision Notes

## 1. Why Do We Need These Communication Methods?

Many applications need to show new information without making the user refresh the page.

Examples:
- A chat app receives a new message.
- A dashboard displays updated statistics.
- A delivery app shows an order's progress.
- A website displays notifications.
- An AI assistant shows its answer as it is being generated.

The browser (client) and the backend (server) need a way to exchange information. Polling, Long Polling, and Server-Sent Events are three ways to deliver updates from a server to a client.

**Client:** Usually the browser or mobile app making a request.

**Server:** The backend that processes requests and sends responses.

---

## 2. Polling (Short Polling)

### What is polling?

Polling is a communication technique in which the client sends requests to the server at regular intervals to ask whether new information is available.

For example, a client might ask the server every five seconds: “Is there anything new?”

### How it works

1. The client sends an HTTP request to the server.
2. The server checks whether new information is available.
3. The server returns a response, even if there is no new information.
4. The client waits for the chosen interval.
5. The client sends another request.
6. This cycle continues while updates are needed.

### Real-world example: checking order status

Imagine that you order food through an app.

1. The app asks the backend for the order status.
2. The backend responds: “Preparing.”
3. A few seconds later, the app asks again.
4. The backend responds: “On the way.”
5. The app displays the updated status.

The app repeatedly checks for changes instead of receiving them automatically.

### Advantages

- Simple to understand and implement.
- Works with ordinary HTTP requests and responses.
- Useful when updates do not need to appear immediately.
- Often suitable for low-frequency status checks.

### Limitations

- Many requests may return without new information.
- Frequent requests create extra network traffic and server work.
- Updates may be delayed until the next scheduled request.
- A short interval can increase server load; a long interval can make updates feel slow.

### When is polling useful?

Use it when updates are infrequent, a small delay is acceptable, and a simple request-response approach is enough.

---

## 3. Long Polling

### What is long polling?

Long polling is similar to ordinary polling, but the server does not respond immediately when there is no new information. Instead, it keeps the HTTP request open and waits for an update or a timeout.

When the server responds, the client sends another request to wait for the next update.

### How it works

1. The client sends an HTTP request asking for new information.
2. If new information already exists, the server returns it.
3. If no new information exists, the server waits while keeping the request open.
4. When new information arrives—or the waiting period ends—the server sends a response.
5. The client processes the response and immediately sends another request.
6. The process repeats.

### Real-world example: a chat application

Imagine that you have a chat window open.

1. Your browser sends a request asking for new messages.
2. The backend keeps the request waiting because there are no new messages.
3. Your friend sends you a message.
4. The backend makes the new message available to your waiting request.
5. The server responds, and your browser displays the message.
6. The browser starts another long-polling request.

This can make messages appear quickly without the browser checking every second.

### Polling vs. long polling

With **ordinary polling**, the server responds immediately, even when nothing has changed.

With **long polling**, the server waits for new information or a timeout before responding.

Long polling reduces repeated empty responses, but it still uses a sequence of separate HTTP requests. It is not one permanent connection.

### Advantages

- Can deliver updates sooner than ordinary polling.
- Avoids many empty responses when updates are infrequent.
- Uses familiar HTTP request-response infrastructure.
- Can be useful when a persistent streaming connection is unavailable or unsuitable.

### Limitations

- The client must send a new request after each response.
- Many waiting requests require careful server resource management.
- Timeouts, proxies, and load balancers can interrupt waiting requests.
- Reconnection and duplicate-message handling need attention.
- It is usually less suitable than a persistent stream for frequent, continuous updates.

### Important implementation considerations

- Set compatible timeouts across the application, reverse proxy, and load balancer.
- Handle network failures and reconnect safely.
- Use message IDs or cursors if the application must detect missed or repeated updates.
- For multiple backend instances, make sure each instance can access the relevant events, often through shared storage or a message broker.

### Where is it used?

Long polling can be used for chat-like updates, notifications, and applications that need near-real-time updates using HTTP request-response cycles. Some real-time libraries, including Socket.IO, can use HTTP long polling as a transport or fallback depending on configuration.

---

## 4. Server-Sent Events (SSE)

### What is SSE?

Server-Sent Events (SSE) is a browser technology that lets a server send a continuous stream of events to a client over a single HTTP response.

Instead of ending the response after one update, the server keeps the response open and can send multiple events as they happen.

SSE is **one-way by design**: the server sends updates to the client. The client can still send information to the server through separate HTTP requests.

### How it works

1. The browser opens an HTTP connection to an SSE endpoint.
2. The server responds with the `text/event-stream` content type.
3. The connection remains open.
4. Whenever an event is ready, the server writes it into the response stream.
5. The browser receives the event and updates the interface.
6. The same connection can carry more events until it closes or fails.

In browsers, JavaScript commonly uses the `EventSource` API to receive SSE messages.

### Real-world example: streaming an AI answer

Imagine asking an AI assistant a question.

1. The browser sends your prompt to the backend.
2. The backend begins receiving generated text from an AI model.
3. Rather than waiting for the entire answer, the backend streams pieces of the answer to the browser.
4. The browser displays each piece as it arrives.
5. The stream ends when generation is complete.

This makes the answer appear progressively instead of all at once. SSE is one common way to implement streaming AI responses.

### Other real-world uses

- Live dashboards and monitoring pages.
- Progress updates for long-running jobs.
- Notifications and activity feeds.
- Live logs from a running process.
- Status updates for uploads, reports, or data processing.

### Advantages

- Multiple updates can be sent over one open HTTP response.
- Avoids repeatedly creating a new request for every event.
- Supports event IDs and browser reconnection behavior.
- Well suited to one-way, server-to-client updates.
- Often simpler than WebSockets when the client does not need a continuous two-way channel.

### Limitations

- SSE is not a two-way communication channel; client-to-server messages need separate requests.
- Network interruptions can close the stream.
- Reconnection alone does not guarantee that no events will be missed.
- Proxies may buffer streamed output unless configured appropriately.
- Many simultaneous connections need efficient server and infrastructure handling.
- The browser's native `EventSource` API does not support arbitrary custom request headers in the same way as a general HTTP client; authentication design needs to account for this.

### Important implementation considerations

- Send the correct `text/event-stream` content type.
- Ensure proxies and hosting infrastructure do not buffer events in a way that defeats streaming.
- Use event IDs and server-side event history when clients need to resume after reconnecting.
- Design authentication and authorization carefully.
- Use heartbeats or suitable keep-alive behavior where necessary to help detect broken connections.
- Consider HTTP/2 and your hosting environment when supporting many concurrent browser connections.

### Does SSE automatically prevent missed events?

No. Browsers can reconnect automatically, and SSE supports event IDs, but the server must retain or retrieve the relevant events if it needs to replay updates. If the server does not support replay, an interruption may still cause an event to be missed.

---

## 5. Quick Comparison

| Feature | Polling | Long Polling | SSE |
|---|---|---|---|
| Who initiates each update check? | Client | Client | Client opens the stream; server sends events |
| Server response | Usually immediate | Waits for an event or timeout | Stream stays open for multiple events |
| Separate request for each update cycle? | Yes | Yes | No, not normally |
| Direction | Client requests; server responds | Client requests; server responds | Server-to-client stream |
| Near-real-time updates | Depends on polling interval | Often good for infrequent updates | Good for ongoing one-way updates |
| Empty responses | Common when nothing changes | Reduced | Usually avoided during an active stream |
| Two-way communication on the same channel | No | No | No |
| Common fit | Infrequent status checks | HTTP-based near-real-time updates | Notifications, live feeds, streaming output |

---

## 6. How to Choose

### Choose ordinary polling when:
- Updates are infrequent.
- A delay is acceptable.
- Simplicity is more important than immediate delivery.

### Choose long polling when:
- Updates should arrive soon after they become available.
- The application uses HTTP request-response cycles.
- A new request after each event is acceptable.

### Choose SSE when:
- The server needs to send repeated updates to the browser.
- The client mostly receives information rather than sending continuous messages.
- A continuously updating feed or streamed response is needed.

### What if the application needs two-way real-time communication?

Consider **WebSockets** when both client and server need to send messages over the same persistent connection—for example, interactive multiplayer games or collaborative applications. The best choice depends on the communication pattern and infrastructure, not simply on which technology is newest.

---

## 7. Common Mistakes to Avoid

1. **Thinking polling is the same as long polling.** Ordinary polling usually gets an immediate response; long polling waits for an event or timeout.
2. **Thinking long polling keeps one connection forever.** Each response ends a request cycle, and the client opens another request.
3. **Thinking SSE is two-way.** SSE streams from server to client; client messages use another request.
4. **Assuming automatic reconnection guarantees delivery.** Reliable recovery requires suitable event IDs and server-side replay or equivalent logic.
5. **Ignoring infrastructure timeouts and buffering.** Proxies and load balancers can affect long polling and SSE.
6. **Choosing based only on trendiness.** Match the technique to the update frequency, direction of communication, reliability requirements, and server capacity.

---

## 8. Quick Revision

- **Polling:** The client asks repeatedly, “Anything new?”
- **Long polling:** The client asks, and the server waits before answering if nothing is new.
- **SSE:** The client opens a stream, and the server sends multiple events through it.
- **Polling and long polling:** Use repeated HTTP request-response cycles.
- **SSE:** Uses one continuing HTTP response for a sequence of server-to-client events.
- **SSE is one-way; WebSockets support two-way communication over a persistent connection.**
- **Near-real-time does not automatically mean guaranteed delivery.** Reliable systems need suitable reconnection, IDs, replay, and error handling.

### Short Answers

1. Polling usually receives an immediate response; long polling waits for new information or a timeout.
2. Many requests may return when there is no new information.
3. The client generally sends another request to wait for the next update.
4. It lets a server stream multiple events over one continuing HTTP response.
5. No. SSE is server-to-client; client-to-server communication uses separate requests.
6. To help recover updates after a connection interruption.
7. SSE can be a good fit when updates are one-way from server to client.
8. Ordinary polling can be sufficient when a delay is acceptable.
