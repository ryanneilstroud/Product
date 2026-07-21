# Creating a Periscope 1.2 Integration

This guide describes how to build a client library that captures HTTP transactions and sends them to the Periscope 1.2 macOS viewer.

> **Version:** Periscope 1.2. The current wire payload does not contain a protocol-version field.

Read the [Periscope 1.2 Integration Contract](periscope-integration-contract.md) first. It defines the required technology, TCP transport, and JSON payloads. This guide focuses on implementation.

## Components

Keep capture and delivery separate:

1. **Interceptor** observes the platform's HTTP client.
2. **Mapper** converts requests and responses to the Periscope contract.
3. **Transport** owns the TCP connection, framing, queue, and retries.
4. **Public API** starts and stops capture and configures the Periscope host and port.

This separation lets the interceptor remain independent of socket state and makes each component testable.

## Public API

A small integration normally needs:

```text
start(host, port = 61337)
stop()
inject(httpClientConfiguration) // if the platform requires explicit interception
```

Use `127.0.0.1` when the application and Periscope run on the same Mac. A physical device normally needs the Mac's LAN IP address.

Make capture opt-in and disabled in production builds by default. The TCP connection is not authenticated or encrypted.

## Capture lifecycle

When a request begins:

1. Generate a `requestID`.
2. Record the start using a monotonic clock.
3. Snapshot the URL, method, headers, and body.
4. Send a `started` event.

When it finishes or fails:

1. Calculate `durationMS` using the monotonic clock.
2. Snapshot the status, headers, body, and error.
3. Send a `completed` event using the same `requestID`.

Generate a different event `id` for each phase. Periscope merges the phases by `requestID`.

```text
requestID = randomUUID()
startedAt = monotonicClock.now()
request = snapshot(outgoingRequest)

transport.send(startedEvent(requestID, request))

try:
  result = performOriginalRequest()
  response = snapshot(result, elapsedMilliseconds(startedAt))
catch error:
  response = snapshot(error, elapsedMilliseconds(startedAt))
finally:
  transport.send(completedEvent(requestID, request, response))
```

Use a wall clock for the event timestamp and a monotonic clock for duration.

## Interception rules

Monitoring must not alter the application request.

- Preserve the URL, method, headers, body, redirects, cancellation, errors, and streaming behavior.
- Mark forwarded requests so the interceptor does not capture them recursively.
- Do not consume a one-shot body stream unless you replace it with equivalent content.
- Do not wait for Periscope before allowing the application request to continue.
- Never surface monitoring connection or encoding failures to the application.
- Document whether interception is global or limited to configured HTTP clients.

## Mapping data

Convert every header name and value to a string. Use an empty object when no headers are present.

Represent bodies as strings. UTF-8 text can be sent directly. Binary content can be base64 encoded, but the current contract has no field that identifies encoding or truncation, so document any such behavior in the client library.

Apply size limits before copying bodies into events. Provide redaction for at least authorization headers, cookies, access tokens, and user-configured sensitive fields.

For failed requests, send a `completed` event with:

- `statusCode` set to the received status or `null`;
- response headers, or an empty object;
- any available response body;
- a useful `error` string; and
- a non-negative `durationMS`.

## Transport behavior

Use one long-lived TCP connection per app process where practical.

When the connection becomes ready:

1. Send `clientHello`.
2. Flush queued events in their original order.
3. Send new events as newline-terminated JSON objects.

When disconnected:

- queue events in memory up to a fixed limit;
- discard the oldest events when the limit is reached;
- reconnect with capped exponential backoff; and
- stop retrying when capture is stopped.

A reasonable initial policy is a maximum of 500 queued events and retry delays from 250 milliseconds up to 8 seconds.

## Testing

At minimum, cover:

- started and completed events share a `requestID`;
- every event has a unique `id`;
- JSON frames end with LF;
- headers are encoded as string values;
- success, HTTP failure, transport failure, and cancellation;
- request and response bodies, including streams and binary content;
- recursive interception prevention;
- queue limits and event ordering;
- disconnect, reconnect, and a new `clientHello`; and
- monitoring failures never fail the host application's request.

Test once against a running Periscope viewer before publishing the integration.

For unusual failures and protocol edge cases, see [Troubleshooting Periscope Integrations](troubleshooting-periscope-integrations.md).
