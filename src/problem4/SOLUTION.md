# SOLUTION 3

## 1. What Problems You Found
The platform was experiencing a **502 Bad Gateway** error when accessing the `/api/users` endpoint, even though the home page (static content) was functioning correctly. Upon investigation, a configuration mismatch was identified between the service layers:
* **Application:** Hardcoded to run on port `3000` (index.js, line 35).
* **Nginx (Reverse Proxy):** Configured with `proxy_pass` pointing to port `3001` in `default.conf`.
* **Dockerfile:** Missing the `EXPOSE` instruction and running as `root` (security risk).

---

## 2. How You Diagnosed Them
I followed a layer-by-layer diagnostic process:
1.  **Reproduction:** Executed `docker compose up --build`. The build succeeded, and `localhost:8080` loaded "Welcome to the platform," but `/api/users` failed with a 502 error.
2.  **Log Analysis:** Inspected Nginx logs, confirming an upstream communication error:
    `172.18.0.1 - ... "GET /api/users HTTP/1.1" 502 ...`
3.  **Code Review:** Checked `index.js` and confirmed the application forces port `3000`.
4.  **Configuration Audit:** Reviewed the Nginx `default.conf` and identified the incorrect `proxy_pass http://api:3001;` directive, which did not align with the application's actual port.

---

## 3. The Fixes You Applied
* **Nginx Configuration:** Corrected the `proxy_pass` from port `3001` to **`3000`** to match the application. Also add `location /status`. 
* **Docker-Compose:** Add `restart: unless-stopped` for each service, and mount volume `./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql` for postgresql service 
* **Deployment:** Performed a clean rebuild using the `--no-cache` flag to ensure all configuration changes were applied.
* **Validation:** Successfully accessed the endpoint and received the expected JSON: `{"ok":true, ...}`.

---

## 4. Monitoring & Alerts (Proposed)
I recommend setting up the following threshold-based alerts:

| Layer | Metric | Alert Threshold |
| :--- | :--- | :--- |
| **Nginx** | HTTP 5xx rate | > 1% over 2 mins |
| **API** | p95 Latency | > 500ms over 5 mins |
| **Postgres** | Connection saturation | > 80% `max_connections` |
| **Postgres** | Long-running query | > 30s |
| **Redis** | Memory usage | > 80% `maxmemory` |
| **Infra** | Disk | < 15% free |
| **Infra** | CPU usage | < 20% free |
| **Infra** | RAM usage | < 20% free |
| **Container** | Restart count | > 3 times / 10 mins |
---

## 5. How You Would Prevent This in Production

### 5.1 CI/CD & Automation
* **Integration Testing:** Implement a pipeline that runs `docker compose up`, followed by `curl -f` health checks on critical endpoints before merging PRs. This catches port mismatches immediately.
* **Secret Management:** Avoid hardcoding ports or credentials. Use **HashiCorp Vault** or **ENV System** to manage environment variables securely.

### 5.2 Docker Compose Optimization
* **Health Checks:** Use `healthcheck` instead of just `depends_on`. Ensure the API only starts after Postgres and Redis are fully ready to accept connections, preventing startup race conditions.
* **Resource Limits:** Define CPU/RAM limits for each container to prevent a single failing service from crashing the entire node.
