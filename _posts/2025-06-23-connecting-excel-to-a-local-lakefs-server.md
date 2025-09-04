---
title: "Connecting Excel to a Local lakeFS Server"
date: 2025-06-23 21:29:00 +0400
categories: [Tech Stack, Troubleshooting]
tags: [cors, docker, excel, lakefs, nginx, xlwings]
render_with_liquid: false
---

### Abstract

This guide provides a definitive solution to a common but complex problem: connecting a modern Excel add-in (specifically, xlwings Lite) to a lakeFS server running on your local machine (`localhost`).

When you attempt this, you are likely to be blocked by browser security errors that can be difficult to diagnose. In your browser's developer console, you will typically see a failed `OPTIONS` preflight request in the network tab, followed by a CORS error message similar to this:

> Access to XMLHttpRequest at 'http://localhost:8000/api/v1/repositories' from origin 'https://addin.xlwings.org' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.

Before we dive into the technical reasons, let's confirm you're facing this exact issue.

---

### Diagnosing the Exact Point of Failure

You can confirm the precise point of failure using a command-line tool like `curl`. This is the best way to distinguish between a server that's down and this specific preflight issue.

#### **Test 1: A Standard API Request**

First, a direct `GET` request to a lakeFS API endpoint should work perfectly. This confirms your lakeFS container is running and accessible.

```bash
curl -i -X GET \
  -u "YOUR_ACCESS_KEY:YOUR_SECRET_KEY" \
  http://host.docker.internal:8000/api/v1/repositories
```

You should receive a **`200 OK`** status code along with a JSON response. **This proves the server itself is running correctly.**

#### **Test 2: Simulating the Browser's Preflight Request**

Now, we'll perfectly simulate the browser's preflight `OPTIONS` request, including all the headers it sends.

```bash
curl -i -X OPTIONS \
  -H "Origin: https://addin.xlwings.org" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: authorization" \
  -H "Access-Control-Request-Private-Network: true" \
  -u "YOUR_ACCESS_KEY:YOUR_SECRET_KEY" \
  http://host.docker.internal:8000/api/v1/repositories
```

When you run this, the lakeFS server (≤ 1.60.0) will return a **`500 Internal Server Error`**.

#### **Conclusion of the Diagnosis**

These two tests point to the key evidence:

* The server is up and responds correctly to direct API calls (`200 OK`).
* The server **cannot** handle the browser's mandatory PNA preflight check and crashes (`500 Internal Server Error`).

Now that we've confirmed the behavior, let's understand the underlying security features that cause it.

---

### Deep Dive: Why the Connection Fails

The failure you just diagnosed is caused by two specific, modern security protocols built into web browsers.

#### **1. CORS (Cross-Origin Resource Sharing)**

A "cross-origin" request happens when a web page from one origin (e.g., `https://addin.xlwings.org`) requests a resource from a different origin (e.g., `http://localhost`). Browsers restrict these requests by default for security.

