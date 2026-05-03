# EC 441 — Web: HTTP & HTML
## Homework Problems + Solved Lab

---

# PART 1: HOMEWORK PROBLEMS

---

## Problem 1: HTTP Fundamentals — Request & Response Format

### Problem Statement

A user types `http://www.bu.edu/index.html` into their browser. The browser has never visited this site before (empty cache). The RTT between client and server is **20 ms**. Assume all objects are small (negligible transmission time).

**(a)** Write out the **exact HTTP/1.1 GET request** the browser sends for the HTML page. Include all required headers and the blank line that terminates the header section. Explain the purpose of each header line you include.

**(b)** The server responds with the following:
```
HTTP/1.1 200 OK
Date: Mon, 03 May 2026 14:00:00 GMT
Content-Type: text/html; charset=UTF-8
Content-Length: 3842
Connection: keep-alive
Last-Modified: Fri, 01 May 2026 09:00:00 GMT

<!DOCTYPE html>
<html> ... </html>
```
Identify and explain the purpose of **each field** in the response. What does `Content-Length` allow the browser to do?

**(c)** The HTML page contains references to **3 CSS files**, **5 images**, and **1 JavaScript file** — 9 embedded objects total. Assuming **HTTP/1.1 with persistent connections and NO pipelining**, how many RTTs does it take to load the full page? Show your reasoning step by step.

**(d)** Repeat (c) for **HTTP/1.1 with persistent connections AND pipelining**. How many RTTs now? Draw a timeline diagram showing when each request and response occurs.

**(e)** Now assume **HTTP/1.0 (non-persistent)** with the browser opening at most **one connection at a time**. How many TCP connections are opened total? How many RTTs?

---

### Solution

**(a) HTTP/1.1 GET Request:**

```
GET /index.html HTTP/1.1
Host: www.bu.edu
Connection: keep-alive
Accept: text/html,application/xhtml+xml,*/*
Accept-Language: en-US,en;q=0.9
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15) Chrome/120
```
*(blank line — marks end of headers)*

**Header explanations:**
- `GET /index.html HTTP/1.1` — the **request line**: method (GET = retrieve), URL path, HTTP version
- `Host: www.bu.edu` — **required in HTTP/1.1**; tells the server which virtual host is being requested (multiple websites can share one IP address; the Host header distinguishes them)
- `Connection: keep-alive` — requests that the TCP connection stay open after this response, enabling persistent connections (HTTP/1.1 default)
- `Accept:` — tells the server what content types the browser can handle
- `User-Agent:` — identifies the browser software; servers sometimes customize responses based on this

The **blank line** after the last header is mandatory — it signals the end of the header section. Without it, the server would wait for more headers forever.

**(b) HTTP Response field-by-field:**

- `HTTP/1.1 200 OK` — **status line**: protocol version, numeric status code (200 = success), human-readable phrase
- `Date:` — timestamp when the response was generated at the server; used for cache validation
- `Content-Type: text/html; charset=UTF-8` — tells the browser how to interpret the body. Without this, the browser wouldn't know whether to render HTML, display plain text, or trigger a download. `charset=UTF-8` specifies the character encoding.
- `Content-Length: 3842` — the exact byte count of the response body. This lets the browser **know when the response is complete** without the server closing the connection — essential for persistent connections. The browser can also show a progress bar and pre-allocate a buffer.
- `Connection: keep-alive` — server agrees to keep the TCP connection open for more requests
- `Last-Modified:` — when the content was last changed; enables **conditional GET** caching. On the next request, the browser can send `If-Modified-Since` and the server returns `304 Not Modified` (no body) if nothing changed, saving bandwidth.

**(c) RTT count — persistent, no pipelining:**

With persistent connections but no pipelining, the browser **must wait for each response before sending the next request** on the same connection.

```
Event                               Cumulative RTTs
─────────────────────────────────   ───────────────
TCP 3-way handshake                       1 RTT
GET /index.html + response                1 RTT     (total: 2)
GET css1 + response                       1 RTT     (total: 3)
GET css2 + response                       1 RTT     (total: 4)
GET css3 + response                       1 RTT     (total: 5)
GET image1 + response                     1 RTT     (total: 6)
GET image2 + response                     1 RTT     (total: 7)
GET image3 + response                     1 RTT     (total: 8)
GET image4 + response                     1 RTT     (total: 9)
GET image5 + response                     1 RTT     (total: 10)
GET script.js + response                  1 RTT     (total: 11)
```

