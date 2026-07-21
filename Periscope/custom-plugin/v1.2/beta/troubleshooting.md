# Troubleshooting Periscope 1.2 Integrations

Use this reference after implementing the [Periscope 1.2 Integration Contract](periscope-integration-contract.md).

> **Version:** These notes apply to Periscope 1.2. The current wire payload does not contain a protocol-version field.

## The source does not appear

Check that:

- Periscope is listening on the configured port;
- the integration is using TCP, not HTTP or WebSocket;
- the host and port are correct;
- each JSON object ends with an LF byte (`0x0A`);
- `clientHello` contains `type`, `deviceName`, and `appName`; and
- a firewall, VPN, or network policy is not blocking the connection.

On a physical device, `127.0.0.1` points to the device itself. Use the Mac's LAN IP address. Emulator networking varies by platform and may require a host alias or forwarded port.

## Events do not appear

Validate the emitted JSON one line at a time. Common causes are:

- an invalid UUID in `id` or `requestID`;
- a missing required field;
- `headers` containing numbers, arrays, or objects instead of string values;
- an unsupported `kind` value;
- an invalid timestamp; or
- a message that was written without its final newline.

Periscope accepts ISO 8601 timestamps with or without fractional seconds, Unix epoch seconds, and Unix epoch milliseconds.

## Started and completed events do not merge

Both phases must use exactly the same `requestID`. Each phase must have a different event `id`.

Use `kind: "started"` for the first phase and `kind: "completed"` for the second. Generate the `requestID` once, store it with the in-flight request, and reuse it for every completion path, including errors and cancellation.

## A source appears more than once

Source identity is derived from the client fields. Keep `deviceName`, `appName`, and `bundleIdentifier` stable for the life of the app process. Avoid values that change per request or connection.

## Bodies are missing or corrupt

- A request body may be a one-shot stream. Reading it for capture can consume it before the real request is sent.
- A response body may be streamed in several chunks. Capture all required chunks before creating the completed event.
- JSON only carries text. Convert binary content to a string representation such as base64.
- Large bodies can increase memory use and socket latency. Truncate or omit them using a documented limit.

The protocol currently has no metadata field for body encoding or truncation.

## Requests repeat indefinitely

The interceptor is intercepting its own forwarded request. Mark captured requests with platform-specific metadata, or use a forwarding client that does not include the interceptor.

## Monitoring slows or breaks application requests

Socket connection, encoding, queueing, and retries must run outside the application's request path. Sending to Periscope should be best-effort: discard or queue monitoring data when necessary, but never fail the original request.

Measure `durationMS` with a monotonic clock. Wall-clock changes can otherwise produce incorrect or negative durations.

## Events are lost after disconnecting

Queue events while disconnected and flush them after the new connection becomes ready. Keep the queue bounded and preserve ordering. Send a new `clientHello` before flushing.

If a socket write fails, return that event to the queue before reconnecting. Expect duplicate delivery to be possible around ambiguous socket failures; stable event IDs make such cases diagnosable.

## Manual smoke test

1. Open Periscope and confirm its listening port.
2. Connect a TCP client to the host and port.
3. Send one LF-terminated `clientHello` message.
4. Send one LF-terminated started event.
5. Send one LF-terminated completed event with the same `requestID`.
6. Confirm that one source and one merged transaction appear.
7. Repeat after disconnecting and reconnecting.

Also test stream framing by splitting one JSON message across multiple socket writes and by sending multiple LF-terminated messages in one write. TCP write boundaries are not message boundaries.

## Compatibility notes

- New integrations should use the `{ "type": "event", "event": ... }` envelope.
- Bare event objects are accepted for backward compatibility.
- Unknown message types and event kinds are not accepted.
- Empty lines are ignored.
- A partial final line remains buffered until its newline arrives.
- The protocol currently has no explicit version field.

Because the connection is unauthenticated and unencrypted, use it only in trusted development environments, redact secrets, and disable capture in production by default.
