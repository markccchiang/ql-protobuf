# QuantLib Service Wire Schema

The Protobuf schema for the interactive QuantLib service: the frames a client
and the pricing backend exchange over a WebSocket, and the market,
instrument, engine and result vocabulary they carry.

This repository holds `.proto` files and nothing else. No generated code is
checked in — each consumer runs `protoc` for its own language, so there is one
source of truth and no stale bindings. It is consumed as a submodule at
`proto/` by [`QuantLib-backend`](https://github.com/markccchiang/QuantLib-backend),
whose `DESIGN.md` records the QuantLib constraints the shapes here are answers
to; the section references in the file comments (`DESIGN §5`, `§6.3`) point
there.

Schema is clean under `protoc 34.0`.

## Layout

```
quantlib/
  v1/
    conventions.proto   day counters, calendars, business-day rules, frequencies
    envelope.proto      the v1 protocol — superseded, kept compiling
  v2/
    envelope.proto      frames, session lifecycle, pricing, sweeps, cancellation
    market.proto        the market namespace: quotes, curves, surfaces, indices
    instrument.proto    payoff x exercise x underlying x style, and the legs
    engine.proto        method x model, and the parameter block each one takes
    results.proto       the Value variant, cash flows, plot series
```

The proto package path is the directory path, so an import reads
`import "quantlib/v2/market.proto";` and the include root is this repository's
top level.

## Versions

`v2` is the live schema. `v1` is the first cut, kept in the tree because it
compiles and because its file comments say what was wrong with it; nothing new
should be built against `quantlib/v1/envelope.proto`.

The break was not additive and could not have been. v1 made the market a
property of the trade (`VanillaOption` carried its own spot, vol and discount
curve ids), wrote one message per product so the count grew as
shape × quanto × exercise, put every engine parameter on every request, and
returned `map<string, double>` for results QuantLib publishes as vectors and
matrices. Fixing any of those changes the meaning of existing field numbers, so
v2 is a new package rather than new fields.

`quantlib/v1/conventions.proto` is the exception, and is imported by v2 rather
than copied. Day counters, calendars, business-day conventions and frequencies
are ISDA facts, not decisions this schema makes; they have no reason to change
when the pricing shapes do, and two copies would have to be kept in step by
hand.

## The rules the shapes follow

**Zero is always `*_UNSPECIFIED`, and the backend rejects it.** Proto3 cannot
distinguish an unset enum from its first value, so a field the client forgot to
set must not resolve to a real convention. This is the difference between a
named error and a silent mispricing.

**Enum values are not QuantLib's own.** `Frequency` alone has `NoFrequency = -1`
and `Once = 0`: one is a negative varint, the other collides with the reserved
zero slot. The backend's registry maps every value by hand and never casts.

**A flag that changes the number is a `Flag`, not a `bool`.** Proto3 cannot tell
an unset bool from a false one, and payer-or-receiver defaulting to one of them
is a sign error. Flags that change only the *reply* — `include_cashflows` — stay
plain bools, because defaulting those to false gives a smaller answer, not a
wrong one.

**Anything the frontend can vary is a quote id, never a literal.** A value
copied into an instrument at construction never invalidates it, which is the
failure mode where the slider moves and the price does not. Where a number
really is constant for the life of the session, `Number.fixed` says so out loud.

**Composition is a `oneof`, not a feature list.** QuantLib has a fixed menu of
instrument classes and there is no quanto-barrier-lookback; a repeated feature
list would promise a product space most of which cannot be built. Quanto sits
outside the `oneof` because it genuinely is orthogonal — `QuantoEngine<Instr,
Engine>` wraps another engine, not another instrument.

## The protocol

One serialized `ClientFrame` or `ServerFrame` per binary WebSocket message.
WebSocket already delimits messages, so there is no length prefix.

Every frame carries a `request_id` chosen by the client, unique per connection
and never reused; the server echoes it on every frame it sends in reply. Every
request gets exactly one frame with `terminal = true`, success or failure,
including a cancel and including a rejection — a client that never sees one
waits forever.

| Client → server | |
| --- | --- |
| `OpenSession` | Evaluation date plus the whole market in dependency order; builds the session's live object graph |
| `UpdateMarket` | Quote writes and fixings, batched so dependents recompute once |
| `PriceRequest` | An instrument, an engine, and what to return besides the NPV |
| `CancelRequest` | Cancels one in-flight `request_id` |
| `CloseSession` | |

| Server → client | |
| --- | --- |
| `SessionOpened` | Session id, bootstrap time, and the market ids actually built |
| `Ack` | Terminal reply to a request with no payload |
| `Progress` | Non-terminal; Monte Carlo path counts and the like |
| `PriceResult` | NPV, requested greeks, cash flows, sampled curves |
| `ScenarioResult` | One price per point of a quote sweep, plus the plot series |
| `Error` | Wire code and the proto field to blame |

A `PriceRequest` carrying a `Scenario` sweeps a single quote — explicit values,
a linear range, or multipliers of the current value — and returns a
`ScenarioResult` instead. That is the point of holding a session open: N lazy
recomputes of only what the quote invalidated, against one graph, rather than N
requests. The sweep restores the quote afterwards unless `keep_final_value`
says otherwise.

`CurveSample` ships curves and surfaces sampled onto a grid alongside the
price, so the frontend draws the term structure the backend priced with rather
than re-implementing QuantLib's interpolation in TypeScript.

## Use as a submodule

```bash
git submodule add https://github.com/markccchiang/ql-protobuf.git proto
git submodule update --init proto
```

In `QuantLib-backend` this lands at `proto/`, which is the `protoc` include
root:

```cmake
add_library(qlservice_proto OBJECT
    proto/quantlib/v1/conventions.proto
    proto/quantlib/v2/market.proto
    proto/quantlib/v2/instrument.proto
    proto/quantlib/v2/engine.proto
    proto/quantlib/v2/results.proto
    proto/quantlib/v2/envelope.proto)

protobuf_generate(
    TARGET qlservice_proto
    IMPORT_DIRS "${CMAKE_CURRENT_SOURCE_DIR}/proto"
    PROTOC_OUT_DIR "${QLSERVICE_PROTO_OUT}")
```

Import paths in generated code keep the `quantlib/v2/` prefix, so the output
directory is added to the include path and headers are included as
`quantlib/v2/envelope.pb.h`.

Pin the consumer to a commit, as with any submodule: a schema change that
compiles is not necessarily a schema change that stays wire-compatible.

## Generating code

Run from this repository's root, or point `-I` at wherever it was checked out.

```bash
protoc -I . --cpp_out=gen      $(find quantlib -name '*.proto')   # C++
protoc -I . --python_out=gen   $(find quantlib -name '*.proto')   # Python
protoc -I . --java_out=gen     $(find quantlib -name '*.proto')   # Java
protoc -I . --go_out=gen       $(find quantlib -name '*.proto')   # Go
```

`option java_multiple_files = true` and `option go_package` are set on every
file. TypeScript is generated with `protoc-gen-ts_proto` or `buf`; the frontend
owns that choice, so no plugin is pinned here.

Generating only `quantlib/v2/envelope.proto` is enough for the v2 protocol —
`protoc` follows the imports and emits the market, instrument, engine, result
and convention files with it.

## Naming

Field names avoid identifiers that are reserved in a generated language even
where they read better: `Scenario.Linear` uses `begin`/`end` rather than
`from`/`to`, because `from` is a keyword in Python and the field there becomes
unreachable or is silently renamed.
