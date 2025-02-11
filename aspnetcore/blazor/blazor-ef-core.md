---
title: ASP.NET Core Blazor with Entity Framework Core (EF Core)
author: guardrex
description: Learn how to use Entity Framework Core (EF Core) in Blazor apps.
monikerRange: '>= aspnetcore-3.1'
ms.author: jeliknes
ms.custom: mvc
ms.date: 11/12/2024
uid: blazor/blazor-ef-core
---
# ASP.NET Core Blazor with Entity Framework Core (EF Core)

[!INCLUDE[](~/includes/not-latest-version.md)]

This article explains how to use [Entity Framework Core (EF Core)](/ef/core/) in server-side Blazor apps.

Server-side Blazor is a stateful app framework. The app maintains an ongoing connection to the server, and the user's state is held in the server's memory in a *circuit*. One example of user state is data held in [dependency injection (DI)](xref:fundamentals/dependency-injection) service instances that are scoped to the circuit. The unique application model that Blazor provides requires a special approach to use Entity Framework Core.

> [!NOTE]
> This article addresses EF Core in server-side Blazor apps. Blazor WebAssembly apps run in a WebAssembly sandbox that prevents most direct database connections. Running EF Core in Blazor WebAssembly is beyond the scope of this article.

:::moniker range=">= aspnetcore-8.0"

This guidance applies to components that adopt interactive server-side rendering (interactive SSR) in a Blazor Web App.

:::moniker-end

:::moniker range="< aspnetcore-8.0"

This guidance applies to the **`Server`** project of a hosted Blazor WebAssembly solution or a Blazor Server app.

:::moniker-end

## Secure authentication flow required for production apps

This article uses a local database that doesn't require user authentication. Production apps should use the most secure authentication flow available. For more information on authentication for deployed test and production Blazor apps, see the articles in the [Blazor *Security and Identity* node](xref:blazor/security/index).

For Microsoft Azure services, we recommend using *managed identities*. Managed identities securely authenticate to Azure services without storing credentials in app code. For more information, see the following resources:

* [What are managed identities for Azure resources? (Microsoft Entra documentation)](/entra/identity/managed-identities-azure-resources/overview)
* Azure services documentation
  * [Managed identities in Microsoft Entra for Azure SQL](/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity)
  * [How to use managed identities for App Service and Azure Functions](/azure/app-service/overview-managed-identity)

## Sample app

The sample app was built as a reference for server-side Blazor apps that use EF Core. The sample app includes a grid with sorting and filtering, delete, add, and update operations.

:::moniker range=">= aspnetcore-8.0"

