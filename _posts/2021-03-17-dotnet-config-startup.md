---
layout: post
title: "Configuring ASP.NET Core: The Startup Class"
categories: Programming
excerpt: Learning .Net Core configuration
---

In my goal to further improve my understanding of C# .NET Core and ASP.NET Core, I've realised that I don't fully understand how the `Startup.cs` class actually works.

This code is usually pre-generated by Visual Studio and I've never really needed to add to or edit this class, including at my job. Whenever I did, it was often to copy/paste code from documentation when using new libraries i.e. AutoFac.

But the truth is I don't understand how this class actually works or how it is intended to be used. So today I had thought to actually read the documentation to learn more.

## The Startup Class &#x1f477;

ASP.NET Core apps uses a Startup class, named `Startup` by convention.

The `Startup` class Includes a Configure method to create the app's request processing pipeline.

It also **optionally** includes a `ConfigureServices` method to configure the app's services.

A service is a reusable component that provides app functionality and are registered in `ConfigureServices` to be consumed across the app via dependency injection (DI) or ApplicationServices.

```c#
    public class Startup
    {
        public IConfigurationRoot Configuration { get; private set; }
        public ILifetimeScope AutofacContainer { get; private set; }

        public Startup(IWebHostEnvironment env)
        {
            var builder = new ConfigurationBuilder()
                .SetBasePath(env.ContentRootPath)
                .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
                .AddEnvironmentVariables();
            Configuration = builder.Build();
        }

        public void ConfigureServices(IServiceCollection services)
        {
            services.AddControllers();
            services.AddDbContext<TGTContext>(options =>
                    options.UseSqlServer(Configuration.GetConnectionString("TGTContext")));
        }

        public void ConfigureContainer(ContainerBuilder builder)
        {
            builder.RegisterModule(new AutofacModule());
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseRouting();
            app.UseAuthorization();
            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });
        }
    }

```

The `Startup` class is specified when the app's **Host** is built and is typically specified by calling the `WebHostBuilderExtensions.UseStartup<TStartup>` method on the host builder:

```c#
    public class Program
    {
        public static void Main(string[] args)
        {
            var host = Host.CreateDefaultBuilder(args)
                .ConfigureLogging(logging =>
                {
                    logging.AddConsole();
                })
                .UseServiceProviderFactory(new AutofacServiceProviderFactory())
                .ConfigureWebHostDefaults(webHostBuilder =>
                {
                    webHostBuilder
                      .UseContentRoot(Directory.GetCurrentDirectory())
                      .UseIISIntegration()
                      .UseStartup<Startup>(); // HERE is where the Startup class is specified
                })
                .Build();

            host.Run();
        }
    }

```

The host provides services that are available to the `Startup` class constructor. The app adds additional services via `ConfigureServices`. Both the host and app services are available in `Configure` and throughout the app.

Only the following service types can be injected into the Startup constructor when using the Generic Host (IHostBuilder):

- `IWebHostEnvironment`
- `IHostEnvironment`
- `IConfiguration`

## The ConfigureServices Method &#x1f6e0;

The `ConfigureServices` method is:

- Optional
- Called by the hose **before** the `Configure` method to configure the app's services.
- Where configuration options are set by convention
  - The host may configure some services before Startup methods are called however.

For features that require substantial setup, there are Add{Service} extension methods on IServiceCollection.
For example, AddDbContext, AddDefaultIdentity, AddEntityFrameworkStores, and AddRazorPages


```c#
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {

        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(
                Configuration.GetConnectionString("DefaultConnection")));
        services.AddDefaultIdentity<IdentityUser>(
            options => options.SignIn.RequireConfirmedAccount = true)
            .AddEntityFrameworkStores<ApplicationDbContext>();

        services.AddRazorPages();
    }
}
```
Adding services to the service container makes them available within the app and in the Configure method. The services are resolved via dependency injection.

## The Configure Method &#x2699;

The Configure method is used to specify how the app responds to HTTP requests.

The request pipeline is configured by adding middleware components to an `IApplicationBuilder` instance.

`IApplicationBuilder` is available to the Configure method, but it isn't registered in the service container. Hosting creates an `IApplicationBuilder` and passes it directly to `Configure`.

```c#
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddRazorPages();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }
        else
        {
            app.UseExceptionHandler("/Error");
            app.UseHsts();
        }

        app.UseHttpsRedirection();
        app.UseStaticFiles();

        app.UseRouting();

        app.UseAuthorization();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapRazorPages();
        });
    }
}
```

Each Use extension method adds one or more middleware components to the request pipeline. For instance, UseStaticFiles configures middleware to serve static files.

Each middleware component in the request pipeline is responsible for invoking the next component in the pipeline or short-circuiting the chain, if appropriate.

Additional services, such as `IWebHostEnvironment`, `ILoggerFactory`, or anything defined in `ConfigureServices`, can be specified in the `Configure` method signature.
These services are injected if they're available.

## Summary &#x1f4dd;

It's interesting how little I knew about configuring a .NET Core app. In many ways, I don't actually *need* to know, I managed to work with .Net Core for a few years and not fully understand how all of this is set up. It's like not needing to know how Windows OS works behind the scenes, you'd just expect it to work "out of the box".

This is probably due to the tooling and pre-generated code that comes with working on .NET and C# which is likely designed to be as streamlined as possible for new devs, which is something I really like about this tech stack.

That said, when it's time to customise or add new configuration or even to read how an existing application is configured, it can lead to confusion and challenges without understanding how that part of the code works, I've also found it difficult to understand exceptions or errors, furthermore, Googling those and finding anything helpful on StackOverflow.

The topic of configuration for .NET Core alone seems to be huge and perhaps learning *everything* is not the most productive use of time, that said, understanding some fundamentals is not a bad idea.

This post in general was *copied* from the Microsoft documentation, but writing it here at least allows me to *read* the docs and try to put it in my own words (although most of it was a direct copy).

There's more to be learnt, but maybe best to do it pragmatically when I encounter an issue.

### External sources &#x1f4a1;

[App startup in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/startup?view=aspnetcore-5.0)
