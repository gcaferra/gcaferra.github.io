---
title: "Container is disposed and should not be used"
tags:
- c#
- azure
---

# Container is disposed and should not be used

if you are you are facing the same problem here I can show you some hint on how I've fixed it.

## We stated to have this error from Azure Function that consume Service Bus messages

``` text
Container is disposed and should not be used: Container is disposed. You may include Dispose stack-trace into the message via: container.With(rules => rules.WithCaptureContainerDisposeStackTrace()) |  
```

Searching aroung I was pointed to this Issue on GitHub:

[Container is disposed and should not be used: Container is disposed](https://github.com/Azure/azure-functions-host/issues/5240)

But I've not found any solution there but following you can find my solution for it after several try I've figured out what to do. 
If you register the Context with the factory like the following

> Code from Startup.cs

``` csharp

public override void Configure(IFunctionsHostBuilder builder)
{
    var configuration = builder.GetContext().Configuration;

    builder.Services.AddDbContextFactory<SampleContext>(
        (IServiceProvider _, DbContextOptionsBuilder opts) =>
        {
            opts.UseSqlServer(configuration.GetConnectionString("ConnectionString"));
        });
}
```

you can have the Error seen above **"Container is disposed and should not be used"** but here there is the working example I've developed and seems to solve the issue.

First you should register your settings as single instance:

```charp
public override void Configure(IFunctionsHostBuilder builder)
{
  builder.Services.AddSingleton(x =>
        {
            CustomSettings myConfig = new();
            builder.GetContext().Configuration.Bind("ConnectionStrings", myConfig);
            return myConfig;
        });
    
```

then during the cold start the Startup.Config is called and you setting is saved into the **CustomSettings** Class

After the Settings are registered you should change also how you register you **DbContext** registering it directly like:

```csharp
builder.Services
.AddDbContext<ConfirmationContext>((IServiceProvider sp, DbContextOptionsBuilder opts) => 
                opts.UseSqlServer(sp.GetRequiredService<AnswersConsumerSettings>().ConfirmationDb));
```

In my understanding after the cold start the function calls Dispose of Context and then this call fails when it happen:

```charp
builder.GetContext().Configuration
```

The factory try to access to the configuration on each new execution of the function run and this accessing the configuration on a Disposed context throw the exception above.

Hope this post helps someone to save some time
