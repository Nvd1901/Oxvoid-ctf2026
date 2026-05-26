[challenge-01-action-packed.md](https://github.com/user-attachments/files/28261188/challenge-01-action-packed.md)
# Challenge 01 — Action Packed

> **Category:** Web · **Points:** 100 · **Difficulty:** Easy

---

## Table of Contents

- [Overview](#overview)
- [Reconnaissance](#reconnaissance)
- [Exploitation Chain](#exploitation-chain)
  - [Step 1 — Interface Reconnaissance](#step-1--interface-reconnaissance)
  - [Step 2 — Intercepting Packets with Burp Suite](#step-2--intercepting-packets-with-burp-suite)
- [Flag](#flag)
- [Takeaways](#takeaways)

---

## Overview

An internal dashboard exposes convenience actions for trusted workflows. The interesting part is not the UI button, but the packet around the request.

| Field       | Detail                                         |
|-------------|------------------------------------------------|
| Category    | Web                                            |
| Points      | 100                                            |
| Technique   | HTTP Traffic Interception / API Response Leak  |
| Tool        | Burp Suite                                     |
| Flag        | `0xV01D{ca45b212-1551-427d-90a6-98d88a4cd027}` |

---

## Reconnaissance

After accessing the challenge link, we are presented with a minimal internal web interface containing two main surfaces:

| Feature             | Description                                        |
|---------------------|----------------------------------------------------|
| **Profile Settings**| Allows updating or modifying user profile info     |
| **API Access**      | Functions related to generating or verifying tokens|

At first glance, neither feature reveals anything sensitive in the browser. The key is to look beneath the surface — at the raw HTTP traffic.

> **Hypothesis:** The token generation feature may return sensitive data in its response body that the UI deliberately hides from the user.

---

## Exploitation Chain

### Step 1 — Interface Reconnaissance

Access the challenge URL and explore the available features. Set up **Burp Suite** as an intercepting proxy to capture all traffic between the browser and the server.

```
Browser  ──(proxy)──►  Burp Suite  ──────────►  Server
                            │
                            ▼
                     Inspect Requests
                      & Responses
```

Navigate to the **API Access / Token Generation** section and prepare to trigger the token creation action.

---

### Step 2 — Intercepting Packets with Burp Suite

With Burp Suite's **Proxy → Intercept** active, click the token generation button. Capture and forward the outgoing `POST` request, then examine the server's response in the **HTTP history** tab.

**Captured POST request:**

```http
POST /api/token/generate HTTP/1.1
Host: <challenge-host>
Content-Type: application/json

{}
```

**Server response (Burp Suite — Response tab):**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "token": "<generated-token>",
  "flag": "0xV01D{ca45b212-1551-427d-90a6-98d88a4cd027}"
}
```

The server leaks the flag directly inside the JSON response body — a field that the frontend silently discards and never displays to the user.

---

## Flag

```
0xV01D{ca45b212-1551-427d-90a6-98d88a4cd027}
```

---

## Takeaways

| # | Lesson |
|---|--------|
| 1 | **Never trust the UI** — the frontend may hide fields that the API still returns |
| 2 | **Always intercept traffic** — Burp Suite reveals what the browser intentionally conceals |
| 3 | **Sensitive data in API responses** should never be sent to the client unless strictly necessary |
| 4 | **Token generation endpoints** are high-value targets; their responses often carry more than just the token |

---

*Writeup by [nvd] · CTF: [OXvoid-ctf] · Date: 2026-05-26*