**Total: 11 RTTs** (1 handshake + 10 sequential requests)
**Total time: 11 × 20 ms = 220 ms**

**(d) RTT count — persistent WITH pipelining:**

With pipelining, after receiving the HTML the browser sends **all 9 object requests back-to-back** without waiting for individual responses. The server processes them in order and sends responses back-to-back.

```
Timeline (time →)
─────────────────────────────────────────────────────────
t=0:     Client sends SYN
t=20ms:  Server sends SYN-ACK; client sends ACK + GET /index.html
t=40ms:  Client receives HTML page
         Client immediately sends ALL 9 GETs in one burst (pipelined)
t=60ms:  Client receives all 9 responses (server sent them back-to-back)
```

```
RTT 1: TCP handshake (SYN → SYN-ACK → ACK)
RTT 2: GET index.html → HTML response
RTT 3: All 9 pipelined GETs → all 9 responses arrive
```

**Total: 3 RTTs**
**Total time: 3 × 20 ms = 60 ms**

*(Note: In practice, browsers open 6 parallel connections per host rather than true pipelining, since HTTP/1.1 pipelining has head-of-line blocking issues. The theoretical min here is 3 RTTs.)*

**(e) HTTP/1.0 non-persistent, one connection at a time:**

Each object requires its own TCP connection: handshake (1 RTT) + request/response (1 RTT) = **2 RTTs per object**.

- 1 HTML file → 2 RTTs, 1 TCP connection
- 9 embedded objects → 9 × 2 = 18 RTTs, 9 TCP connections

**Total: 1 + 9 = 10 TCP connections opened**
**Total RTTs: 2 + 18 = 20 RTTs**
**Total time: 20 × 20 ms = 400 ms**

**Summary comparison:**

| Mode | TCP Connections | Total RTTs | Time (RTT=20ms) |
|------|----------------|------------|-----------------|
| HTTP/1.0, non-persistent, sequential | 10 | 20 | 400 ms |
| HTTP/1.1, persistent, no pipelining | 1 | 11 | 220 ms |
| HTTP/1.1, persistent, pipelined | 1 | 3 | 60 ms |

---

## Problem 2: HTTP Methods, Status Codes, and Caching

### Problem Statement

**(a)** HTTP defines several **methods** (also called "verbs"). Describe the purpose of each of the following, and give a concrete real-world example of when a browser or application would use each:
- `GET`
- `POST`
- `PUT`
- `DELETE`
- `HEAD`

**(b)** Match each HTTP status code to its meaning, and describe a concrete scenario where it occurs:

| Code | Name | Scenario |
|------|------|----------|
| 200 | | |
| 301 | | |
| 304 | | |
| 404 | | |
| 500 | | |
| 503 | | |

**(c)** Explain the **conditional GET** mechanism. A browser cached an image at `http://bu.edu/logo.png` on Monday. On Wednesday it requests the same image. Write out the exact HTTP exchange (both request and response) for the case where the image has **not** changed, and the case where it **has** changed.

**(d)** Web **caches (proxy servers)** sit between clients and origin servers. Explain how a shared campus web cache reduces both latency and bandwidth usage. Draw the two possible request paths: cache hit vs. cache miss.

**(e)** A page loads correctly over `http://` but is broken over `https://`. Name **three possible causes** rooted in the HTTP/TLS layer (not network/firewall issues).

---

### Solution

**(a) HTTP Methods:**

- **GET** — retrieve a resource. The request has no body. Idempotent: calling it 10 times has the same effect as calling it once. Example: typing a URL in the address bar, clicking a link, loading an image — every browser navigation uses GET.

- **POST** — send data to the server to be processed; typically creates or updates a resource. Has a request body. Not idempotent — submitting a form twice may create two records. Example: submitting a login form, posting a tweet, uploading a file.

- **PUT** — replace a resource at a known URL. Idempotent — putting the same data twice results in the same server state. Example: a REST API updating a user's profile (`PUT /users/42` with a JSON body). The distinction from POST: PUT targets a specific known URL; POST lets the server decide where to store the result.

- **DELETE** — remove a resource. Idempotent — deleting something that's already gone is still "success." Example: a REST API deleting a record (`DELETE /posts/17`).

- **HEAD** — identical to GET but the server returns **only the headers, no body**. Used to check metadata without downloading the content. Example: checking if a large file has changed before downloading it (`Content-Length`, `Last-Modified`), or checking if a URL is valid for link-checking tools.

