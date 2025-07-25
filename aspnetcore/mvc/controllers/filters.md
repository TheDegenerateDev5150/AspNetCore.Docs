---
title: Filters in ASP.NET Core
author: tdykstra
description: Learn how filters work and how to use them in ASP.NET Core.
monikerRange: '>= aspnetcore-3.1'
ms.author: tdykstra
ms.custom: mvc
ms.date: 6/20/2023
uid: mvc/controllers/filters
---
# Filters in ASP.NET Core

:::moniker range=">= aspnetcore-8.0"

By [Kirk Larkin](https://github.com/serpent5), [Rick Anderson](https://twitter.com/RickAndMSFT), [Tom Dykstra](https://github.com/tdykstra/), and [Steve Smith](https://ardalis.com/)

*Filters* in ASP.NET Core allow code to run before or after specific stages in the request processing pipeline.

Built-in filters handle tasks such as:

* Authorization, preventing access to resources a user isn't authorized for.
* Response caching, short-circuiting the request pipeline to return a cached response.

Custom filters can be created to handle cross-cutting concerns. Examples of cross-cutting concerns include error handling, caching, configuration, authorization, and logging. Filters avoid duplicating code. For example, an error handling exception filter could consolidate error handling.

This document applies to Razor Pages, API controllers, and controllers with views. Filters don't work directly with [Razor components](xref:blazor/components/index). A filter can only indirectly affect a component when:

* The component is embedded in a page or view.
* The page or controller and view uses the filter.

## How filters work

Filters run within the *ASP.NET Core action invocation pipeline*, sometimes referred to as the *filter pipeline*. The filter pipeline runs after ASP.NET Core selects the action to execute:

:::image source="~/mvc/controllers/filters/_static/filter-pipeline-1.png" alt-text="The request is processed through Other Middleware, Routing Middleware, Action Selection, and the Action Invocation Pipeline. The request processing continues back through Action Selection, Routing Middleware, and various Other Middleware before becoming a response sent to the client.":::

### Filter types

Each filter type is executed at a different stage in the filter pipeline:

* [Authorization filters](#authorization-filters):

  * Run first.
  * Determine whether the user is authorized for the request.
  * Short-circuit the pipeline if the request is not authorized.

* [Resource filters](#resource-filters):

  * Run after authorization.
  * <xref:Microsoft.AspNetCore.Mvc.Filters.IResourceFilter.OnResourceExecuting%2A> runs code before the rest of the filter pipeline. For example, `OnResourceExecuting` runs code before model binding.
  * <xref:Microsoft.AspNetCore.Mvc.Filters.IResourceFilter.OnResourceExecuted%2A> runs code after the rest of the pipeline has completed.

* [Action filters](#action-filters):

  * Run immediately before and after an action method is called.
  * Can change the arguments passed into an action.
  * Can change the result returned from the action.
  * Are **not** supported in Razor Pages.

* [Endpoint filters](/aspnet/core/fundamentals/minimal-apis/min-api-filters):

  * Run immediately before and after an action method is called.
  * Can change the arguments passed into an action.
  * Can change the result returned from the action.
  * Are **not** supported in Razor Pages.
  * Can be invoked on both actions and route handler-based endpoints.

* [Exception filters](#exception-filters):
  * Apply global policies to unhandled exceptions that occur before the response body has been written to.
  * Run after model binding and action filters, but before the action result is executed.
  * Run only if an unhandled exception occurs during action execution or action result execution.
  * Do not run for exceptions thrown during middleware execution, routing, or model binding.

* [Result filters](#result-filters):
  
  * Run immediately before and after the execution of action results.
  * Run only when the action method executes successfully.
  * Are useful for logic that must surround view or formatter execution.

Razor Pages also support [Razor Page filters](xref:razor-pages/filter), which run before and after a Razor Page handler.

## Implementation

Filters support both synchronous and asynchronous implementations through different interface definitions.

Synchronous filters run before and after their pipeline stage. For example, <xref:Microsoft.AspNetCore.Mvc.Filters.IActionFilter.OnActionExecuting%2A> is called before the action method is called. <xref:Microsoft.AspNetCore.Mvc.Filters.IActionFilter.OnActionExecuted%2A> is called after the action method returns:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Filters/SampleActionFilter.cs" id="snippet_Class":::

Asynchronous filters define an `On-Stage-ExecutionAsync` method. For example, <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncActionFilter.OnActionExecutionAsync%2A>:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Filters/SampleAsyncActionFilter.cs" id="snippet_Class":::

In the preceding code, the `SampleAsyncActionFilter` has an <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutionDelegate>, `next`, which executes the action method.

### Multiple filter stages

Interfaces for multiple filter stages can be implemented in a single class. For example, the <xref:Microsoft.AspNetCore.Mvc.Filters.ActionFilterAttribute> class implements:

* Synchronous: <xref:Microsoft.AspNetCore.Mvc.Filters.IActionFilter> and <xref:Microsoft.AspNetCore.Mvc.Filters.IResultFilter>
* Asynchronous: <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncActionFilter> and <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncResultFilter>
* <xref:Microsoft.AspNetCore.Mvc.Filters.IOrderedFilter>

Implement **either** the synchronous or the async version of a filter interface, **not** both. The runtime checks first to see if the filter implements the async interface, and if so, it calls that. If not, it calls the synchronous interface's method(s). If both asynchronous and synchronous interfaces are implemented in one class, only the async method is called. When using abstract classes like <xref:Microsoft.AspNetCore.Mvc.Filters.ActionFilterAttribute>, override only the synchronous methods or the asynchronous methods for each filter type.

### Built-in filter attributes

ASP.NET Core includes built-in attribute-based filters that can be subclassed and customized. For example, the following result filter adds a header to the response:

<a name="response-header-attribute"></a>

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Filters/ResponseHeaderAttribute.cs" id="snippet_Class":::

Attributes allow filters to accept arguments, as shown in the preceding example. Apply the `ResponseHeaderAttribute` to a controller or action method and specify the name and value of the HTTP header:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Controllers/ResponseHeaderController.cs" id="snippet_ClassIndex" highlight="1":::

Use a tool such as the [browser developer tools](https://developer.mozilla.org/docs/Learn/Common_questions/What_are_browser_developer_tools) to examine the headers. Under **Response Headers**, `filter-header: Filter Value` is displayed.

The following code applies `ResponseHeaderAttribute` to both a controller and an action:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Controllers/ResponseHeaderController.cs" id="snippet_Class" highlight="1,9":::

Responses from the `Multiple` action include the following headers:

* `filter-header: Filter Value`
* `another-filter-header: Another Filter Value`

Several of the filter interfaces have corresponding attributes that can be used as base classes for custom implementations.

Filter attributes:

* <xref:Microsoft.AspNetCore.Mvc.Filters.ActionFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.Filters.ExceptionFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.Filters.ResultFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.FormatFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.TypeFilterAttribute>

Filters cannot be applied to Razor Page handler methods. They can be applied either to the Razor Page model or globally.

## Filter scopes and order of execution

A filter can be added to the pipeline at one of three *scopes*:

* Using an attribute on a controller or Razor Page.
* Using an attribute on a controller action. Filter attributes cannot be applied to Razor Pages handler methods.
* Globally for all controllers, actions, and Razor Pages as shown in the following code:
  :::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Program.cs" id="snippet_GlobalFilter" highlight="6":::

### Default order of execution

When there are multiple filters for a particular stage of the pipeline, scope determines the default order of filter execution. Global filters surround class filters, which in turn surround method filters.

As a result of filter nesting, the *after* code of filters runs in the reverse order of the *before* code. The filter sequence:

* The *before* code of global filters.
  * The *before* code of controller filters.
    * The *before* code of action method filters.
    * The *after* code of action method filters.
  * The *after* code of controller filters.
* The *after* code of global filters.

The following example illustrates the order in which filter methods run for synchronous action filters:

| Sequence | Filter scope | Filter method       |
|:--------:|:------------:|:-------------------:|
| 1        | Global       | `OnActionExecuting` |
| 2        | Controller   | `OnActionExecuting` |
| 3        | Action       | `OnActionExecuting` |
| 4        | Action       | `OnActionExecuted`  |
| 5        | Controller   | `OnActionExecuted`  |
| 6        | Global       | `OnActionExecuted`  |

### Controller level filters

Every controller that inherits from <xref:Microsoft.AspNetCore.Mvc.Controller> includes the <xref:Microsoft.AspNetCore.Mvc.Controller.OnActionExecuting%2A>, <xref:Microsoft.AspNetCore.Mvc.Controller.OnActionExecutionAsync%2A>, and <xref:Microsoft.AspNetCore.Mvc.Controller.OnActionExecuted%2A> methods. These methods wrap the filters that run for a given action:

* `OnActionExecuting` runs before any of the action's filters.
* `OnActionExecuted` runs after all of the action's filters.
* `OnActionExecutionAsync` runs before any of the action's filters. Code after a call to `next` runs after the action's filters.

The following `ControllerFiltersController` class:

* Applies the `SampleActionFilterAttribute` (`[SampleActionFilter]`) to the controller.
* Overrides `OnActionExecuting` and `OnActionExecuted`.

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Controllers/ControllerFiltersController.cs" id="snippet_Class" highlight="1,4,12":::

Navigating to `https://localhost:<port>/ControllerFilters` runs the following code:

* `ControllerFiltersController.OnActionExecuting`
  * `GlobalSampleActionFilter.OnActionExecuting`
    * `SampleActionFilterAttribute.OnActionExecuting`
      * `ControllerFiltersController.Index`
    * `SampleActionFilterAttribute.OnActionExecuted`
  * `GlobalSampleActionFilter.OnActionExecuted`
* `ControllerFiltersController.OnActionExecuted`

Controller level filters set the [Order](https://github.com/dotnet/AspNetCore/blob/main/src/Mvc/Mvc.Core/src/Filters/ControllerActionFilter.cs#L15-L17) property to `int.MinValue`. Controller level filters can **not** be set to run after filters applied to methods. Order is explained in the next section.

For Razor Pages, see [Implement Razor Page filters by overriding filter methods](xref:razor-pages/filter#implement-razor-page-filters-by-overriding-filter-methods).

### Override the default order

The default sequence of execution can be overridden by implementing <xref:Microsoft.AspNetCore.Mvc.Filters.IOrderedFilter>. `IOrderedFilter` exposes the <xref:Microsoft.AspNetCore.Mvc.Filters.IOrderedFilter.Order> property that takes precedence over scope to determine the order of execution. A filter with a lower `Order` value:

* Runs the *before* code before that of a filter with a higher value of `Order`.
* Runs the *after* code after that of a filter with a higher `Order` value.

In the [Controller level filters](#controller-level-filters) example, `GlobalSampleActionFilter` has global scope so it runs before `SampleActionFilterAttribute`, which has controller scope. To make `SampleActionFilterAttribute` run first, set its order to `int.MinValue`:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Snippets/Controllers/ControllerFiltersController.cs" id="snippet_Class" highlight="1":::

To make the global filter `GlobalSampleActionFilter` run first, set its `Order` to `int.MinValue`:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Snippets/Program.cs" id="snippet_AddFilterOrder" highlight="3":::

## Cancellation and short-circuiting

The filter pipeline can be short-circuited by setting the <xref:Microsoft.AspNetCore.Mvc.Filters.ResourceExecutingContext.Result> property on the <xref:Microsoft.AspNetCore.Mvc.Filters.ResourceExecutingContext> parameter provided to the filter method. For example, the following Resource filter prevents the rest of the pipeline from executing:

<a name="short-circuiting-resource-filter"></a>

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Filters/ShortCircuitingResourceFilterAttribute.cs" id="snippet_Class" highlight="5-8":::

In the following code, both the `[ShortCircuitingResourceFilter]` and the `[ResponseHeader]` filter target the `Index` action method. The `ShortCircuitingResourceFilterAttribute` filter:

* Runs first, because it's a Resource Filter and `ResponseHeaderAttribute` is an Action Filter.
* Short-circuits the rest of the pipeline.

Therefore the `ResponseHeaderAttribute` filter never runs for the `Index` action. This behavior would be the same if both filters were applied at the action method level, provided the `ShortCircuitingResourceFilterAttribute` ran first. The `ShortCircuitingResourceFilterAttribute` runs first because of its filter type:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Controllers/ShortCircuitingController.cs" id="snippet_Class":::

## Dependency injection

Filters can be added by type or by instance. If an instance is added, that instance is used for every request. If a type is added, it's type-activated. A type-activated filter means:

* An instance is created for each request.
* Any constructor dependencies are populated by [dependency injection](xref:fundamentals/dependency-injection) (DI).

Filters that are implemented as attributes and added directly to controller classes or action methods cannot have constructor dependencies provided by [dependency injection](xref:fundamentals/dependency-injection) (DI). Constructor dependencies cannot be provided by DI because attributes must have their constructor parameters supplied where they're applied. 

The following filters support constructor dependencies provided from DI:

* <xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.TypeFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory> implemented on the attribute.

The preceding filters can be applied to a controller or an action.

Loggers are available from DI. However, avoid creating and using filters purely for logging purposes. The [built-in framework logging](xref:fundamentals/logging/index) typically provides what's needed for logging. Logging added to filters:

* Should focus on business domain concerns or behavior specific to the filter.
* Should **not** log actions or other framework events. The built-in filters already log actions and framework events.

### ServiceFilterAttribute

Service filter implementation types are registered in `Program.cs`. A <xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute> retrieves an instance of the filter from DI.

The following code shows the `LoggingResponseHeaderFilterService` class, which uses DI:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Filters/LoggingResponseHeaderFilterService.cs" id="snippet_Class" highlight="5-6":::

In the following code, `LoggingResponseHeaderFilterService` is added to the DI container:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Program.cs" id="snippet_ResponseHeaderFilterService":::

In the following code, the `ServiceFilter` attribute retrieves an instance of the `LoggingResponseHeaderFilterService` filter from DI:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Controllers/FilterDependenciesController.cs" id="snippet_ServiceFilter" highlight="1":::

When using `ServiceFilterAttribute`, setting <xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute.IsReusable?displayProperty=nameWithType>:

* Provides a hint that the filter instance *may* be reused outside of the request scope it was created within. The ASP.NET Core runtime doesn't guarantee:
  * That a single instance of the filter will be created.
  * The filter will not be re-requested from the DI container at some later point.
* Shouldn't be used with a filter that depends on services with a lifetime other than singleton.

 <xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute> implements <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory>. `IFilterFactory` exposes the <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory.CreateInstance%2A> method for creating an <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterMetadata> instance. `CreateInstance` loads the specified type from DI.

### TypeFilterAttribute

<xref:Microsoft.AspNetCore.Mvc.TypeFilterAttribute> is similar to <xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute>, but its type isn't resolved directly from the DI container. It instantiates the type by using <xref:Microsoft.Extensions.DependencyInjection.ObjectFactory?displayProperty=fullName>.

Because `TypeFilterAttribute` types aren't resolved directly from the DI container:

* Types that are referenced using the `TypeFilterAttribute` don't need to be registered with the DI container. They do have their dependencies fulfilled by the DI container.
* `TypeFilterAttribute` can optionally accept constructor arguments for the type.

When using `TypeFilterAttribute`, setting <xref:Microsoft.AspNetCore.Mvc.TypeFilterAttribute.IsReusable?displayProperty=nameWithType>:
* Provides hint that the filter instance *may* be reused outside of the request scope it was created within. The ASP.NET Core runtime provides no guarantees that a single instance of the filter will be created.

* Should not be used with a filter that depends on services with a lifetime other than singleton.

The following example shows how to pass arguments to a type using `TypeFilterAttribute`:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Controllers/FilterDependenciesController.cs" id="snippet_TypeFilter" highlight="1-2":::

## Authorization filters

Authorization filters:

* Are the first filters run in the filter pipeline.
* Control access to action methods.
* Have a before method, but no after method.

Custom authorization filters require a custom authorization framework. Prefer configuring the authorization policies or writing a custom authorization policy over writing a custom filter. The built-in authorization filter:

* Calls the authorization system.
* Does not authorize requests.

Do **not** throw exceptions within authorization filters:

* The exception will not be handled.
* Exception filters will not handle the exception.

Consider issuing a challenge when an exception occurs in an authorization filter.

Learn more about [Authorization](xref:security/authorization/introduction).

## Resource filters

Resource filters:

* Implement either the <xref:Microsoft.AspNetCore.Mvc.Filters.IResourceFilter> or <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncResourceFilter> interface.
* Execution wraps most of the filter pipeline.
* Only [Authorization filters](#authorization-filters) run before resource filters.

Resource filters are useful to short-circuit most of the pipeline. For example, a caching filter can avoid the rest of the pipeline on a cache hit.

Resource filter examples:

* [The short-circuiting resource filter](#short-circuiting-resource-filter) shown previously.
* [DisableFormValueModelBindingAttribute](https://github.com/aspnet/Entropy/blob/master/samples/Mvc.FileUpload/Filters/DisableFormValueModelBindingAttribute.cs):

  * Prevents model binding from accessing the form data.
  * Used for large file uploads to prevent the form data from being read into memory.

## Action filters

Action filters do **not** apply to Razor Pages. Razor Pages supports <xref:Microsoft.AspNetCore.Mvc.Filters.IPageFilter> and <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncPageFilter>. For more information, see [Filter methods for Razor Pages](xref:razor-pages/filter).

Action filters:

* Implement either the <xref:Microsoft.AspNetCore.Mvc.Filters.IActionFilter> or <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncActionFilter> interface.
* Their execution surrounds the execution of action methods.

The following code shows a sample action filter:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Filters/SampleActionFilter.cs" id="snippet_Class":::

The <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutingContext> provides the following properties:

* <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutingContext.ActionArguments> - enables reading the inputs to an action method.
* <xref:Microsoft.AspNetCore.Mvc.Controller> - enables manipulating the controller instance.
* <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutingContext.Result%2A> - setting `Result` short-circuits execution of the action method and subsequent action filters.

Throwing an exception in an action method:

* Prevents running of subsequent filters.
* Unlike setting `Result`, is treated as a failure instead of a successful result.

The <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutedContext> provides `Controller` and `Result` plus the following properties:

* <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutedContext.Canceled%2A> - True if the action execution was short-circuited by another filter.
* <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutedContext.Exception%2A> - Non-null if the action or a previously run action filter threw an exception. Setting this property to null:
  * Effectively handles the exception.
  * `Result` is executed as if it was returned from the action method.

For an `IAsyncActionFilter`, a call to the <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutionDelegate>:

* Executes any subsequent action filters and the action method.
* Returns `ActionExecutedContext`.

To short-circuit, assign <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutingContext.Result?displayProperty=fullName> to a result instance and don't call `next` (the `ActionExecutionDelegate`).

The framework provides an abstract <xref:Microsoft.AspNetCore.Mvc.Filters.ActionFilterAttribute> that can be subclassed.

The `OnActionExecuting` action filter can be used to:

* Validate model state.
* Return an error if the state is invalid.

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Filters/ValidateModelAttribute.cs" id="snippet_Class":::

> [!NOTE]
> Controllers annotated with the `[ApiController]` attribute automatically validate model state and return a 400 response. For more information, see [Automatic HTTP 400 responses](xref:web-api/index#automatic-http-400-responses).

The `OnActionExecuted` method runs after the action method:

* And can see and manipulate the results of the action through the <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutedContext.Result> property.
* <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutedContext.Canceled> is set to true if the action execution was short-circuited by another filter.
* <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutedContext.Exception> is set to a non-null value if the action or a subsequent action filter threw an exception. Setting `Exception` to null:
  * Effectively handles an exception.
  * `ActionExecutedContext.Result` is executed as if it were returned normally from the action method.

## Exception filters

Exception filters:

* Implement <xref:Microsoft.AspNetCore.Mvc.Filters.IExceptionFilter> or <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncExceptionFilter>.
* Can be used to implement common error handling policies.

The following sample exception filter displays details about exceptions that occur when the app is in development:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Filters/SampleExceptionFilter.cs" id="snippet_Class":::

The following code tests the exception filter:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Controllers/ExceptionController.cs" id="snippet_Class" highlight="1":::

Exception filters:

* Don't have before and after events.
* Implement <xref:Microsoft.AspNetCore.Mvc.Filters.IExceptionFilter.OnException%2A> or <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncExceptionFilter.OnExceptionAsync%2A>.
* Handle unhandled exceptions that occur in Razor Page or controller creation, [model binding](xref:mvc/models/model-binding), action filters, or action methods.
* Do **not** catch exceptions that occur in resource filters, result filters, or MVC result execution.

To handle an exception, set the <xref:Microsoft.AspNetCore.Mvc.Filters.ExceptionContext.ExceptionHandled%2A> property to `true` or assign the <xref:Microsoft.AspNetCore.Mvc.Filters.ExceptionContext.Result%2A> property. This stops propagation of the exception. An exception filter can't turn an exception into a "success". Only an action filter can do that.

Exception filters:

* Are good for trapping exceptions that occur within actions.
* Are not as flexible as error handling middleware.

Prefer middleware for exception handling. Use exception filters only where error handling *differs* based on which action method is called. For example, an app might have action methods for both API endpoints and for views/HTML. The API endpoints could return error information as JSON, while the view-based actions could return an error page as HTML.

## Result filters

Result filters:

* Implement an interface:
  * <xref:Microsoft.AspNetCore.Mvc.Filters.IResultFilter> or <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncResultFilter>
  * <xref:Microsoft.AspNetCore.Mvc.Filters.IAlwaysRunResultFilter> or <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncAlwaysRunResultFilter>
* Their execution surrounds the execution of action results.

### IResultFilter and IAsyncResultFilter

The following code shows a sample result filter:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Snippets/Filters/SampleResultFilter.cs" id="snippet_Class":::

The kind of result being executed depends on the action. An action returning a view includes all razor processing as part of the <xref:Microsoft.AspNetCore.Mvc.ViewResult> being executed. An API method might perform some serialization as part of the execution of the result. Learn more about [action results](xref:mvc/controllers/actions).

Result filters are only executed when an action or action filter produces an action result. Result filters are not executed when:

* An authorization filter or resource filter short-circuits the pipeline.
* An exception filter handles an exception by producing an action result.

The <xref:Microsoft.AspNetCore.Mvc.Filters.IResultFilter.OnResultExecuting%2A?displayProperty=fullName> method can short-circuit execution of the action result and subsequent result filters by setting <xref:Microsoft.AspNetCore.Mvc.Filters.ResultExecutingContext.Cancel?displayProperty=fullName> to `true`. Write to the response object when short-circuiting to avoid generating an empty response. Throwing an exception in `IResultFilter.OnResultExecuting`:

* Prevents execution of the action result and subsequent filters.
* Is treated as a failure instead of a successful result.

When the <xref:Microsoft.AspNetCore.Mvc.Filters.IResultFilter.OnResultExecuted%2A?displayProperty=fullName> method runs, the response has probably already been sent to the client. If the response has already been sent to the client, it cannot be changed.

`ResultExecutedContext.Canceled` is set to `true` if the action result execution was short-circuited by another filter.

`ResultExecutedContext.Exception` is set to a non-null value if the action result or a subsequent result filter threw an exception. Setting `Exception` to null effectively handles an exception and prevents the exception from being thrown again later in the pipeline. There is no reliable way to write data to a response when handling an exception in a result filter. If the headers have been flushed to the client when an action result throws an exception, there's no reliable mechanism to send a failure code.

For an <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncResultFilter>, a call to `await next` on the <xref:Microsoft.AspNetCore.Mvc.Filters.ResultExecutionDelegate> executes any subsequent result filters and the action result. To short-circuit, set <xref:Microsoft.AspNetCore.Mvc.Filters.ResultExecutingContext.Cancel?displayProperty=nameWithType> to `true` and don't call the `ResultExecutionDelegate`:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Snippets/Filters/SampleAsyncResultFilter.cs" id="snippet_Class":::

The framework provides an abstract `ResultFilterAttribute` that can be subclassed. The [ResponseHeaderAttribute](#response-header-attribute) class shown previously is an example of a result filter attribute.

### IAlwaysRunResultFilter and IAsyncAlwaysRunResultFilter

The <xref:Microsoft.AspNetCore.Mvc.Filters.IAlwaysRunResultFilter> and <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncAlwaysRunResultFilter> interfaces declare an <xref:Microsoft.AspNetCore.Mvc.Filters.IResultFilter> implementation that runs for all action results. This includes action results produced by:

* Authorization filters and resource filters that short-circuit.
* Exception filters.

For example, the following filter always runs and sets an action result (<xref:Microsoft.AspNetCore.Mvc.ObjectResult>) with a *422 Unprocessable Entity* status code when content negotiation fails:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Snippets/Filters/UnprocessableResultFilter.cs" id="snippet_Class":::

## IFilterFactory

<xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory> implements <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterMetadata>. Therefore, an `IFilterFactory` instance can be used as an `IFilterMetadata` instance anywhere in the filter pipeline. When the runtime prepares to invoke the filter, it attempts to cast it to an `IFilterFactory`. If that cast succeeds, the <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory.CreateInstance%2A> method is called to create the `IFilterMetadata` instance that is invoked. This provides a flexible design, since the precise filter pipeline doesn't need to be set explicitly when the app starts.

`IFilterFactory.IsReusable`:

* Is a hint by the factory that the filter instance created by the factory may be reused outside of the request scope it was created within.
* Should ***not*** be used with a filter that depends on services with a lifetime other than singleton.

The ASP.NET Core runtime doesn't guarantee:

* That a single instance of the filter will be created.
* The filter will not be re-requested from the DI container at some later point.

> [!WARNING] 
> Only configure <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory.IsReusable?displayProperty=nameWithType> to return `true` if the source of the filters is unambiguous, the filters are stateless, and the filters are safe to use across multiple HTTP requests. For instance, don't return filters from DI that are registered as scoped or transient if `IFilterFactory.IsReusable` returns `true`.

`IFilterFactory` can be implemented using custom attribute implementations as another approach to creating filters:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Filters/ResponseHeaderFactoryAttribute.cs" id="snippet_Class" highlight="1,5-6":::

The filter is applied in the following code:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Controllers/FilterFactoryController.cs" id="snippet_Index" highlight="1":::

### IFilterFactory implemented on an attribute

Filters that implement `IFilterFactory` are useful for filters that:

* Don't require passing parameters.
* Have constructor dependencies that need to be filled by DI.

<xref:Microsoft.AspNetCore.Mvc.TypeFilterAttribute> implements <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory>. `IFilterFactory` exposes the <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory.CreateInstance%2A> method for creating an <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterMetadata> instance. `CreateInstance` loads the specified type from the services container (DI).

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Filters/SampleActionTypeFilterAttribute.cs" id="snippet_Class" highlight="1,3-4":::

The following code shows three approaches to applying the filter:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Controllers/FilterFactoryController.cs" id="snippet_TypeFilterAttribute" highlight="1,5,9":::

In the preceding code, the first approach to applying the filter is preferred.

## Use middleware in the filter pipeline

Resource filters work like [middleware](xref:fundamentals/middleware/index) in that they surround the execution of everything that comes later in the pipeline. But filters differ from middleware in that they're part of the runtime, which means that they have access to context and constructs.

To use middleware as a filter, create a type with a `Configure` method that specifies the middleware to inject into the filter pipeline. The following example uses middleware to set a response header:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/FilterMiddlewarePipeline.cs" id="snippet_Class":::

Use the <xref:Microsoft.AspNetCore.Mvc.MiddlewareFilterAttribute> to run the middleware:

:::code language="csharp" source="~/mvc/controllers/filters/samples/8.x/FiltersSample/Controllers/FilterMiddlewareController.cs" id="snippet_Class" highlight="1":::

Middleware filters run at the same stage of the filter pipeline as Resource filters, before model binding and after the rest of the pipeline.

## Thread safety

When passing an *instance* of a filter into `Add`, instead of its `Type`, the filter is a singleton and is **not** thread-safe.

## Additional resources

* [View or download sample](https://github.com/dotnet/AspNetCore.Docs/tree/main/aspnetcore/mvc/controllers/filters/samples) ([how to download](xref:index#how-to-download-a-sample)).
* <xref:razor-pages/filter>

:::moniker-end
[!INCLUDE[](~/mvc/controllers/filters/includes/filters7.md)]
