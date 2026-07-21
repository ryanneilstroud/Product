# Periscope 1.2 Integration Contract

This is the minimum contract for sending captured HTTP transactions to Periscope 1.2 from any platform.

> **Version:** Periscope 1.2. The current wire payload does not contain a protocol-version field.

## Required technology

Your client library needs:

- an HTTP interceptor or middleware;
- a TCP socket client;
- UTF-8 JSON encoding;
- UUID generation;
- ISO 8601 or Unix timestamps;
- a monotonic clock for durations; and
- a bounded in-memory event queue.

No HTTP server, WebSocket library, database, authentication layer, or Periscope-specific dependency is required.

## Transport

| Setting | Value |
| --- | --- |
| Protocol | TCP |
| Default port | `61337` |
| Local host | `127.0.0.1` |
| Physical device host | Mac's LAN IP address |
| Encoding | UTF-8 |
| Framing | One JSON object followed by LF (`\n`) |

Keep the connection open while capture is active. Send a `clientHello` after connecting, followed by event messages.

## Messages

Client hello:

```json
{
  "type": "clientHello",
  "client": {
    "deviceName": "Pixel 10 Pro",
    "appName": "Example App",
    "bundleIdentifier": "com.example.app"
  }
}
```

Event:

```json
{
  "type": "event",
  "event": {
    "id": "61B13066-7FC1-4F47-BBE7-D9E98E919E7A",
    "kind": "started",
    "timestamp": "2026-07-21T12:34:56.123Z",
    "requestID": "18C4AE53-6250-4873-950B-5E8F3D54C952",
    "request": {
      "url": "https://api.example.com/users",
      "method": "GET",
      "headers": {},
      "body": null
    },
    "response": null,
    "client": {
      "deviceName": "Pixel 10 Pro",
      "appName": "Example App",
      "bundleIdentifier": "com.example.app"
    }
  }
}
```

Append an actual LF byte after each JSON object.

## Definitions

### Event

| Field | Type | Required | Definition |
| --- | --- | --- | --- |
| `id` | UUID string | Yes | Unique ID for this event |
| `kind` | `started` or `completed` | Yes | Request lifecycle phase |
| `timestamp` | date | Yes | Time the event was emitted |
| `requestID` | UUID string | Yes | Correlates both request phases |
| `request` | request object | Yes | Captured request |
| `response` | response object or null | No | Response for a completed event |
| `client` | client object or null | No | Source identity; recommended |

### Request

| Field | Type | Required |
| --- | --- | --- |
| `url` | string | Yes |
| `method` | string | Yes |
| `headers` | string-to-string object | Yes |
| `body` | string or null | No |

### Response

| Field | Type | Required |
| --- | --- | --- |
| `statusCode` | integer or null | No |
| `headers` | string-to-string object | Yes |
| `body` | string or null | No |
| `error` | string or null | No |
| `durationMS` | non-negative integer | Yes |

### Client

| Field | Type | Required |
| --- | --- | --- |
| `deviceName` | string | Yes |
| `appName` | string | Yes |
| `bundleIdentifier` | string or null | No |

## Request lifecycle

For each HTTP request:

1. Generate one `requestID`.
2. Send a `started` event with no response.
3. Perform the HTTP request.
4. Send a `completed` event containing the response or error.
5. Reuse the same `requestID`, but generate a new event `id`.

Periscope merges both events using `requestID`.

Use ISO 8601 UTC timestamps where possible. Unix epoch seconds and milliseconds are also accepted.

## Minimum behavior

- Never fail or delay the application's request when Periscope is unavailable.
- Reconnect after losing the TCP connection and resend `clientHello`.
- Bound the pending event queue.
- Prevent recursive interception.
- Redact credentials and sensitive data before sending.

For implementation guidance and complete examples, see [Creating a Periscope Integration](creating-periscope-integration.md).
