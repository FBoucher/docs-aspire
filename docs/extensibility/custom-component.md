---
title: Create custom .NET Aspire component
description: Learn how to create a custom .NET Aspire component for an existing containerized application.
ms.date: 07/16/2024
ms.topic: how-to
---

# Create custom .NET Aspire component

This article is a continuation of the [Create custom resource types for .NET Aspire](custom-resources.md) article. It guides you through creating a .NET Aspire component that uses [MailKit](https://github.com/jstedfast/MailKit) to send emails. This component is then integrated into the Newsletter app you previously built. The previous example, omitted the creation of a component and instead relied on the existing .NET `SmtpClient`. It's best to use MailKit's `SmtpClient` over the official .NET `SmtpClient` for sending emails, as it's more modern and supports more features/protocols. For more information, see [.NET SmtpClient: Remarks](/dotnet/api/system.net.mail.smtpclient#remarks).

## Prerequisites

If you're following along, you should have a Newsletter app from the steps in the [Create custom resource types for .NET Aspire](custom-resources.md) article.

> [!TIP]
> This article is inspired by existing .NET Aspire components, and based on the teams official guidance. There are places where said guidance varies, and it's important to understand the reasoning behind the differences. For more information, see [.NET Aspire component requirements](https://github.com/dotnet/aspire/blob/f38b6cba86942ad1c45fc04fe7170f0fd4ba7c0b/src/Components/Aspire_Components_Progress.md#net-aspire-component-requirements).

## Create library for component

[.NET Aspire components](../fundamentals/components-overview.md) are delivered as NuGet packages, but in this example, it's beyond the scope of this article to publish a NuGet package. Instead, you create a class library project that contains the component and reference it as a project. .NET Aspire component packages are intended to wrap a client library, such as MailKit, and provide production-ready telemetry, health checks, configurability, and testability. Let's start by creating a new class library project.

1. Create a new class library project named `MailKit.Client` in the same directory as the _MailDevResource.sln_ from the previous article.

    ```dotnetcli
    dotnet new classlib -o MailKit.Client
    ```

1. Add the project to the solution.

    ```dotnetcli
    dotnet sln /MailDevResource.sln add MailKit.Client/MailKit.Client.csproj
    ```

The next step is to add all the NuGet packages that the component relies on. Rather than having you add each package one-by-one from the .NET CLI, it's likely easier to copy and paste the following XML into the _MailKit.Client.csproj_ file.

```xml
  <ItemGroup>
    <PackageReference Include="MailKit" Version="4.7.0" />
    <PackageReference Include="Microsoft.Extensions.Configuration.Binder" Version="8.0.2" />
    <PackageReference Include="Microsoft.Extensions.Resilience" Version="8.7.0" />
    <PackageReference Include="Microsoft.Extensions.Hosting.Abstractions" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.Diagnostics.HealthChecks" Version="8.0.7" />
    <PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.9.0" />
  </ItemGroup>
```

## Define component settings

Whenever you're creating a .NET Aspire component, it's best to understand the client library that you're mapping to. With MailKit, you need to understand the configuration settings that are required to connect to a Simple Mail Transfer Protocol (SMTP) server. But it's also important to understand if the library has support for _health checks_, _tracing_ and _metrics_. MailKit supports _tracing_ and _metrics_, through its [`Telemetry.SmtpClient` class](https://github.com/jstedfast/MailKit/blob/master/MailKit/Telemetry.cs#L112-L189). When adding _health checks_, you should use any established or existing health checks where possible. Otherwise, you might consider implementing your own in the component. Add the following code to the `MailKit.Client` project in a file named _MailKitClientSettings.cs_:

:::code source="snippets/MailDevResource/MailKit.Client/MailKitClientSettings.cs":::

The preceding code defines the `MailKitClientSettings` class with:

- `Endpoint` property that represents the connection string to the SMTP server.
- `Credentials` property that represents the credentials to authenticate with the SMTP server.
- `DisableHealthChecks` property that determines whether health checks are enabled.
- `DisableTracing` property that determines whether tracing is enabled.
- `DisableMetrics` property that determines whether metrics are enabled.

### Parse connection string logic

The settings class also contains a `ParseConnectionString` method that parses the connection string into a valid `Uri`. The configuration is expected to be provided in the following format:

- `ConnectionStrings:<connectionName>`: The connection string to the SMTP server.
- `MailKit:Client:Endpoint`: The connection string to the SMTP server.

If neither of these values are provided, an exception is thrown. Likewise, if there's a value but it's not a valid URI, an exception is thrown.

### Parse credentials logic

The settings class also contains a `ParseCredentials` method that parses the credentials into a valid `NetworkCredential`. The configuration is expected to be provided in the following format:

- `MailKit:Client:Credentials:UserName`: The username to authenticate with the SMTP server.
- `MailKit:Client:Credentials:Password`: The password to authenticate with the SMTP server.

When credentials are configured, the `ParseCredentials` method attempts to parse the username and password from the configuration. If either the username or password is missing, an exception is thrown.

## Expose component wrapper functionality

The goal of .NET Aspire components is to expose the underlying client library to consumers through dependency injection. With MailKit and for this example, the `SmtpClient` class is what you want to expose. You're not wrapping any functionality, but rather mapping configuration settings to an `SmtpClient` class. It's common to expose both standard and keyed-service registrations for components. Standard registrations are used when there's only one instance of a service, and keyed-service registrations are used when there are multiple instances of a service. Sometimes, to achieve multiple registrations of the same type you use a factory pattern. Add the following code to the `MailKit.Client` project in a file named _MailKitClientFactory.cs_:

:::code source="snippets/MailDevResource/MailKit.Client/MailKitClientFactory.cs":::

The `MailKitClientFactory` class is a factory that creates an `ISmtpClient` instance based on the configuration settings. It's responsible for returning an `ISmtpClient` implementation that has an active connection to a configured SMTP server and optionally authenticated. Next, you need to expose the functionality for the consumers to register this factory with the dependency injection container. Add the following code to the `MailKit.Client` project in a file named _MailKitExtensions.cs_:

:::code source="snippets/MailDevResource/MailKit.Client/MailKitExtensions.cs":::

The preceding code adds two extension methods on the `IHostApplicationBuilder` type, one for the standard registration of MailKit and another for keyed-registration of MailKit.

> [!TIP]
> Extension methods for .NET Aspire components should extend the `IHostApplicationBuilder` type and follow the `Add<MeaningfulName>` naming convention where the `<MeaningfulName>` is the type or functionality you're adding. For this article, the `AddMailKitClient` extension method is used to add the MailKit client. It's likely more in-line with the official guidance to use `AddMailKitSmtpClient` instead of `AddMailKitClient`, since this only registers the `SmtpClient` and not the entire MailKit library.

Both extensions ultimately rely on the private `AddMailKitClient` method to register the `MailKitClientFactory` with the dependency injection container as a [scoped service](/dotnet/core/extensions/dependency-injection#scoped). The reason for registering the `MailKitClientFactory` as a scoped service is because the connection (and authentication) operations are considered expensive and should be reused within the same scope where possible. In other words, for a single request, the same `ISmtpClient` instance should be used. The factory holds on to the instance of the `SmtpClient` that it creates and disposes of it.

### Configuration binding

One of the first things that the private implementation of the `AddMailKitClient` methods does, is to bind the configuration settings to the `MailKitClientSettings` class. The settings class is instantiated and then `Bind` is called with the specific section of configuration. Then the optional `configureSettings` delegate is invoked with the current settings. This allows the consumer to further configure the settings, ensuring that manual code settings are honored over configuration settings. After that, depending on whether the `serviceKey` value was provided, the `MailKitClientFactory` should be registered with the dependency injection container as either a standard or keyed service.

> [!IMPORTANT]
> It's intentional that the `implementationFactory` overload is called when registering services. The `CreateMailKitClientFactory` method throws when the configuration is invalid. This ensures that creation of the `MailKitClientFactory` is deferred until it's needed and it prevents the app from erroring out before logging is available.

The registration of health checks, and telemetry are described in a bit more detail in the following sections.

### Add health checks

[Health checks](../fundamentals/health-checks.md) are a way to monitor the health of a component. With MailKit, you can check if the connection to the SMTP server is healthy. Add the following code to the `MailKit.Client` project in a file named _MailKitHealthCheck.cs_:

:::code source="snippets/MailDevResource/MailKit.Client/MailKitHealthCheck.cs":::

The preceding health check implementation:

- Implements the `IHealthCheck` interface.
- Accepts the `MailKitClientFactory` as a primary constructor parameter.
- Satisfies the `CheckHealthAsync` method by:
  - Attempting to get an `ISmtpClient` instance from the `factory`. If successful, it returns `HealthCheckResult.Healthy`.
  - If an exception is thrown, it returns `HealthCheckResult.Unhealthy`.

As previously shared in the registration of the `MailKitClientFactory`, the `MailKitHealthCheck` is conditionally registered with the `IHeathChecksBuilder`:

```csharp
if (settings.DisableHealthChecks is false)
{
    builder.Services.AddHealthChecks()
        .AddCheck<MailKitHealthCheck>(
            name: serviceKey is null ? "MailKit" : $"MailKit_{connectionName}",
            failureStatus: default,
            tags: []);
}
```

The consumer could choose to omit health checks by setting the `DisableHealthChecks` property to `true` in the configuration. A common pattern for components is to have optional features and .NET Aspire components strongly encourages these types of configurations. For more information on health checks and a working sample that includes a user interface, see [.NET Aspire ASP.NET Core HealthChecksUI sample](/samples/dotnet/aspire-samples/aspire-health-checks-ui/).

### Wire up telemetry

As a best practice, the [MailKit client library exposes telemetry](https://github.com/jstedfast/MailKit/blob/master/Telemetry.md). .NET Aspire can take advantage of this telemetry and display it in the [.NET Aspire dashboard](../fundamentals/dashboard/overview.md). Depending on whether or not tracing and metrics are enabled, telemetry is wired up as shown in the following code snippet:

```csharp
if (settings.DisableTracing is false)
{
    builder.Services.AddOpenTelemetry()
        .WithTracing(
            traceBuilder => traceBuilder.AddSource(
                Telemetry.SmtpClient.ActivitySourceName));
}

if (settings.DisableMetrics is false)
{
    // Required by MailKit to enable metrics
    Telemetry.SmtpClient.Configure();

    builder.Services.AddOpenTelemetry()
        .WithMetrics(
            metricsBuilder => metricsBuilder.AddMeter(
                Telemetry.SmtpClient.MeterName));
}
```

## Update the Newsletter service

With the component library created, you can now update the Newsletter service to use the MailKit client. The first step is to add a reference to the `MailKit.Client` project. Add the _MailKit.Client.csproj_ project reference to the `MailDevResource.NewsletterService` project:

```dotnetcli
dotnet add ./MailDevResource.NewsletterService/MailDevResource.NewsletterService.csproj reference MailKit.Client/MailKit.Client.csproj
```

The final step is to replace the existing _Program.cs_ file in the `MailDevResource.NewsletterService` project with the following C# code:

:::code source="snippets/MailDevResource/MailDevResource.NewsletterService/Program.cs":::

The most notable changes in the preceding code are:

- The updated `using` statements that include the `MailKit.Client`, `MailKit.Net.Smtp`, and `MimeKit` namespaces.
- The replacement of the registration for the official .NET `SmtpClient` with the call to the `AddMailKitClient` extension method.
- The replacement of both `/subscribe` and `/unsubscribe` map post calls to instead inject the `MailKitClientFactory` and use the `ISmtpClient` instance to send the email.

## Run the sample

Now that you've created the MailKit client component and updated the Newsletter service to use it, you can run the sample. From your IDE, select <kbd>F5</kbd> or run `dotnet run` from the root directory of the solution to start the application—you should see the [.NET Aspire dashboard](../fundamentals/dashboard/overview.md):

:::image type="content" source="./media/maildev-with-newsletterservice-dashboard.png" lightbox="./media/maildev-with-newsletterservice-dashboard.png" alt-text=".NET Aspire dashboard: MailDev and Newsletter resources running.":::

Once the application is running, navigate to the Swagger UI at [https://localhost:7251/swagger](https://localhost:7251/swagger) and test the `/subscribe` and `/unsubscribe` endpoints. Select the down arrow to expand the endpoint:

:::image type="content" source="./media/swagger-ui.png" lightbox="./media/swagger-ui.png" alt-text="Swagger UI: Subscribe endpoint.":::

Then select the `Try it out` button. Enter an email address, and then select the `Execute` button.

:::image type="content" source="./media/swagger-ui-try.png" lightbox="./media/swagger-ui-try.png" alt-text="Swagger UI: Subscribe endpoint with email address.":::

Repeat this several times, to add multiple email addresses. You should see the email sent to the MailDev inbox:

:::image type="content" source="./media/maildev-inbox.png" alt-text="MailDev inbox with multiple emails.":::

Stop the application by selecting <kbd>Ctrl</kbd>+<kbd>C</kbd> in the terminal window where the application is running, or by selecting the stop button in your IDE.

### Configure MailDev credentials

The MailDev container supports basic authentication for both incoming and outgoing SMTP. To configure the credentials for incoming, you need to set the `MAILDEV_INCOMING_USER` and `MAILDEV_INCOMING_PASS` environment variables. For more information, see [MailDev: Usage](https://maildev.github.io/maildev/#usage).

To configure these credentials, update the _Program.cs_ file in the `MailDevResource.AppHost` project with the following code:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var mailDevUsername = builder.AddParameter("maildev-username");
var mailDevPassword = builder.AddParameter("maildev-password");

var maildev = builder.AddMailDev("maildev")
    .WithEnvironment("MAILDEV_INCOMING_USER", mailDevUsername)
    .WithEnvironment("MAILDEV_INCOMING_PASS", mailDevPassword);

builder.AddProject<Projects.MailDevResource_NewsletterService>("newsletterservice")
       .WithReference(maildev);

builder.Build().Run();
```

The preceding code adds two parameters for the MailDev username and password. It assigns these parameters to the `MAILDEV_INCOMING_USER` and `MAILDEV_INCOMING_PASS` environment variables. The `AddMailDev` method has two chained calls to `WithEnvironment` which includes these environment variables. For more information on parameters, see [External parameters](../fundamentals/external-parameters.md).

Next, configure the secrets for these paremeters. Right-click on the `MailDevResource.AppHost` project and select `Manage User Secrets`. Add the following JSON to the `secrets.json` file:

```json
{
  "Parameters:maildev-username": "@admin",
  "Parameters:maildev-password": "t3st1ng"
}
```

> [!WARNING]
> These credentials are for demonstration purposes only and MailDev is intended for local development. These crednetials are fictitious and shouldn't be used in a production environment.

If you're to run the sample now, the client wouldn't be able to connect to the MailDev container. This is because the MailDev container is configured to require authentication for incoming SMTP connections. The MailKit client configuration also needs to be updated to include the credentials.

To configure the credentials in the client, right-click on the `MailDevResource.NewsletterService` project and select `Manage User Secrets`. Add the following JSON to the `secrets.json` file:

```json
{
  "MailKit:Client": {
    "Credentials": {
      "UserName": "@admin",
      "Password": "t3st1ng"
    }
  }
}
```

Run the app again, and everything works as it did before, but now with authentication enabled.

### View MailKit telemetry

The MailKit client library exposes telemetry that can be viewed in the .NET Aspire dashboard. To view the telemetry, navigate to the .NET Aspire dashboard at [https://localhost:7251](https://localhost:7251). Select the `newsletter` resource to view the telemetry on the **Metrics** page:

:::image type="content" source="./media/mailkit-metrics-dashboard.png" lightbox="./media/mailkit-metrics-dashboard.png" alt-text=".NET Aspire dashboard: MailKit telemetry.":::

Open up the Swagger UI again, and make some requests to the `/subscribe` and `/unsubscribe` endpoints. Then, navigate back to the .NET Aspire dashboard and select the `newsletter` resource. Select a metric under the **mailkit.net.smtp** node, such as `mailkit.net.smtp.client.operation.count`. You should see the telemetry for the MailKit client:

:::image type="content" source="./media/mailkit-metrics-graph-dashboard.png" lightbox="./media/mailkit-metrics-graph-dashboard.png" alt-text=".NET Aspire dashboard: MailKit telemetry for operation count.":::

## Summary

In this article, you learned how to create a .NET Aspire component that uses MailKit to send emails. You also learned how to integrate this component into the Newsletter app you previously built. You learned about the core principles of .NET Aspire components, such as exposing the underlying client library to consumers through dependency injection, and how to add health checks and telemetry to the component. You also learned how to update the Newsletter service to use the MailKit client.

Go forth and build your own .NET Aspire components. If you believe that there's enough community value in the component you're building, consider publishing it as a [NuGet package](/dotnet/standard/library-guidance/nuget) for others to use. Furthermore, consider submitting a pull request to the [.NET Aspire GitHub repository](https://github.com/dotnet/aspire) for consideration to be included in the official .NET Aspire components.
