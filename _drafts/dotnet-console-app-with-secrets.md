---
layout: post
title: Using .NET Secret Manager with console applications
tags: [dotnet]
---

A few hours ago I was starting to build a small demo that use a secret that I don't want to commit in the related repository (like a device connection string for instance).  
So I asked myself *Why not try to use this .NET secret thing that I never use normally ?*  
Well, as it was not as simple a I initially though, I have ended up with the idea of writing this post to share what I have learnt.


## What is the .NET Secret Manager ?

It's a tool to store secrets away from the repository structure during development. The full documentation is available [here](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets), here is what it does in a few bullet points:
- The `dotnet user-secrets init` command generates a *UserSecretsId*, a GUID stored in a element of the *csproj*
- Setting a secret value is done using the `dotnet user-secrets "<key>" "<value>"` command, or `dotnet user-secrets "<section>:<key>" "<value>"` if you want to use section or map to a POCO (more on this below)
- Secrets are stored in a *json* file in a folder named after the *UserSecretsId*, located somewhere in you home directory (depending on you OS)

The official documentation is centered around ASP.NET Core, and the information I have found elsewhere was a little out of date, or missing using statements or packages references. So as using the Secret Manager for a console application was not that simple, I'll try to illustrate it with 3 samples. Let's start with the most basic one.


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

Then you can access a secret value:
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