[View or download sample code](https://github.com/dotnet/blazor-samples) ([how to download](xref:blazor/fundamentals/index#sample-apps)): Select the folder that matches the version of .NET that you're adopting. Within the version folder, access the sample named `BlazorWebAppEFCore`.

:::moniker-end

:::moniker range="< aspnetcore-8.0"

[View or download sample code](https://github.com/dotnet/blazor-samples) ([how to download](xref:blazor/fundamentals/index#sample-apps)): Select the folder that matches the version of .NET that you're adopting. Within the version folder, access the sample named `BlazorServerEFCoreSample`.

:::moniker-end

### Use the sample with SQLite

The sample uses a local [SQLite](https://www.sqlite.org/index.html) database so that it can be used on any platform.

The sample demonstrates use of EF Core to handle optimistic concurrency. However, [native database-generated concurrency tokens](/ef/core/saving/concurrency?tabs=fluent-api#native-database-generated-concurrency-tokens) aren't supported for SQLite databases, which is the database provider for the sample app. To demonstrate concurrency with the sample app, adopt a different database provider that supports database-generated concurrency tokens (for example, the [SQL Server provider](/ef/core/providers/sql-server)). You can adopt SQL Server for the sample app by following the guidance in the next section, *Use the sample with SQL Server and optimistic concurrency*.

### Use the sample with SQL Server and optimistic concurrency

The sample demonstrates use of EF Core to handle optimistic concurrency but only for a database provider that uses [native database-generated concurrency tokens](/ef/core/saving/concurrency?tabs=fluent-api#native-database-generated-concurrency-tokens), which is a feature supported for SQL Server. To demonstrate concurrency with the sample app, the sample app can be converted from the SQLite provider to use the [SQL Server provider](/ef/core/providers/sql-server) with a new SQL Server database created using .NET scaffolding.

Use the following guidance to adopt SQL Server for the sample app using Visual Studio.

Open the `Program` file (`Program.cs`) and comment out the lines that add the database context factory with the SQLite provider:

```diff
- builder.Services.AddDbContextFactory<ContactContext>(opt =>
-     opt.UseSqlite($"Data Source={nameof(ContactContext.ContactsDb)}.db"));
+ //builder.Services.AddDbContextFactory<ContactContext>(opt =>
+ //    opt.UseSqlite($"Data Source={nameof(ContactContext.ContactsDb)}.db"));
```

Save the `Program.cs` file.

Right-click the project in **Solution Explorer** and select **Add** > **New Scaffolded Item**.

In the **Add New Scaffolded Item** dialog, select **Installed** > **Common** > **Blazor** > **Razor Component** > **Razor Components Using Entity Framework (CRUD)**. Select the **Add** button.

In the **Add Razor Components Using Entity Framework (CRUD)** dialog, use the following settings:

* **Template**: Use the default selection (**CRUD**).
* **Model class**: Select the `Contact` model from the dropdown list.
* **DbContext class**: Select the plus sign (`+`) and select **Add** using the default context class name generated by the scaffolder.
* **Database provider**: Use the default selection (**SQL Server**).

Select **Add** to scaffold the model and create the SQL Server database.

When the scaffolding operation completes, delete the generated context class from the `Data` folder (`Data/{PROJECT NAME}Context.cs`, where the `{PROJECT NAME}` placeholder is the project's name/namespace).

Delete the `ContactPages` folder from the `Components/Pages` folder, which contains [QuickGrid](xref:blazor/components/quickgrid)-based pages for contact management. For a complete demonstration of QuickGrid-based pages for managing data, use the <xref:blazor/tutorials/movie-database-app/index> tutorial.

Open the `Program` file (`Program.cs`) and find the line that scaffolding added to create a database context factory using the SQL Server provider. Change the context from the generated context class (deleted earlier) to the app's existing `ContactContext` class:

```diff
- builder.Services.AddDbContextFactory<BlazorWebAppEFCoreContext>(options =>
+ builder.Services.AddDbContextFactory<ContactContext>(options =>
```

At this point, the app is using the SQL Server provider and a SQL Server database created for the `Contact` model class. Optimistic concurrency works using [native database-generated concurrency tokens](/ef/core/saving/concurrency?tabs=fluent-api#native-database-generated-concurrency-tokens) that are already implemented in the sample app's `ContactContext` class.

### Database logging

The sample also configures database logging to show the SQL queries that are generated. This is configured in `appsettings.Development.json`:

:::moniker range=">= aspnetcore-9.0"

:::code language="json" source="~/../blazor-samples/9.0/BlazorWebAppEFCore/appsettings.Development.json":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="json" source="~/../blazor-samples/8.0/BlazorWebAppEFCore/appsettings.Development.json":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

:::code language="json" source="~/../blazor-samples/7.0/BlazorServerEFCoreSample/appsettings.Development.json":::

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-7.0"

:::code language="json" source="~/../blazor-samples/6.0/BlazorServerEFCoreSample/appsettings.Development.json":::

:::moniker-end

:::moniker range=">= aspnetcore-5.0 < aspnetcore-6.0"

:::code language="json" source="~/../blazor-samples/5.0/BlazorServerEFCoreSample/appsettings.Development.json":::

:::moniker-end

:::moniker range="< aspnetcore-5.0"

:::code language="json" source="~/../blazor-samples/3.1/BlazorServerEFCoreSample/appsettings.Development.json":::

:::moniker-end

The grid, add, and view components use the "context-per-operation" pattern, where a context is created for each operation. The edit component uses the "context-per-component" pattern, where a context is created for each component.

> [!NOTE]
> Some of the code examples in this topic require namespaces and services that aren't shown. To inspect the fully working code, including the required [`@using`](xref:mvc/views/razor#using) and [`@inject`](xref:mvc/views/razor#inject) directives for Razor examples, see the [sample app](#sample-app).

:::moniker range=">= aspnetcore-8.0"

## Build a Blazor movie database app tutorial

For a tutorial experience building an app that uses EF Core to work with a database, see <xref:blazor/tutorials/movie-database-app/index>. The tutorial shows you how to create a Blazor Web App that can display and manage movies in a movie database.

:::moniker-end

## Database access

EF Core relies on a <xref:Microsoft.EntityFrameworkCore.DbContext> as the means to [configure database access](/ef/core/miscellaneous/configuring-dbcontext) and act as a [*unit of work*](https://martinfowler.com/eaaCatalog/unitOfWork.html). EF Core provides the <xref:Microsoft.Extensions.DependencyInjection.EntityFrameworkServiceCollectionExtensions.AddDbContext%2A> extension for ASP.NET Core apps that registers the context as a *scoped* service. In server-side Blazor apps, scoped service registrations can be problematic because the instance is shared across components within the user's circuit. <xref:Microsoft.EntityFrameworkCore.DbContext> isn't thread safe and isn't designed for concurrent use. The existing lifetimes are inappropriate for these reasons:

* **Singleton** shares state across all users of the app and leads to inappropriate concurrent use.
* **Scoped** (the default) poses a similar issue between components for the same user.
* **Transient** results in a new instance per request; but as components can be long-lived, this results in a longer-lived context than may be intended.

The following recommendations are designed to provide a consistent approach to using EF Core in server-side Blazor apps.

* Consider using one context per operation. The context is designed for fast, low overhead instantiation:

  ```csharp
  using var context = new MyContext();

  return await context.MyEntities.ToListAsync();
  ```

* Use a flag to prevent multiple concurrent operations:

  ```csharp
  if (Loading)
  {
      return;
  }

  try
  {
      Loading = true;

      ...
  }
  finally
  {
      Loading = false;
  }
  ```

  Place operations after the `Loading = true;` line in the `try` block.
  
  Thread safety isn't a concern, so loading logic doesn't require locking database records. The loading logic is used to disable UI controls so that users don't inadvertently select buttons or update fields while data is fetched.
  
* If there's any chance that multiple threads may access the same code block, [inject a factory](#scope-to-the-component-lifetime) and make a new instance per operation. Otherwise, injecting and using the context is usually sufficient.

* For longer-lived operations that take advantage of EF Core's [change tracking](/ef/core/querying/tracking) or [concurrency control](/ef/core/saving/concurrency), [scope the context to the lifetime of the component](#scope-to-the-component-lifetime).

## New `DbContext` instances

The fastest way to create a new <xref:Microsoft.EntityFrameworkCore.DbContext> instance is by using `new` to create a new instance. However, there are scenarios that require resolving additional dependencies:

* Using [`DbContextOptions`](/ef/core/miscellaneous/configuring-dbcontext#configuring-dbcontextoptions) to configure the context.
* Using a connection string per <xref:Microsoft.EntityFrameworkCore.DbContext>, such as when you use [ASP.NET Core's Identity model](xref:security/authentication/customize_identity_model). For more information, see [Multi-tenancy (EF Core documentation)](/ef/core/miscellaneous/multitenancy).

[!INCLUDE[](~/blazor/security/includes/secure-authentication-flows.md)]

The recommended approach to create a new <xref:Microsoft.EntityFrameworkCore.DbContext> with dependencies is to use a factory. EF Core 5.0 or later provides a built-in factory for creating new contexts.

:::moniker range="< aspnetcore-5.0"

:::code language="csharp" source="~/../blazor-samples/3.1/BlazorServerEFCoreSample/Data/DbContextFactory.cs":::

In the preceding factory:

* <xref:Microsoft.Extensions.DependencyInjection.ActivatorUtilities.CreateInstance%2A?displayProperty=nameWithType> satisfies any dependencies via the service provider.
* <xref:Microsoft.EntityFrameworkCore.IDbContextFactory%601> is available in EF Core ASP.NET Core 5.0 or later, so the interface is [implemented in the sample app for ASP.NET Core 3.x](https://github.com/dotnet/blazor-samples/blob/main/3.1/BlazorServerEFCoreSample/Data/IDbContextFactory.cs).

:::moniker-end

The following example configures [SQLite](https://www.sqlite.org/index.html) and enables data logging. The code uses an extension method (`AddDbContextFactory`) to configure the database factory for DI and provide default options:

:::moniker range=">= aspnetcore-9.0"

:::code language="csharp" source="~/../blazor-samples/9.0/BlazorWebAppEFCore/Program.cs" id="snippet1":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="csharp" source="~/../blazor-samples/8.0/BlazorWebAppEFCore/Program.cs" id="snippet1":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

:::code language="csharp" source="~/../blazor-samples/7.0/BlazorServerEFCoreSample/Program.cs" id="snippet1":::

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-7.0"

:::code language="csharp" source="~/../blazor-samples/6.0/BlazorServerEFCoreSample/Program.cs" id="snippet1":::

:::moniker-end

:::moniker range=">= aspnetcore-5.0 < aspnetcore-6.0"

:::code language="csharp" source="~/../blazor-samples/5.0/BlazorServerEFCoreSample/Startup.cs" id="snippet1":::

:::moniker-end

:::moniker range="< aspnetcore-5.0"

:::code language="csharp" source="~/../blazor-samples/3.1/BlazorServerEFCoreSample/Startup.cs" id="snippet1":::

:::moniker-end

The factory is injected into components and used to create new `DbContext` instances.

In the home page of the sample app, `IDbContextFactory<ContactContext>` is injected into the component:

```razor
@inject IDbContextFactory<ContactContext> DbFactory
```

A `DbContext` is created using the factory (`DbFactory`) to delete a contact in the `DeleteContactAsync` method:

:::moniker range=">= aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/9.0/BlazorWebAppEFCore/Components/Pages/Home.razor" id="snippet1":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="razor" source="~/../blazor-samples/8.0/BlazorWebAppEFCore/Components/Pages/Home.razor" id="snippet1":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

:::code language="razor" source="~/../blazor-samples/7.0/BlazorServerEFCoreSample/Pages/Index.razor" id="snippet1":::

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-7.0"

:::code language="razor" source="~/../blazor-samples/6.0/BlazorServerEFCoreSample/Pages/Index.razor" id="snippet1":::

:::moniker-end

:::moniker range=">= aspnetcore-5.0 < aspnetcore-6.0"

:::code language="razor" source="~/../blazor-samples/5.0/BlazorServerEFCoreSample/Pages/Index.razor" id="snippet1":::

:::moniker-end

:::moniker range="< aspnetcore-5.0"

:::code language="razor" source="~/../blazor-samples/3.1/BlazorServerEFCoreSample/Pages/Index.razor" id="snippet1":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0"

> [!NOTE]
> `Filters` is an injected `IContactFilters`, and `Wrapper` is a [component reference](xref:blazor/components/index#capture-references-to-components) to the `GridWrapper` component. See the `Home` component (`Components/Pages/Home.razor`) in the sample app.

:::moniker-end

:::moniker range="< aspnetcore-8.0"

> [!NOTE]
> `Filters` is an injected `IContactFilters`, and `Wrapper` is a [component reference](xref:blazor/components/index#capture-references-to-components) to the `GridWrapper` component. See the `Index` component (`Pages/Index.razor`) in the sample app.

:::moniker-end

## Scope to the component lifetime

You may wish to create a <xref:Microsoft.EntityFrameworkCore.DbContext> that exists for the lifetime of a component. This allows you to use it as a [unit of work](https://martinfowler.com/eaaCatalog/unitOfWork.html) and take advantage of built-in features, such as change tracking and concurrency resolution.

:::moniker range=">= aspnetcore-8.0"

You can use the factory to create a context and track it for the lifetime of the component. First, implement <xref:System.IDisposable> and inject the factory as shown in the `EditContact` component (`Components/Pages/EditContact.razor`):

:::moniker-end

:::moniker range="< aspnetcore-8.0"

You can use the factory to create a context and track it for the lifetime of the component. First, implement <xref:System.IDisposable> and inject the factory as shown in the `EditContact` component (`Pages/EditContact.razor`):

:::moniker-end

```razor
@implements IDisposable
@inject IDbContextFactory<ContactContext> DbFactory
```

The sample app ensures the context is disposed when the component is disposed:

:::moniker range=">= aspnetcore-9.0"

:::code language="csharp" source="~/../blazor-samples/9.0/BlazorWebAppEFCore/Components/Pages/EditContact.razor" id="snippet1":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="csharp" source="~/../blazor-samples/8.0/BlazorWebAppEFCore/Components/Pages/EditContact.razor" id="snippet1":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

:::code language="csharp" source="~/../blazor-samples/7.0/BlazorServerEFCoreSample/Pages/EditContact.razor" id="snippet1":::

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-7.0"

:::code language="csharp" source="~/../blazor-samples/6.0/BlazorServerEFCoreSample/Pages/EditContact.razor" id="snippet1":::

:::moniker-end

:::moniker range=">= aspnetcore-5.0 < aspnetcore-6.0"

:::code language="csharp" source="~/../blazor-samples/5.0/BlazorServerEFCoreSample/Pages/EditContact.razor" id="snippet1":::

:::moniker-end

:::moniker range="< aspnetcore-5.0"

:::code language="csharp" source="~/../blazor-samples/3.1/BlazorServerEFCoreSample/Pages/EditContact.razor" id="snippet1":::

:::moniker-end

Finally, [`OnInitializedAsync`](xref:blazor/components/lifecycle) is overridden to create a new context. In the sample app, [`OnInitializedAsync`](xref:blazor/components/lifecycle) loads the contact in the same method:

:::moniker range=">= aspnetcore-9.0"

:::code language="csharp" source="~/../blazor-samples/9.0/BlazorWebAppEFCore/Components/Pages/EditContact.razor" id="snippet2":::

:::moniker-end

:::moniker range=">= aspnetcore-8.0 < aspnetcore-9.0"

:::code language="csharp" source="~/../blazor-samples/8.0/BlazorWebAppEFCore/Components/Pages/EditContact.razor" id="snippet2":::

:::moniker-end

:::moniker range=">= aspnetcore-7.0 < aspnetcore-8.0"

:::code language="csharp" source="~/../blazor-samples/7.0/BlazorServerEFCoreSample/Pages/EditContact.razor" id="snippet2":::

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-7.0"

:::code language="csharp" source="~/../blazor-samples/6.0/BlazorServerEFCoreSample/Pages/EditContact.razor" id="snippet2":::

:::moniker-end

:::moniker range=">= aspnetcore-5.0 < aspnetcore-6.0"

:::code language="csharp" source="~/../blazor-samples/5.0/BlazorServerEFCoreSample/Pages/EditContact.razor" id="snippet2":::

:::moniker-end

:::moniker range="< aspnetcore-5.0"

:::code language="csharp" source="~/../blazor-samples/3.1/BlazorServerEFCoreSample/Pages/EditContact.razor" id="snippet2":::

In the preceding example:

* When `Busy` is set to `true`, asynchronous operations may begin. When `Busy` is set back to `false`, asynchronous operations should be finished.
* Place additional error handling logic in a `catch` block.

:::moniker-end

## Enable sensitive data logging

<xref:Microsoft.EntityFrameworkCore.DbContextOptionsBuilder.EnableSensitiveDataLogging%2A> includes application data in exception messages and framework logging. The logged data can include the values assigned to properties of entity instances and parameter values for commands sent to the database. Logging data with <xref:Microsoft.EntityFrameworkCore.DbContextOptionsBuilder.EnableSensitiveDataLogging%2A> is a **security risk**, as it may expose passwords and other [Personally Identifiable Information (PII)](xref:blazor/security/index#personally-identifiable-information-pii) when it logs SQL statements executed against the database.

We recommend only enabling <xref:Microsoft.EntityFrameworkCore.DbContextOptionsBuilder.EnableSensitiveDataLogging%2A> for development and testing:

```csharp
#if DEBUG
    services.AddDbContextFactory<ContactContext>(opt =>
        opt.UseSqlite($"Data Source={nameof(ContactContext.ContactsDb)}.db")
        .EnableSensitiveDataLogging());
#else
    services.AddDbContextFactory<ContactContext>(opt =>
        opt.UseSqlite($"Data Source={nameof(ContactContext.ContactsDb)}.db"));
#endif
```

## Additional resources

* [EF Core documentation](/ef/)
* [Blazor samples GitHub repository (`dotnet/blazor-samples`)](https://github.com/dotnet/blazor-samples)
* [Blazor *Security and Identity* node articles](xref:blazor/security/index)