**(b) Status codes:**

| Code | Name | Meaning | Concrete Scenario |
|------|------|---------|-------------------|
| 200 | OK | Request succeeded; response body contains the requested resource | Loading `google.com` successfully |
| 301 | Moved Permanently | Resource has permanently moved to a new URL (given in `Location` header); browser should update its bookmark | `http://bu.edu` → redirects to `https://bu.edu` permanently |
| 304 | Not Modified | Conditional GET: resource hasn't changed since cached copy; no body sent | Browser sends `If-Modified-Since`; server says "your cache is still valid" |
| 404 | Not Found | The server cannot find the requested resource at that URL | Typing a misspelled URL; a deleted page |
| 500 | Internal Server Error | The server encountered an error processing the request — a bug in server-side code | An unhandled exception in a Python Flask app |
| 503 | Service Unavailable | Server temporarily cannot handle requests — overloaded or down for maintenance | A web server during a traffic spike or deployment window |

**(c) Conditional GET — exact HTTP exchange:**

**First visit (Monday) — server caches the response headers:**
```
GET /logo.png HTTP/1.1
Host: bu.edu

HTTP/1.1 200 OK
Last-Modified: Sat, 01 May 2026 08:00:00 GMT
Content-Type: image/png
Content-Length: 8240
[image bytes]
```
The browser stores the image AND the `Last-Modified` value in its local cache.

---

**Wednesday revisit — image has NOT changed:**
```
GET /logo.png HTTP/1.1
Host: bu.edu
If-Modified-Since: Sat, 01 May 2026 08:00:00 GMT

HTTP/1.1 304 Not Modified
Date: Wed, 05 May 2026 10:00:00 GMT
```
The server checks: has `logo.png` been modified since Saturday? No → return `304` with **no body**. The browser uses its cached copy. Zero bytes of image data transmitted — huge bandwidth saving.

---

**Wednesday revisit — image HAS changed:**
```
GET /logo.png HTTP/1.1
Host: bu.edu
If-Modified-Since: Sat, 01 May 2026 08:00:00 GMT

HTTP/1.1 200 OK
Last-Modified: Tue, 04 May 2026 15:30:00 GMT
Content-Type: image/png
Content-Length: 9102
[new image bytes]
```
The server checks: modified since Saturday? Yes → return `200` with the full new image. The browser replaces its cached copy and updates the stored `Last-Modified` date.

**(d) Web cache (proxy) — hit vs. miss:**

**Cache HIT path** (fast, cheap):
```
Browser → Campus Cache → [found in cache!] → Response back to Browser
```
- Latency: RTT to the campus cache (maybe 1–2 ms, same building)
- No internet bandwidth used
- Origin server receives zero requests for this object

**Cache MISS path** (slower, uses bandwidth):
```
Browser → Campus Cache → [not found] → Origin Server → Cache stores copy → Browser
```
- Latency: RTT to origin server (maybe 50–200 ms, across internet)
- Internet link bandwidth consumed once
- On the next request for same object → cache hit for all subsequent users

**Two benefits:**
1. **Reduced latency** for cache hits — the campus cache is nearby; the origin server may be across the world. A 2 ms cache hit vs. a 100 ms round-trip to the origin is a 50× improvement.
2. **Reduced bandwidth** on the campus→internet uplink — if 500 students all request the same YouTube thumbnail, the campus cache fetches it once from YouTube and serves it 500 times locally. The internet link only sees 1 request instead of 500.

This is exactly why CDN edge servers exist globally — they are institutionalized, strategically placed web caches.

**(e) Three causes of http:// working but https:// broken:**

1. **Certificate error** — the server's TLS certificate is expired, self-signed (not trusted by the browser's CA store), or issued for a different hostname (CN mismatch). The browser shows a security warning and blocks the connection. HTTP has no certificate requirement so it always connects.

2. **Mixed content blocking** — the HTTPS page tries to load resources (images, scripts) over `http://`. Modern browsers block these "mixed content" requests because loading insecure resources inside a secure page would undermine HTTPS's security guarantee. HTTP pages have no such restriction.

3. **HTTP Strict Transport Security (HSTS) misconfiguration** — the server previously sent an HSTS header telling the browser to *only* use HTTPS for this domain for a set duration. If the HTTPS certificate later becomes invalid (expired, revoked), the browser cannot fall back to HTTP — HSTS forbids it. The site is inaccessible until the certificate is fixed. Under plain HTTP, no such enforcement exists.

