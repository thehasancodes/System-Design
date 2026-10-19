## 1. Core Architecture & Traffic Flow

A **load balancer** is a dedicated networking device or software application that acts as a central **traffic controller** between incoming client devices and backend application server pools [1]. Its primary objective is to distribute network or application traffic across multiple servers, ensuring no single server becomes overwhelmed while optimizing overall application performance, reliability, and availability.

        Clients / Browsers 
            │ 
            ▼ 
     ┌──────────────┐ 
     │ Load Balancer│ ◄─── Health Checks &amp; Routing Algorithms 
     └──────┬───────┘ 
            ├─────────────────┬─────────────────┐ 
            ▼                 ▼                 ▼ 
      ┌───────────┐     ┌───────────┐      ┌───────────┐ 
      │ Server 1  │     │ Server 2  │      │ Server 3  │ 
      └───────────┘     └───────────┘      └───────────┘

### Request Lifecycle

1. **Client request**: A client browser sends an HTTP/HTTPS or TCP request to access a web service.
2. **Load balancer interception**: The request lands on the load balancer's public entry point rather than reaching an individual backend server directly.
3. **Health and routing evaluation**: The load balancer evaluates backend health and applies its routing algorithm to select an eligible instance.
4. **Backend forwarding**: The request is forwarded to the selected healthy backend server.
5. **Execution and response**: The backend processes the request and returns a response through the load balancer to the client.

## 2. Systems With vs. Without Load Balancing

| Architectural metric | Without load balancer | With load balancer |
| --- | --- | --- |
| **Traffic distribution** | All client traffic reaches a single backend application server. | Traffic is distributed across a pool of backend instances. |
| **System reliability** | A **single point of failure (SPOF)**: if the server crashes, the application becomes unavailable. | Health checks allow traffic to bypass unhealthy instances, improving availability and fault tolerance. |
| **Scalability** | Limited to **vertical scaling** (adding CPU or RAM to one machine), which eventually reaches hardware limits [7]. | Supports **horizontal scaling** by adding or removing servers from the pool. |
| **Resource utilization** | Uneven load can cause server overload, high latency, and timeouts. | Requests are spread across available instances for more balanced resource utilization. |

                     ┌─────────────────────────────────────┐ 
                     │ Real-World Load Balancing Use Cases │ 
                     └──────────────────┬──────────────────┘ 
               ┌────────────────────────┼────────────────────────┐
               ▼                        ▼                        ▼ 
        Restaurant Kitchen      E-Commerce Flash Sales      Banking & ATM Infrastructure 
        (Head Chef at Pass)       (Dynamic Failover)           (App Tier vs Database)

- **Restaurant kitchen analogy**: In a busy kitchen, a head chef at the pass routes incoming orders to available line chefs based on their workload. A load balancer similarly directs requests to available servers to help keep processing fast and smooth.
- **E-commerce flash sales**: During a high-demand sale, a load balancer distributes search, cart, and checkout requests across application servers. Health monitoring can redirect traffic away from a failed server.
- **ATM transaction networks**: Payment switches can send ATM requests to backend application servers through a load balancer. The load balancer manages application-tier traffic, while backend database clusters enforce transaction consistency and concurrency controls.

## 3. Classification Models: Deployment & OSI Layers

### A. Classification by Deployment Model

| Deployment model | Description | Common use |
| --- | --- | --- |
| **Hardware-based** | A dedicated appliance designed to process high volumes of network traffic. | Data centers that need predictable performance and specialized features. |
| **Software-based** | A load-balancing application running on standard servers or virtual machines. | Private infrastructure that needs flexible configuration and scaling. |
| **Cloud-based** | A managed service provided by a cloud platform, typically integrated with its networking and scaling features. | Applications hosted in public cloud environments. |

### B. Classification by OSI Network Layer

| Layer | Also known as | How it routes traffic | Typical use |
| --- | --- | --- | --- |
| **Layer 4** | Transport layer | Routes TCP or UDP connections using information such as IP addresses and ports, without inspecting application content. | High-throughput routing with low processing overhead. |
| **Layer 7** | Application layer | Routes requests using application data such as HTTP paths, hostnames, headers, or cookies. | Content-based routing, TLS termination, and application-aware policies. |

## 4. In-Depth Breakdown of Routing Algorithms

Routing algorithms are commonly grouped by whether they use a fixed allocation pattern or respond to current server conditions:

```text
                         Routing Algorithm Types
                                     |
           +-------------------------+-------------------------+
           |                                                   |
           v                                                   v
  Static Algorithms                                   Dynamic Algorithms
  Fixed allocation patterns                           Current health and metrics
   - Round Robin                                        - Least Connections
   - Weighted Round Robin                               - Least Response Time
   - IP / URL Hashing
   - Session persistence (sticky sessions)
```

### Static Algorithms (Fixed Allocation Patterns)

These algorithms follow configured rules rather than continuously choosing a server based on its current workload.

