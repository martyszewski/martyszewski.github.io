
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







