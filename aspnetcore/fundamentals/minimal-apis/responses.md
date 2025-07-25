---
title: Create responses in Minimal API applications
author: brunolins16
description: Learn how to create responses for minimal APIs in ASP.NET Core.
ms.author: brolivei
monikerRange: '>= aspnetcore-7.0'
ms.date: 05/09/2025
uid: fundamentals/minimal-apis/responses
---

# How to create responses in Minimal API apps

[!INCLUDE[](~/includes/not-latest-version.md)]

:::moniker range=">= aspnetcore-10.0"

Minimal endpoints support the following types of return values:

1. `string` - This includes `Task<string>` and `ValueTask<string>`.
1. `T` (Any other type) - This includes `Task<T>` and `ValueTask<T>`.
1. `IResult` based - This includes `Task<IResult>` and `ValueTask<IResult>`.

## `string` return values

|Behavior|Content-Type|
|--|--|
| The framework writes the string directly to the response. | `text/plain`

Consider the following route handler, which returns a `Hello world` text. 

:::code language="csharp" source="~/fundamentals/minimal-apis/9.0-samples/Snippets/Program.cs" id="snippet_01":::

The `200` status code is returned with `text/plain` Content-Type header and the following content.

```text
Hello World
```

## `T` (Any other type) return values

|Behavior|Content-Type|
|--|--|
| The framework JSON-serializes the response.| `application/json`

Consider the following route handler, which returns an anonymous type containing a `Message` string property.

:::code language="csharp" source="~/fundamentals/minimal-apis/9.0-samples/Snippets/Program.cs" id="snippet_02":::

The `200` status code is returned with `application/json` Content-Type header and the following content.

```json
{"message":"Hello World"}
```

## `IResult` return values

|Behavior|Content-Type|
|--|--|
| The framework calls [IResult.ExecuteAsync](xref:Microsoft.AspNetCore.Http.IResult.ExecuteAsync%2A).| Decided by the `IResult` implementation.

The `IResult` interface defines a contract that represents the result of an HTTP endpoint. The static [Results](<xref:Microsoft.AspNetCore.Http.Results>) class and the static [TypedResults](<xref:Microsoft.AspNetCore.Http.TypedResults>) are used to create various `IResult` objects that represent different types of responses.

### TypedResults vs Results

The <xref:Microsoft.AspNetCore.Http.Results> and <xref:Microsoft.AspNetCore.Http.TypedResults> static classes provide similar sets of results helpers. The `TypedResults` class is the *typed* equivalent of the `Results` class. However, the `Results` helpers' return type is <xref:Microsoft.AspNetCore.Http.IResult>, while each `TypedResults` helper's return type is one of the `IResult` implementation types. The difference means that for `Results` helpers a conversion is needed when the concrete type is needed, for example, for unit testing. The implementation types are defined in the <xref:Microsoft.AspNetCore.Http.HttpResults> namespace.

Returning `TypedResults` rather than `Results` has the following advantages:

