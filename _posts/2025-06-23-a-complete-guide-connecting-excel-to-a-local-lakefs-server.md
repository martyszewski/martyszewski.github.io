---
layout: post
title: "A Complete Guide: Connecting Excel to a local LakeFS Server"
date: 2025-06-23 21:29:00 +0400
categories: [Blogging, Tutorial]
tags: [lakefs, xlwings]
render_with_liquid: false
---

This guide provides a comprehensive solution to a common but tricky problem: connecting a modern Excel add-in (like xlwings Lite) to a lakeFS server running on your own computer (`localhost`). When you attempt this, you'll likely run into frustrating security errors from your web browser.

Here, we'll break down *why* this happens and provide a detailed, step-by-step solution using a simple but powerful tool: **NGINX**.

-----

### The Core Problem: Modern Browser Security Rules

When your Excel add-in tries to communicate with your local lakeFS server, it gets blocked by two security rules built into modern browsers to protect you from malicious websites.

1.  **CORS (Cross-Origin Resource Sharing):** Think of this as a security check at a border. Your add-in is from one web address (the "origin," `https://addin.xlwings.org`), and your lakeFS server is at another (`http://localhost`). Because your add-in sends login credentials (an `Authorization` header), the browser enforces a strict rule: the server's response *must* explicitly name the add-in's origin in the `Access-Control-Allow-Origin` header. A generic wildcard (`*`) is not allowed for credentialed requests.

2.  **PNA (Private Network Access):** This is a newer security layer. It prevents public websites from making requests to servers on your private network (like `localhost`) without permission. To enforce this, the browser sends a preliminary "preflight" `OPTIONS` request before the *actual* request. This preflight includes the header `Access-Control-Request-Private-Network: true`. For the connection to proceed, the server *must* respond with `Access-Control-Allow-Private-Network: true`.

The issue is that the lakeFS server (tested on v1.60.0) doesn't correctly handle this PNA preflight request, returning a `500` error and causing the browser to block the connection entirely.

-----

### The Best Solution: An NGINX "Sidecar" Proxy

The most robust and portable solution is to use an **NGINX reverse proxy**.

Think of NGINX as a smart receptionist sitting in front of your lakeFS server. It intercepts all incoming requests from the Excel add-in. This receptionist is specifically trained to handle the browser's tricky CORS and PNA security questions. It provides the correct headers in its response, satisfying the browser. Once the browser is happy, NGINX simply passes the real request straight through to the lakeFS server.

This approach is ideal because **you don't have to modify the lakeFS server at all**. It works seamlessly within a Docker environment.

#### Step-by-Step Implementation

To get this working, you'll run NGINX in a "sidecar" container alongside your lakeFS container using Docker Compose.

**1. Create the NGINX Configuration File**

First, create a file named `cors.conf` in the same directory as your `docker-compose.yml` file. This file contains the rules NGINX will follow.

```nginx
# cors.conf

server {
    listen 80;
    listen [::]:80;

    location / {
        # Handle the browser's "preflight" OPTIONS request
        if ($request_method = 'OPTIONS') {
            # Respond with all necessary CORS and PNA headers
            add_header 'Access-Control-Allow-Origin' 'https://addin.xlwings.org';
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization';
            add_header 'Access-Control-Allow-Private-Network' 'true';
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain; charset=utf-8';
            add_header 'Content-Length' 0;
            
            # End the request here with a 204 No Content response
            return 204;
        }

        # For all other requests (GET, POST, etc.), 
        # add the origin header and forward to the lakeFS server.
        add_header 'Access-Control-Allow-Origin' 'https://addin.xlwings.org';
        proxy_pass http://lakefs:8000;
    }
}
```

**2. Update Your Docker Compose File**

Next, modify your `docker-compose.yml` file to include the NGINX proxy service. You'll add a new `nginx` service and make your local port `8000` point to the NGINX container instead of directly to the lakeFS container.

```yaml
# docker-compose.yml

version: "3"
services:
  lakefs:
    image: treeverse/lakefs:latest
    # ... (keep your existing lakefs configuration)
    ports:
      - "18000:8000" # Expose lakeFS on a different host port if needed for direct access
    environment:
      # ... (your lakeFS environment variables)

  nginx:
    image: nginx:latest
    volumes:
      - ./cors.conf:/etc/nginx/conf.d/default.conf
    ports:
      - "8000:80" # Map host port 8000 to the NGINX container's port 80
    depends_on:
      - lakefs

# ... (other services if you have them)
```

**Key Changes in `docker-compose.yml`:**

  * **`nginx` service:** We define a new service using the official `nginx` image.
  * **`volumes`:** We mount our `cors.conf` file into the NGINX container, replacing its default configuration.
  * **`ports`:** The crucial change. Your machine's `localhost:8000` now points to the NGINX proxy. The proxy then forwards traffic internally to `lakefs:8000`.
  * **`depends_on`:** This ensures the `lakefs` container starts up before the `nginx` container does.

Now, when you run `docker-compose up`, both containers will start. Your Excel add-in will talk to `localhost:8000` (NGINX), which will handle the security headers and seamlessly pass the request to lakeFS, solving the connection issue.
