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
curl -X GET -u "YOUR_ACCESS_KEY:YOUR_SECRET_KEY" \
  http://localhost:8000/api/v1/repositories
```

You should receive a **`200 OK`** status code along with a JSON response. **This proves the server itself is running correctly.**

#### **Test 2: Simulating the Browser's Preflight Request**

Now, we'll perfectly simulate the browser's preflight `OPTIONS` request, including all the headers it sends.

```bash
curl -X OPTIONS -v \
  -H "Origin: https://addin.xlwings.org" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: authorization" \
  -H "Access-Control-Request-Private-Network: true" \
  http://localhost:8000/api/v1/repositories
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

