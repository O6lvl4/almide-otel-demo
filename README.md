# almide-otel-demo

日本語版: [README.ja.md](./README.ja.md)

A from-scratch OpenTelemetry tracing SDK written in pure [Almide](https://github.com/almide/almide) — about 166 lines across four modules — plus a two-service demo that produces one distributed trace in Jaeger.

There is no OpenTelemetry Collector and no SDK dependency. Spans are encoded as OTLP/HTTP + JSON and posted straight to Jaeger's native OTLP endpoint. Context crosses the HTTP boundary as a W3C `traceparent` header.

## What you get

Four spans, two processes, one trace:

```
almide-checkout    checkout         50.3ms   root
almide-checkout    GET /inventory   37.7ms   └─ client span
almide-inventory   GET /inventory   30.1ms      └─ server span, linked via traceparent
almide-inventory   SELECT items     30.1ms         └─ simulated DB work
```

Attributes arrive as their real proto3 types, not as stringified everything:

```
order.id         = "A-1042"  (string)
order.items      = 2         (int64)
order.total      = 38.5      (float64)
order.express    = true      (bool)
```

And a failed call is a failed span. Stop the inventory service and run checkout alone:

```
GET /inventory   otel.status_code        = ERROR
                 otel.status_description = connection failed: Connection refused (os error 61)
```

## Run it

You need Docker and almide **v0.53.2 or newer**:

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

## Run the tests

The encoder is a pure function, so the wire is asserted without a collector, a socket, or a clock. No Docker needed:

```bash
almide test          # 36 tests
```

They pin the things a tracing SDK gets quietly wrong: that each oneof arm emits exactly one key, that a root span has no `parentSpanId` at all, that a status message appears only on the error arm, that a malformed `traceparent` is rejected rather than repaired, and one golden OTLP document byte for byte.

## Layout

The SDK is 166 lines across four modules, split by what each one is *responsible for*. Three of them are pure — no clock, no socket, no randomness — which is exactly why 31 of the 36 tests are plain value comparisons.

```
span ← w3c
span ← otlp ← otel
```

| module | responsibility | impure? |
|---|---|---|
| `src/span.almd` | what a span IS: the kind/value/status matrices and the span's own verbs | no |
| `src/w3c.almd` | the W3C Trace Context header, inject and parse | no |
| `src/otlp.almd` | the wire: proto3 mirror types and the domain→wire bridges | no |
| `src/otel.almd` | the tracer: clock, id generation, and the socket | **yes** |

| program | |
|---|---|
| `service_a.almd` | checkout — root + client span, injects `traceparent` into the outbound request |
| `service_b.almd` | inventory — extracts `traceparent`, records a server span and a DB child span |
| `minimal.almd` | smallest possible program that puts a span in Jaeger; no SDK, no imports from `src/` |

A service imports `otel` for the tracer, `span` for the span's verbs, and `w3c` for the header. It never imports `otlp` — the wire format is an implementation detail of `export`, and nothing outside that one function names it.

## How it works

**The wire is a type.** The OTLP document is declared as Codec types whose field names ARE the wire names (`traceId`, `startTimeUnixNano`), so the derived encode emits the document with no mapping layer. A `none` Option field omits its key — exactly proto3 "unset" — which is why a root span simply has no `parentSpanId` (almide's encode-omission law, [C-209](https://github.com/almide/almide/blob/main/docs/contracts/contracts.toml)).

**A oneof is a sum type.** proto3's `AnyValue` is a oneof, so in the SDK it is `Str | Int64 | Bool | Double` — exactly one arm exists by construction. On the wire it becomes an all-Option mirror record where exactly one key survives. The bridge between them is a single `match`, and the oneof discipline is that match rather than a runtime check.

**Status is the full matrix.** `Unset | Success | Failure(String)` covers all three proto3 status codes, and only the failure arm carries a message — which is exactly what the protocol says, since `message` is meaningful only when `code = 2`. Because the outbound request's `Result` is matched instead of unwrapped, a failed call still produces a finished, exported span that says why.

**Timestamps.** `datetime.now()` is second-precision, which is useless for span durations. The SDK anchors the wall clock once at tracer creation and derives every timestamp from `datetime.monotonic_ns()` — the same trick production OTel SDKs use. OTLP's fixed64 nano fields are JSON *strings*, per the proto3 JSON mapping.

**Propagation.** The client span serializes to `00-{trace_id}-{span_id}-01` and rides the request as a `traceparent` header. The server reads it back with `http.req_header(req, "traceparent")` and parents its span there. A missing *or* malformed header yields `none`, which starts a new trace rather than corrupting the caller's — every check in `parse_traceparent` is a MUST from the W3C recommendation, including the all-zero ids that the spec defines as invalid.

**Export.** One `POST` per batch to `http://localhost:4318/v1/traces`. A `{"partialSuccess":{}}` response means Jaeger accepted everything.

## The API

One rule decides every signature: a function that **creates** takes the tracer first, a function that **transforms** a span takes the span first. So a span's whole life story is one pipe chain, ending in the single effectful step that stamps the clock.

```almide
let checkout = otel.root(t, "checkout")!
  |> span.attr("order.id", span.Str("A-1042"))
  |> span.attr("order.total", span.Double(38.5))
  |> span.succeed
  |> otel.finish(t)!
```

The module prefixes in that chain are not noise — they say which layer each step belongs to. Everything spelled `span.` is a pure value transform; the two spelled `otel.` are the only ones that touch a clock.

`root` and `child` are the two entry points: `child` takes a `SpanCtx` rather than an `Option`, so a child without a parent is unwritable. The raw `start` — the one that takes `Option[SpanCtx]` — is for the single case where a parent genuinely may not exist, which is the inbound server handler.

## Provenance

Writing the first version surfaced six almide compiler bugs ([#1049–#1054](https://github.com/almide/almide/issues/1049)), all fixed within a day. The typed handler signature in `service_b.almd` exists because of that batch.

Splitting it into modules surfaced six more, filed as [#1087–#1092](https://github.com/almide/almide/issues/1087). **All six are fixed in v0.53.2**, and this repo is written against the fixed compiler rather than around it:

| was | the code now |
|---|---|
| [#1087](https://github.com/almide/almide/issues/1087) convention methods unreachable across an import | `p.encode()`, `x.repr()` and a `[T: Codec]` bound all resolve; `SpanKind` overrides `repr`, so `"${kind}"` prints `client` in any module |
| [#1088](https://github.com/almide/almide/issues/1088) default arguments dropped from an exported signature | `otel.tracer("almide-checkout")` and `otel.root(t, "checkout")` take their defaults |
| [#1089](https://github.com/almide/almide/issues/1089) `json.encode` missing across an import | encoding is one call, everywhere |
| [#1090](https://github.com/almide/almide/issues/1090) `almide fmt` deleted comments inside record bodies | field comments survive a format |
| [#1091](https://github.com/almide/almide/issues/1091) multi-line method chains did not parse | chains stay `\|>`, which reads better here, but both forms work |
| [#1092](https://github.com/almide/almide/issues/1092) a docs example that could not compile | — |

One shape is still open, and the code says so where it bites: `otlp`'s span mirror is `WireSpan`, not `Span`, because the domain module is itself called `span` and its type is `Span` — with both spelled bare, a reference inside that file still resolves to the wrong one ([#1093](https://github.com/almide/almide/issues/1093), [#1094](https://github.com/almide/almide/issues/1094)). Everything else in the OTLP mirror keeps the wire's own names.

## License

MIT
