# almide-otel-demo

日本語版: [README.ja.md](./README.ja.md)

A from-scratch OpenTelemetry tracing SDK written in pure [Almide](https://github.com/almide/almide) — about 150 lines — plus a two-service demo that produces one distributed trace in Jaeger.

There is no OpenTelemetry Collector and no SDK dependency. Spans are encoded as OTLP/HTTP + JSON and posted straight to Jaeger's native OTLP endpoint. Context crosses the HTTP boundary as a W3C `traceparent` header.

## What you get

Four spans, two processes, one trace:

```
almide-checkout    checkout         59.2ms   root
almide-checkout    GET /inventory   47.8ms   └─ client span
almide-inventory   GET /inventory   29.8ms      └─ server span, linked via traceparent
almide-inventory   SELECT items     29.8ms         └─ simulated DB work
```

## Run it

You need Docker and almide v0.52.0 or newer — the wire types lean on the v0.52.0 Codec semantics (a `none` field omits its key, proto3-style, which is also what the AnyValue oneof rides on):

```bash
curl -fsSL https://raw.githubusercontent.com/almide/almide/main/tools/install.sh | sh
```

Then:

```bash
docker run -d --name jaeger -p 16686:16686 -p 4318:4318 jaegertracing/all-in-one:latest

almide run service_b.almd   # terminal 1: inventory service on :7002
almide run service_a.almd   # terminal 2: checkout — prints the trace id
```

Open http://localhost:16686, pick the `almide-checkout` service, and the waterfall is there.

`minimal.almd` is the single-file starting point: one span, no SDK — the OTLP wire declared as Codec types that mirror it, so the derived encode is the whole exporter.

## Layout

| file | |
|---|---|
| `src/otel.almd` | the SDK: Tracer, spans, id generation, OTLP/HTTP JSON export, traceparent inject/parse |
| `service_a.almd` | checkout — root + client span, injects `traceparent` into the outbound request |
| `service_b.almd` | inventory — extracts `traceparent`, records a server span and a DB child span |
| `minimal.almd` | smallest possible program that puts a span in Jaeger |

## How it works

**Timestamps.** `datetime.now()` is second-precision, which is useless for span durations. The SDK anchors the wall clock once at tracer creation and derives every timestamp from `datetime.monotonic_ns()` — the same trick production OTel SDKs use. OTLP's fixed64 nano fields are JSON *strings*, per the proto3 JSON mapping.

**Propagation.** The client span serializes to `00-{trace_id}-{span_id}-01` and rides the request as a `traceparent` header. The server reads it back with `http.req_header(req, "traceparent")` and parents its span there. That single header is what joins the two processes into one trace.

**Export.** One `POST` per batch to `http://localhost:4318/v1/traces`. A `{"partialSuccess":{}}` response means Jaeger accepted everything.

**The wire is a type.** The OTLP document is declared as Codec types whose field names ARE the wire names (`traceId`, `startTimeUnixNano`), so the derived encode emits the document with no mapping layer. A `none` Option field omits its key — exactly proto3 "unset" — which is why a root span simply has no `parentSpanId`, and why `AnyValue` is a real proto3 oneof: an all-Option mirror type where exactly the alternative you construct hits the wire (almide v0.52.0's encode-omission law, [C-209](https://github.com/almide/almide/blob/main/docs/contracts/contracts.toml)).

## Provenance

Writing this surfaced six almide compiler bugs ([#1049–#1054](https://github.com/almide/almide/issues/1049)), all fixed within a day. The typed handler signature in `service_b.almd` exists because of that batch.

## License

MIT