- **Round Robin**: Sends each new request to the next server in sequence. It works well when servers have similar capacity and requests have similar cost.
- **Weighted Round Robin**: Assigns more requests to servers with higher configured weights, making it useful when backend capacity differs.
- **IP / URL Hashing**: Uses a client IP address or request URL to select a server. This can help keep related traffic on the same server, although server membership changes may alter the mapping.
- **Session persistence (sticky sessions)**: Keeps a client associated with the same backend for a period of time, often using a cookie or client identifier. Persistence can be combined with a distribution algorithm; it is not itself a round-robin variant.

### Dynamic Algorithms (Real-Time Performance Metrics)

These algorithms use current or recent backend measurements to make routing decisions.

- **Least Connections**: Selects the server with the fewest active connections. This can help when requests take different amounts of time to complete.
- **Least Response Time**: Prefers servers with low observed response times, often considering active connections as well. The measurement window and health of the monitoring data affect how well it performs.

## 5. Health Monitoring, Metrics & Automatic Failover

Load balancers monitor backend instances so they can stop routing traffic to servers that are unavailable or unhealthy.

```text
                      Health Monitoring Mechanisms
                                   |
         +-------------------------+-------------------------+
         |                         |                         |
         v                         v                         v
 Active health checks       Passive health checks      Heartbeat monitoring
 Periodic HTTP/TCP probes   Observes live requests     Periodic status signals
```

- **Active health checks**: Send periodic probes, such as an HTTP request to a health endpoint or a TCP connection attempt. A server that fails configured checks is marked unhealthy.
- **Passive health checks**: Infer health from real traffic, such as repeated connection failures or error responses. This can reveal problems between active probes.
- **Heartbeat monitoring**: Receives periodic status signals from backend instances or monitoring agents. Missing heartbeats can indicate that an instance is unavailable.

When a backend fails its health criteria, the load balancer removes it from the eligible pool and routes new traffic to healthy instances. It can add the backend back after it passes recovery checks. Detection thresholds and probe intervals should balance quick failover against false alarms.

## 6. Risk Analysis & Mitigation Strategies

| Risk / challenge | Potential impact | Recommended mitigation |
| --- | --- | --- |
| **Load balancer as a single point of failure (SPOF)** | If the only load balancer fails, clients may lose access to the application even when backend servers are healthy. | Deploy redundant load balancers in an active-passive or active-active configuration, with health-based failover such as a floating IP or DNS failover. |
| **Performance bottlenecks** | A traffic spike can exceed the load balancer's throughput, increasing latency or causing requests to fail. | Size and monitor capacity, scale out where supported, and use Layer 4 routing for suitable high-throughput workloads. |
| **Security targeting** | As a public entry point, the load balancer may be targeted by denial-of-service attacks or malicious requests. | Apply rate limits, integrate DDoS protection and a Web Application Firewall (WAF) where appropriate, and keep configurations and software up to date. |

## 7. Practical Implementation: Build a Load Balancer Locally

The concepts above can be tested by running multiple backend application servers and sending requests through one entry point. This lab uses Python and Flask to create three small backend servers and a simple Round Robin load balancer.

> **Learning goal:** Observe request distribution, health checks, and basic failover on your own computer. This is a learning prototype, not a production-ready load balancer.

### 7.1 Architecture

```text
                         Client / Browser
                                |
                                | HTTP requests
                                v
                     Load Balancer :8080
                     Round Robin + health checks
                         /       |       \
                        /        |        \
                       v         v         v
                 Server 1    Server 2    Server 3
                 :5001       :5002       :5003
```

- **Client**: Sends requests to `http://127.0.0.1:8080/`.
- **Load balancer**: Chooses a healthy backend server.
- **Backend servers**: Each responds with its port number so that the routing decision is visible.
- **Health endpoint**: Each backend exposes `/health`, which the load balancer checks before forwarding a request.

### 7.2 Prerequisites

- Python 3 installed.
- Visual Studio Code or another code editor.
- A terminal. The commands below include Windows PowerShell commands.

Create a project folder with these files:

```text
load-balancer-project/
├── server.py
├── load_balancer.py
└── requirements.txt
```

### 7.3 Install the Dependencies

Add the following to `requirements.txt`:

```text
Flask
requests
```

Open a terminal in the project folder and run:

```powershell
python -m venv venv
.\venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

If PowerShell blocks virtual-environment activation, you can use Command Prompt instead:

```bat
venv\Scripts\activate.bat
```

### 7.4 Create the Backend Server

Save this code as `server.py`:

```python
from flask import Flask, jsonify
import sys

app = Flask(__name__)
PORT = int(sys.argv[1])


@app.route("/")
def home():
    return jsonify({
        "message": "Request processed successfully",
        "server": f"Server running on port {PORT}"
    })


@app.route("/health")
def health():
    return jsonify({"status": "healthy"}), 200


if __name__ == "__main__":
    app.run(host="127.0.0.1", port=PORT, debug=False, threaded=True)
```

This is a single server program that can be started several times with different port numbers. Each running process acts as a separate backend instance.

### 7.5 Start Three Backend Instances

Open three separate terminals in the project folder. Activate the virtual environment in each terminal if needed, then run one command per terminal:

**Terminal 1**

```powershell
python server.py 5001
```

**Terminal 2**

```powershell
python server.py 5002
```

**Terminal 3**

```powershell
python server.py 5003
```

Test a backend directly by opening `http://127.0.0.1:5001/` in a browser. The response should identify the server running on port `5001`. Repeat with ports `5002` and `5003`.