---

## Problem 3: RTT Calculation — Exam-Style

### Problem Statement

A web client has an RTT of **RTT = 10 ms** to a web server. DNS resolution is **already complete** (no DNS RTT needed). The web page `index.html` contains **1 base HTML file** and **8 embedded objects** (images and scripts).

Assume transmission time for all objects is negligible (small objects).

**(a)** The client connects using **HTTP/1.0 non-persistent**. The browser opens **up to 5 parallel connections**. How many RTTs does the full page load take? Show all work.

**(b)** The client switches to **HTTP/1.1 persistent, no pipelining, 1 connection**. How many RTTs now?

**(c)** With **HTTP/1.1 persistent and pipelining (1 connection)**:
- RTTs to load the full page?
- Total time in ms?

**(d)** The client now uses **HTTPS (TLS 1.2)** with HTTP/1.1 persistent. TLS 1.2 requires **2 additional RTTs** for the TLS handshake (after the TCP handshake). How many total RTTs to load the full page with pipelining?

**(e)** Finally, **HTTPS with TLS 1.3** reduces the TLS overhead to **1 additional RTT**. And **QUIC/HTTP/3** eliminates the transport+TLS handshake overhead entirely (1 RTT for the first request on a new connection, 0 RTT on resumed connections). Assuming this is the client's **first visit**, how many RTTs under QUIC?

---

### Solution

**(a) HTTP/1.0 non-persistent, up to 5 parallel connections:**

Step 1: Fetch `index.html`:
- 1 TCP connection: 1 RTT (handshake) + 1 RTT (GET + response) = **2 RTTs**

Step 2: Fetch 8 embedded objects with up to 5 parallel connections:
- Round 1: Open 5 connections in parallel → fetch 5 objects simultaneously → 2 RTTs each, but they happen in parallel = **2 RTTs**
- Round 2: Open 3 more connections → fetch remaining 3 objects → **2 RTTs**

Total: 2 (HTML) + 2 (round 1) + 2 (round 2) = **6 RTTs**
**Total time: 6 × 10 ms = 60 ms**

**(b) HTTP/1.1 persistent, no pipelining, 1 connection:**

1 RTT for TCP handshake
+ 1 RTT per object (sequential, no pipelining), 9 objects total

Total: 1 + 9 = **10 RTTs**
**Total time: 10 × 10 ms = 100 ms**

**(c) HTTP/1.1 persistent, pipelined, 1 connection:**

```
RTT 1: TCP handshake
RTT 2: GET index.html → receive HTML
RTT 3: Send all 8 GETs in one burst → receive all 8 responses
```

Total: **3 RTTs**
**Total time: 3 × 10 ms = 30 ms**

**(d) HTTPS with TLS 1.2 + HTTP/1.1 persistent pipelined:**

TLS 1.2 adds 2 RTTs *after* the TCP handshake, *before* any HTTP data:

```
RTT 1: TCP handshake (SYN/SYN-ACK/ACK)
RTT 2: TLS ClientHello → ServerHello + Certificate
RTT 3: TLS key exchange → TLS Finished (connection secured)
RTT 4: GET index.html → HTML response
RTT 5: All 8 pipelined GETs → all responses
```

Total: **5 RTTs**
**Total time: 5 × 10 ms = 50 ms**

**(e) HTTPS with TLS 1.3 and QUIC/HTTP/3 (first visit):**

**TLS 1.3** reduces handshake to 1 RTT (combines key exchange into one round-trip):
```
RTT 1: TCP handshake
RTT 2: TLS 1.3 combined handshake (1-RTT mode)
RTT 3: GET index.html → HTML
RTT 4: 8 pipelined GETs → responses
Total: 4 RTTs, 40 ms
```

**QUIC/HTTP/3** (first visit — 1-RTT connection establishment):
- QUIC combines TCP handshake + TLS 1.3 into a single 1-RTT exchange
- HTTP/3 has no HOL blocking — all 8 objects stream independently

```
RTT 1: QUIC handshake (transport + TLS combined) + GET index.html sent
RTT 2: HTML received; 8 pipelined/multiplexed GETs sent
RTT 3: All 8 responses received
Total: 3 RTTs (approximately), 30 ms
```

**Summary table:**

