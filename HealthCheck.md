# HealthChecks

The guidance generates a health check strategy based on:
- AspNetCore.HealthChecks
- HealthChecks from [Xabaril](https://github.com/xabaril/AspNetCore.Diagnostics.HealthChecks).
- HealthCheck UI

Based on this we will be able to implement 3 things:

- Self monitoring of the services
  - on Kubernetes:
    - Readiness probe
    - Liveness probe.
  - Business process checks.
- Alerting in conjunction with Seq
- Portal to see the current status.

## Strategy

Each services will contain the capability to monitor itself and to report the status to an external system. Currently Seq will be used.

2 self monitorings will be implemented:
- Technical monitoring (Redis, Mongo, Sql, etc...)
- Business monitoring (by coding some defined business process defined by the business).

The Yarp service will add one feature which is a healthcheck UI offering a dashboard with the status of all the services running for the app.

## HealthChecks

See the [Xabaril](https://github.com/xabaril/AspNetCore.Diagnostics.HealthChecks) nuget packages available for standard components we use.

The Guidance creates 3 endpoints:
- readiness
- liveness
- health summary

```csharp

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapHealthChecks("/health/ready", new HealthCheckOptions()
                {
                    Predicate = (check) => check.Tags.Contains("ready"),
                });

                endpoints.MapHealthChecks("/health/live", new HealthCheckOptions()
                {
                    Predicate = (check) => check.Tags.Contains("live")
                });
                endpoints.MapHealthChecks("healthz", new HealthCheckOptions()
                {
                    Predicate = _ => true,
                    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
                });
                endpoints.MapReverseProxy();
            });

```

The predicate is used to filter the healthcheck rules defined and where those rules must be applied. One rule can be
applied to more than one endpoint.

By design the healthz endpoint is the summary of all.

The liveness and readiness ones are used in kubernetes to determine if the service can be used when a new instance of a pod is used (readiness) and if the service is
still valid (liveness).

Probably technical component (dependencies) like Sql Server, MongoDB, Redis, etc... will determine if the system is healthy for readiness and liveness probe. 
Other rules (more business oriented) will be used by the healtz endpoint to determine if the service is performing its job correctly.

3 States are possible:
- healthy
- degraded
- unhealthy

Degarded means that there is a problem but the service is still considered as operational. This can be the case if a circuit breaker is implemented for a cache by example and
the system can still continue to work without cache but with less performance. 

### Rules

The guidance will define the entry point in the startup.cs file to enter the the standard healthchecks and your custom ones plus to report this to Seq.

```csharp

           services.AddHealthChecks()
                .AddSeqPublisher((options) => { options.Endpoint = Configuration["AppSettings:seq_ingest"]; });

```

As you see no healthchecks are defined...

Up to you based on your dependencies to add them.

Here an example with Redis.

```csharp
           services.AddHealthChecks()
                .AddRedis(Configuration["Redis.Settings:ConnectionString"], "Redis", tags: new[] { "live", "ready" })
                .AddSeqPublisher((options) => { options.Endpoint = Configuration["AppSettings:seq_ingest"]; });
```

As you see here I have defined that Redis is mandatory when the service is starting and started!

## Healthcheck UI.

On the Yarp project the guidance add the UI capability. This will in fact add the following service.

```csharp

            services
                .AddHealthChecksUI()
                .AddInMemoryStorage();

```

The InMemory storage is used but for production I recommand to persist this in a database like SqlServer.

Each micro-services must be now registered in the UI to reach the healthz endpoint!

This is done at the level of the appsettings file where the entry
When the project is created the following entry will be added with only Yarp registered.

```json

{
  "HealthChecksUI": {
    "HealthChecks": [
      {
        "Name": "Yarp",
        "Uri": "https://localhost:44???/healthz"
      }
    ],
    "EvaluationTimeinSeconds": 6,
    "MinimumSecondsBetweenFailureNotifications": 60
  }
}
```

Each time a service is added an entry is also added in the section.

The configuration for Kubernetes is more or less the same but the internal 8080 port (ClusterIP) is used.

```json

{
  "HealthChecksUI": {
    "HealthChecks": [
      {
        "Name": "Yarp",
        "Uri": "http://yarpinternal:8080/healthz"
      },
      {
        "Name": "Stock",
        "Uri": "http://stock:8080/healthz"
      },
      {
        "Name": "order",
        "Uri": "http://order:8080/healthz"
      }
    ],
    "EvaluationTimeinSeconds": 6,
    "MinimumSecondsBetweenFailureNotifications": 60
  }
}
```



The endpoint endpoints.MapHealthChecksUI() enable the default endpoint healthchecks-ui that you can reach from the Yarp portal.

```csharp

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapHealthChecks("/health/ready", new HealthCheckOptions()
                {
                    Predicate = (check) => check.Tags.Contains("ready"),
                });

                endpoints.MapHealthChecks("/health/live", new HealthCheckOptions()
                {
                    Predicate = (check) => check.Tags.Contains("live")
                });
                endpoints.MapHealthChecksUI();
                endpoints.MapHealthChecks("healthz", new HealthCheckOptions()
                {
                    Predicate = _ => true,
                    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
                });

            });

```

So havind this in mind you are able to fully configured an application with each services self monitored and a global overview of the state of them.