* **Credentialed Requests:** The request sent by the add-in includes an `Authorization` header, making it a **"credentialed request."** According to the [Fetch Standard](https://fetch.spec.whatwg.org/), for such requests, the server cannot use a wildcard (`*`) in the `Access-Control-Allow-Origin` response header. It *must* specify the exact origin of the client (`https://addin.xlwings.org`).

#### **2. PNA (Private Network Access)**

Introduced to prevent attacks where public websites could exploit services on your private network, Private Network Access ([PNA](https://developer.chrome.com/blog/private-network-access-preflight/)) adds another layer of security.

* **Preflight `OPTIONS` Request:** Because the request is from a public origin (`https`) to a private one (`http://localhost`), the browser sends a preliminary "preflight" request using the `OPTIONS` method. This is the exact request that we simulated with `curl` and which failed with a `500` error. It includes a new header: `Access-Control-Request-Private-Network: true`.
* **Server Requirement:** To grant access, the server must respond to this `OPTIONS` request with the header: `Access-Control-Allow-Private-Network: true`. Since lakeFS fails to do this, the entire process is halted.

---

### Solution Options

Before diving into the implementation details, here is a summary of the possible solutions to help you choose the best approach for your situation.

| Option                           | When to use                                                                                                    | What it does                                                                                                    | Pros                                                            | Cons                                                                        |
| :------------------------------- | :------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------- | :-------------------------------------------------------------------------- |
| **NGINX Proxy<br>(Recommended)** | **Local development<br>& testing.** The standard,<br>recommended approach.                                     | Intercepts requests,<br>correctly handles<br>the security preflight,<br>then forwards<br>the request to lakeFS. | ✅ **Non-invasive**<br>✅ **Portable & Secure**<br>✅ **Reliable** | Adds one extra<br>(but lightweight)<br>container to your setup.             |
| **Patch lakeFS**                 | When you have full<br>ownership of the<br>production code<br>and cannot use a proxy.<br>(Rare for this issue). | Modifies the lakeFS<br>source code to handle<br>`OPTIONS` requests and<br>PNA headers natively.                 | The fix is self-contained<br>within the application.            | ❌ **Very complex**<br>❌ **High maintenance**<br>❌ **Requires Go expertise** |
| **Disable PNA<br>in Browser**    | For a **quick, temporary,<br>personal test only.**<br>Never for team<br>or development work.                   | Turns off the Private<br>Network Access<br>security feature in your<br>browser's settings.                      | Very fast to implement<br>for a one-off test.                   | ❌ **Insecure**<br>❌ **Not portable**<br>❌ **Affects only you**              |

---

### Solution 1 (Recommended): The NGINX "Sidecar" Proxy

The most effective and non-intrusive solution is to deploy an **NGINX reverse proxy**. The idea is not new; [**Maximillian Xavier’s 2020 blog post**](https://maximillianxavier.medium.com/solving-cors-problem-on-local-development-with-docker-4d4a25cd8cfe "Solving CORS problem on local development with Docker | by Maximillian Xavier | Medium") already suggested to *“run a Docker container with NGINX acting as reverse proxy. Two files, one container, one simple solution\!”* We’ve adapted this elegant idea for **2025-era browsers** that now enforce the stricter Private Network Access (PNA) checks.

Notably, this aligns perfectly with official guidance. The [**xlwings Lite documentation**](https://lite.xlwings.org/web_requests?utm_source=chatgpt.com "Web API Requests - xlwings Lite documentation") explicitly suggests using a **CORS proxy** if you don’t control the API, and recommends **hosting your own** for privacy reasons—which is exactly what this solution accomplishes.

#### **Step-by-Step Implementation**

**1. Create the NGINX Configuration File**

Create a file named `cors.conf`. This is the rulebook for our NGINX proxy.

```nginx
# cors.conf
server {
  listen 80;
  listen [::]:80;

  # Fixed allowed origin
  set $cors_origin "https://addin.xlwings.org";

  # Send on ALL responses (2xx/3xx/4xx/5xx)
  add_header Access-Control-Allow-Origin $cors_origin always;
  add_header Access-Control-Allow-Private-Network true always;
  add_header Access-Control-Allow-Methods "GET,POST,OPTIONS" always;
  add_header Access-Control-Allow-Headers "authorization,content-type,x-lakefs-client" always;
  add_header Access-Control-Max-Age 86400 always;   # 1 day (browsers often cap here)
  add_header Vary Origin always;

  # Preflight
  if ($request_method = OPTIONS) {
    return 204;
  }

  # forward everything else to LakeFS
  location / {
    proxy_pass http://lakefs:8000;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 300;
    proxy_send_timeout 300;
    proxy_redirect off;
    # client_max_body_size 100m;  # uncomment/tune if uploads happen
  }
}
```

**2. Update Your `compose.yaml`**

Modify your Docker Compose setup to run the NGINX container alongside lakeFS.

```yaml
# compose.yaml
services:
  lakefs:
    image: treeverse/lakefs:latest
    # ... (your existing lakefs configuration)
    ports:
      - "${LAKEFS_PORT:-8000}:8000" # LakeFS UI and API port
    environment:
      # Database config
      LAKEFS_DATABASE_TYPE: "..."
      # ... other environment variables

  nginx:
    image: nginx:latest
    ports:
      - "${LAKEFS_CORS_PORT:-8010}:80" # CORS proxy for LakeFS
    volumes:
      - ./cors.conf:/etc/nginx/conf.d/default.conf # Custom CORS configuration
    depends_on:
      - lakefs

# ... (other services like postgres)
```

**3. Verifying the Fix (Smoke Test)**

After running `docker-compose up` with the new configuration, you can immediately verify that the NGINX proxy is working correctly. Run the same preflight request that was failing before (change the host's port number from 8000 to 8010); it will now be handled by NGINX.

```bash
curl -i -X OPTIONS \
  -H "Origin: https://addin.xlwings.org" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-control-request-headers: authorization,x-lakefs-client" \
  -H "Access-Control-Request-Private-Network: true" \
  -u "YOUR_ACCESS_KEY:YOUR_SECRET_KEY" \
  http://host.docker.internal:8010/api/v1/repositories
```

You should see a successful response directly from NGINX:

```
HTTP/1.1 204 No Content
Server: nginx/1.25.5
Date: Mon, 01 Sep 2025 18:52:23 GMT
Connection: keep-alive
Access-Control-Allow-Origin: https://addin.xlwings.org
Access-Control-Allow-Private-Network: true
Access-Control-Allow-Methods: GET,POST,OPTIONS
Access-Control-Allow-Headers: authorization,content-type,x-lakefs-client
Access-Control-Max-Age: 86400
Vary: Origin
```

**4. Configure Your Client Environment**

The final step is to tell your Python code, running via xlwings, to communicate with the NGINX proxy instead of directly with the lakeFS server. You can do this by setting environment variables directly within the xlwings Lite add-in.

1.  In Excel, go to the xlwings Lite panel.

2.  Click the menu icon (**☰**) and select **Environment Variables**.

3.  Add the appropriate variable based on the Python library you are using:

      * For the **`lakefs`** package, set the following variable:

          * **Name:** `LAKECTL_SERVER_ENDPOINT_URL`
          * **Value:** `http://localhost:8010`

      * For the **`lakefs-spec`** (used by `fsspec`) package, set this variable instead:

          * **Name:** `LAKEFS_HOST`
          * **Value:** `http://localhost:8010`

Once set, your Python code will send its requests to the correct endpoint, allowing your connection to work seamlessly.

---

### Alternative (Not Recommended) Solutions

Here we discuss the other options from the matrix in more detail.

  * **Solution 2: Patching lakeFS:** One could modify the lakeFS Go source code to handle `OPTIONS` requests and PNA headers correctly.

      * **Why it's a bad idea:** This is a complex undertaking that requires knowledge of the Go ecosystem and the lakeFS codebase. It creates a maintenance nightmare, as your custom patch would need to be re-applied and tested with every lakeFS update.

  * **Solution 3: Disabling PNA in the Browser:** You can disable this security feature in Chrome/Edge via `chrome://flags/#private-network-access-send-preflights`.

      * **Why it's a bad idea:** This is a client-side workaround that only applies to your machine. It lowers your browser's security posture and is not a portable or scalable solution for a team.

---

### Conclusion

The browser security landscape is constantly evolving with standards like CORS and PNA. While these protections are essential, they can create challenges in local development environments. For connecting a public-origin client like an Excel add-in to a local server, using a **sidecar NGINX proxy is the cleanest, most robust, and most maintainable solution.** It requires no modification to the target application and neatly solves the problem by satisfying the browser's security requirements at the network edge.

---

#### **References**

  * [Xavier, M. (2020). *Simple NGINX Reverse Proxy for Docker*](https://www.google.com/search?q=https://maximillianxavier.medium.com/simple-nginx-reverse-proxy-for-docker-a634431e779a)
  * [xlwings Lite Documentation: Web Requests & CORS](https://lite.xlwings.org/web_requests?utm_source=chatgpt.com)
  * [xlwings Lite Documentation: Environment Variables](https://lite.xlwings.org/environment_variables)
  * [Private Network Access (PNA) specification](https://wicg.github.io/private-network-access/)
  * [Google Developers Blog: Private Network Access preflight](https://developer.chrome.com/blog/private-network-access-preflight/)
  * [Fetch Standard (regarding credentialed requests)](https://fetch.spec.whatwg.org/)