| Mode | RTTs | Time (RTT=10ms) |
|------|------|-----------------|
| HTTP/1.0, non-persistent, 5 parallel | 6 | 60 ms |
| HTTP/1.1, persistent, no pipeline | 10 | 100 ms |
| HTTP/1.1, persistent, pipelined | 3 | 30 ms |
| HTTPS TLS 1.2, pipelined | 5 | 50 ms |
| HTTPS TLS 1.3, pipelined | 4 | 40 ms |
| QUIC/HTTP/3 (first visit) | ~3 | ~30 ms |

Key takeaway: **the TCP + TLS handshake overhead is the biggest bottleneck for short interactions**, which is exactly why QUIC was designed.

---

## Problem 4: HTTP Headers Deep Dive

### Problem Statement

A browser sends the following request, and receives the following response. Answer the questions below.

**Request:**
```
GET /api/profile HTTP/1.1
Host: api.example.com
Cookie: session_id=abc123xyz; theme=dark
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Accept: application/json
Accept-Encoding: gzip, deflate, br
If-None-Match: "a3f2bc9d"
Connection: keep-alive
```

**Response:**
```
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Content-Encoding: gzip
Content-Length: 412
ETag: "a3f2bc9d"
Cache-Control: private, max-age=300
Set-Cookie: last_login=2026-05-03T14:00:00Z; HttpOnly; Secure; SameSite=Strict
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

**(a)** The response has `ETag: "a3f2bc9d"` and the request sent `If-None-Match: "a3f2bc9d"`. The server returned `200 OK` anyway. What does this tell you? Under what condition would the server return `304 Not Modified` instead?

**(b)** The response says `Content-Encoding: gzip` but `Content-Length: 412`. Does `Content-Length` refer to the compressed size or the uncompressed size? What must the browser do before parsing the JSON?

**(c)** `Cache-Control: private, max-age=300` — explain each directive. Why would `private` matter for a `/api/profile` endpoint specifically?

**(d)** The `Set-Cookie` header has three attributes: `HttpOnly`, `Secure`, and `SameSite=Strict`. Explain what attack each attribute defends against.

**(e)** `Strict-Transport-Security: max-age=31536000; includeSubDomains` — what is this header doing? What happens the next time a user types `http://api.example.com` in their browser? How long does this protection last?

---

### Solution

**(a) ETag and conditional GET:**

The client sent `If-None-Match: "a3f2bc9d"` — this is a conditional GET saying "only send me the full response if the content's ETag is different from this value." The server's response ETag is `"a3f2bc9d"` — the **same value**. 

Yet the server returned `200 OK` with a full body. This means one of two things:
1. The server **doesn't support conditional GET** for this endpoint and ignores `If-None-Match`
2. The content **did change** but the developer forgot to update the ETag (a bug)

The server should have returned **`304 Not Modified`** (with no body) when it sees `If-None-Match` matches the current ETag. It would return `200 OK` (with full body) if the ETag had changed — i.e., `"a3f2bc9d"` in the request but `"f9bc2a3e"` on the server would trigger a full response.

**(b) Content-Encoding and Content-Length:**

`Content-Length: 412` refers to the **compressed (gzip) size** — it is the number of bytes actually transmitted over the wire. This is what matters for TCP framing (knowing when the response body ends).

The browser must:
1. Receive all 412 bytes
2. **Decompress the gzip data** to get the actual JSON
3. Then parse the JSON

The uncompressed JSON might be 2,000+ bytes; gzip commonly achieves 70–80% compression on text/JSON. The browser knew compression was acceptable because it sent `Accept-Encoding: gzip` in the request.

**(c) Cache-Control directives:**

- `private` — this response **may only be cached by the browser** (or other end-user caches), **not by shared caches** like CDN edge servers or corporate proxy caches. The reason: `/api/profile` returns **user-specific data** (your profile, not a public page). If a CDN cached it and served it to a different user, that user would see your profile data — a serious privacy/security violation.

- `max-age=300` — the cached response is valid for **300 seconds (5 minutes)**. During this window the browser can re-use the cached response without contacting the server. After 300 seconds the cache entry is stale and must be revalidated.

Together: "cache this locally for up to 5 minutes, but never put it in a shared cache."

**(d) Cookie security attributes:**

- **`HttpOnly`** — defends against **Cross-Site Scripting (XSS)**. Normally, JavaScript running on the page can read cookies via `document.cookie`. `HttpOnly` makes the cookie invisible to JavaScript — it's only sent automatically by the browser in HTTP requests. A malicious script injected into the page cannot steal the `session_id` cookie if it has `HttpOnly`.

