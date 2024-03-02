---
title: "My experience on implementing tracing at scale"
date: 2024-03-02T14:23:00+02:00
draft: false
summary: 
tags: 
- dotnet
- activity
- tracing
- opentelemetry
- observability
- telemetry
- distributed systems
---

Tracing is a telemetry concept that gives us the big picture of what happens when a request is made on a distributed system.  

Several solutions exist to add "automated" tracing. Abstracting and hiding things away from you does indeed make it easier to use, but as soon as you need to adapt something to your specific needs, it will start to be a pain. If you have the human resources and the will to have a proper tracing, my advice is to understand the concepts and implement parts of it yourself. The OpenTelemetry defines standards, semantics, and protocols and provides some libraries and tools to help you with that.

In this article, I'll share my experience 2 years after starting to implement at scale, in a backend handling hundreds of billions of requests per day, and more specifically in .NET.

# The .NET Activity API

Since .NET 6, the [Activity API](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.activity?view=net-6.0) (System.Diagnostics) serves as a built-in and platform-agnostic way to record a trace.  

While the Activity API namings sometimes differ from the OpenTelemetry ones, the concepts are the same.  

Here is a non-exhaustive mapping table:
| Activity API | OpenTelemetry | Comment |
|-|-|-|
| Activity | Span | |
| Activity Tag | Span Attribute | |
| Missing | Span Resource | OpenTelemetry .NET libraries add resources alongside what activities holds when creating spans. |
| Activity Baggage | Missing | I don't recommend using Activity Baggages at all since there isn't really any equivalent on the Opentelemetry side. |
| Activity DisplayName | Span Name | |

## How It Works

This API is a little uncommon, to say the least. I'll spare you the details. To make it simple, it works with 3 components:
- `Activity`: The equivalent of a Span in OpenTelemetry. It represents a unit of work.
- `ActivitySource`: It's used to create Activities.
- `ActivityListener`: It's used to receive callbacks when Activities are created, started, stopped, etc.

**Example →** Listening to Activities
```csharp
// Create a source "MySource"
ActivitySource activitySource = new("MySource");

// Create a listener
ActivityListener activityListener = new()
{
    // This listener only listens to Activities created by sources with name "MySource"
    ShouldListenTo = source => source.Name == activitySource.Name,
    // Any Activity it listens to is marked as "AllDataAndRecorded"
    Sample = (ref ActivityCreationOptions<ActivityContext> _) => ActivitySamplingResult.AllDataAndRecorded
    // Receive callbacks to know when a listened Activity is started or stopped
    ActivityStarted = activity => Console.WriteLine($"Activity {activity.OperationName} started")
    ActivityStopped = activity => Console.WriteLine($"Activity {activity.OperationName} stopped")
};
// Register the listener (yes this is statis 🙄)
ActivitySource.AddActivityListener(activityListener);

// Then anywhere in your code, you can create an Activity
var activity = activitySource.StartActivity("Activity");
// StartActivity automatically sets the Current Activity
Assert.AreEqual(Activity.Current, activity);
// Activity is not null because it's listened by at least one listener
Assert.NotNull(activity);
// Activity is marked as recorded because the listener sets it as "AllDataAndRecorded"
Assert.IsTrue(activity.Recorded);
```

Here you can see that you can, with a few tweaks on this example:
- Listen to sources you are interested in in the `ShouldListenTo` callback. 
- Implement a sampling logic in the `Sample` callback
- Implement the logic to convert the `Activity` to an OpenTelemetry Span in the `ActivityStopped` callback and send it (in batches) to your observability backend.

### Existing Activity Sources

