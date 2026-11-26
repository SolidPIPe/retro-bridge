# Retro-Bridge Developer Guide

Retro-Bridge connects apps to a RetroTINK over Wi-Fi. It works with both
the **RetroTINK 4K** and **RetroTINK 6X** series of scalers. This guide calls either model
"the RetroTINK".

This guide covers the Retro-Bridge protocol itself (connecting to the bridge, sharing the
RetroTINK with other apps, sending commands, binary transfers). It does not
include all the RetroTINK's serial commands.

Applies to firmware **1.02 and later**.

## Contents

- [Choose an interface](#choose-an-interface)
- [Essentials](#essentials)
- [HTTP API](#http-api)
  - [Quick start](#quick-start) · [Endpoints](#endpoints) · [Device info](#device-info) · [Clients and IDs](#clients-and-ids)
  - [Send a command](#send-a-command) · [Check status](#check-status) · [Wake the RetroTINK](#wake-the-retrotink)
  - [Leases and file transfers](#leases-and-file-transfers) · [Monitoring](#monitoring) · [Errors](#errors) · [HTTP client rules](#http-client-rules)
- [Raw TCP serial (port 2323)](#raw-tcp-serial-port-2323)
- [Troubleshooting](#troubleshooting)

## Choose an interface

| Interface | Best for |
| --- | --- |
| [HTTP API](#http-api) (port 80) | New apps and integrations, browser, desktop & mobile apps. |
| [Raw TCP](#raw-tcp-serial-port-2323) (port 2323) | Existing serial software, terminals, plain byte streams. |

All interfaces share the bus fairly, choose based on your usecase.

## Essentials

- **Address:** The default is `http://retro-bridge.local`, and users can rename
  it. mDNS doesn't work on every network, so always let users enter an IP
  address or a custom mDNS address instead.
- **Security:** There's no authentication or TLS. Anyone on the LAN
  can control the bridge. Never expose ports 80 or 2323 to the internet, unless you want strangers to change your profiles. :P
- **Line endings:** Sent bytes reach the RetroTINK unchanged, so end every
  command with CR, LF or CR LF. In code, use your language's escape sequence, such as `"ver\r\n"`.
- **Binary-safe replies:** Replies are raw bytes. Decode them as text only when
  you expect or know the reply is text.
- **Automatic routing & baud negotiation.** The bridge reaches the RetroTINK over USB or HD-15,
  whichever is ready. Apps can't choose the link. Unlike a physical HD-15 connection, there is **no need to send
  `baud` commands** over HTTP or raw TCP.
- **Replies only.** You receive only what the RetroTINK sends during your own turn.
  Output sent while no app has a turn is discarded.
- **Feature detection.** Read `capabilities` and `limits` from
  [`/api/v1/info`](#device-info) instead of hard-coding them as they can change between firmware versions.

## HTTP API

### Quick start

```sh
# 1. Identify the bridge
curl http://retro-bridge.local/api/v1/info

# 2. Create a client  →  {"client_id":12345,"expires_ms":300000}
curl -X POST http://retro-bridge.local/api/v1/clients

# 3. Send a command and print the reply
printf 'ver\r\n' | curl --data-binary @- \
  -H 'Content-Type: application/octet-stream' \
  'http://retro-bridge.local/api/v1/clients/12345/transactions?request_id=1'
```

### Endpoints

All paths start with `/api/v1`. `{c}` is a client ID and `{l}` is a lease ID.

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/info` | Identity, capabilities, limits |
| `GET` | `/status` | Link and scheduler state |
| `GET` | `/metrics` | Traffic counters |
| `GET` | `/events?cursor=N` | Recent event log |
| `POST` | `/clients` | Create a client |
| `DELETE` | `/clients/{c}` | Remove a client and cancel its work |
| `POST` | `/clients/{c}/transactions?request_id=N` | Send a command and wait for the reply |
| `POST` | `/clients/{c}/transactions/{id}/cancel` | Cancel a transaction |
| `POST` | `/clients/{c}/wake` | Wake a RetroTINK |
| `POST` | `/clients/{c}/leases?lease_id=N` | Request a lease |
| `GET` | `/clients/{c}/leases/{l}` | Lease state |
| `POST` | `/clients/{c}/leases/{l}/write` | Send bytes |
| `GET` | `/clients/{c}/leases/{l}/read?cursor=N` | Receive bytes |
| `POST` | `/clients/{c}/leases/{l}/heartbeat` | Keep an idle lease alive |
| `POST` | `/clients/{c}/leases/{l}/release` | Release a lease |
| `POST` | `/clients/{c}/leases/{l}/cancel` | Cancel a queued or active lease |

Other routes, such as `/config/*` and `/debug.txt`, serve the built-in
dashboard only.

### Device info

`GET /api/v1/info` (trimmed):

```json
{
  "name": "Retro-Bridge",
  "firmware": "1.02",
  "api_version": 1,
  "hostname": "retro-bridge.local",
  "device_id": "0123456789abcdef",
  "capabilities": ["transactions", "binary_leases", "no_ack_transfers", "rt4k_wake"],
  "limits": {"clients": 8, "transaction_bytes": 1024, "response_bytes": 4096}
}
```

- Require `api_version` to be `1`.
- `device_id` is fixed for each bridge. Use it to recognize a saved device
  after its IP address changes.

| Capability | Meaning |
| --- | --- |
| `transactions` | [Send a command](#send-a-command) |
| `binary_leases` | [Exclusive leases](#leases-and-file-transfers) |
| `no_ack_transfers` | `no_ack` leases |
| `lease_write_wait` | `wait_ms` on lease writes |
| `queued_lease_cancel` | Cancel works on queued leases |
| `parallel_exclusive_leases` | Two exclusive leases can be active at once, one per link |
| `lease_cursor_64` | Read cursors are 64-bit |
| `rt4k_wake`, `rt4k_wake_readiness_probe` | [Wake](#wake-the-retrotink) returns only when the RetroTINK is ready |
| `transparent_tcp_serial` | [Raw TCP](#raw-tcp-serial-port-2323) on port 2323 |

| Limit | Value | Meaning |
| --- | --- | --- |
| `clients` | 8 | Clients, including raw TCP connections |
| `queue_per_client` | 4 | Pending transactions and leases per client |
| `transaction_bytes` | 1024 | Maximum transaction request |
| `response_bytes` | 4096 | Maximum transaction reply |
| `lease_write_bytes` / `lease_read_bytes` | 4096 | Maximum bytes per lease write or read |
| `lease_write_wait_ms` | 5000 | Maximum write `wait_ms` |
| `http_connections` | 16 | Open HTTP connections, all apps combined |
| `raw_tcp_connections` | 3 | Raw TCP connections |

### Clients and IDs

Every HTTP app needs a client ID. Create one per app session (not per command), and reuse it.

```http
POST /api/v1/clients HTTP/1.1
```

```json
{"client_id":12345,"expires_ms":300000}
```

- A client expires after **5 minutes** without transactions or lease
  activity. On `404 no_client`, create a new client and continue with it.
- Client IDs don't survive a bridge restart.
- When the eight slots are full, which includes raw TCP connections, creation
  returns `429 client_limit`.
- Call `DELETE /api/v1/clients/{c}` when your app closes. It cancels the
  client's queued and active work.

**Request and lease IDs.** Every [transaction](#send-a-command) and
[lease](#leases-and-file-transfers) you start also needs its own ID:

| ID | Chosen by | Identifies | Sent as |
| --- | --- | --- | --- |
| Client ID | The bridge | Your app's session | `{c}` in the path |
| Request ID | Your app | One transaction | `request_id` query parameter |
| Lease ID | Your app | One lease | `lease_id` query parameter, then `{l}` in the path |

Request and lease IDs belong to your client and share one set of numbers. Keep
a single counter starting at 1, take the next number for every transaction or
lease, and don't reuse a number. Reusing a number that's still in use, or that
finished recently, returns `409 duplicate_request`. The bridge can't detect
older reuse, so don't rely on that error. You use the same ID later to cancel
a transaction, or to read from, write to and release a lease.

### Send a command

A transaction sends one command and returns its reply in the HTTP response.

```http
POST /api/v1/clients/12345/transactions?request_id=7&completion=first_reply_line HTTP/1.1
Content-Type: application/octet-stream
Content-Length: 13

remote menu
```

The body is `remote menu` followed by CR LF. Response:

```http
HTTP/1.1 200 OK
Content-Type: application/octet-stream
X-Retro-Bridge-Result: complete
X-Retro-Bridge-Transport: usb_ftdi

[COM] Serial Remote: menu
```

| Parameter | Default | Values |
| --- | --- | --- |
| `request_id` | required | Nonzero, unused ID |
| `completion` | `idle_gap` | When the reply counts as complete (see below) |
| `idle_ms` | `30` | Silence that ends an `idle_gap` reply, `0`–`timeout_ms` |
| `timeout_ms` | `2000` | `1`–`30000` |
| `priority` | `interactive` | Use `background` for work that the user isn't waiting on |

| `completion` | Completes when | Use for |
| --- | --- | --- |
| `idle_gap` | Reply bytes arrive, then `idle_ms` of silence | Multi-line or unknown-length replies |
| `first_reply_line` | The first CR or LF arrives | One-line replies. The body can include extra bytes after that line. |
| `no_reply` | Your bytes have been sent | Commands that never reply |

| HTTP | `X-Retro-Bridge-Result` | Meaning |
| --- | --- | --- |
| `200` | `complete` | Reply received |
| `413` | `overflow` | Reply exceeded 4,096 bytes. The body holds the first part. |
| `504` | `timeout` | No complete reply in time. The body can hold a partial reply. |
| `503` | `disconnected` | The RetroTINK link dropped |
| `503` | `canceled` | Canceled by you or by deleting the client |

`X-Retro-Bridge-Transport` reports the link used: `usb_ftdi` or `hd15_uart`.
Browser apps can't read this header; desktop and Node apps can.

**Watch out for:**

- With `idle_gap` and `first_reply_line`, a command that sends nothing back
  always ends in `504 timeout`. Use `no_reply` for silent commands.
- Time spent waiting in the queue counts toward `timeout_ms`. Set your HTTP
  timeout at least one second longer than `timeout_ms`.
- Transactions don't queue behind another app's lease or raw TCP stream. If
  both USB and HD-15 are connected, the command runs on the free link.
  Otherwise you get `409 serial_reserved` or `409 binary_transfer_active` with
  `retry_after_ms`. Wait at least that long, then resend with a **new**
  `request_id`.
- A timeout doesn't mean the command never ran. Don't resend commands that
  change state automatically.
- Use a [lease](#leases-and-file-transfers) for requests over 1,024 bytes or
  replies over 4,096 bytes.

To cancel, call `POST .../transactions/{id}/cancel`. It returns `204`, or
`404 transaction_not_found`. The waiting call then returns `503 canceled`.

#### JavaScript Example

A minimal client commonly used in browsers and Node apps. The lease example(s) below reuse its
`api()` helper, `client_id` and ID counter.

```js
const BASE = "http://retro-bridge.local";
let nextId = 1; // Shared by request IDs and lease IDs.

const api = (path, init = {}) =>
  fetch(BASE + path, { cache: "no-store", targetAddressSpace: "local", ...init });

const { client_id } = await (await api("/api/v1/clients", { method: "POST" })).json();

async function send(command, options = {}) {
  const query = new URLSearchParams({ request_id: nextId++, ...options });
  const res = await api(`/api/v1/clients/${client_id}/transactions?${query}`, {
    method: "POST",
    headers: { "Content-Type": "application/octet-stream" },
    body: new TextEncoder().encode(command),
  });
  const reply = new Uint8Array(await res.arrayBuffer());
  if (!res.ok) {
    const reason = res.headers.get("X-Retro-Bridge-Result") ?? new TextDecoder().decode(reply);
    throw new Error(`HTTP ${res.status}: ${reason}`);
  }
  return reply;
}

console.log(new TextDecoder().decode(await send("ver\r\n")));
await send("remote menu\r\n", { completion: "first_reply_line" });
```

### Check status

`GET /api/v1/status` returns a large object, most of them being for diagnostics. Apps only need these fields:

| Field | Meaning |
| --- | --- |
| `wifi.ip` | Numeric address, the fallback when mDNS fails |
| `serial.state` | `"ready"` when the RetroTINK can be reached |
| `serial_links.usb.ready`, `serial_links.hd15.ready` | Readiness of each link |
| `scheduler.owner` | Who holds the RetroTINK right now; empty when idle |
| `scheduler.wait_reason` | `binary_transfer` or `tcp_serial` while one of them holds the RetroTINK; otherwise empty. A queued lease reports its own [`wait_reason`](#request-a-lease). |
| `scheduler.maintenance_active` | The bridge is busy briefly; retry shortly |
| `rt4k_wake.state` | Result of the latest wake |

Poll it no more than once per second.

### Wake the RetroTINK

Use this endpoint instead of sending `pwr on`.

```http
POST /api/v1/clients/12345/wake?timeout_ms=30000 HTTP/1.1
Content-Length: 0
```

Response (trimmed):

```json
{"state":"ready","acknowledged":true,"ready_transport":"usb_ftdi","ready_baud":2000000}
```

- The call returns only once the RetroTINK accepts normal commands. If the RetroTINK is
  already awake, it returns quickly without sending a power command.
- `timeout_ms` defaults to `30000` and accepts `1000`–`30000`. Set your HTTP
  timeout at least two seconds longer.
- Proceed only on `200` with `"state":"ready"`. On its own,
  `acknowledged: true` means the RetroTINK received the power request and may still
  be booting.
- Apps that call at the same time share one wake.

| HTTP | `state` | Action |
| --- | --- | --- |
| `200` | `ready` | Send commands |
| `409` | `busy` | Another app is using the RetroTINK. Retry after it finishes. |
| `503` | `unavailable` | No RetroTINK link is connected |
| `504` | `timeout` | The RetroTINK didn't become ready. Retry or tell the user. |

### Leases and file transfers

A lease reserves the RetroTINK for a multi-step exchange: file transfers, OSD or
font streams, ACK/NAK protocols, or anything larger than a transaction.

| Mode | Use for | Your app sends RTL1 ACKs | Ends |
| --- | --- | --- | --- |
| `exclusive` (default) | ACK-mode transfers (`-a`) and any multi-step exchange | Yes | When you release it |
| `no_ack` | Streaming transfers without `-a`, such as `get`, `font`, `osd`, `osd2` and uploads | No | Automatically, after the final frame and RetroTINK terminal line |

```text
queued → active → complete | canceled | timeout | disconnected | overflow
```

#### Request a lease

```http
POST /api/v1/clients/12345/leases?lease_id=8&mode=no_ack HTTP/1.1
```

```json
{"lease_id":8,"state":"queued","mode":"no_ack","transport":"none","automatic_release":true}
```

| Parameter | Default | Values |
| --- | --- | --- |
| `lease_id` | required | Next ID from your shared counter |
| `mode` | `exclusive` | `exclusive` or `no_ack` |
| `idle_ms` | `15000` | `250`–`lifetime_ms`. Ends the lease after this long with no reads, writes, heartbeats or serial traffic. |
| `lifetime_ms` | `120000` | `1000`–`300000`. How long the lease can go without moving data at the minimum rate (see [Expiry](#expiry)). |
| `completion_idle_ms` | `1000` | `no_ack` only: `20`–`5000`, and no more than `idle_ms` |

If the lease is `queued`, poll `GET .../leases/{l}` every 50–250 ms until it's
`active`. Don't write while it's queued, and don't request a duplicate. A
queued lease reports what it's waiting for:

```json
{"lease_id":8,"state":"queued","mode":"no_ack","transport":"none","automatic_release":true,
 "bridge_paced_download":false,"blocked":true,"wait_reason":"binary_transfer",
 "blocking_client_id":4711,"blocking_id":3}
```

#### Write

```http
POST /api/v1/clients/12345/leases/8/write?wait_ms=1000 HTTP/1.1
Content-Type: application/octet-stream

<up to 4096 bytes>
```

```json
{"written":2058}
```

- Writes are all or nothing. A full 2,058-byte RTL1 frame fits in one write.
- `wait_ms` (up to `5000`) waits for buffer space instead of failing at once.
- `429 lease_not_writable`: nothing was accepted. Retry the same chunk.
- `409 lease_not_writable`: the lease is no longer active.
- If the connection drops during a write, you can't tell whether the bridge
  accepted it. Don't resend; abort the transfer.

#### Read

```http
GET /api/v1/clients/12345/leases/8/read?cursor=0&wait_ms=1000 HTTP/1.1
```

```http
HTTP/1.1 200 OK
Content-Type: application/octet-stream
X-Retro-Bridge-Cursor: 4096
X-Retro-Bridge-Overflow: false
X-Retro-Bridge-Complete: false

<up to 4096 bytes>
```

- Start at cursor `0`. Pass each `X-Retro-Bridge-Cursor` value to the next
  read, and check that it equals the old cursor plus the bytes received.
- `wait_ms` (up to `5000`) waits for data. An empty `200` means nothing arrived
  yet.
- `Overflow: true` means bytes were lost. Abort, release, and restart the
  transfer.
- `Complete: true` means a `no_ack` lease has finished. Stop reading.
- Run exactly **one** read loop per lease.
- Cursors are 64-bit. In JavaScript, use `BigInt` if a single lease could pass
  2<sup>53</sup> bytes.

#### Heartbeat, release, cancel

| Call | Returns | Notes |
| --- | --- | --- |
| `heartbeat` | `204` | Resets the `idle_ms` timer. Reads and writes already do this. It doesn't extend `lifetime_ms`. |
| `release` | `204` | Call it in `finally`. It's safe after automatic completion. Returns `409 lease_not_active` for a lease that isn't active yet. |
| `cancel` | `204` | Ends a queued or active lease. Returns `409 lease_not_found` for an unknown ID. |

#### Expiry

- A lease has no total time limit while data keeps moving at a minimum rate:
  5 KiB/s on USB, and less on slow HD-15 links.
- Whenever data moves slower than that, the lease ends with `timeout` after
  `lifetime_ms`, unless the rate recovers first. This includes slow interactive
  exchanges, so a lease can't stay open longer than `lifetime_ms` (at most 5
  minutes) without moving data.
- With no reads, writes, heartbeats or serial traffic at all, the lease ends
  sooner, after `idle_ms`.
- Heartbeats reset only the `idle_ms` timer. They don't count as moving data.
- Don't put a total-duration limit on transfers in your app. Use an inactivity
  timeout instead.

#### ACK-mode transfer (`exclusive`)

1. Request an `exclusive` lease and wait for `active`.
2. Write the command with `-a`, for example `get -a -- /path/file\r\n`.
3. Read frames, and write an ACK for each one as the RetroTINK protocol requires.
4. If the reply is `[COM] <command>: busy`, the RetroTINK is already running
   another transfer. Release the lease and retry later.
5. Release the lease in `finally`.

When both links are connected, two exclusive leases can be active at once.
The RetroTINK still runs only one file transfer at a time, so step 4 applies.

#### No-ACK mode download (`no_ack`)

1. Request a `no_ack` lease and wait for `active`.
2. Start the read loop at cursor `0` **before** writing the command.
3. Write the command **without** `-a`, for example `get -- /path/file\r\n`.
4. Pass every byte, in order, to your RTL1 decoder: a text `ready` line, then
   binary frames. Don't send ACKs.
5. If the RetroTINK replies with another line instead of `ready`, such as
   `[COM] get: busy`, the transfer didn't start and the lease won't end by
   itself. Stop reading, release the lease and retry later.
6. The transfer has succeeded only when your decoder has validated it **and** a
   read has returned `Complete: true`. Either one alone isn't enough.
7. Release the lease in `finally`.

> [!WARNING]
> HD-15 has no flow control. If a `no_ack` lease becomes active with
> `"transport":"hd15_uart"`, release it and use an `exclusive` lease in ACK mode.

##### JavaScript Example

The read loop for steps 2, 4, 5 and 6, followed by the full download sequence.
It reuses `api()` and `client_id` from the [first example](#javascript-example).
`decoder` is your own RTL1 decoder: it takes bytes through `push()`, throws if
the first line isn't `ready` (step 5), and sets `complete` once it has
validated the file.

```js
async function readLease(leaseId, decoder) {
  let cursor = 0;
  for (;;) {
    const res = await api(
      `/api/v1/clients/${client_id}/leases/${leaseId}/read?cursor=${cursor}&wait_ms=1000`);
    if (!res.ok) throw new Error(`read failed: HTTP ${res.status}`);
    const bytes = new Uint8Array(await res.arrayBuffer());
    const next = Number(res.headers.get("X-Retro-Bridge-Cursor"));
    if (res.headers.get("X-Retro-Bridge-Overflow") === "true") throw new Error("data lost");
    if (next !== cursor + bytes.length) throw new Error("cursor gap");
    cursor = next;
    if (bytes.length) decoder.push(bytes);
    if (res.headers.get("X-Retro-Bridge-Complete") === "true") {
      if (!decoder.complete) throw new Error("transfer incomplete");
      return;
    }
  }
}

// Step 1 is done: lease `leaseId` is active.
const lease = `/api/v1/clients/${client_id}/leases/${leaseId}`;
const decoder = new MyRtl1Decoder(); // Your own RTL1 decoder.

async function writeCommand(command) {
  const res = await api(`${lease}/write?wait_ms=1000`, {
    method: "POST",
    headers: { "Content-Type": "application/octet-stream" },
    body: new TextEncoder().encode(command),
  });
  if (!res.ok) throw new Error(`write failed: HTTP ${res.status}`);
}

try {
  await Promise.all([
    readLease(leaseId, decoder),           // Step 2: start reading first.
    writeCommand("get -- /path/file\r\n"), // Step 3: then send the command.
  ]);
} finally {
  await api(`${lease}/release`, { method: "POST" }); // Step 7.
}
```

#### No-ACK mode upload (`no_ack`)

1. Request a `no_ack` lease, wait for `active`, and start the read loop.
2. Write the upload command without `-a`, then wait for the RetroTINK's `ready`
   line. If another line arrives instead, such as `[COM] put: busy`, release the
   lease and retry later.
3. Write the DATA frames one per write, using `wait_ms`. A full frame is 2,058
   bytes. Finish with a zero-length DATA frame.
4. Keep reading until the RetroTINK's final line and `Complete: true`.

### Monitoring

- **`GET /api/v1/metrics`**: cumulative byte counters, intended for polling
  once or twice a second. Work out rates from the differences between polls.
  Counters wrap at 32 bits, so use unsigned subtraction. Reset your baseline
  when `serial_epoch`, `usb_epoch` or `hd15_epoch` changes.
- **`GET /api/v1/events?cursor=N`**: up to 32 recent events, each with
  `cursor`, `timestamp_ms`, `severity` and `message`. On the next call, pass
  the largest `events[].cursor` you've processed, not the top-level `cursor`.

### Errors

Errors return `{"error":"code"}`. A failed transaction instead returns the
reply bytes and `X-Retro-Bridge-Result`. Check the HTTP status and the code
together.

| HTTP | Code | Meaning and action |
| --- | --- | --- |
| `400` | `invalid_request` | Bad ID, size, timeout or lease parameter |
| `400` | `invalid_completion`, `invalid_lease_mode`, `invalid_cursor` | Bad parameter value |
| `400` | `invalid_timeout_ms` | Wake timeout outside `1000`–`30000` |
| `400` | `malformed_request` | Bad HTTP, headers over 2 KiB or body over 4 KiB |
| `404` | `no_client` | Client expired or the bridge restarted. Create a new client. |
| `404` | `lease_not_found`, `transaction_not_found` | Unknown ID |
| `409` | `duplicate_request` | ID already used. Take the next ID. |
| `409` | `serial_reserved`, `binary_transfer_active` | The RetroTINK is held by another app, or the bridge is busy briefly. Wait `retry_after_ms`, then retry with a new ID. |
| `409` | `lease_not_active` | The lease has ended |
| `409` | `lease_not_found` | Cancel called with an unknown lease ID |
| `409` / `429` | `lease_not_writable` | `409`: the lease is inactive. `429`: the buffer is full; retry the same chunk. |
| `409` | `wake_superseded` | Request the wake again |
| `429` | `client_limit` | All 8 client slots are in use |
| `429` | `queue_full` | Your client has 4 pending operations. Wait for replies. |
| `503` | `http_connections_full` | Too many HTTP connections. The request didn't run; retry after 1 second. |
| `503` | `status_report_busy` | Retry shortly |
| `504` | `bridge_timeout`, `wake_bridge_timeout` | No result in time. The outcome is unknown. |

### HTTP client rules

- **Reuse connections.** Keep one pooled HTTP client per bridge: browser
  `fetch`, a shared .NET `HttpClient`, or a Python `requests.Session`. Don't
  create a new HTTP client per request.
- **Expect closed connections.** The bridge closes connections after 2 seconds
  idle and after 1,024 requests. Your client must reconnect without fuss. A
  closed connection doesn't end your client ID or your leases.
- **Limit slow requests.** Keep no more than three slow requests open at once:
  transactions, waiting reads and wake.
- **Plain requests only.** Send `Content-Length`. Don't use chunked encoding,
  `Expect: 100-continue` or pipelining.
- **Retry only when the request didn't run.** That's true for
  `503 http_connections_full`, `409 serial_reserved`,
  `409 binary_transfer_active` and `429 queue_full`. After a timeout or dropped
  connection, the outcome is unknown.
- **Browsers.** API routes allow any origin (CORS). Chromium asks the user for
  Local Network Access. Use a private IP address in the URL, or pass
  `targetAddressSpace: "local"` to `fetch`. Chromium-based browsers are
  supported.

## Raw TCP serial (port 2323)

Open a plain TCP socket to port 2323 and exchange bytes as you would over a
direct connection to the RetroTINK's USB serial port. In PuTTY, choose **Raw** and
port `2323`.

- **Raw (not Telnet)** Turn off Telnet and RFC 2217 negotiation as they inject
  bytes.
- **Three connections at most.** Each one uses one of the eight client slots.
  Extra connections close as soon as they connect.
- **Turns.** An idle connection doesn't hold the RetroTINK. Each command takes a
  turn that lasts through its full reply or transfer. Other apps can run
  between your commands.
- **Transfers.** ACK mode and streaming both pass through unchanged. Prefer
  ACK mode (`-a`) for reliable file transfers.
- **Waiting.** By default, a command waits silently while another app holds
  the RetroTINK. If it can't start within 30 seconds, the connection closes. Use
  [`rbctl 1`](#cooperative-mode-rbctl-1) to get an immediate busy reply
  instead.
- **Keep reading.** The connection closes if your app stops reading for 30
  seconds, or when Wi-Fi or the RetroTINK link drops. Reconnect and carry on.

### Cooperative mode (`rbctl 1`)

By default, a command waits silently while another app is using the RetroTINK,
so your app can't tell whether it was sent. Cooperative mode makes the bridge
answer each command immediately with `accepted` or `busy`, so your app never
waits blindly for a turn and always knows whether the command reached the
RetroTINK.

It applies only to raw TCP connections, and only to the connection
that turns it on. Other connections keep the default behavior.

Send `rbctl 1` once. From then on, the bridge tells you
whether each command got a turn before anything reaches the RetroTINK.

```text
→ rbctl 1\r
← \r\n[COM] rbctl 1 ready\r\n
→ ver\r
← \r\n[COM] rbctl 1 accepted ver\r\n
← …RetroTINK reply…
→ osd\r
← \r\n[COM] rbctl 1 busy osd\r\n
```

- `busy`: the command was discarded and never reached the RetroTINK. It won't run
  later. Retry when you're ready.
- `accepted`: the command got its turn. That doesn't mean the RetroTINK succeeded,
  so read its reply as usual.
- The decision line repeats your command without line endings, so you can
  match decisions to commands.
- The mode stays on until the connection closes.
- Each command, including its line ending, must fit in 4,096 bytes, or the
  connection closes.
- Decision lines start with `[COM]` like RetroTINK output. Check for
  `[COM] rbctl 1` before you parse a line as a RetroTINK reply.

#### Python Example

A minimal client that turns on cooperative mode and sends one command.

```python
import socket

sock = socket.create_connection(("retro-bridge.local", 2323), timeout=5)
rx = sock.makefile("rb")

def next_line():
    while True:
        line = rx.readline().strip()
        if line:
            return line.decode(errors="replace")

sock.sendall(b"rbctl 1\r\n")
assert next_line() == "[COM] rbctl 1 ready"

sock.sendall(b"ver\r\n")
decision = next_line()
if decision == "[COM] rbctl 1 accepted ver":
    print(next_line())                # First line of the RetroTINK reply
elif decision == "[COM] rbctl 1 busy ver":
    print("RetroTINK busy; ver was not sent")
```

### Virtual COM ports

Apps that can only open a COM port may work through virtual COM port software
that forwards to port 2323. This setup isn't officially supported. If you try
it:

- Use the app's USB mode at **2,000,000 baud**.
- Don't send `baud` commands or use the app's HD-15 mode. Its baud changes
  fail.
- Set the virtual port to plain raw TCP, with Telnet/NVT and RFC 2217 off.
- Turn off hardware flow control, or make sure the port reports CTS as ready.
  Otherwise the app can stall.
- Give each app its own COM port.

## Troubleshooting

| Symptom | Try |
| --- | --- |
| `retro-bridge.local` doesn't resolve | Use the IP address from the dashboard or router. Some networks block mDNS. The name works about 5 seconds after the bridge joins Wi-Fi. Check whether the user renamed it. |
| Browser requests fail with no response | Allow Local Network Access when Chromium asks. Use the IP address in the URL. |
| Every command returns `504 timeout` | The RetroTINK may be in standby: call [wake](#wake-the-retrotink). Check `serial.state` in `/status`. Silent commands need `completion=no_reply`. |
| `409 serial_reserved` keeps repeating | Another app is transferring. Check `scheduler.owner` and retry once it's idle. |
| `429 client_limit` | Close unused apps and raw TCP connections. Abandoned clients expire after 5 minutes. |
| `404 no_client` | The client expired or the bridge restarted. Create a new client. |
| Reply text is garbled | Replies can include binary data. Decode only the text part. |
| `bad baud ...` reply | Don't send `baud` over the network. |
| Raw TCP connection closes immediately | All 3 TCP connections or all 8 client slots are in use. |
| Characters such as `ÿû` in a terminal | Switch the terminal from Telnet to Raw. |
| `Overflow: true` during a transfer | Start the read loop before writing the command, and read continuously. On HD-15, use ACK mode. |
| A lease ends with `timeout` mid-transfer | Keep reading and writing steadily. The transfer was too slow or stalled. |
| An exclusive lease is active, but the RetroTINK replies `busy` | Another transfer is running on the other link. Release and retry later. |
| `503 http_connections_full` | Your app opens too many connections. Reuse one pooled client. |