- **`Secure`** — defends against **network eavesdropping (man-in-the-middle attacks)**. The cookie is only sent over HTTPS connections, never over plain HTTP. Without this, a user who accidentally visits `http://` (unencrypted) would have their cookie visible to anyone on the network.

- **`SameSite=Strict`** — defends against **Cross-Site Request Forgery (CSRF)**. With `Strict`, the browser only sends this cookie when the request originates from the same site that set it. If `evil.com` has a hidden form that submits to `api.example.com`, the browser will not attach the cookie to that cross-site request. The attacker cannot forge authenticated requests on the user's behalf.

**(e) HTTP Strict Transport Security (HSTS):**

`Strict-Transport-Security` is a security policy the server sends to the browser saying: **"For the next year (31,536,000 seconds = 365 days), you must only connect to me over HTTPS. Never use HTTP."**

**What happens next time a user types `http://api.example.com`:**
The browser doesn't even send the HTTP request. It internally rewrites the URL to `https://api.example.com` and connects over TLS directly — **without ever sending a cleartext HTTP request**. The user's browser does this transformation locally, before any network traffic.

This matters because a redirect from `http://` to `https://` still sends one unencrypted HTTP request (the redirect request itself) which an attacker on the network could intercept. HSTS eliminates this window entirely.

**`includeSubDomains`** extends the policy to all subdomains (`mail.example.com`, `cdn.example.com`, etc.) — they are all forced to HTTPS as well.

**Duration: 31,536,000 seconds = 1 year.** Every time the user visits the site (over HTTPS), the timer is reset. HSTS entries can be preloaded into browsers at factory-settings time (the HSTS preload list) so the protection applies even on the very first visit.

---

---

# PART 2: SOLVED LAB

## Lab: HTTP in Action — Using `curl` to Observe the Protocol

### Overview

In this lab, you will use `curl` (a command-line HTTP client) to observe real HTTP requests and responses. You will see persistent vs. non-persistent connections, caching headers, redirects, and HTTPS behavior — all the concepts from lecture made visible at the wire level.

**Tools needed:** `curl` (pre-installed on macOS/Linux; on Windows use WSL or Git Bash)

---

### Lab Part 1: Your First Raw HTTP Request

**Task:** Fetch a web page and see the full HTTP headers.

**Command:**
```bash
curl -v http://httpbin.org/get
```

The `-v` flag (verbose) shows everything: the TCP connection, the request headers curl sends, and the response headers the server returns.

**Expected output (annotated):**

```
*   Trying 52.2.228.189:80...             ← TCP SYN being sent
* Connected to httpbin.org port 80        ← TCP 3-way handshake complete

> GET /get HTTP/1.1                       ← Request line (method, path, version)
> Host: httpbin.org                       ← Required HTTP/1.1 header
> User-Agent: curl/7.88.1                 ← curl identifies itself
> Accept: */*                             ← Accept any content type

< HTTP/1.1 200 OK                         ← Status line (200 = success)
< Date: Sun, 03 May 2026 14:00:00 GMT     ← When server generated response
< Content-Type: application/json          ← Body is JSON
< Content-Length: 307                     ← Body is exactly 307 bytes
< Connection: keep-alive                  ← TCP stays open (HTTP/1.1 default)
<                                         ← Blank line = end of headers
{                                         ← JSON body begins
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.org",
    "User-Agent": "curl/7.88.1"
  },
  ...
}
```

**What to observe:**
- The `>` lines are what **curl sent** (the HTTP request)
- The `<` lines are what **the server sent back** (the HTTP response)
- The blank line after the last `<` header separates headers from body
- `Connection: keep-alive` confirms the TCP connection stays open

**Exercise question:** What would change in the output if you used `curl -v http://httpbin.org/get http://httpbin.org/ip` (two URLs)? Would curl open a second TCP connection or reuse the first?

**Answer:** curl reuses the first TCP connection (persistent connection). You will see `Connected to httpbin.org` only once, but two request/response cycles. The `Connection: keep-alive` from the server told curl to keep the socket open.

---

### Lab Part 2: Observing HTTP Redirects

**Task:** Watch a 301 redirect happen in real time.

**Command (follow redirects):**
```bash
curl -v -L http://bu.edu
```

**Command (do NOT follow redirects — see the redirect itself):**
```bash
curl -v http://bu.edu
```

