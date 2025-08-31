---
layout: post
title: "Connecting Excel to a Local lakeFS Server: A Clearer Guide"
date: 2025-06-23 21:29:00 +0400
categories: [Blogging, Tutorial]
tags: [lakefs, xlwings]
render_with_liquid: false
---

This guide solves a common but tricky problem: connecting a modern Excel add-in (like xlwings Lite) to a lakeFS server running on your own computer (`localhost`). When you try this, you'll likely run into security errors from your web browser.

Here, we'll break down why this happens and provide a clear, step-by-step solution using a simple tool called NGINX.

### The Core Problem: Modern Browser Security

When your Excel add-in tries to talk to your local lakeFS server, it's blocked by two modern browser security rules designed to protect you:

1.  **CORS (Cross-Origin Resource Sharing):** Think of this like a bouncer at a club. Your add-in is from one web address (the "origin"), and your lakeFS server is at another (`localhost`). Because your add-in is sending login details (an `Authorization` header), the bouncer (your browser) is extra strict. It requires the lakeFS server to explicitly say, "Yes, I trust the add-in from that specific address." The default lakeFS setup doesn't do this, so the request is blocked.
2.  **PNA (Private Network Access):** This is a newer rule. It's like a doorman asking for extra credentials before letting someone from a public place (the internet, where your add-in technically lives) into a private building (your computer, or `localhost`). The browser sends a special "preflight" question to the server first. The lakeFS server (in version v1.60.0, at least) doesn't know how to answer this question correctly, so it returns an error, and the browser cancels the connection.

### The Best Solution: An NGINX "Sidecar"

The most reliable way to fix this is to use a simple **NGINX reverse proxy**.

Think of NGINX as a helpful receptionist. It sits in front of your lakeFS server and handles all the incoming requests from your Excel add-in. This receptionist knows exactly how to answer the browser's tricky CORS and PNA security questions. Once it gives the browser the "all clear," it simply passes the real request straight through to the lakeFS server.

This is the best approach because **you don't have to modify the lakeFS server at all**.

#### Step-by-Step Implementation

To get this working, you'll run NGINX in a "sidecar" container using Docker.

1.  **Create the NGINX Configuration File:**
    * Create a file named `cors.conf`.
    * This file tells NGINX how to handle the security preflight questions and forward everything else. It ensures the correct `Access-Control-Allow-Origin` and `Access-Control-Allow-Private-Network` headers are sent back to the browser.

2.  **Set Up Your Docker Compose File:**
    * Edit your `docker-compose.yml` file.
    * Add the NGINX service alongside your existing lakeFS service. This configuration will tell Docker to:
        * Run the official NGINX image.
        * Use the `cors.conf` file you just created.
        * Listen for requests and forward them to your lakeFS container.

By following these steps, your browser will get the security assurances it needs, and your Excel add-in will be able to connect to your local lakeFS server smoothly.

### What About Other Options?

The original article briefly mentions alternatives. Here’s a more detailed look at why they aren't as good:

* **Disabling PNA in Your Browser:** You could turn off the Private Network Access security feature in your browser's settings (e.g., via `chrome://flags`).
    * **Why it's not ideal:** This is a temporary fix that only works on your machine. It also lowers your browser's security, which is not recommended.
* **Patching lakeFS:** You could try to modify the lakeFS code itself to handle these requests correctly.
    * **Why it's not ideal:** This is complex, could introduce bugs, and would need to be re-applied every time you update lakeFS.

The NGINX sidecar avoids all of these problems, making it the superior solution for a development environment.
