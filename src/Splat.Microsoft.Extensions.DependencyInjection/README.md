# Splat.Microsoft.Extensions.DependencyInjection

## Using Autofac

`Splat.Microsoft.Extensions.DependencyInjection` is an adapter for `IMutableDependencyResolver`.
It allows you to register your application dependencies in a MS DI `Container`.  You can then use the container as Splat's internal dependency resolver.
You can also choose to have the conainer controlled externally, for example using a [Generic Host](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host?view=aspnetcore-2.2#configureappconfiguration).

### Register the Container

```cs
// call this method from your apps constructor or early initialization code

using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging.EventLog;
using ReactiveUI;
using Splat;
using Splat.Microsoft.Extensions.DependencyInjection;
using Splat.Microsoft.Extensions.Logging;
using System;
using System.Linq;
// using Windows.UI.Xaml

sealed partial class App // : Application
{   
  public App()
  {
    Init();
    /* Some other initialization stuff */
  }                     

  public IServiceProvider Container { get; private set; }

  void Init()
  {
    var host = Host
      .CreateDefaultBuilder()
      .ConfigureServices(services =>
      {
        services.UseMicrosoftDependencyResolver();
        var resolver = Locator.CurrentMutable;
        resolver.InitializeSplat();
        resolver.InitializeReactiveUI();

        // Configure our local services and access the host configuration
        ConfigureServices(services);
      })
      .ConfigureLogging(loggingBuilder =>
      {
        /*
        //remove loggers incompatible with UWP
        {
          var eventLoggers = loggingBuilder.Services
            .Where(l => l.ImplementationType == typeof(EventLogLoggerProvider))
            .ToList();

          foreach (var el in eventLoggers)
            loggingBuilder.Services.Remove(el);
        }
        */

        loggingBuilder.AddSplat();
      })
      .UseEnvironment(Environments.Development)
      .Build();

    // Since MS DI container is a different type,
    // we need to re-register the built container with Splat again
    Container = host.Services;
    Container.UseMicrosoftDependencyResolver();
  }

  void ConfigureServices(IServiceCollection services)
  {
    /* register your personal services here, for example */
    services.AddSingleton<MainViewModel>();
    services.AddTransient<MainPage, IViewFor<MainViewModel>>();
    services.AddTransient<SecondaryPage, IViewFor<SecondaryViewModel>>();    
    services.AddTransient<SecondaryViewModel>();
  }
}  
```

### Register the Adapter to Splat

First, call:

```cs
IServiceCollection services = ...
services.UseAMicrosoftDependencyResolver();
```

then, if you wish to have the MS DI container controlled by an external service other than Splat, re-register it as above (using Generic Host), or as follows:

```cs
IServiceProvider container = services.BuildServiceProvider();
container.UseAMicrosoftDependencyResolver();
```

### Use the Locator

Now calls to `Locator.Current` will resolve to the underlying Microsoft DI container.  In the case of ReactiveUI, platform registrations will now happen in the MS DI container.  So when the platform calls to resolve dependencies, the will resolve from the MS DI container.