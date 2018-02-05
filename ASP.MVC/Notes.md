# Thoughts on ASP.NET MVC Core

This is a collection of my thoughts and understanding of ASP.NET MVC Core. It is meant as a refernce for myself and a learning aid.

## The Startup Class

The `Startup` class is where all of the configuration for the web app is done. There are three methods that are called as a part of spin up of the website: `Startup` (the constructor), `ConfigureServices`, and `Configure`.

### Startup

The constructor is obviously the first thing that is called. It is recommended to create a private property to hold the `IConfiguration` object. This will make is easy for the services that will be instantiated in the next two calls to lookup configuration settings.

```csharp
private IConfiguration _configuration;

public Startup(IConfiguration configuration)
{
    _configuration = configuration;
}
```

### ConfigureServices

The next method that is called is the `ConfigureServices` method. This method takes a `IServiceCollection` which is used to attach services to the web app. These services can use the `_configuration` property to look up how they are to be setup. These are the services that the app will use to perform the Dependency Injection throughout the rest of the app.

One of the important things about adding services is the lifetime of the service. There are three different options: Singleton, Scoped, and Transient.

A Singleton service will only have a single instance of the service available to the app.

A Scoped service will instantiate a instance for each call. The same instance will be used throughout the call. This means that when a request comes in, a single instance will be created and will be reused for everything that requires an instance of that service. A good example is a data provider which connects to a database.

A Transient service will have a new instance created for every time it is used.

### Configure

The configure method is where the middleware is setup for the app. When someone makes a call to the webserver the first thing that is going to happen is that the call is going to start making its way through the middleware. It is evaluated in the order that the middleware is added to the app. In the following example the `Rewriter` middleware is called before the `StaticFiles` middleware. In the following code the `app` object is an instance of `IApplicationBuilder` which has been injected into the method.

```csharp
app.UseRewriter(new RewriteOptions()
    .AddRedirectToHttpsPermanent());

app.UseStaticFiles();

app.UseNodeModules(env.ContentRootPath);

app.UseAuthentication();

app.UseMvc(ConfigureRoutes);
```

This means that the actual call to the controller happens after all of the other middleware has been passed through. This is because the `Mvc` middleware is the last one added to the `app` object. This is why it is important that the `Authentication` middleware is added to the app before the `Mvc` middleware. The order they are added to the `IApplicationBuilder` is the order they are called.

## The Request Lifecycle

This next section discusses what actually happens to a request that the `Mvc` middleware handles. This means that all of the other middleware have already been passed through.
