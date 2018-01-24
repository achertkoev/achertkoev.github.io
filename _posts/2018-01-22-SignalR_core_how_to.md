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

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddSignalR();
    services.AddMvc();
}
```

Let's take a look on internal details of `AddSignalR`. It's just an extension method over `IServiceCollection` interface and is used for a default signalr services registration:

```csharp
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

```csharp
public class CurrencyHub : Hub
{

}
```

The last but not least we need to update the method `Configure` with the `UseSignalR` method call:

```csharp
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

I'd like to implement a pretty straightforward stock currencies display. This display should contains information about a set of currencies and has an ability to update it in real-time (once per second) for every connected user simultaneously.

The sample contains three main parts:
- Broadcaster as a component, which should fetch or listen an information about currency updates;
- Notifier as a component, which should notify all active connections with up-to-date data;
- Display table as a currency information view;

I've implemented a broadcaster with the usage of `IHostedService` ([https://blogs.msdn.microsoft.com/cesardelatorre/2017/11/18/implementing-background-tasks-in-microservices-with-ihostedservice-and-the-backgroundservice-class-net-core-2-x/](read more)). It's pretty usefull interface with a great benefit like hosting your component as an ASP.NET Core background worker:

```csharp
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
            {"EUR", Get(69) }, {"USD", Get(56) }, {"GBR", Get(79) },
            {"INR", Get(1) }, {"CAD", Get(45) }, {"MAD", Get(6) },
            {"AUD", Get(45) }, {"TRY", Get(15) }, {"AZN", Get(33) },
        };

        return Task.FromResult(data);
    }

    //..
}
```

The broadcaster part of this service have to be implemented with the usage of `HubLifetimeManager<THub>`. Based on generic type `THub` this class contains an information about all active connections of a particular hub and has an opportunity to notify them throught a function call, which is declared at frontend:

```csharp
private async Task Broadcast(Dictionary<string, double> data)
{
    await _hubManager.InvokeAllAsync("currenciesUpdated", 
        new object[] { data });
}
```

To host this service we need to register it at `IServiceCollection` like this:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton<IHostedService, CurrencyListenerService>();

    //..
}
```

As I've already mentioned before we need to declare a function at frontend, which could be called from a backend side for a notifier purposes (let it be `currenciesUpdated` function).

Recently i've taken a look at [Vue.js](https://vuejs.org/) and found it pretty nice and straightforward. Bellow you could see several parts of code with the usage of this library:

```javascript
<script>
    var app = new Vue({
        el: "#app",
        data: {
            currencies: null,
            changes: {}
        },
        methods: {
            hubConnect: function() {
                var connection = new signalR.HubConnection('/currency');

                connection.on('currenciesUpdated',
                    function(data) {
                        this.currencies = data;
                    }.bind(this));

                connection.onclose(e => {
                    if (e) {
                        console.log('Connection closed with error: ' + e, 'red');
                    } else {
                        console.log('Disconnected', 'green');
                    }
                });

                connection.start();
            }
        },
        watch: {
            currencies: function (newVal, oldVal) {
                if (!newVal || !oldVal) return;
                for (var key in newVal) {
                    if (newVal.hasOwnProperty(key)) {
                        this.changes[key] = newVal[key] > oldVal[key];
                    }
                }
            }
        },
        mounted: function() {
            this.hubConnect();
        }
    });
</script>
```

And a layout:

```html
<table class="table table-bordered table-striped" v-if="currencies">
    <tr>
        <th></th>
        <th v-for="(value, key) in currencies">{{ key.toUpperCase() }}</th>
    </tr>

    <tr>
        <th>
            <img src="http://www.xe.com/themes/xe/images/flags/rub.png" /> RUB
        </th>
        <td v-for="(value, key) in currencies"
            v-bind:class="{ 'up': changes[key], 'down': !changes[key] }">
            {{ value }}
        </td>
    </tr>
</table>
```

Looks really nice, isn't it?

It doesn't really matter which frontend framework you are going to use. The only thing is **really important** is do not forget to append a reference to signalr library, because otherwise a console output won't let you got to sleep for a while:

```html
<script src="~/lib/signalr-client-1.0.0-alpha2-final.min.js"></script>
```

For now it's still in alpha (and more than 158 issues is still open), but npm module is already available: ([@aspnet/signalr-client](https://www.npmjs.com/package/@aspnet/signalr-client)):

>
> **Usage**:
> 
> To use the client in a browser, copy *.js files from the dist/browser folder to your script folder include on your page using the *script* tag.
>
> **Example**:
> ```javascript
> let connection = new signalR.HubConnection('/chat');
>  
> connection.on('send', data => {
>   console.log(data);
> });
> 
> connection.start()
>     .then(() => connection.invoke('send', 'Hello'));
> ```
>

And finally here is a demo of the result:

![chsarp_71_ep](/images/post/currency-signalr-result.gif)

Thanks for reading, I really hope you found it usefull and feel free to comment!

Summary:

1. [Github repository](https://github.com/FSou1/CoreSignalR.Sample);
2. [Announcing SignalR (alpha) for ASP.NET Core 2.0](https://blogs.msdn.microsoft.com/webdev/2017/09/14/announcing-signalr-for-asp-net-core-2-0/);
2. [Implementing background tasks in .NET Core 2.x webapps or microservices with IHostedService and the BackgroundService class](https://blogs.msdn.microsoft.com/cesardelatorre/2017/11/18/implementing-background-tasks-in-microservices-with-ihostedservice-and-the-backgroundservice-class-net-core-2-x/);

