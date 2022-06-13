# Context-scoped attributes

Add Context-scoped telemetry attributes which typically apply to all signals associated with a trace as it crosses a single service.

## Motivation

This OTEP aims to address various related demands that have been brought up in the past, where the scope of resource attributes is too broad, but the scope of span attributes is too narrow. For example, this happens where there is a mismatch between the OpenTelemetry SDK’s (and thus TracerProvider’s, MeterProvider’s) process-wide initialization and the semantic scope of a (sub)service.

A concrete example is posed in open issue [open-telemetry/opentelemetry-specification#335](https://github.com/open-telemetry/opentelemetry-specification/issues/335) “Consider adding a resource (semantic convention) to distinguish HTTP applications (http.app attribute)”.
If the opentelemetry-java agent is injected in a JVM running a classical Java Application Server (such as tomcat), there will be a single resource for the whole JVM.
The application server can host multiple independent applications.
There is currently no way to distinguish between these, other than with the generic HTTP attributes like http.route, http.target.
The issue proposes to add an http.app attribute but it is unclear where this could be placed.
The resource cannot be used, since there is only one that is shared between multiple applications.
A span attribute on the root span could be used.
However, logically, the app attribute would apply to all spans within the trace as it crosses the app server, as shown in the diagram below:

[Diagram showing how two traces cross a single service, having a http.app attribute applied to all spans within that service of each trace](!img/../0207-context-scoped-attributes.md)

This example shows two traces, with two "HTTP GET" root spans, one originating from service `frontend` and another from `user-verification-batchjob`.
Each of these HTTP GET spans calls into a third service `my-teams-tomcat` which is a Java Tomcat application server.
That service hosts two distinct HTTP applications, and as each of the traces crosses it, all the spans have a
respective `http.app` associated with it. When the `my-teams-tomcat` service makes a call to another service `authservice`,
that attribute does *not* apply to the remote child spans.

Using only currently specified mechanisms, the http.app attribute could be set, e.g., on the `/myapp/users/{userid}` root span, but it logically applies also to the `HTTP GET` child span which is executed in the same app.

A similar problem occurs with `faas.id` and `faas.name` ([Function as a Service resource semantic conventions](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/resource/semantic_conventions/faas.md)) on Azure Functions.
While both are defined as resource attributes, an Azure function app hosts multiple co-deployed functions with different names and IDs.
In PR [open-telemetry/opentelemetry-specification#2502](https://github.com/open-telemetry/opentelemetry-specification/pull/2502), this was solved by allowing to set these resource attributes on the FaaS root span instead.
This has the drawback that the context of which function a span is executed in is lost in any child spans that may occur within the function (app).

There is also issue [open-telemetry/opentelemetry-specification#1089](https://github.com/open-telemetry/opentelemetry-specification/issues/1089) “Add BeforeEnd to have a callback where the span is still writeable” which seems to be motivated by essentially the same problem, though the motivation is stated more on a technical level: “What I tried to implement is a feature called trace field in Honeycomb (reference: [AddFieldToTrace](https://godoc.org/github.com/honeycombio/beeline-go#AddFieldToTrace))”, though lately the issue seems to go into a slightly different direction.

This motivation analogously also applies to other signals (metrics, logs) which will have very similar problems. E.g. the http.app attribute also looks like something you may want to split metrics by or use as a query for logs (“I want to see all logs produced by the `http.app=myapp`”).

## Explanation

The context-scoped attributes allows you to attach attributes to all telemetry signals emitted within a Context (or, following the usual rules of Context values, any child context thereof unless overridden).  Context-scoped attributes are normal attributes, which means you can use strings, integers, floating point numbers, booleans or arrays thereof, just like for span or resource attributes. Context-scoped attributes are associated with all telemetry signals emitted while the Context containing the Context-scoped attributes is active and are available to telemetry exporters. For spans, the context within which the span is started applies. Like other telemetry APIs, Context-scoped attributes are write-only for applications. You cannot query the currently set Context-scoped attributes, they are only available on the SDK level (e.g. to telemetry exporters, Samplers and SpanProcessors).

Context-scoped attributes should be thought of equivalent to adding the attribute directly to each single telemetry item it applies to.

### Comparing Context-scoped attributes to Baggage

Context-scoped attributes and [baggage](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/baggage/api.md) have two major commonalities:

* Both are "bags of attributes"
* Both are propagated identically in-process (Baggage also lives on the Context).

However, the use cases are very different: Baggage is meant to transport app-level data along the path of execution, while execution-scoped attributes are annotations for all telemetry in the program execution "below" this point.

This also explains the functional & API differences:

* Most important point: Baggage is meant to be propagated; in fact, this is the main use case of baggage. While a propagator for Context-scoped attributes would be implementable, it would break the intended meaning by extending the scope of attributes to called services and would also raise many security concerns.
* Baggage is both readable and writable for the application, while execution-scoped attributes, like all other telemetry attributes, are write-only.
* Baggage only supports string values.
* Baggage is not available to telemetry exporters (although e.g., a SpanProcessor could be used to change that), while that's the whole purpose of execution-scoped attributes.

### Comparing Context-scoped attributes to Scope Attributes

Context-scoped attributes and the recently added [Scope attributes](https://github.com/open-telemetry/oteps/pull/201) have these commonalities:

* Both have "scope" in the name
* Both apply a set of telemetry attributes to to a set of telemetry items (those in their "scope")

What is different is the scope. While Scope attributes apply to all items emitted by the same Tracer, Meter or LogEmitter (i.e., typically the same telemetry-implementing unit of code),
the scope of Context-scoped attributes is determined at runtime by the Context.

In practice, these scopes will [often be completely orthogonal](#scope-and-context), i.e., as the Context is "propagated" in-process through through a single
service, there will often be exactly one span per Tracer (e.g., one span from the HTTP server instrumentation as the request arrives,
and another one from a HTTP client instrumentation, as a downstream service is called)

I suggest calling Scope Attributes emitter scope attributes for better distinguishability.

## Internal details

This section explains the API-level, SDK-level and exporter-level changes that are expected to implement this.

### API changes

The API would be extended with a variation of the following:

* `Context AddContextScopedAttributes(Context context, Attributes attributes)`

`AddContextScopedAttributes` takes a set of attributes and a Context and returns a new Context that has the given context attributes set in addition to any already set ones.
If the context already contains any attributes with that name, they MUST be overwritten with the attributes of the later call.
If an associated telemetry item already has an attribute with a name that is also in the Context-scoped attributes, the telemetry attribute MUST take precedence.
Since the typical use case for this function is expected to be just before a local root span for the trace is created, no particular care needs to be taken to optimize merging of attributes from calls to this function on a Context that already has Context-scoped attributes.
As for all APIs, care should be taken to expose it in a way that API-users do not bind themselves to a particular SDK implementation, such as the OpenTelemetry default SDK (for example, by going through a `GlobalOpenTelemetry.GetInstance()` or `GlobalOpenTelemetry.GetInstance().GetContextAttributeWriter()` singleton that returns an interface implementation).
As this is a write-only API, the API-only implementation should do nothing at all.

### SDK changes

The SDK-implementation could look like this:

```C#
Context AddContextScopedAttributes(Context context, Attributes attributes) {
    return context.WithValue(
        _CTX_KEY_SCOPED_ATTRS,
        MergeAttributes(context.GetValue(_CTX_KEY_SCOPED_ATTRS), attributes));
}
```

The Context-scoped attributes must be available for all telemetry items they are logically associated with.
For example, the `ReadableSpan` should include the context attributes in the list of span attributes, and they must be included into the list of attributes that the sampler receives.
This will be an implementation-level change without any changes in the API-surface of the SDK (i.e., it is not necessary to make context-scoped attributes distinguishable from “direct” telemetry attributes).

### Exporter changes (none required)

Exporters should set the attributes on all telemetry items they belong to as if they were set directly on them.
For example, a span with span attribute `foo=bar` and Context attribute `x=y` would be exported with span attributes `foo=bar, x=y`.
This should happen by default as the telemetry items will include the context-scoped attributes among their “direct” attributes.

## Trade-offs and mitigations

This OTEP suggests making context-scoped attributes indistinguishable from attributes set directly on the telemetry items.
This should ease implementation on both backends and OpenTelemetry client libraries, and hopefully also reduces conceptual complexity, but it does makes the feature less powerful.
No use case is known to the proposer of the OTEP that would require this power though.
Nevertheless, some implications of making the Context-scoped attributes distinguishable are explored in [Prior art and alternatives](#prior-art-and-alternatives).

This OTEP relies on the [`Context`](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/context/README.md) concept that has been in the spec since 1.0.
However, at least .NET went 1.0 before the spec and does not include a conforming Context concept.
For .NET, an async-local variable that holds the Context-scoped attributes seems to be the most useful implementation of this OTEP (similar to how Baggage is implemented there).

## Prior art and alternatives

### Alternative: Making Context-scoped attributes semantically distinct from other telemetry attributes

As explained in trace-offs and mitigations, this alternative was not selected for the main text, because it seems to add more effort and complexity than value.

#### SDK-changes for semantically distinct Context-scoped attributes

The context-scoped attributes would not be included with the other telemetry attributes.
Instead, there would be separate getters for the context-scoped attributes on all telemetry items.

Additionally, at the SDK-level, the following could be provided, mainly for access from samplers:

* `Attributes GetContextScopedAttributes(Context context)`

Alternatively, a new argument could be added to the sampler input (languages that implement the sampler arguments with a single object with multiple properties can implement this in a backwards-compatible way).

While in theory, applications and instrumentations could take a dependency on the SDK, and call this function from anywhere, this is strongly discouraged and not guaranteed to work.
Languages may even opt for an unorthodox way to expose this that only works from within SDK callbacks (e.g. by having a separate type SdkContext / ReadableContext passed to them).

#### Exporter changes for semantically distinct Context-scoped attributes

Exporters would need to explicitly handle these attributes. As most protocols won’t have support for this concept, the output should probably end up the same as with the indistinguishable attributes.

#### OTLP protocol changes for semantically distinct Context-scoped attributes (optional)

With the implementation suggested in the main section of the OTEP, there is no need to change the OTLP protocol for this feature, as the Context attributes should semantically be equivalent to adding the attribute to every associated telemetry item.

If context attributes were distinguishable, the OTLP protocol could be extended with special support for context-scoped attributes. The scope of Context-scoped attributes is orthogonal to the existing hierarchy of Resource -> (Emitter)Scope -> item. Thus, each single telemetry item needs to get a new field with its context-scoped attributes. This could be just a repeated opentelemetry.proto.common.v1.KeyValue contextAttributes

If a more deduplicated representation is desired, it could alternatively be an index into a list of Context-scoped attributes that is sent once in the root message to be shared among multiple telemetry items.

```protobuf
message TracesData {
    // ...
    repeated opentelemetry.proto.common.v1.KeyValueList contexts = 12345;
}
```

As mentioned in the API-changes, it is expected that no particular care has to be taken to optimize for merging attributes, so a repeated KeyValueList seems to fit the bill.
If a need for such optimization is found, a representation of a tree-structure of these Contexts might be needed, e.g., by storing a dedicated submessage that also contains a parent index.
This could become complex to implement in the exporter though, and generic compression (e.g., gzip) may be more effective.

<a name="scope-and-context"></a>

Yet another alternative, that would also have implications for semantics, would be to merge Context-scoped attributes with the Resource of all associated telemetry items, or with the emitter scope (formerly instrumentation library info, currently only named “Scope”), instead of their own attributes.
This would mean that every distinct set of Context-scoped attributes gets sent within a distinct ResourceSpans/ScopeSpans message, duplicating the resource and (emitter-)scope attributes that many times (a cross-product, if you will).
This approach seems semantically interesting, but could lead to larger export messages, especially if we assume that very often there will only be one span of the same emitter-scope per Context-scope (e.g., you typically will only have one single Span produced by the HTTP server instrumentation as the trace crosses a single service).

#### Alternative: Associating Context-scoped attributes with the span on end and setting attributes on parent Contexts

This alternative seems to be a valid option which is just as easy to implement as the approach in the main context, but it feels a bit less natural.

For spec-conforming implementations of tracing (which excludes .NET in this case), ending/starting a span is independent from setting it as active in a Context.
This allows ending a span in a different context from starting it in.
One use case this could enable is pushing attributes up to a parent span.
For example, an incoming web request might have a low-level HTTP span as root, which might then be dispatched to a HTTP handler method that logically is a Function as a Service span.
In this case, one might want to associate the HTTP parent span with the FaaS function (faas.id, faas.name, …) as well.
This could be implemented by leaving the context with these context-scoped attributes active after returning from the handler method.

This approach would be more similar to what is suggested in [open-telemetry/opentelemetry-specification#1089](https://github.com/open-telemetry/opentelemetry-specification/issues/1089).
However, the approach used in the motivation for that issue seems to go even further and makes pushing attributes up to the local root span the default.
See the [next alternative](#alternative-with-prior-art-bubbling-attributes-up-to-the-root-span).

### Alternative with prior art: Bubbling attributes up to the root span

This alternative is explored because it came up in a spec issue before, but it seems to be quite complex and not clearly any more useful than the semantics in the main OTEP.

[open-telemetry/opentelemetry-specification#1089](https://github.com/open-telemetry/opentelemetry-specification/issues/1089) implements “trace-fields” (trace attributes) and seems to be based on a shared mutable set of Attributes created together with the root span instead of having an immutable set per (child) Context. If we wanted these semantics, that would probably warrant a modified API replacing AddContextScopedAttributes:

* `Context CreateContextScopedRoot(Context)`
* `void SetAttributes(Context context, Attributes newAttributes)`

CreateContextScopedRoot would return a new context with an empty set of mutable attributes.
Child contexts would inherit a reference to the same mutable set. Additionally, it would seem appropriate that setting a span as active in a context that has neither an active span, nor a mutable attribute set, would implicitly also CreateContextScopedRoot.

These semantics seem to be more complex and prone to surprises though.
A typical “failure mode” when trying to associate attributes with the whole trace (or rather whole local part of the trace within a service) including parent spans would be when some in-process parent span has already ended (e.g. asynchronous fire-and-forget invocation), or if the parent span has created another child span that already ended at the time SetAttributes is called.

One also needs to define what happens when attributes are added after a span has ended (or a metric was recorded) but before it is exported.
Probably you would want to have a snapshot of the mutable attributes at time of emitting the telemetry item (ending the span).

Another difficult question would be what to do if SetAttributes is called with a context that does not have mutable attributes yet.
Should this case be prevented by having every child of the root context eagerly create mutable attributes? Should mutable attributes be created in-place, modifying the Context (deliberately made impossible with the current context API, where setting a key always returns a new Context).
Should it always return (new) Context then?
Should it be a no-op?

Do note that with the `CreateContextScopedRoot` API, it would be possible to use this approach even independently of the tracing signal, just as the main OTEP.

### Alternative: Associating attributes through indirectly through spans

This does not seem particularly useful, but the existence of this alternative should serve as a reminder that the trace structure might be different from the (child) context structure.

If going for the [“Making context-scoped attributes semantically distinct from other telemetry attributes” alternative](#alternative-making-context-scoped-attributes-semantically-distinct-from-other-telemetry-attributes), a possibility would pop up to store context-scoped attributes not on the context, but on spans (in a dedicated property).
Other telemetry signals could still go through the active span to associate subtrace-scoped attributes.
The (subtle?) difference would be that these attributes follow the trace tree instead of the context tree.

#### Alternative: An implementation for traces on the Backend

A receiver of trace data (and only trace data!) has information about the execution flow through the parent-child relationship of spans.
It could use that to propagate/copy attributes from the parent span to children after receiving them.
However, there are several problems/differences in what is possible:

* The backend needs to determine which attributes only apply to a particular span, and which apply to a whole subtrace.
  In absence of a dedicated API that allows instrumentation libraries to add this information, it can only use pre-determined rules (e.g. fixed list of attribute keys) for that.
* Spans are only sent (by usual SpanProcessor+Exporter pipelines) when they are ended, meaning that the parent will usually arrive after the children.
  This complicates processing, and can make additional buffering and/or reprocessing (e.g. repeated on-read processing) necessary.
* To only propagate the attributes within a service, [open-telemetry/oteps#182](https://github.com/open-telemetry/oteps/pull/182) (sending the `IsRemote` property of the (parent) span Context in the OTLP protocol) would be required.
  This would also require using OTLP, or another protocol (is there any?) that transports this information.
  Also, the OTLP exporters & language SDKs used in the participating services need to actually implement the feature.
  In absence of that, the span kind (server/consumer as child of client/producer) would be the only heuristic to try working around this.

Additionally, Context-scoped attributes would have the advantage of being usable for metrics and logs as well.
The backend could alternatively try to re-integrate attributes from matched spans, if it's a single backend for all signals, but this would probably become even more costly to implement.

## Open questions

None known at the moment (but see [Trade-offs and mitigations](#trade-offs-and-mitigations) and [Prior art and alternatives](#prior-art-and-alternatives)).

## Future possibilities

With the changes implemented in this OTEP, we will hopefully be unblock some long-standing specification issues. See [Motivation](#motivation).