### 7.6 Implement a Simple Round Robin Load Balancer

Save this code as `load_balancer.py`:

```python
from flask import Flask, jsonify
import requests
from itertools import count
from threading import Lock

app = Flask(__name__)

SERVERS = [
    "http://127.0.0.1:5001",
    "http://127.0.0.1:5002",
    "http://127.0.0.1:5003",
]

counter = count()
counter_lock = Lock()


def choose_healthy_server():
    # Reserve a starting position in the Round Robin sequence.
    with counter_lock:
        start = next(counter) % len(SERVERS)

    # Try each backend at most once, beginning at that position.
    for offset in range(len(SERVERS)):
        index = (start + offset) % len(SERVERS)
        server = SERVERS[index]

        try:
            response = requests.get(server + "/health", timeout=1)
            if response.ok:
                return server
        except requests.RequestException:
            continue

    return None


@app.route("/")
def home():
    server = choose_healthy_server()

    if server is None:
        return jsonify({
            "error": "No healthy backend servers available"
        }), 503

    try:
        response = requests.get(server + "/", timeout=3)
        return (
            response.content,
            response.status_code,
            {
                "Content-Type": response.headers.get(
                    "Content-Type", "application/json"
                )
            },
        )
    except requests.RequestException:
        return jsonify({"error": "Backend request failed"}), 502


@app.route("/health")
def load_balancer_health():
    return jsonify({"status": "load balancer running"}), 200


if __name__ == "__main__":
    app.run(host="127.0.0.1", port=8080, debug=False, threaded=True)
```

### 7.7 Start the Load Balancer

Keep the three backend processes running. Open a fourth terminal and run:

```powershell
python load_balancer.py
```

Visit `http://127.0.0.1:8080/`. This is the load balancer's entry point; clients do not need to choose a backend port themselves.

To send several requests from another terminal, run:

```powershell
python -c "import requests; [print(requests.get('http://127.0.0.1:8080/').json()) for _ in range(9)]"
```

The output should show requests being routed among ports `5001`, `5002`, and `5003`. The exact first server may vary, but with all servers healthy the selection should rotate through the list.

### 7.8 Test Failover

1. Leave the load balancer running.
2. Stop the backend on port `5002` with `Ctrl+C` in its terminal.
3. Send more requests to `http://127.0.0.1:8080/`.
4. Confirm that requests continue through the remaining healthy backends.
5. Restart the stopped backend using `python server.py 5002` and test again.

Because the prototype checks `/health` before forwarding, it should skip a stopped backend. Once that backend is running and its health endpoint responds successfully, it can receive requests again.

### 7.9 What This Practical Demonstrates

| Concept | What you observe |
| --- | --- |
| **Reverse proxy / single entry point** | Clients send requests to port `8080`, not directly to each backend. |
| **Round Robin** | Backend selection rotates through the configured server list. |
| **Health checking** | The load balancer requests `/health` on each candidate backend. |
| **Basic failover** | A stopped backend is skipped if another healthy backend is available. |
| **Horizontal scaling concept** | Another backend can be added by starting it on a new port and adding its URL to `SERVERS`. |

### 7.10 Limitations and Production Considerations

This implementation is intentionally small to make the concepts easy to inspect. It has important limitations:

- It checks health for every incoming request rather than using a separate periodic health-monitoring loop.
- A backend can fail after its health check but before the actual request completes.
- It forwards only the `/` route and does not preserve all original request headers, methods, query strings, or response headers.
- Its Round Robin counter is shared only within this one Python process; multiple load-balancer processes would not share the counter.
- The load balancer itself is a single point of failure in this lab.
- It does not implement TLS termination, rate limiting, authentication, distributed tracing, or production-grade retry policies.

For a more realistic next step, configure **Nginx** as a reverse proxy in front of the same backend servers. Nginx supports upstream server pools and health/failure handling options and is closer to a commonly deployed production component.

### 7.11 Practical Exercises

Try these exercises after the basic lab works:

1. **Change the number of backends**: Add a fourth server on port `5004` and include it in `SERVERS`.
2. **Observe failure**: Stop one backend and record which servers receive the next requests.
3. **Compare algorithms**: Replace Round Robin with a Least Connections strategy, then explain why this simple Flask prototype does not yet track active connections accurately.
4. **Measure response time**: Send repeated requests and record the response times. Compare results with one backend versus three.
5. **Explore the load-balancer SPOF**: Explain why the backend pool can remain healthy while the whole application becomes unreachable if the only load balancer stops.

### Practical Conclusion

This lab demonstrates the basic flow of a load-balanced web application: a client sends a request to one entry point, the load balancer selects a backend, and the backend returns a response. By stopping a backend and observing the remaining servers, you can see how health checks and basic failover improve availability. The prototype is intended for learning; production systems require stronger monitoring, request forwarding, security, capacity planning, and redundancy.