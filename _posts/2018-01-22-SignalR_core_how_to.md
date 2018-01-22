---
layout: post
title: SignalR Core- how to start on currency broadcaster sample
tags: .NET SignalR Core
---

Today I'd like to take a look on a new version of SignalR package (to be honest, it's official name is `Microsoft.AspNetCore.Signalr`) and implement currency updates broadcaster sample. 

For now (22 Jan 2018) the package is still in alpha version, but the release one is going to be published very soon with an official AspNetCore release.

## Default startup

Let's start with adding the nuget package to our newly created project with the usage of CLI:

`Install-Package Microsoft.AspNetCore.SignalR -Version 1.0.0-alpha2-final`

Or a nuget package manager:

![signalr_nuget](/images/post/signalr_nuget.png)

After that we have to tell to our application that we are going to use recently added SignalR package. First of all, we need to update *Startup.cs*'s `ConfigureServices` method like this:

```c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddSignalR();
    services.AddMvc();
}
```

Let's take a look on internal details of `AddSignalR`. It's just an extension method over `IServiceCollection` interface and is used for a default signalr services registration:

```c#
services.Configure<HubOptions>(configure);
services.AddSockets();
services.AddSingleton(typeof (HubLifetimeManager<>), typeof (DefaultHubLifetimeManager<>));
services.AddSingleton(typeof (IHubProtocolResolver), typeof (DefaultHubProtocolResolver));
services.AddSingleton(typeof (IHubContext<>), typeof (HubContext<>));
services.AddSingleton(typeof (IHubContext<,>), typeof (HubContext<,>));
services.AddSingleton(typeof (HubEndPoint<>), typeof (HubEndPoint<>));
services.AddScoped(typeof (IHubActivator<>), typeof (DefaultHubActivator<>));
services.AddAuthorization();
```

Then we have to declare a new class which is derived from the `Hub` one. At this sample the only purpose is to handle a particular endpoint requests, but it also could contains a logic, which should be accessable from a frontend (or fired due to `OnConnected` and `OnDisconnected` events). I've named it as `CurrencyHub` and left empty:

```c#
public class CurrencyHub : Hub
{

}
```

The last but not least we need to update the method `Configure` with the `UseSignalR` method call:

```c#
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    // ..   
            
    app.UseSignalR(routes => {
        routes.MapHub<CurrencyHub>("currency");
    });

    // ..
}
```

We use the `UseSignalR` method for configuring our routes table: every `/currency` http request will be handled by our `CurrencyHub` class.

If everything have been done in a proper way then the next output should be appeared:

![signalr_first_currency](/images/post/signalr_first_currency.png)

Fow now your backend part of the application is fully configured and could be used as a base for a futher implementation. But as I've mentioned before, I'd like to implement a working sample and it's only a half of way ;)

## Sample implementation

I'd like to implement a pretty straightforward stock currencies display. This display should contains information about a set of currencies (let it be 5 pairs) and has an ability to update it in real-time (twice per second) for every connected user simultaneously.

The sample contains three main parts:
- Broadcaster as a component, which should fetch or listen an information about currency updates;
- Notifier as a component, which should notify all active connections with up-to-date data;
- Display table as a currency information view;

I've implemented a broadcaster with the usage of `IHostedService`. It's pretty usefull interface with a great benefit like hosting your component as an ASP.NET Core background worker:

```c#
public class CurrencyListenerService : IHostedService, IDisposable
{
    //..

    public async Task StartAsync(CancellationToken cts)
    {
        while (!cts.IsCancellationRequested)
        {
            var currencies = await FetchCurrencyUpdates();

            await Broadcast(currencies);

            await Task.Delay(_delay, cts);
        }
    }

    private Task<Dictionary<string, double>> FetchCurrencyUpdates()
    {
        var data = new Dictionary<string, double>
        {
            {"EUR", 80 + _rnd.NextDouble()},
            {"USD", 57 + _rnd.NextDouble()},
            {"BYN", 28 + _rnd.NextDouble()}
        };

        return Task.FromResult(data);
    }

    //..
}
```

The broadcaster part of this service have to be implemented with the usage of `HubLifetimeManager<THub>`. Based on generic type `THub` this class contains an information about all active connections of a particular hub and has an opportunity to notify them throught a function call, which is declared at frontend:

```c#
private async Task Broadcast(Dictionary<string, double> data)
{
    await _hubManager.InvokeAllAsync("currenciesUpdated", 
        new object[] { data });
}
```

To host this service we need to register it at `IServiceCollection` like this:

```c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton<IHostedService, CurrencyListenerService>();

    //..
}
```

As I've already mentioned before we need to declare a function at frontend, which could be called from a backend side for a notifier purposes (let it be `currenciesUpdated` function).

 