- The source named ["Microsoft.AspNetCore"](https://github.com/dotnet/aspnetcore/blob/c7c76d7d3bb30e29e8386d1a3e6dae9e307fd0de/src/Hosting/Hosting/src/WebHostBuilder.cs#L288) is used by ASP.NET Core to create Activities for incoming HTTP requests.
- The source named ["System.Net.Http"](https://github.com/dotnet/runtime/blob/ce6cf4b2e2a8c7d80ed7b31487eaab574d9007fa/src/libraries/System.Net.Http/src/System/Net/Http/DiagnosticsHandler.cs#L19) is used by `HttpClient` to create Activities for outgoing HTTP requests.

Third-party libraries that are up-to-date with tracing will usually define their own `ActivitySource` and create Activities for their operations.
Unfortunately, not all libraries are up-to-date with tracing, and you might have to create your own `ActivitySource` and create Activities for their operations, sometimes requiring you to monkey-patch the library or make wrappers around it 😢.  
Fortunately, the OpenTelemetry for .NET packages helps you in this situation with what they call "instrumentation". An example is the SQL instrumentation where [they listen to diagnostics events](https://github.com/dotnet/SqlClient/issues/2210) (a different piece of tech than the Activity API) emitted from `Microsoft.Data.Sql`. Hopefully one day more libraries will (properly) implement tracing, and we won't have to rely on third parties to do it for us.

### The OpenTelemetry .NET Libraries

OpenTelemetry has open-sourced a set of libraries to help you with tracing in .NET: https://github.com/open-telemetry/opentelemetry-dotnet.  
They aim to help listen to several sources, simplify setup via DI extensions, and provide exporters to send the traces to your observability backend.  
In my personal opinion, the libraries are still a bit rough around the edges, and you might end up complexifying your code instead. I recommend using the exporters and some instrumentation packages if not trivial to implement yourself, such as the one for `Microsoft.Data.Sql`.

### AsyncLocal Magic

The Activity API uses `AsyncLocal<T>` to store the current `Activity`. `AsyncLocal<T>` is a powerful construct, but it's also a bit of a black box for the uninitiated. It's an async-context-local storage that works with async/await. What makes it very interesting in the context of tracing is that it allows you to create Activities that are bound to an async context without having to refactor all of your code to pass the Activity around. You just use it as if it was a global variable.  

**Example →** Creating Activities in async code
```csharp
Activity.Current = source.StartActivity("A");
await DoWorkAsync();
Console.WriteLine(Activity.Current.OperationName); // "A"

static async Task DoWorkAsync()
{
    Activity.Current = source.StartActivity("B");
    await SetThirdValueAsync();
    Console.WriteLine(Activity.Current.OperationName); // "B"
}

static async Task DoMoreWorkAsync()
{
    Activity.Current = source.StartActivity("C");
    await Task.Delay(1000);
    Console.WriteLine(Activity.Current.OperationName); // "C"
}
```

It is very powerful, but it can also be unintuitive for the non-initiated. Here is an example of a pitfall: https://x.com/MrPeterLMorris/status/1763502114184053198?s=20

### When to Create Activities?

Tracing is meant to get a "big picture" of a distributed system. In this context, while it can be tempting to create an Activity for every method call, I think it's not a good idea. My rule of thumb is to have traces depend on architecture but not code. My advice is to create an Activity only in such cases:
- An incoming I/O, such as a request received in an ASP NET Core service. This way you have the server-side spans.
- An outgoing I/O, such as an HTTP request sent by an `HttpClient`, or an SQL query. This way you want client-side spans.

With client-side and server-side spans and tracing implemented across all services, you can create a map of your services and pinpoint complex issues in distributed systems, such as answering where the latency is coming from.  

Exceptionally, it can make sense to have an `Activity` for some "internal" asynchronous operations. For instance, a buffering middleware may introduce latency, but it is not per se considered an outgoing I/O (or difficult to consider as such).

# Sampling

Sampling is maybe one of the most complex aspects of tracing. It's the process of deciding which traces to keep and which to discard. It's a trade-off between the amount of data you want to collect and the performance overhead of collecting it.  

There are two types of sampling:
- **Head-based sampling**: the decision to sample is made at the beginning of the trace.
- **Tail-based sampling**: the decision to sample is made at the end of the trace.

Tail-based sampling implies that every trace must be built and stored in memory before the decision to sample is made. This is not efficient and can lead to memory issues. Also, tail-based sampling implies that a trace might be sampled depending on some criteria, which would make the traces as a whole not representative of the whole traffic.  

There is a nice write-up on the OpenTelemetry website about sampling: https://opentelemetry.io/docs/concepts/sampling/. However, it can be a bit overwhelming. I think what they call ["Consistent Probability Sampling"](https://opentelemetry.io/docs/specs/otel/trace/tracestate-probability-sampling/#consistent-probability-sampling) is the best approach to sampling for the majority of use cases, but their implementation proposal is overly complex IMHO, so I'll propose my own.

## Consistent Probability Sampling

Because tracing is distributed, it implies that the sampling decision but be consistent across all services taking part in the trace. We could pass a sampling decision from one service to the next, but it implies that the first service would need to know it is indeed the first service in the trace, which can add some complexity. Instead, **we chose to use consistent probability sampling**.

Consistent probability sampling means that the sampling decision is made based on the trace ID and some constant sampling rate. This means that the same trace ID will always be sampled or not sampled, no matter the service. This is achieved by using a hash of the trace ID and the sampling rate.

The hashing must be ubiquitous across all services so that the same trace ID will always be sampled or not sampled. This is why **we chose to use the [FNV-1a](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function) hash function**, which is a non-cryptographic hash function that is easy to implement and is widely used in distributed systems and which should be sufficient for our use case. **The trace ID must be uppercased** before being hashed.

**Example →** FNV-1a-based sampling in C#
```csharp
public static bool Sample(string traceId, uint samplingRate)
{
    uint h = 0x811c9dc5;
    for (var i = 0; i < traceId.Length; i++) {
        h ^= traceId[i];
        h *= 16777619u;
    }
    return h % samplingRate == 0;
}

public static void Main()
{
    Console.WriteLine(Sample("A12C78D8", 2)); // False
    Console.WriteLine(Sample("7129034239FCB", 2)); // True
}
```

## Bypassing The Sampling

In some cases, we might want to bypass the sampling. For example, when we are debugging a specific request, or for test automation purposes, we might want to see to force the sampling for a trace. 

The W3C defines a "sampling flag" for the `traceparent` header: https://www.w3.org/TR/trace-context/#sampled-flag. However, I think relying on it can have collateral effects. For instance, if a B2B customer calls your API with the flag set because he uses tracing on his side, it will force the trace to bypass the sampling on your side.  

I prefer using the `tracestate` header with a value such as `no-sampling` for my vendor name `my-vendor`. Then I can look for this value in my `Sample` delegate and bypass the sampling if it's present.

**Example →** `tracestate: my-vendor=no-sampling,other-vendor=whatever`

# Context propagation

Context propagation is what makes tracing distributed: it's the process of passing the trace context from one service to the next. See the OpenTelemetry documentation about context propagation: https://opentelemetry.io/docs/concepts/context-propagation/.

In .NET, the `Activity` class has a `Context` property that holds the trace context. The OpenTelemetry .NET libraries provide "Propagators" to automatically copy the context into HTTP headers when making outgoing HTTP requests, and to extract the context from incoming HTTP requests. It's nice, but you may not want to inject these headers in every request you perform (EG you call a third-party API, they don't need your tracing headers). Also, it will only work for HTTP requests, which is far from covering all the I/Os you might have in your system (gRPC + protobuf, RabbitMQ, Kafka, ...).  

In my opinion, the best way to propagate the context is to do it manually. You can use the `Activity.Current.GetW3CTraceParent()` extension method I provided at the end of this article to get a string that complies with the W3C `traceparent` header format. Then you can pass this string to the next service in the request, either as a header or in any other form for protocols not covered by OpenTelemetry specifications. The `tracestate` header can be read as is from `Activity.Current.TraceStateString`.

# Bonus

I'd like to share some resources that I found useful when implementing tracing in .NET.

## Useful Extensions

```csharp
// Extension to check if an Activity is sampled before adding tags or events (avoid unecessary work and allocations in case trace is going to be discarded).
public static bool IsRecorded([NotNullWhen(true)] this Activity? activity)
{
    return activity != null && activity.Recorded;
}

// Extension to start an "internal" activity (no I/Os). Automatically picks the name of the caller class and method.
public static Activity StartActivityInternal(this ActivitySource source, [CallerFilePath] string callerFilePathAttribute = "", [CallerMemberName] string callerMemberName = "")
{
    var activity = source.StartActivity(ActivityKind.Internal);
    if (activity.IsRecorded())
    {
        string fileName = Path.GetFileNameWithoutExtension(callerFilePathAttribute);
        activity.DisplayName = $"{fileName}.{callerMemberName}";
    }
    return activity;
}

// Extension to start an "external" activity (involving I/O). Sets a bunch of OpenTelemetry tags to help the tracing backend understand the context and display it properly.
public static Activity StartActivityExternal(this ActivitySource source, string serviceName, string operationKind, bool isDatabase = false)
{
    var activity = source.StartActivity(operationKind, ActivityKind.Client);
    if (activity.IsRecorded())
    {
        // This is a special tag, see https://github.com/open-telemetry/opentelemetry-specification/blob/6ce62202e5407518e19c56c445c13682ef51a51d/specification/trace/semantic_conventions/span-general.md#general-remote-service-attributes
        activity.SetTag("peer.service", serviceName);

        if (isDatabase)
        {
            // This is a special tag, see https://github.com/open-telemetry/opentelemetry-specification/blob/6ce62202e5407518e19c56c445c13682ef51a51d/specification/trace/semantic_conventions/database.md#connection-level-attributes
            activity.SetTag("db.system", serviceName);
        }
    }
    return activity;
}

// Extension a string that complies with the W3C traceparent header format
public static string GetW3CTraceParent(this Activity activity)
{
    return $"00-{activity.Context.TraceId}-{activity.Context.SpanId}-{(byte)activity.Context.TraceFlags:x2}";
}

// Extension to set an Activity in an error state
public static void SetError(this Activity activity, Exception exception, string message = "")
{
    string description = string.IsNullOrEmpty(message) ? string.Empty : $"{message}. ";
    description += $"Exception: {exception}";
    // Extension method from OpenTelemetry libraries that sets some tags
    activity.SetStatus(Status.Error.WithDescription(description));
    // Specific to the Activity API
    activity.SetStatus(ActivityStatusCode.Error, description);
}
```

## Useful Links

- [Grafana Tempo](https://grafana.com/docs/tempo/latest/#tempo-documentation). A distributed tracing backend that is easy to set up and use. Available as a self-hosted solution (open source) or as a managed service (paid). In my opinion, it is the best tracing backend available and follows OpenTelemetry specifications quite well. The TraceQL language is very powerful. It does take some time to set up and learn however.
- [opentelemetry.io](https://opentelemetry.io/). Always a good starting point for learning about tracing.
- [Article on hidden working of AsyncLocal and ExecutionContext](https://medium.com/net-under-the-hood/hidden-workings-of-execution-context-in-net-43b491726c65) by Nakib. A good read to understand the magic behind `AsyncLocal<T>` and `ExecutionContext`.