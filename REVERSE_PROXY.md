# Reverse Proxy & Proxy Server Notes

## 1. What Is a Proxy Server?

### Overview and definition

A **proxy server** is an intermediary device or server that sits between a user's client device and the internet (1). Instead of connecting directly to an external website or web service, the user's web traffic is routed through the proxy server, which forwards the request on the client's behalf. This setup manages traffic flow, can hide user identity, and adds network controls.

### How a proxy server works

1. **User request:** The user initiates a web request (for example, by entering a URL in a browser).
2. **Interception:** The request goes to the proxy server first rather than directly to the target server on the public internet.
3. **Forwarding and IP substitution:** The proxy forwards the request to the target web server and may mask or alter request details, such as the user's IP address.
4. **Response handling:** The target web server sends the requested data back to the proxy server.
5. **Validation and delivery:** The proxy can inspect the returned data for security compliance or content filtering before delivering it to the user.

### Key benefits and use cases

- **Privacy and anonymity:** Hides the client's true IP address from external websites, which can reduce tracking and mask location data.
- **Content filtering and access control:** Organizations and parents use proxies to monitor web traffic, inspect packet headers or payloads, and restrict access to harmful or distracting websites.
- **Caching and bandwidth savings:** Stores local copies of frequently requested web pages to accelerate browsing and reduce bandwidth consumption.
- **Security and encryption:** A proxy can be configured to encrypt outbound web requests and shield internal clients from direct exposure to internet threats.
- **Web scraping and geo-targeting:** Allows automated data collection and viewing of location-specific search results by rotating or assigning IP addresses across different regions.

## 2. Types of Proxy Servers

| Proxy type | Core function and characteristics |
| --- | --- |
| **Forward proxy** | Sits between internal users and the public internet to hide client identities, filter outgoing traffic, and enforce browsing policies. |
| **Reverse proxy** | Sits in front of internal backend servers to shield them, route incoming traffic, balance workloads, and accelerate web content delivery. |
| **Web proxy** | Specializes in handling HTTP/HTTPS URL-based requests, often through a web browser interface. |
| **Anonymous proxy** | Hides the client's real IP address, though target web servers can still identify that a proxy is being used. |
| **High-anonymity (elite) proxy** | Hides both the client's real IP address and the fact that a proxy server is being used, offering greater privacy. |
| **Transparent proxy** | Does not hide the user's IP address or offer anonymity; primarily used in schools and offices for caching, traffic logging, and filtering. |
| **CGI proxy** | Uses a web form interface to access websites without installing software; generally outdated and slow. |
| **Suffix proxy** | Appends a custom suffix to target URLs to bypass basic domain filters, with minimal anonymity. |
| **Distorting proxy** | Passes a fake IP address in request headers to protect the client's real IP while attempting to bypass geographic blocks. |
| **Tor onion proxy** | Routes web traffic through multiple encrypted network layers to provide hard-to-track user anonymity. |
| **I2P anonymous proxy** | Directs traffic through a decentralized, encrypted network for censorship resistance and privacy. |
| **DNS proxy** | Handles DNS queries on behalf of clients, caching responses to speed up domain lookups and block malicious domains. |
| **Rotating proxy** | Assigns a different IP address for every request or user connection; often used for automated web scraping. |

## 3. Deep Dive into Reverse Proxies

### What is a reverse proxy?

A **reverse proxy** (sometimes called a **surrogate server**) is a server positioned in front of one or more backend web servers. To external clients, the reverse proxy appears to be the destination web server itself. It acts as a traffic controller that receives incoming public requests, screens them, forwards them to the appropriate internal server, and sends the server's response back to the client.

While a **forward proxy acts on behalf of clients**, a **reverse proxy acts on behalf of origin web servers**.

### Step-by-step working mechanism

1. **Request interception:** The client sends an HTTP/HTTPS request over the internet. The request hits the reverse proxy's public IP address instead of reaching the backend server directly.
2. **Server IP masking:** The proxy conceals the real IP addresses and internal architecture of the backend origin servers behind a firewall.
3. **Request evaluation and routing:** The reverse proxy inspects request headers, target paths (such as `/api`), or server health status to select a backend server.
4. **Backend processing:** The proxy relays the request across the private internal network to the selected backend server, which processes the request and sends the response back to the proxy.
5. **Response inspection and delivery:** The proxy can check the response for errors or security issues before transmitting it back to the client.

### Core features and benefits

