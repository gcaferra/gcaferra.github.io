---
title: "Stop using IConfiguration to configure your Services"
tags: 
- dotnet
- configuration
- c#
- core
- best practices
---

# Stop using IConfiguration to configure you Services

In several projects I've found using the IConfiguration directly and pass it to services with somethig like this:

``` csharp
appBuilder.Services.ConfigureMyServices1(configuration);
```

This approach is simple but it open to several issues I'll try to show you in the following chapters

## A .net Core basic Configuration

Dotnet offers a starndardized way to configure the applications by simply writing somethig like:

``` csharp
var configurationBuilder = new ConfigurationBuilder();
```

You can add json configuration files using the package `Microsoft.Extensions.Configuration.Json` and that will give you the option to add configuration files:

``` csharp
.AddJsonFile("appSettings.json", true);
```

control your application from the Command Line using `Microsoft.Extensions.Configuration.CommandLine` package and adding the following code to the above statement:

``` csharp
.AddCommandLine(args);
```

or in addition you can use Environmental variables with the package `Microsoft.Extensions.Configuration.EnvironmentVariables` useful for containers or in cloud for example

``` csharp
.AddEnvironmentVariables();
```

These are just few examples on how configuration can work in .net core

Please get more information on the configuragion on [Miscrosoft](https://learn.microsoft.com/en-us/dotnet/core/extensions/configuration) web site

## Configuring your Modules (The lazy way)

In several projects I've found `IConfiguration` passed directly around the modules with something like

``` csharp
services.ConfigureMyServices(configuration);
```

and then used directly reading settings in this way in the service

``` csharp
    public static IServiceCollection ConfigureMyServices(this IServiceCollection services, IConfiguration configuration)
    {
        var readMySettings = configuration["MyServiceLevelSetting"];
        services.AddScoped<IMyFirstService, MyFirstService>();
        return services;
    }
```

or in the classes like the following:

```csharp
public class MyFirstService : IMyFirstService
{
    string _externalUrl;
    string _connectionString;
    string _anotherConfig;

    public MyFirstService(IConfiguration configuration)
    {
        _externalUrl = configuration["ExternalUrl"] ?? throw new InvalidOperationException();
        _connectionString = configuration["ConnectionString"] ?? throw new InvalidOperationException();
        _anotherConfig = configuration["AnotherConfig"] ?? throw new InvalidOperationException();
    }

    public Task DoSomething(string action)
    {
        //Do something with the configured url;
        Console.WriteLine($"Performed {action} to Url {_externalUrl}");
        return Task.CompletedTask;
    }
}
```

At first glace it seems an easy trick when you need to pass configuration around it's simple as it look like.
Another advantage is when you add configuration setting you don't touch the `services.ConfigureMyServices(configuration);` part and it will always work, right?

Let me show you why it's a bad practice.

## Runtime Exceptions

Your `MyFirstService` class become useful for another project and you create a nuget package for it and publish it in you private nuget repository to make it available for other teams.

After few minutes you published the package the other team start to have runtime NullReferenceExceptions or in the snipped above `InvalidOperationException` and will come back to you to ask why your library is causing that exceptions.
After a while checking the code you will figure out the problem: There are missing settings in the other's team project.
Then to solve the issue You will give to him the list of settings to add in it's appSettings.json for example:

``` csharp
ExternalUrl
ConnectionString
AnotherConfig

//and

MyServiceLevelSetting
```

Issue solved! Right?

But you library is really useful and more teams need it, you need to give them the same hint again and again adding the settings to their configurations or else they will have runtime exceptions.

## How to make the configuration clear for everyone?

With the following method for example everyone will be clear in what is needed to configure your service making it clear to everyone how to configure it:

``` csharp
    public static IServiceCollection ConfigureMyServices2(this IServiceCollection services, string myServiceLevelSetting, string externalUrl, string connectionString, string anotherConfig)
    {
        var readMySettings = myServiceLevelSetting;
        services.AddScoped<IMyFirstService>(c => new MySecondService(externalUrl, connectionString, anotherConfig));
        return services;
    }
```

The Service configuration now will look like to this:

``` csharp
appBuilder.Services.ConfigureMyServices(configuration["MyServiceLevelSetting"],
    configuration["ExternalUrl"],
    configuration["ConnectionString"],
    configuration["AnotherConfig"]);
```

You can validate the settings on the application startup to not have runtime errors when the application is running and the first usage of your Service will cause Exceptions due to the missing settings to avoid that you can have something like this:

``` csharp
appBuilder.Services.ConfigureMyServices(configuration["MyServiceLevelSetting"] ?? throw new InvalidOperationException(),
    configuration["ExternalUrl"] ?? throw new InvalidOperationException(),
    configuration["ConnectionString"] ?? throw new InvalidOperationException(),
    configuration["AnotherConfig"] ?? throw new InvalidOperationException());
```

In this way if a setting is missing the application will throw an exception.

The service need to be refactored in the following way for example:

``` csharp
public class MySecondService : IMyFirstService
{
    string _externalUrl;
    string _connectionString;
    string _anotherConfig;
    public MySecondService(string externalUrl, string connectionString, string anotherConfig)
    {
        _externalUrl = externalUrl;
        _connectionString = connectionString;
        _anotherConfig = anotherConfig;
    }

    public Task DoSomething(string action)
    {
        //Do something with the configured url;
        Console.WriteLine($"Performed {action} to Url {_externalUrl}");
        return Task.CompletedTask;
    }
}
```

This approach will and will not start preventing any other disruption later and you will quickly discover that there are missing settings.

Of course isn't optimal for all scenarios when you have tens configurations and/or more complex scenarios.

## A slight Improvement: Typed Settings

Having your setting file for example:

```json
{
    "MyServiceLevelSetting" : 10,
    "ExternalUrl" : "https://localhost",
    "ConnectionString": "ThisIsAConnectionString",
    "AnotherConfig": "ConfigExample"
  }
```

and a `class` that will represent your settings:

``` csharp
public class MyConfiguration
{
    public int MyServiceLevelSetting { get; set; }
    public string ExternalUrl { get; set; }
    public string ConnectionString { get; set; }
    public string AnotherConfig { get; set; }
}
```

yoy can now load it via the `Bind` method that will match the class properties names to the Settings names and assign their values respecting their types:

``` csharp
var myConfiguration = new MyConfiguration();
configuration.Bind(myConfiguration); //Match class properties to settings

appBuilder.Services.ConfigureMyServices(myConfiguration);
```

Now the Service implementation need to be updated and receive the MyConfiguration class directly:

``` csharp
public class MyThirdService : IMyFirstService
{
    string _externalUrl;
    string _connectionString;
    string _anotherConfig;
    public MyThirdService(MyConfiguration configuration)
    {
        _externalUrl = configuration.ExternalUrl;
        _connectionString = configuration.ConnectionString;
        _anotherConfig = configuration.AnotherConfig;
    }

    public Task DoSomething(string action)
    {
        //Do something with the configured url;
        Console.WriteLine($"Performed {action} to Url {_externalUrl}");
        return Task.CompletedTask;
    }
}

```

Using this approach you will have the same simplicity as the initial statement but now the properties are strongly typed and everyone will know what your library need to be configured in order to run properly.

## Another Improvements: using Options

Of course you can configure some custom validation rules in your class to avoid to have missing or wrong settings but the .net framework provide already a mecchanism for validate the settings using the Data Annotations and you can have them just installing the package `Microsoft.Extensions.Options.DataAnnotations` and decorate your `MyConfiguration` class with annotations like:

``` csharp
public class MyConfiguration
{
    [Required]
    [Range(1, 100)]
    public int MyServiceLevelSetting { get; set; }
    [Required]
    public string ExternalUrl { get; set; }
    [Required]
    public string ConnectionString { get; set; }
    public string AnotherConfig { get; set; }
}
```

As you can see above you can make some properties `required` some other that define `ranges` or several other annotations are available, see more in the reference link down here.

Now the service configuration need to be changed in somethig like the following example:

``` csharp
appBuilder.Services.AddOptions<MyConfiguration>()
    .Bind(configuration.GetSection(nameof(MyConfiguration)))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

Your json file need to have a secion named `MyConfiguration` as specified above (`configuration.GetSection(nameof(MyConfiguration))`) and all settings will be loaded

``` json
{
  "MyConfiguration": {
    "MyServiceLevelSetting" : 10,
    "ExternalUrl" : "http://localhost",
    "ConnectionString": "ThisIsAConnectionString",
    "AnotherConfig": "ConfigExample"
  }
}
```

Calling `.ValidateDataAnnotations().ValidateOnStart()` will ensure the data anotations are respected on the application startup or a Validation error will be thrown and the application will crash and you will know that some settings are missing or wrong right after the deployment for example.

Now your Service class need to be update like the following example:

``` csharp
public class MyFourthService : IMyFirstService
{
    string _externalUrl;
    string _connectionString;
    string _anotherConfig;

    public MyFourthService(IOptions<MyConfiguration> configuration)
    {
        _externalUrl = configuration.Value.ExternalUrl;
        _connectionString = configuration.Value.ConnectionString;
        _anotherConfig = configuration.Value.AnotherConfig;
    }

    public Task DoSomething(string action)
    {
        //Do something with the configured url;
        Console.WriteLine($"Performed {action} to Url {_externalUrl}");
        return Task.CompletedTask;
    }
}
```

Using this approach you will have strongly typed settings, validation on start up and will be clear to everyone what is needed to make you library run.

## Conclusions

Here, you have some examples of how we can configure services. However, to ensure clarity for everyone, you need to cease propagating `IConfiguration` as the "Configuration Dispatcher." Doing so will alleviate all the problems we have discussed here. Instead, start utilizing strongly typed configuration and ensure that your application won't start if some settings are invalid.

### References

- [Options pattern in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/options)

### Source code

- [GitHub](https://github.com/gcaferra/ConfigurationsBadPractices)