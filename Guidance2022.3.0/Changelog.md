# What's different

- appsettings configuration file.


## appsettings configuration file.

In .NET 7 its recommended to change the configuration based on the builder.configuration. => ASP0013 warning.

Previously the code generated was 
```chsarp

var builder = WebApplication.CreateBuilder(args);
	// Remove sensitive information that can be exploit by hackers.
	builder.WebHost.ConfigureKestrel(options => options.AddServerHeader = false);

    builder.Host
		.ConfigureAppConfiguration((hostingContext, config) =>
		{
			//foreach (var source in config.Sources.Where(s => s.GetType().Name.Contains("Json")).ToList())
			//{
			//	config.Sources.Remove(source);
			//}

			config.Sources.Clear();

			var env = hostingContext.HostingEnvironment.EnvironmentName;

			config.AddJsonFile("configs/appsettings.json", true, true);
			config.AddJsonFile($"configs/appsettings.{env}.json", true, true);
			config.AddJsonFile("configs/reverseproxy.json", true, true);
			config.AddJsonFile($"configs/reverseproxy.{env}.json", true, true);

			config.AddEnvironmentVariables();
		})
		.UseSerilog((ctx, lc) => lc
					.ReadFrom.Configuration(ctx.Configuration))

```

Now the code is

```chsarp
var builder = WebApplication.CreateBuilder(args);
	// Remove sensitive information that can be exploit by hackers.
	builder.WebHost.ConfigureKestrel(options => options.AddServerHeader = false);

    var Configuration = builder.Configuration;
    Configuration.Sources.Clear();

	var env = builder.Environment.EnvironmentName;

	Configuration.AddJsonFile("configs/appsettings.json", true, true);
	Configuration.AddJsonFile($"configs/appsettings.{env}.json", true, true);
	Configuration.AddJsonFile("configs/reverseproxy.json", true, true);
	Configuration.AddJsonFile($"configs/reverseproxy.{env}.json", true, true);

	Configuration.AddEnvironmentVariables();

    builder.Host
		.UseSerilog((ctx, lc) => lc
					.ReadFrom.Configuration(ctx.Configuration))

```

=> you have also to remove the statement assigning Configuration <=</br>
~~IConfiguration Configuration = builder.Configuration;~~


```chsarp
```


# What's new

# Breaking changes

- WebSdkAnalyzersPath

## WebSdkAnalyzersPath

Up to .NET 6, it was possible to have the WebSdkAnalyzersPath activated. Now this is not supported.
The configuration has been removed from the facade and interface project.