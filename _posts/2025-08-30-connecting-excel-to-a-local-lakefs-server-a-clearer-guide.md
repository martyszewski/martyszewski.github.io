---
layout: post
title: Connecting Excel to a Local lakeFS Server: A Clearer Guide
---

## Connecting Excel to a Local lakeFS Server: A Clearer Guide

This guide solves a common but tricky problem: connecting a modern Excel add-in (like xlwings Lite) to a lakeFS server running on your own computer (`localhost`). When you try this, you'll likely run into security errors from your web browser.

Here, we'll break down why this happens and provide a clear, step-by-step solution using a simple tool called NGINX.

### The Core Problem: Modern Browser Security

When your Excel add-in tries to talk to your local lakeFS server, it's blocked by two modern browser security rules designed to protect you:

1.  **CORS (Cross-Origin Resource Sharing):** Think of this like a bouncer at a club. Your add-in is from one web address (the "origin"), and your lakeFS server is at another (`localhost`). Because your add-in is sending login details (an `Authorization` header), the bouncer (your browser) is extra strict. It requires the lakeFS server to explicitly say, "Yes, I trust the add-in from that specific address." The default lakeFS setup doesn't do this, so the request is blocked.
2.  **PNA (Private Network Access):** This is a newer rule. It's like a doorman asking for extra credentials before letting someone from a public place (the internet, where your add-in technically lives) into a private building (your computer, or `localhost`). The browser sends a special "preflight" question to the server first. The lakeFS server (in version v1.60.0, at least) doesn't know how to answer this question correctly, so it returns an error, and the browser cancels the connection.

