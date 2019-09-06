# Allowing to implement Span fields as special attributes

**Status:** `proposed`

Ease decisions between making something a new field on a Span or a semantic convention for an attribute
and allowing for simpler implementations
by reserving names for fields
and specifying existing "fields" as type-safe APIs for these.

## Motivation

I will motivate this OTEP in three ways: First, I show that there is no clear-cut criteria (at least currently) to decide between special fields and semantic conventions. Then I will present the advantages of semantic conventions and the disadvantages of fields. Lastly, while this PR won't address that issue itself, I will argue that the disadvantages of semantic conventions can be overcome with typed spans.

Note that while this section touches on multiple intertwined problems, this OTEP will only offer a partial solution, with the hope that this can become the basis for a more complete solution in later OTEPs.

### The fine line between fields and attribute conventions

Currently, a Span encapsulates nine different things, that fall into these categories:

* Data that identifies the Span ("primary key"): The SpanContext.
* Data that defines the relationship of the Span with other Spans ("foreign keys"): parent Span or SpanContext, Links.
* Data about the Span: operation name, status, start timestamp, end timestamp, Events, (other) attributes.

Note that another element in the last category is currently ready-to-merge in the OTEP-less [specification PR #226 "Add spankind as an option during span creation"][KindPR], growing it to seven items.

This OTEP will only consider the last category.

The [semantic conventions][] about Spans, defined in their own document in the specification, define more closely how these fields should be set for certain types of operations. However, the `component` attribute is used for the same purpose in each of the currently defined conventions
and the `peer.*` attributes look like they could be useful for more than just gRPC (e.g., why not record the peer address for a HTTP request?).

In summary, the line between a semantic convention and a special field seems to be drawn somewhat arbitrary currently and there is no clearly defined policy for deciding between the two.

### Reasons for semantic conventions

The advantage of a semantic convention is that it can be established more easily, as they are "softer".
A new semantic convention will never break existing code.
At worst, if an existing convention is changed, backends will not be able to fully understand spans that are based on the old convention.

Fields have the the major disadvantage that they are hard or even impossible to add without breaking backwards compatibility and even harder to remove or change.
Also, a lot of special fields are the cause of a lot of boilerplate code (depending on the programming language) that is contagious (SpanData must duplicate the fields and each field needs to be handled specifically in exporters).

### The advantages of special fields

The advantage of mere conventions is also the cause of their disadvantage:
There is nothing that enforces them (they are conventions, not rules)
and Span producers may misinterprets a convention, have a bug that leads to a violation or even deliberately violate it while claiming to abide it.
On the other hand, special fields offer strong type checking (including the possibility to force the user to set a value, like for the name)
and may have useful defaults (timestamps).

These are important advantages and we should try to not get them for all our semantic conventions, not just an arbitrary set of specially blessed ones that have become cast in fields. However, I argue that this responsibility can be moved away from the core Span data object to another component, e.g. a special SpanBuilder classes. [OTEP PR #25][OTEPTypedSpans] was also motivated by this issue and is a direction that should be pursued further.

[OTEPTypedSpans]: https://github.com/open-telemetry/oteps/pull/25
[KindPR]: https://github.com/open-telemetry/opentelemetry-specification/pull/226
[semantic conventions]: https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/data-semantic-conventions.md

## Explanation

A first step, and hopefully an uncontroversial one, towards a solution of this problem is to reserve attributes for each special fields. This way, implementations are allowed to use the attribute map as a backing storage for special fields and hopefully reduce boilerplate code.

To make this useful, this OTEP also proposes that, on the SDK layer, the readable Span(-like) object that is passed to a Span Exporter MUST allow accessing all special fields via the span attribute map (this implies that they MUST also be included when the map is iterated over). Implementations MAY allow filtering out certain sets of fields, e.g. by ensuring that the reserved attributes come first in the iteration and providing special iterators for, e.g.:

* All attributes except the special fields in Vx.y of the OpenTelemetry-Spec
* All attributes except start and end-timestamp
* All attributes except name, start and end-timestamp

## Internal details

I propose to reserve the following attribute keys (taken from the [OpenCensus protobuf definition][OCSpanProto]; see below for prefixes/namespacing):

| Field | Attribute key | Type | Note |
| --- | --- | --- | --- |
| Start timestamp | start_time | timestamp | |
| End timestamp | end_time | timestamp | |
| Operation name | name | string | |
| Status | status | enum | |
| Kind | kind | enum | |

If non-scalar attributes come to be supported, we could also consider reserving the name "events" for a (ordered) mapping from timestamps to maps that represent the event attributes, again with a reserved attribute key "name".

Similarly, we might also consider reserving names for Links and parent Span or even the Span Context, but somewhere that would cross the line to just specifying that a Span is essentially a JSON object with a certain schema.

### Namespacing

To keep the above-specified names unique, they should be in some kind of namespace. If/when nested attributes are supported, just one top-level attribute would need to be reserved. In that case, this OTEP proposes the name "otel.vxyz", where "xyz" is the (first) version of the specification that specifies that attribute (it cannot be the version of the language binding that implements it, because that would make the name dependent on the language from which the telemetry comes, which is something we definitely should avoid).

If nested attributes do *not* exist, then instead the above-defined names shall be prefixed with "otel.vxyz." and all attributes that start with the regex `otel\b` are reserved.

### Types: enums and timestamps

It might be worth considering adding a new attribute type "timestamp" that represents exactly that. Otherwise a nanosecond-precision int64 shall be used.

The enums may be represented as integers or short string identifiers. Since such identifiers are already specified (Ok, Cancelled, NotFound, ...), this OTEP proposes the latter. This also helps human-readability of raw traces.

### Setting a reserved attribute

Setting the defined attributes SHOULD act as if the corresponding setter has been called, if there is any. If there is no setter, or the name is reserved but not assigned to a field, or if the implementation cannot efficiently support calling the setter, it MUST threat such a call as an error and act accordingly (a single equality or `startsWith` check is enough to determine if a name is reserved, no complete list is needed).

## Trade-offs and mitigations

If API implementations start returning start timestamps, etc. when iterating over the attribute map, this might force exporters to manually filter them back out. The suggested special iterators provide a way around this, but may be clunky or inefficient to implement in some programming languages. A partial mitigation would be to at least keep timestamps special-field-only because they are the most likely to require special handling all the time. Another possibility would be to handle only status and kind as proposed in this OTEP, as they seem the most "specialized" attributes.

When setters and reserved attributes coexist on the same Span class, implementation complexity of that class increases. However, as discussed above, Span-using code can become simpler, especially if it needs to handle fields and attributes uniformly. If type-safe setters are moved to a different object, this becomes even less of an issue.

## Prior art and alternatives

Historically, OpenTelemetry eschewed special fields (it did have name, events and start & end timestamps) and [its semantic conventions][OTracSemConv] do in part map one-to-one to the currently specified OpenTelemetry special fields.

On the other hand OpenCensus had a lot more attributes on the Span, which can be seen in the [Protobuf-based protocol definition][OCSpanProto]. In addition to all of the ones currently in OpenTelemetry (including SpanKind), it also had built-in support for stack traces and messages (a special kind of network event).

Note that the ideas of this OTEP were briefly discussed at the Data Formats and Semantic Conventions Meeting on 8/22/2019 (see [meeting notes][datasig])

[OTracSemConv]: https://github.com/opentracing/specification/blob/master/semantic_conventions.md
[OCSpanProto]: https://github.com/census-instrumentation/opencensus-proto/blob/master/src/opencensus/proto/trace/v1/trace.proto#L41-L314
[datasig]: https://docs.google.com/document/d/1D4a5U9nnswAo3mp35rF-q4tRKVFZ9KAAZAY4eZYwZkc/view#heading=h.88ek0il9ey47

## Open questions

Summary of already mentioned ones:

* Should timestamps be excluded from this OTEP?
* Should a new timestamp attribute type be added?
* Should events be included in this OTEP?

## Future possibilities

In the [discussion for OTEP PR #12 "Propose consolidating into a single implementation"][OTEPRmApi], it was suggested to look into proposing a "layered SDK". In that spirit, we could split a core SpanApi class from the rich, type-safe Span that has all kinds of special fields. The rich Span would be implemented directly in the API layer, but internally uses a minimal core SpanApi class. Then vendors (including the OpenTelemetry SDK) only need to implement this far smaller and probably less-changing API surface.

[OTEPRmApi]: https://github.com/open-telemetry/oteps/pull/12#issuecomment-527283844