- **Load balancing:** Distributes incoming request traffic across multiple backend servers using algorithms such as round-robin or sticky sessions, helping prevent single-server overloads and maintain availability (6).
- **SSL/TLS termination:** Handles HTTPS encryption and decryption at the proxy edge (26). Internal services may communicate over HTTP behind the proxy boundary, centralizing certificate management and reducing encryption work on backend services (26)(28).
- **Caching and web acceleration:** Saves local copies of static files (such as images and stylesheets) and, where appropriate, dynamic content to serve repeated requests quickly without burdening origin servers (7).
- **URL rewriting and path routing:** Directs client requests based on URL subpaths (for example, routing `/api` to an API service and other paths to a frontend app) while serving everything under one domain (22).
- **Security and DDoS protection:** Can act as a Web Application Firewall (WAF) barrier, hide origin server IPs, and filter malicious traffic before it reaches internal networks. DDoS absorption depends on the proxy's capacity and deployment (10).
- **Compression and "spoon-feeding":** Compresses payloads to speed up page loads (7). With "spoon-feeding," a reverse proxy buffers dynamically generated pages and feeds them slowly to low-bandwidth clients, freeing backend application worker threads sooner (30).
- **Centralized authentication and A/B testing:** Can add access authentication controls to backend services that lack native login features (26)(31) and run A/B or multivariate testing without changing application code (31).

### Forward proxy vs. reverse proxy

| Feature | Forward proxy | Reverse proxy |
| --- | --- | --- |
| **Primary direction** | Manages **outbound** requests from internal clients to the internet (6). | Manages **inbound** requests from the public internet to internal servers (6). |
| **Entity protected/hidden** | Hides the **client's identity and IP address** (6). | Hides the **backend server's IP address and internal network topology** (9). |
| **Main objectives** | Anonymity, accessing geographically restricted content, and monitoring employee or student internet usage (3). | Load balancing, backend security, SSL termination, caching, and path routing (6). |
| **Control point** | Configured on the **client side** (browser or local network) (7)(9). | Deployed on the **server side** (infrastructure edge or CDN) (9)(10). |

### Common real-world use cases

- **High-traffic web applications:** Spreads heavy traffic across backend server clusters to prevent site crashes during peak load spikes (33)(34).
- **Multi-service microservices architecture:** Routes incoming traffic to separate microservices (such as authentication, payments, and static content) through a single unified IP address and domain (20).
- **Content Delivery Networks (CDNs):** Uses distributed reverse proxy nodes worldwide to serve cached static assets closer to users (7)(34).
- **Multi-domain hosting:** Hosts several websites or domain names on one physical server IP address by routing incoming HTTP host headers appropriately (30)(34).

### Popular reverse proxy tools

- **Nginx:** A lightweight, high-performance web server and reverse proxy known for managing many concurrent connections efficiently.
- **Apache:** A feature-rich web server that implements reverse proxying using modules such as `mod_proxy`.
- **HAProxy:** A high-speed TCP/HTTP load balancer and reverse proxy designed for high-availability environments.
- **Traefik:** A cloud-native reverse proxy with automatic container service discovery and dynamic routing, commonly used with Docker and Kubernetes.
- **Caddy:** A user-friendly, modern web server and proxy that can automatically provision and renew SSL/TLS certificates via Let's Encrypt.

### Security risks and trade-offs

- **Single point of failure:** Because traffic passes through the reverse proxy, an outage or misconfiguration at the proxy level can make backend applications inaccessible (37).
- **TLS private key exposure:** SSL/TLS termination requires certificates and private keys on the proxy, increasing the impact if that server is compromised (31)(38).
- **Expanded attack surface:** Exposing internal tools to external users through a reverse proxy can invite security threats if the internal applications were not hardened for public access (39).

## 4. Practical Implementation: Build a Reverse Proxy with Python

### Objective

Implement and test a reverse proxy that accepts client HTTP requests on a single port and forwards them to the appropriate backend service based on the URL path.

### Tools and technologies

- Python
- Flask — to run the proxy and backend HTTP servers
- Requests — to forward HTTP requests
- VS Code terminal
- Browser or Postman for testing

### Architecture

```text
                         Client / Browser
                                |
                                | HTTP request
                                v
                    Reverse Proxy (Port 8080)
                         /             \
             /api/users                 /api/products
                    |                         |
                    v                         v
             Backend A                   Backend B
             Port 5001                   Port 5002
```

The client sends requests only to the reverse proxy at `http://127.0.0.1:8080`. The proxy selects a backend and forwards the request. The backend response is then returned to the client through the proxy.

### Project structure

```text
reverse-proxy-project/
├── reverse_proxy.py
├── server.py
├── requirements.txt
└── README.md
```

### Step 1: Install dependencies

Create `requirements.txt`:

```text
Flask
requests
```

Create and activate a virtual environment on Windows PowerShell, then install the dependencies:

```powershell
python -m venv venv
.\venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

### Step 2: Create the backend application

Save the following as `server.py`. The same application is run twice with different environment variables so each instance identifies itself separately.

```python
from flask import Flask, jsonify
import os

app = Flask(__name__)

SERVER_NAME = os.getenv("SERVER_NAME", "Backend")
PORT = int(os.getenv("PORT", "5001"))


@app.get("/")
def home():
    return jsonify({
        "server": SERVER_NAME,
        "message": "Request reached the backend"
    })


@app.get("/api/users")
def users():
    return jsonify({
        "server": SERVER_NAME,
        "users": ["Kapil", "Rahul", "Aman"]
    })


@app.get("/api/products")
def products():
    return jsonify({
        "server": SERVER_NAME,
        "products": ["Laptop", "Phone", "Keyboard"]
    })


