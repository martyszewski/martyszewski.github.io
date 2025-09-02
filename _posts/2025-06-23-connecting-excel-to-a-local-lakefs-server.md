---

title: "Connecting Excel to a Local lakeFS Server"
date: 2025-06-23 21:29:00 +0400
categories: [Tech Stack, Troubleshooting]
tags: [cors, docker, excel, lakefs, nginx, xlwings]
render_with_liquid: false

---

This guide provides a definitive solution to a common but complex problem: connecting a modern Excel add-in (specifically, xlwings Lite) to a lakeFS server running on your local machine (`localhost`).

When you attempt this, you are likely to be blocked by browser security errors that can be difficult to diagnose. In your browser's developer console, you will typically see a failed `OPTIONS` preflight request in the network tab, followed by a CORS error message similar to this:

> Access to XMLHttpRequest at 'http://localhost:8000/api/v1/repositories' from origin 'https://addin.xlwings.org' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.

Before we dive into the technical reasons, let's confirm you're facing this exact issue.