* `TypedResults` helpers return strongly typed objects, which can improve code readability, unit testing, and reduce the chance of runtime errors.
* The implementation type [automatically provides the response type metadata for OpenAPI](/aspnet/core/fundamentals/openapi/aspnetcore-openapi#describe-response-types) to describe the endpoint.

Consider the following endpoint, for which a `200 OK` status code with the expected JSON response is produced.

:::code language="csharp" source="~/tutorials/min-web-api/samples/7.x/todo/Program.cs" id="snippet_11b":::

In order to document this endpoint correctly the extensions method `Produces` is called. However, it's not necessary to call `Produces` if `TypedResults` is used instead of `Results`, as shown in the following code. `TypedResults` automatically provides the metadata for the endpoint.

:::code language="csharp" source="~/tutorials/min-web-api/samples/7.x/todo/Program.cs" id="snippet_112b":::

For more information about describing a response type, see [OpenAPI support in minimal APIs](/aspnet/core/fundamentals/openapi/aspnetcore-openapi#describe-response-types-1).

As mentioned previously, when using `TypedResults`, a conversion is not needed. Consider the following minimal API which returns a `TypedResults` class

:::code language="csharp" source="~/../AspNetCore.Docs.Samples/fundamentals/minimal-apis/samples/MinApiTestsSample/WebMinRouteGroup/TodoEndpointsV1.cs" id="snippet_1":::

The following test checks for the full concrete type:

:::code language="csharp" source="~/../AspNetCore.Docs.Samples/fundamentals/minimal-apis/samples/MinApiTestsSample/UnitTests/TodoInMemoryTests.cs" id="snippet_11" highlight="26":::

Because all methods on `Results` return `IResult` in their signature, the compiler automatically infers that as the request delegate return type when returning different results from a single endpoint. `TypedResults` requires the use of `Results<T1, TN>` from such delegates.

The following method compiles because both [`Results.Ok`](xref:Microsoft.AspNetCore.Http.Results.Ok%2A) and [`Results.NotFound`](xref:Microsoft.AspNetCore.Http.Results.NotFound%2A) are declared as returning `IResult`, even though the actual concrete types of the objects returned are different:

:::code language="csharp" source="~/tutorials/min-web-api/samples/7.x/todo/Program.cs" id="snippet_1a":::

The following method does not compile, because `TypedResults.Ok` and `TypedResults.NotFound` are declared as returning different types and the compiler won't attempt to infer the best matching type:

:::code language="csharp" source="~/tutorials/min-web-api/samples/7.x/todo/Program.cs" id="snippet_111":::

To use `TypedResults`, the return type must be fully declared; when the method is asynchronous, the declaration requires wrapping the return type in a `Task<>`. Using `TypedResults` is more verbose, but that's the trade-off for having the type information be statically available and thus capable of self-describing to OpenAPI:

:::code language="csharp" source="~/tutorials/min-web-api/samples/7.x/todo/Program.cs" id="snippet_1b":::

### Results<TResult1, TResultN>

Use [`Results<TResult1, TResultN>`](/dotnet/api/microsoft.aspnetcore.http.httpresults.results-2) as the endpoint handler return type instead of `IResult` when:

* Multiple `IResult` implementation types are returned from the endpoint handler. 
* The static `TypedResult` class is used to create the `IResult` objects.

This alternative is better than returning `IResult` because the generic union types automatically retain the endpoint metadata. And since the `Results<TResult1, TResultN>` union types implement implicit cast operators, the compiler can automatically convert the types specified in the generic arguments to an instance of the union type. 

This has the added benefit of providing compile-time checking that a route handler actually only returns the results that it declares it does. Attempting to return a type that isn't declared as one of the generic arguments to `Results<>` results in a compilation error.

Consider the following endpoint, for which a `400 BadRequest` status code is returned when the `orderId` is greater than `999`. Otherwise, it produces a `200 OK` with the expected content.

:::code language="csharp" source="~/fundamentals/minimal-apis/9.0-samples/Snippets/Program.cs" id="snippet_03":::

In order to document this endpoint correctly the extension method `Produces` is called. However, since the `TypedResults` helper automatically includes the metadata for the endpoint, you can return the `Results<T1, Tn>` union type instead, as shown in the following code.

:::code language="csharp" source="~/fundamentals/minimal-apis/9.0-samples/Snippets/Program.cs" id="snippet_04":::

<a name="binr10"></a>

### Built-in results

[!INCLUDE [results-helpers](~/fundamentals/minimal-apis/includes/results-helpers.md)]

The following sections demonstrate the usage of the common result helpers.

#### JSON

:::code language="csharp" source="~/fundamentals/minimal-apis/9.0-samples/Snippets/Program.cs" id="snippet_05":::

<xref:Microsoft.AspNetCore.Http.HttpResponseJsonExtensions.WriteAsJsonAsync%2A> is an alternative way to return JSON:

:::code language="csharp" source="~/fundamentals/minimal-apis/7.0-samples/WebMinJson/Program.cs" id="snippet_writeasjsonasync":::

#### Custom Status Code

:::code language="csharp" source="~/fundamentals/minimal-apis/9.0-samples/Snippets/Program.cs" id="snippet_06":::

#### Internal Server Error

:::code language="csharp" source="~/fundamentals/minimal-apis/9.0-samples/Snippets/Program.cs" id="snippet_07":::

The preceding example returns a 500 status code.

#### Problem and ValidationProblem

:::code language="csharp" source="~/fundamentals/minimal-apis/9.0-samples/Snippets/Program.cs" id="snippet_12":::

#### Text

:::code language="csharp" source="~/fundamentals/minimal-apis/9.0-samples/Snippets/Program.cs" id="snippet_08":::

<a name="stream7"></a>

#### Stream

[!code-csharp[](~/fundamentals/minimal-apis/7.0-samples/WebMinAPIs/Program.cs?name=snippet_stream)]

[`Results.Stream`](/dotnet/api/microsoft.aspnetcore.http.results.stream?view=aspnetcore-7.0&preserve-view=true) overloads allow access to the underlying HTTP response stream without buffering. The following example uses [ImageSharp](https://sixlabors.com/products/imagesharp) to return a reduced size of the specified image:

[!code-csharp[](~/fundamentals/minimal-apis/resultsStream/7.0-samples/ResultsStreamSample/Program.cs?name=snippet)]

The following example streams an image from [Azure Blob storage](/azure/storage/blobs/storage-blobs-introduction):

[!code-csharp[](~/fundamentals/minimal-apis/resultsStream/7.0-samples/ResultsStreamSample/Program.cs?name=snippet_abs)]

The following example streams a video from an Azure Blob:

[!code-csharp[](~/fundamentals/minimal-apis/resultsStream/7.0-samples/ResultsStreamSample/Program.cs?name=snippet_video)]

#### Server-Sent Events (SSE)

The [TypedResults.ServerSentEvents](https://source.dot.net/#Microsoft.AspNetCore.Http.Results/TypedResults.cs,051e6796e1492f84) API supports returning a [ServerSentEvents](xref:System.Net.ServerSentEvents) result.

[Server-Sent Events](https://developer.mozilla.org/docs/Web/API/Server-sent_events) is a server push technology that allows a server to send a stream of event messages to a client over a single HTTP connection. In .NET, the event messages are represented as [`SseItem<T>`](/dotnet/api/system.net.serversentevents.sseitem-1) objects, which may contain an event type, an ID, and a data payload of type `T`.

The [TypedResults](xref:Microsoft.AspNetCore.Http.TypedResults) class has a static method called [ServerSentEvents](https://source.dot.net/#Microsoft.AspNetCore.Http.Results/TypedResults.cs,ceb980606eb9e295) that can be used to return a `ServerSentEvents` result. The first parameter to this method is an `IAsyncEnumerable<SseItem<T>>` that represents the stream of event messages to be sent to the client.

The following example illustrates how to use the `TypedResults.ServerSentEvents` API to return a stream of heart rate events as JSON objects to the client:

:::code language="csharp" source="~/fundamentals/minimal-apis/10.0-samples/MinimalServerSentEvents/Program.cs" id="snippet_item" :::

For more information, see the [Minimal API sample app](https://github.com/dotnet/AspNetCore.Docs/blob/main/aspnetcore/fundamentals/minimal-apis/10.0-samples/MinimalServerSentEvents/Program.cs) using the `TypedResults.ServerSentEvents` API to return a stream of heart rate events as string, `ServerSentEvents`, and JSON objects to the client.

#### Redirect

:::code language="csharp" source="~/fundamentals/minimal-apis/9.0-samples/Snippets/Program.cs" id="snippet_09":::

#### File

:::code language="csharp" source="~/fundamentals/minimal-apis/9.0-samples/Snippets/Program.cs" id="snippet_10":::

<a name="httpresultinterfaces7"></a>

### HttpResult interfaces

The following interfaces in the <xref:Microsoft.AspNetCore.Http> namespace provide a way to detect the `IResult` type at runtime, which is a common pattern in filter implementations:

* <xref:Microsoft.AspNetCore.Http.IContentTypeHttpResult>
* <xref:Microsoft.AspNetCore.Http.IFileHttpResult>
* <xref:Microsoft.AspNetCore.Http.INestedHttpResult>
* <xref:Microsoft.AspNetCore.Http.IStatusCodeHttpResult>
* <xref:Microsoft.AspNetCore.Http.IValueHttpResult>
* <xref:Microsoft.AspNetCore.Http.IValueHttpResult%601>

Here's an example of a filter that uses one of these interfaces:

:::code language="csharp" source="~/fundamentals/minimal-apis/7.0-samples/HttpResultInterfaces/Program.cs" id="snippet_filter":::

For more information, see [Filters in Minimal API apps](xref:fundamentals/minimal-apis/min-api-filters) and [IResult implementation types](xref:fundamentals/minimal-apis/test-min-api#iresult-implementation-types).

## Modifying Headers

Use the `HttpResponse` object to modify response headers:

```csharp
app.MapGet("/", (HttpContext context) => {
    // Set a custom header
    context.Response.Headers["X-Custom-Header"] = "CustomValue";

    // Set a known header
    context.Response.Headers.CacheControl = $"public,max-age=3600";

    return "Hello World";
});
```

## Customizing responses

Applications can control responses by implementing a custom <xref:Microsoft.AspNetCore.Http.IResult> type. The following code is an example of an HTML result type:

[!code-csharp[](~/fundamentals/minimal-apis/7.0-samples/WebMinAPIs/ResultsExtensions.cs)]

We recommend adding an extension method to <xref:Microsoft.AspNetCore.Http.IResultExtensions?displayProperty=fullName> to make these custom results more discoverable.

[!code-csharp[](~/fundamentals/minimal-apis/7.0-samples/WebMinAPIs/Program.cs?name=snippet_xtn)]

Also, a custom `IResult` type can provide its own annotation by implementing the <xref:Microsoft.AspNetCore.Http.Metadata.IEndpointMetadataProvider> interface. For example, the following code adds an annotation to the preceding `HtmlResult` type that describes the response produced by the endpoint.

[!code-csharp[](~/fundamentals/minimal-apis/7.0-samples/WebMinAPIs/Snippets/ResultsExtensions.cs?name=snippet_IEndpointMetadataProvider&highlight=1,17-20)]

The `ProducesHtmlMetadata` is an implementation of <xref:Microsoft.AspNetCore.Http.Metadata.IProducesResponseTypeMetadata> that defines the produced response content type `text/html` and the status code `200 OK`.

[!code-csharp[](~/fundamentals/minimal-apis/7.0-samples/WebMinAPIs/Snippets/ResultsExtensions.cs?name=snippet_ProducesHtmlMetadata&highlight=5,7)]

An alternative approach is using the <xref:Microsoft.AspNetCore.Mvc.ProducesAttribute?displayProperty=fullName> to describe the produced response. The following code changes the `PopulateMetadata` method to use `ProducesAttribute`.

:::code language="csharp" source="~/fundamentals/minimal-apis/9.0-samples/Snippets/Program.cs" id="snippet_11":::

## Configure JSON serialization options

By default, minimal API apps use [`Web defaults`](/dotnet/standard/serialization/system-text-json-configure-options#web-defaults-for-jsonserializeroptions) options during JSON serialization and deserialization.

### Configure JSON serialization options globally

Options can be configured globally for an app by invoking <xref:Microsoft.Extensions.DependencyInjection.HttpJsonServiceExtensions.ConfigureHttpJsonOptions%2A>. The following example includes public fields and formats JSON output.

:::code language="csharp" source="~/fundamentals/minimal-apis/7.0-samples/WebMinJson/Program.cs" id="snippet_confighttpjsonoptions" highlight="3-6":::

Since fields are included, the preceding code reads `NameField` and includes it in the output JSON.

### Configure JSON serialization options for an endpoint

To configure serialization options for an endpoint, invoke <xref:Microsoft.AspNetCore.Http.Results.Json%2A?displayProperty=nameWithType> and pass to it a <xref:System.Text.Json.JsonSerializerOptions> object, as shown in the following example:

:::code language="csharp" source="~/fundamentals/minimal-apis/7.0-samples/WebMinJson/Program.cs" id="snippet_resultsjsonwithoptions" highlight="5-6,9":::

As an alternative, use an overload of <xref:Microsoft.AspNetCore.Http.HttpResponseJsonExtensions.WriteAsJsonAsync%2A> that accepts a <xref:System.Text.Json.JsonSerializerOptions> object. The following example uses this overload to format the output JSON:

:::code language="csharp" source="~/fundamentals/minimal-apis/7.0-samples/WebMinJson/Program.cs" id="snippet_writeasjsonasyncwithoptions" highlight="5-6,10":::

## Additional Resources

* <xref:fundamentals/minimal-apis/security>

:::moniker-end

[!INCLUDE[](~/fundamentals/minimal-apis/includes/responses9.md)]

[!INCLUDE[](~/fundamentals/minimal-apis/includes/responses7-8.md)]