if __name__ == "__main__":
    app.run(host="127.0.0.1", port=PORT, debug=False)
```

### Step 3: Create the reverse proxy

Save the following as `reverse_proxy.py`:

```python
from flask import Flask, request, Response
import requests

app = Flask(__name__)

BACKENDS = {
    "/api/users": "http://127.0.0.1:5001",
    "/api/products": "http://127.0.0.1:5002",
}

DEFAULT_BACKEND = "http://127.0.0.1:5001"

HOP_BY_HOP_HEADERS = {
    "connection",
    "keep-alive",
    "proxy-authenticate",
    "proxy-authorization",
    "te",
    "trailers",
    "transfer-encoding",
    "upgrade",
}


def get_backend(path):
    for prefix, backend in BACKENDS.items():
        if path == prefix or path.startswith(prefix + "/"):
            return backend
    return DEFAULT_BACKEND


@app.route("/", defaults={"path": ""}, methods=[
    "GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"
])
@app.route("/<path:path>", methods=[
    "GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"
])
def proxy(path):
    backend = get_backend(request.path)
    target_url = backend + request.full_path

    if target_url.endswith("?"):
        target_url = target_url[:-1]

    headers = {
        key: value
        for key, value in request.headers.items()
        if key.lower() not in HOP_BY_HOP_HEADERS
        and key.lower() not in {"host", "content-length"}
    }

    try:
        upstream = requests.request(
            method=request.method,
            url=target_url,
            headers=headers,
            data=request.get_data(),
            cookies=request.cookies,
            allow_redirects=False,
            timeout=10,
        )

        response_headers = [
            (key, value)
            for key, value in upstream.headers.items()
            if key.lower() not in HOP_BY_HOP_HEADERS
            and key.lower() not in {"content-length", "content-encoding"}
        ]

        response = Response(
            upstream.content,
            status=upstream.status_code,
            headers=response_headers,
        )
        response.headers["X-Reverse-Proxy"] = "Python-Flask"
        return response

    except requests.RequestException:
        return {
            "error": "Backend unavailable",
            "message": "The reverse proxy could not reach the backend.",
        }, 502


if __name__ == "__main__":
    app.run(host="127.0.0.1", port=8080, debug=False)
```

### Step 4: Run the three processes

Open three separate VS Code terminals. Activate the virtual environment in each terminal.

**Terminal 1 — Backend A (port 5001):**

```powershell
$env:SERVER_NAME="Backend A"
$env:PORT="5001"
python server.py
```

**Terminal 2 — Backend B (port 5002):**

```powershell
$env:SERVER_NAME="Backend B"
$env:PORT="5002"
python server.py
```

**Terminal 3 — Reverse proxy (port 8080):**

```powershell
python reverse_proxy.py
```

### Step 5: Test the routes

Send these requests in a browser or Postman:

| Request URL | Expected destination | Expected result |
| --- | --- | --- |
| `http://127.0.0.1:8080/` | Backend A | Backend A response |
| `http://127.0.0.1:8080/api/users` | Backend A | JSON containing the users list and `"server": "Backend A"` |
| `http://127.0.0.1:8080/api/products` | Backend B | JSON containing the products list and `"server": "Backend B"` |
| `http://127.0.0.1:8080/unknown` | Backend A (default) | Backend A returns a 404 response |
| `http://127.0.0.1:8080/api/products` while Backend B is stopped | Backend B, unavailable | Proxy returns HTTP 502 Bad Gateway |

Successful proxied responses include the `X-Reverse-Proxy: Python-Flask` response header.

### How the implementation works

1. The client sends a request to port 8080.
2. Flask receives the request at the proxy.
3. `get_backend()` chooses a backend using the URL path.
4. `requests.request()` forwards the method, headers, body, and query string to that backend.
5. The proxy creates a response using the backend's response body and status code.
6. If the backend cannot be reached before the timeout, the proxy returns a 502 Bad Gateway response.

### Important distinction: reverse proxy vs. load balancer

This implementation performs path-based routing, not load balancing. The users endpoint always maps to Backend A and the products endpoint always maps to Backend B. It does not distribute requests between multiple equivalent instances of the same service, and it does not automatically fail over when a backend is unavailable.

### Limitations and production considerations

- This is a local learning implementation, not a production-ready proxy.
- Production deployments need careful handling of hop-by-hop headers, client IP forwarding, streaming, connection management, request size limits, logging, and security.
- The proxy and backend applications should be protected appropriately before being exposed to a network.
- A single proxy process can become a single point of failure.
- HTTPS/TLS termination, caching, authentication, load balancing, health checks, and retry policies are not implemented here.

### Result

A basic Python/Flask reverse proxy was implemented with one entry point on port 8080 and two backend instances on ports 5001 and 5002. URL-path routing, response forwarding, default routing, and the backend-unavailable 502 case were defined for testing.

### Conclusion

A reverse proxy provides a single entry point for client requests and forwards them to internal backend services. This practical demonstrates path-based routing and response forwarding. The next extension can add round-robin load balancing, backend health checks, and failover.