**Expected output without `-L`:**
```
> GET / HTTP/1.1
> Host: bu.edu

< HTTP/1.1 301 Moved Permanently
< Location: https://www.bu.edu/          ← New URL to follow
< Content-Length: 0
```

**With `-L` (follow redirect):**
```
* Issue another request to this URL: https://www.bu.edu/
*   Trying 128.197.x.x:443...            ← Now connecting to HTTPS port 443
* SSL connection using TLSv1.3           ← TLS negotiated

> GET / HTTP/1.1
> Host: www.bu.edu

< HTTP/1.1 200 OK
...
```

**What happened:**
1. `http://bu.edu` → server responds `301 Moved Permanently` → `https://www.bu.edu/`
2. curl (with `-L`) follows the redirect and now opens a **new TCP connection** to port 443
3. TLS handshake occurs (you see `SSL connection using TLSv1.3`)
4. A second `GET /` request goes over the encrypted connection
5. Finally `200 OK` with the actual page

This is why your browser always shows `https://www.bu.edu` even when you type just `bu.edu` — it received a 301 and followed it automatically.

---

### Lab Part 3: Conditional GET and Caching

**Task:** Observe how `If-Modified-Since` works to avoid re-downloading unchanged content.

**Step 1 — First request (get the Last-Modified header):**
```bash
curl -v http://httpbin.org/cache
```

Look for the `Last-Modified` header in the response:
```
< Last-Modified: Sun, 03 May 2026 14:00:00 GMT
< ETag: "bfc13a64729c4290ef5b2c2730249c88ba4"
```

**Step 2 — Conditional GET (send If-Modified-Since):**
```bash
curl -v -H "If-Modified-Since: Sun, 03 May 2026 14:00:00 GMT" \
        http://httpbin.org/cache
```

The `-H` flag adds a custom header to the request.

**Expected response:**
```
< HTTP/1.1 304 Not Modified
< ETag: "bfc13a64729c4290ef5b2c2730249c88ba4"
```

**No body is returned** — just the status line and a few headers. The content hasn't changed, so the server sends back `304` with zero bytes of body data. This is exactly what browsers do on repeat visits — checking if cached content is still fresh.

**Step 3 — Force a fresh download:**
```bash
curl -v -H "Cache-Control: no-cache" http://httpbin.org/cache
```

The `Cache-Control: no-cache` header tells the server: "ignore any caching, send me the full response even if nothing changed." Response: `200 OK` with full body.

---

### Lab Part 4: Seeing HTTP vs HTTPS at the Protocol Level

**Task:** Compare the difference in connection setup between HTTP and HTTPS.

**HTTP (plain):**
```bash
curl -v --trace-time http://httpbin.org/get 2>&1 | head -30
```

**HTTPS:**
```bash
curl -v --trace-time https://httpbin.org/get 2>&1 | head -40
```

**What to look for in HTTPS output:**
```
* Connected to httpbin.org port 443
* ALPN: offers h2,http/1.1               ← Negotiating HTTP/2 or 1.1 over TLS
* TLSv1.3 (OUT), TLS handshake, Client hello
* TLSv1.3 (IN),  TLS handshake, Server hello
* TLSv1.3 (IN),  TLS handshake, Encrypted Extensions
* TLSv1.3 (IN),  TLS handshake, Certificate
* TLSv1.3 (IN),  TLS handshake, CERT verify
* TLSv1.3 (IN),  TLS handshake, Finished
* TLSv1.3 (OUT), TLS handshake, Finished
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
```

You can see the TLS handshake happening — **Client hello → Server hello → Certificate → Verify → Finished** — all before a single byte of HTTP data is exchanged. This is the extra RTT(s) that HTTPS adds on top of TCP.

**Counting the RTTs from the trace:**
- TCP connect: 1 RTT
- TLS 1.3 handshake: 1 RTT (see one round of Client hello / Server hello)
- HTTP GET / response: 1 RTT
- **Total for first request: 3 RTTs minimum**

Compare with plain HTTP: 2 RTTs (TCP + HTTP request).

---

### Lab Part 5: Timing a Page Load — Measuring RTT Components

**Task:** Use `curl`'s built-in timing output to break down where time goes in an HTTP request.

```bash
curl -w "\n\nTiming breakdown:\n\
  DNS lookup:        %{time_namelookup}s\n\
  TCP connect:       %{time_connect}s\n\
  TLS handshake:     %{time_appconnect}s\n\
  Time to first byte:%{time_starttransfer}s\n\
  Total time:        %{time_total}s\n\
  Bytes downloaded:  %{size_download} bytes\n" \
  -o /dev/null -s https://www.bu.edu/
```

