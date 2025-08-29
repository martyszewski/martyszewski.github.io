
---
title: Excel ↔ LakeFS on localhost: a practical guide to CORS & Private Network Access (PNA) — with a tiny NGINX sidecar
date: 2025-08-24
categories: [Technical]
tags: [lakefs], [excel], [xlwings]
---

# Excel ↔ LakeFS on localhost: a practical guide to CORS & Private Network Access (PNA) — with a tiny NGINX sidecar

**TL;DR:** When calling a local **LakeFS** server from an **Excel (xlwings Lite)** add-in, the browser’s **CORS** preflight (and today’s **Private Network Access** rules) block requests. In our tests with LakeFS **v1.60.0**, `OPTIONS /api/v1/*` returned `500 {"message": "invalid API endpoint"}`, so the preflight failed and the real `GET` never ran. The most robust dev-time fix is an **ultra-small NGINX reverse proxy** that answers the preflight and forwards real requests to LakeFS.

---

## Why this happens (short version)

1. **CORS + credentials:** Because the add-in sends `Authorization`, browsers treat requests as *credentialed*. You **must** echo a **specific** origin (not `*`).

   > “When responding to a credentialed request, the server must specify an origin … instead of the `*` wildcard.” ([MDN Web Docs][1])

2. **Private Network Access (PNA):** Modern Chrome/Edge send a *new* preflight header when an HTTPS page calls `http://localhost`:

   > “The preflight request will carry `Access-Control-Request-Private-Network: true`, and the response … must carry `Access-Control-Allow-Private-Network: true`.” ([Chrome for Developers][2])
   > Enforcement has been tightening since 2024. ([Chrome for Developers][3])

3. **LakeFS REST base path:** Client calls are under `/api/v1` (e.g., `GET /repositories`). ([PyPI][4])

