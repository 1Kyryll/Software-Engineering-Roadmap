---
tags: [system-design, architecture, infrastructure]
---

# Reverse Proxy with Nginx

A reverse proxy sits in front of backend servers and forwards client requests to them. Clients only ever see the proxy. Uses: single entry point/port, path-based routing to multiple upstreams, TLS termination, buffering slow clients, serving static files, load balancing.

## Where I built it

### Lychee-Chat — routing three upstreams through one port
[`nginx/nginx.conf`](https://github.com/1Kyryll/Lychee-Chat/blob/main/nginx/nginx.conf) exposes everything on `:8080` and routes by path:

```nginx
location / {              # Next.js frontend
    proxy_pass http://web:3000;
    proxy_set_header Upgrade $http_upgrade;      # allow WS upgrade for HMR etc.
    proxy_set_header Connection 'upgrade';
}
location /socket.io/ {    # Socket.IO WebSocket server
    proxy_pass http://ws:4000/socket.io/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;      # WebSocket upgrade handshake
    proxy_set_header Connection "upgrade";
}
```

Two details worth being able to explain:
- **WebSocket proxying needs explicit config.** WS starts as an HTTP request with `Upgrade: websocket` / `Connection: upgrade` headers, and Nginx doesn't forward hop-by-hop headers by default — hence `proxy_http_version 1.1` + the two `proxy_set_header` lines. Without them the Socket.IO handshake falls back to polling or fails.
- **`X-Real-IP` / `X-Forwarded-For`** — the backend otherwise sees every request coming from the proxy's IP. These headers preserve the real client address for logging/rate limiting.

### Personal-Finance-Tracker — classic SPA + API split
[`nginx/nginx.conf`](https://github.com/1Kyryll/Personal-Finance-Tracker/blob/main/nginx/nginx.conf): Nginx serves the built React static files directly and proxies `/api/*` to Flask. This is the standard monolith deployment shape — static assets never touch the app server.

## Interview talking points
- **Reverse vs forward proxy**: reverse proxy hides *servers* from clients; forward proxy hides *clients* from servers.
- **Why not expose the app server directly?** TLS termination in one place, connection buffering (Nginx absorbs slow clients so app workers aren't held hostage), static file serving without hitting Node/Python, one origin → no CORS issues between the SPA and API.
- **Load balancing**: Nginx `upstream` blocks give round-robin/least-conn across replicas — the natural next step when a service needs horizontal scaling (listed in my E-Commerce TODO).
- Docker Compose service names (`web`, `ws`) resolve via the internal DNS of the compose network — that's why `proxy_pass http://ws:4000` works. See [[Docker Compose Orchestration]].

## Related
- [[API Gateway Pattern]] — the smarter, API-aware sibling
- [[Real-Time Communication]] — why WS upgrade headers matter
- [[Architecture Index]]