**Sample output:**
```
Timing breakdown:
  DNS lookup:         0.012s
  TCP connect:        0.031s
  TLS handshake:      0.068s
  Time to first byte: 0.102s
  Total time:         0.287s
  Bytes downloaded:   48293 bytes
```

**Interpretation:**

| Component | Time | What it means |
|-----------|------|---------------|
| DNS lookup | 12 ms | Time to resolve `www.bu.edu` → IP address |
| TCP connect | 31 ms | Time from first SYN to completing 3-way handshake |
| TLS handshake | 68 ms | Time from TCP connected to TLS secured (includes TCP time) |
| Time to first byte | 102 ms | Time from request sent to first byte of response received |
| Total | 287 ms | Full download of all 48 KB of HTML |

**Things to notice:**
- TLS handshake = 68 ms − 31 ms = **37 ms** additional for TLS (≈ 1–2 RTTs)
- Time to first byte = 102 ms − 68 ms = **34 ms** waiting for the server to respond (processing + RTT)
- 48 KB took 287 − 102 = **185 ms** to download (transmission time, not negligible here!)

**Try it yourself:** Run the command against `http://` (no TLS) vs `https://` and compute how many milliseconds the TLS handshake adds. Run it multiple times and observe how DNS lookup drops to near-zero after the first run (cached by your OS resolver).

---

### Lab Part 6: POST vs GET

**Task:** Send a POST request with a JSON body and compare it to GET.

**GET request — data in the URL:**
```bash
curl -v "http://httpbin.org/get?name=Alice&age=20"
```

You'll see the data in the request line:
```
> GET /get?name=Alice&age=20 HTTP/1.1
```

**POST request — data in the body:**
```bash
curl -v -X POST \
     -H "Content-Type: application/json" \
     -d '{"name": "Alice", "age": 20}' \
     http://httpbin.org/post
```

You'll see:
```
> POST /post HTTP/1.1
> Content-Type: application/json
> Content-Length: 27
>
> {"name": "Alice", "age": 20}    ← request body after blank line
```

**Key differences to observe:**
1. GET data is visible in the URL (and in server logs, browser history, proxy logs) — **not private**
2. POST data is in the request body — not in the URL, but **still plaintext over HTTP** (use HTTPS for actual privacy)
3. POST has a `Content-Length` header; GET does not (GET has no body)
4. The server response from httpbin echoes back what it received — you can see your JSON in the `json` field of the response

**Lab question:** You submit a login form (username + password). Should the form use `GET` or `POST`? Why would using `GET` be a security problem even over HTTPS?

**Answer:** Always use **POST**. With GET, the password appears in the URL: `https://example.com/login?user=alice&password=secret123`. This URL gets stored in browser history, appears in server access logs, and could appear in the HTTP `Referer` header if the user clicks a link after logging in. Even over HTTPS (which encrypts the body), URLs can leak in various ways. POST puts the credentials in the encrypted body, invisible to logs and history.

---

### Lab Summary & Key Observations

After completing this lab, you should have observed:

| Concept | Where you saw it |
|---------|-----------------|
| HTTP request/response format | Part 1: `-v` output showing `>` and `<` lines |
| Persistent connections | Part 1: two URLs, one TCP connection |
| 301 redirect | Part 2: `Location` header, curl following it |
| HTTP → HTTPS upgrade | Part 2: redirect to port 443 |
| Conditional GET / 304 | Part 3: `If-Modified-Since` → `304 Not Modified`, no body |
| TLS handshake overhead | Part 4: Client hello / Server hello messages before HTTP |
| RTT breakdown | Part 5: `curl -w` timing output |
| GET vs POST | Part 6: data in URL vs. data in body |

---

*End of HTTP/HTML Problems & Lab*

**Topics covered:**
- HTTP/1.0 vs HTTP/1.1 vs HTTP/2 vs QUIC — RTT analysis
- Persistent connections, pipelining, parallel connections
- HTTP request format, HTTP response format, all common headers
- Status codes (200, 301, 304, 404, 500, 503)
- Conditional GET and caching (ETag, If-None-Match, Last-Modified, If-Modified-Since)
- Cache-Control, HSTS, cookie security (HttpOnly, Secure, SameSite)
- GET vs POST and security implications
- `curl` as a tool for directly observing HTTP at the protocol level