4. **What xlwings Lite expects:** If you don’t control the API, **use a CORS proxy**; and when you do, allow the add-in origin `https://addin.xlwings.org`.

   > “Access-Control-Allow-Origin: **[https://addin.xlwings.org](https://addin.xlwings.org)** … If your API request fails with CORS errors and you don’t control the server, you can use a **CORS proxy** … it’s recommended to host your own.” ([lite.xlwings.org][5])

---

## How the failure looks

* Browser console:
  `Access to XMLHttpRequest … has been blocked by CORS policy: Response to preflight request doesn't pass access control check …`
* `curl GET http://localhost:8000/api/v1/repositories` → **200**
* `curl -X OPTIONS http://localhost:8000/api/v1/repositories …` → **500** with `{"message":"invalid API endpoint"}` (observed locally on v1.60.0).

**Bottom line:** The preflight dies before your real call starts.

---

## The working fix we shipped: a 15-line NGINX sidecar

**What it does**

* Replies to **preflight** (`OPTIONS`) with the 4 headers the browser wants.
* **Proxies** everything else to `lakefs:8000`.

### `cors.conf`

```nginx
server {
    listen 80 default_server;

    # allow exactly the Excel add-in origin (no wildcard with credentials)
    set $cors_origin 'https://addin.xlwings.org';

    # send CORS + PNA headers on every response
    add_header Access-Control-Allow-Origin            $cors_origin                always;
    add_header Access-Control-Allow-Methods           'GET,OPTIONS'               always;
    add_header Access-Control-Allow-Headers           'authorization,x-lakefs-client' always;
    add_header Access-Control-Allow-Private-Network   true                        always;

    # short-circuit the preflight
    if ($request_method = OPTIONS) { return 204; }

    # proxy the real request to LakeFS
    location / {
        proxy_pass http://lakefs:8000;
        proxy_set_header Host            $host;
        proxy_set_header X-Real-IP       $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

> NGINX `add_header` is valid in `http|server|location|if in location`, and `always` ensures headers are added even on non-2xx responses. ([Nginx][6])

### `docker-compose.yml` (excerpt)

```yaml
services:
  lakefs:
    image: treeverse/lakefs:1.60.0
    ports: ["8000:8000"]

  corsproxy:
    image: nginx:1.26-alpine
    volumes:
      - ./cors.conf:/etc/nginx/conf.d/default.conf:ro
    ports: ["8010:80"]
    depends_on: [lakefs]
```

### Point Excel/xlwings at the proxy

```bash
# example env used by your client code
LAKECTL_SERVER_ENDPOINT_URL=http://localhost:8010/api/v1
```

### Smoke test

```bash
curl -i -X OPTIONS http://localhost:8010/api/v1/repositories \
  -H "Origin: https://addin.xlwings.org" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: authorization,x-lakefs-client" \
  -H "Access-Control-Request-Private-Network: true"
```

You should see:

```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://addin.xlwings.org
Access-Control-Allow-Methods: GET,OPTIONS
Access-Control-Allow-Headers: authorization,x-lakefs-client
Access-Control-Allow-Private-Network: true
```

---

## Alternative approaches (trade-offs)

| Option                                                        | When to choose                        | Notes / references                                                                                                                                                                                                           |
| ------------------------------------------------------------- | ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **CORS proxy container** (no NGINX config)                    | You want “batteries included.”        | Same idea as NGINX: forwards requests and injects headers; many dev write-ups use this pattern — *“run a Docker container with NGINX acting as reverse proxy. Two files, one container, one simple solution!”* ([Medium][7]) |
| **Patch LakeFS** to implement `OPTIONS /api/*` (+ PNA header) | You can build custom LakeFS binaries. | Cleanest long-term: handle preflights server-side and echo the PNA response header as Chrome recommends. ([Chrome for Developers][2])                                                                                        |
| **Disable/relax PNA** via enterprise policies                 | Lab environments only.                | Reduces security; still need proper CORS for credentials. ([Chrome for Developers][3])                                                                                                                                       |

---

## Debug checklist (copy/paste friendly)

1. **Confirm the base path**: LakeFS client URIs are under `/api/v1`. ([PyPI][4])
2. **Preflight curl** (simulate the browser): run the OPTIONS command above and inspect headers.
3. **No wildcards with credentials**: ensure `Access-Control-Allow-Origin` is the **exact** origin, not `*`. ([MDN Web Docs][1])
4. **Add the PNA header**: `Access-Control-Allow-Private-Network: true`. ([Chrome for Developers][2])
5. **If NGINX headers “don’t stick”**: ensure `add_header … always;` and put them in `server`/`location` (valid contexts). ([Nginx][6])

---

## Appendix — xlwings Lite specifics

* The add-in runs from `https://addin.xlwings.org` by default, so your server/proxy must allow that origin.
  The docs also suggest using a **self-hosted CORS proxy** if you cannot change the API: *“If your API request fails with CORS errors and you don’t control the server, you can use a CORS proxy … it’s recommended to host your own.”* ([lite.xlwings.org][5])

---

## References & further reading

* **Chrome Developers** — *Private Network Access: introducing preflights*: *“…response … must carry `Access-Control-Allow-Private-Network: true`.”* ([Chrome for Developers][2])
* **Chrome Developers** — *PNA update & deprecation trial* (enforcement timeline). ([Chrome for Developers][3])
* **MDN Web Docs** — *CORS guide*: *“When responding to a credentialed request, the server must specify an origin … instead of `*`.”* ([MDN Web Docs][1])
* **NGINX docs** — `add_header` directive (valid contexts; `always` parameter). ([Nginx][6])
* **lakeFS Python client (PyPI)** — “All URIs are relative to `/api/v1` … `GET /repositories`.” ([PyPI][4])
* **Medium (Maximillian Xavier)** — *Solving CORS problem on local development with Docker* (reverse-proxy pattern). ([Medium][7])

---

### Final takeaway

For local Excel ↔ LakeFS automation, the **least-moving-parts** solution is a tiny **NGINX sidecar** that:

* **answers preflights** with `Allow-Origin` (exact origin), `Allow-Methods`, `Allow-Headers`, and **`Allow-Private-Network`**, and
* **forwards** the real calls to LakeFS.

It’s explicit, easy to version with your `docker-compose`, and portable across machines — no changes to LakeFS required.

[1]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS?utm_source=chatgpt.com "Cross-Origin Resource Sharing (CORS) - MDN"
[2]: https://developer.chrome.com/blog/private-network-access-preflight?utm_source=chatgpt.com "Private Network Access: introducing preflights | Blog"
[3]: https://developer.chrome.com/blog/private-network-access-update?utm_source=chatgpt.com "Private Network Access update: Introducing a deprecation trial"
[4]: https://pypi.org/project/lakefs-client/?utm_source=chatgpt.com "lakefs-client"
[5]: https://lite.xlwings.org/web_requests?utm_source=chatgpt.com "Web API Requests - xlwings Lite documentation"
[6]: https://nginx.org/en/docs/http/ngx_http_headers_module.html?utm_source=chatgpt.com "Module ngx_http_headers_module"
[7]: https://maximillianxavier.medium.com/solving-cors-problem-on-local-development-with-docker-4d4a25cd8cfe?utm_source=chatgpt.com "Solving CORS problem on local development with Docker"
