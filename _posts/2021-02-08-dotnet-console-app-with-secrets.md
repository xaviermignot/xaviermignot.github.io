---
layout: post
title: Using .NET Secret Manager with console applications
tags: [dotnet, csharp]
---

A few ~~hours~~ days ago I was starting to build a small demo as a console app that uses a secret that I didn't want to commit in the related repository (like a device connection string for instance).  
So I asked myself *Why not try to use this .NET secret thing that I never use normally ?*  
Well, as it was not as simple a I initially thought, I have ended up with the idea of writing this post to share what I have learnt.

All the samples are written in .NET 5 and are available [here](https://github.com/xaviermignot/dotnet-console-secret-samples){:target="_blank"}.  
I have put everything in the `Program.cs` file for each sample for readability, I don't do this normally 😉

## What is the .NET Secret Manager ?

It's a tool to store secrets away from the repository structure during development. The full documentation is available [here](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets){:target="_blank"}, here is what it does in a few bullet points:
- The `dotnet user-secrets init` command generates a *UserSecretsId*, a GUID stored in a element of the *csproj*
- Setting a secret value is done using the `dotnet user-secrets "<key>" "<value>"` command, or `dotnet user-secrets "<section>:<key>" "<value>"` if you want to use section or map to a POCO (more on this below)
- Secrets are stored in a *json* file in a folder named after the *UserSecretsId*, located somewhere in you home directory (depending on you OS)

The official documentation is centered around ASP.NET Core, and the information I have found elsewhere was a little out of date, or missing using statements or package references. So as using the Secret Manager for a console application was not that simple, I'll try to illustrate it with 3 samples. Let's start with the most basic one.


## Sample 1: Store a single value as a secret

For this first sample it's pretty simple, first install the `Microsoft.Extensions.Configuration.UserSecrets` package, then initialize the Secret Manager and add a secret with the following commands:
```console
$ dotnet user-secrets init
$ dotnet user-secrets set "MySecret" "my secret value"
```

In the code, you'll need 3 things:
- A reference to the `Microsoft.Extensions.Configuration.UserSecrets` package
- A using statement to the `Microsoft.Extensions.Configuration` namespace
- Call the `AddUsersSecrets<Program>()` before building the `ConfigurationBuilder` like this:
```csharp
var configuration = new ConfigurationBuilder()
    .AddUserSecrets<Program>()
    .Build();
```

Then you can access a secret value as simple as that:
```csharp
var secretValue = configuration["MySecret"];
```

Here is the full code:
```csharp
using System;
using Microsoft.Extensions.Configuration;

namespace SingleValueSample
{
    class Program
    {
        static void Main(string[] args)
        {
            var configuration = new ConfigurationBuilder()
                .AddUserSecrets<Program>()
                .Build();

            var secretValue = configuration["MySecret"];

            Console.WriteLine($"The secret value is: {secretValue}");
        }
    }
}
```


## Sample 2: Map to a POCO object

Getting a single secret is a first step, if you have many secrets it's probably a good idea to map them to a POCO like this class:
```csharp
class MyConfiguration
{
    public string MyFirstSecret { get; set; }
    public string MySecondSecret { get; set; }
}
```

To tell the Secret Manager about your POCO, simply name your secrets with the `<class>:<property>` pattern like this:
```console
$ dotnet user-secrets init
$ dotnet user-secrets set "MyConfiguration:MyFirstSecret" "my first secret value"
$ dotnet user-secrets set "MyConfiguration:MySecondSecret" "my second secret value"
```

Starting from the previous sample you'll need to add two more packages references (in addition to `Microsoft.Extensions.Configuration.UserSecrets`):
- `Microsoft.Extensions.DependencyInjection`
- `Microsoft.Extensions.Options.ConfigurationExtensions`

In the code, when you add the user secrets configuration source, use you POCO class instead of `Program` in the type parameter:
```csharp
var configuration = new ConfigurationBuilder()
    .AddUserSecrets<MyConfiguration>()
    .Build();
```

Then, using the DI and configuration packages, register a configuration instance for your POCO class and build a `ServiceProvider` that will let you get your secrets as a POCO like any other service:
```csharp
var services = new ServiceCollection()
    .Configure<MyConfiguration>(configuration.GetSection(nameof(MyConfiguration)))
    .AddOptions()
    .BuildServiceProvider();

var myConf = services.GetService<IOptions<MyConfiguration>>();
```

Here is the full code with the usings I haven't mentioned yet:
```csharp
using System;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;

namespace MapToPocoSample
{
    class Program
    {
        static void Main(string[] args)
        {
            var configuration = new ConfigurationBuilder()
                .AddUserSecrets<MyConfiguration>()
                .Build();

            var services = new ServiceCollection()
                .Configure<MyConfiguration>(configuration.GetSection(nameof(MyConfiguration)))
                .AddOptions()
                .BuildServiceProvider();
            
            var myConf = services.GetService<IOptions<MyConfiguration>>();
            Console.WriteLine($"The first secret is: {myConf.Value.MyFirstSecret}");
            Console.WriteLine($"The second secret is: {myConf.Value.MySecondSecret}");
        }
    }

    class MyConfiguration
    {
        public string MyFirstSecret { get; set; }
        public string MySecondSecret { get; set; }
    }
}
```


## Sample 3: Using the .NET Generic Host

In this last sample we will use the [.NET Generic Host](https://docs.microsoft.com/en-us/dotnet/core/extensions/generic-host){:target="_blank"} with a `BackgroundService` implementation. This is the step where your console app moves from the "script-style console app" stage to the "real-world console app with full DI power & stuff" level.

We will use the default host builder, which as stated in the [documentation](https://docs.microsoft.com/en-us/dotnet/core/extensions/generic-host#default-builder-settings){:target="_blank"} comes with some nice features out of the box: console logging, environment variables, *appsettings.json* configuration, and Secret Manager __when the app runs in the *Development* environment__.  

So if you have an environment variable `DOTNET_ENVIRONMENT` whose value is `Development`, you don't even need to reference the `Microsoft.Extensions.Configuration.UserSecrets` package and to call the `AddUserSecrets` method to get your secrets, the default host will do that for you.  
You will find more informations on environments in .NET [here](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/environments){:target="_blank"}, but just to set the environment variable you need to do:
- `export DOTNET_ENVIRONMENT=Development` if you use Bash
- `$env:DOTNET_ENVIRONMENT='Development'` if you use Powershell

Starting from the previous sample with the POCO class, the Secret Manager commands are the same. You need to reference the following packages:
- `Microsoft.Extensions.Hosting`
- `Microsoft.Extensions.Options.ConfigurationExtensions`

In the code of the `Program` class, there is no mention about user secrets, we use the default host builder like this in the `Main` method:
```csharp
Host.CreateDefaultBuilder()
    .ConfigureServices((hostContext, services) =>
    {
        services.Configure<MyConfiguration>(hostContext.Configuration.GetSection(nameof(MyConfiguration)));
        services.AddHostedService<ConsoleWorker>();
    }).Build().Run();
```
This code creates the default host builder, registers the configuration for the POCO class, and registers the `ConsoleWorker` class as a hosted service. This class is an implementation of `BackgroundService`, and contains the logic of the console app.  
The dependencies, including the POCO instance containing the secrets, can be injected directly in the constructor.  

Here is the whole code of this sample:
```csharp
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace UseHostBuilder
{
    class Program
    {
        static void Main(string[] args)
        {
            Host.CreateDefaultBuilder()
                .ConfigureServices((hostContext, services) =>
                {
                    services.Configure<MyConfiguration>(hostContext.Configuration.GetSection(nameof(MyConfiguration)));
                    services.AddHostedService<ConsoleWorker>();
                }).Build().Run();
        }
    }

    class MyConfiguration
    {
        public string MyFirstSecret { get; set; }
        public string MySecondSecret { get; set; }
    }

    class ConsoleWorker : BackgroundService
    {
        private readonly MyConfiguration _myConfiguration;
        private readonly ILogger _logger;

        public ConsoleWorker(IOptions<MyConfiguration> myConfiguration, ILogger<ConsoleWorker> logger)
        {
            _myConfiguration = myConfiguration.Value;
            _logger = logger;
        }

        protected override Task ExecuteAsync(CancellationToken stoppingToken)
        {
            _logger.LogInformation($"The first secret is: {_myConfiguration.MyFirstSecret}");
            _logger.LogInformation($"The second secret is: {_myConfiguration.MySecondSecret}");

            return Task.CompletedTask;
        }
    }
}
```


## Wrapping up

I hope you will find this post useful, all the information was already available online but I needed some work to properly understand how the Secret Manager works with console apps.  
Nothing stops me from using it event in small POCs, I hope it's the same for you now.