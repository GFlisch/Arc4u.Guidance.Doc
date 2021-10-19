October 2021.

# Guidance 2021.2.4

This version adds the following features to the 2021.2.0:
</br>

- Do not allow to create a UI if only a Yarp service exists.
- Use relative path for Blazor.
- Use of Arc4u 5.10.3
- Use of FluentValidation in the Business layer.
- Use of Async for console ogging in Serilog.
- Use of IAsyncEnumerable<T> in DL and BL.
- Add Unit Test project

## Do not allow to create a UI if only a Yarp service exists.

A front end app (Uwp, Wpf, Blazor, Xamarin Forms) will checj that a service exists in the
list of services the application has => the Yarp is excluded from the list. This is just a
Reverse Proxy service and it is considered as a technical component.

## AspNet Core 5.0?

This version of the Guidance is using the .net5.0 Arc4u dlls.
Those dlls are versioned like this:
- netStandardVersion: '5.0.5'
- iOSVersion: '11.1.5000.0'
- AndroidVersion: '9.0.5000.0'
- UwpVersion: '10.18362.5000.0'
- PrismVersion: '7.2.5000.0'

The migration from .netCore 3.1 to .net5.0 is not complex.
There is more subtleties related to the introduction of http/2 and gRPC than the use of ASPNet Core 5.0.

The first thing to do is to change the target framework from .netcore3.1 to .net5.0.
As in the feature we will have one .net (.net 6.0) the use of .net standard 2.1 from the different projects are replaced by the .net 5.0 one.
Only the Sdks dlls are differents and will be explained later.

This is an example from a Solution6 generated code.

```csharp

The project structure is now like this:

├───Solution6.Yarp.Business
│   └───net50
├───Solution6.Yarp.Domain
│   └───net50
├───Solution6.Yarp.Facade
│   └───net50
├───Solution6.Yarp.IBusiness
│   └───net50
├───Solution6.Yarp.IISHost
│   └───net50
├───Sdks
│   └───Solution6.Yarp.Facade.Sdk
│       ├───net50
│       └───netstandard2.0
└───Tests
    └───Solution6.Yarp.IntegrationTest
        └───net50

```

As the sdks are consumed by clients using different technologies (.Net, Uwp, Xamarin.Forms) it is important to keep .netStandard 2.0.

The fact that we use .net50 for the proxy is to force the usage of http2 when we do a call from a micro-service to another one.

The the kestrel configuration  regarding the usage of http/2 is detailed [here](./../Framework/Net50/gRPC.md).

After, you have to update the program.cs and startup.cs code to integrate the gRPC capabilities.

## Kestrel.

From this version of the guidance we don't use IIS Express anymore but Kestrel.

The good question can be why?

As we have now gRPC, IIS Express and IIS are not valid anymore. gRPC on HTTP2 is not supported and kestrel (which is used when we deploy on linux and k8s pods) is the way to do.

If you deploy a version of a service with no gRPC => IIS will work perfectly. It means that if you do the upgrade of an existing .Net Core 3.1 project (no gRPC was used in previous version of the Guidance),
your project will continue to be able to deploy on IIS.

You can find more info on [gRPC](./../Framework/Net50/gRPC.md) and [Kestrel](./../Framework/Net50/Kestrel.md) here.


### Http/2.

Http/2 is now the standard protocol to use for our communication between clients and services but also from services to services.

We have 2 behaviors to define:
- Rest Api.
- gRpc.

And those 2 behaviors are dependent of the context: UI or backend.

For details about the gRPC integration in Arc4u, please read this section : [gRPC](./../Framework/Net50/gRPC.md).

#### Rest Api.

At the level of the micro-service, the Guidance will generate the definition using the IServiceCollection extension method.

Adding a Rest proxy will:
- add the HttpClient defintion in the Startup.cs class.
- Define a basic Polly retry policy.
- Create the proxy (based on NSwag) using the HttpClient definition.
- Create HttpHandler that you can use in the HttpClient definition.

The advantage of the HtppClient definition is the integration of the Polly and the capability on each request to define a retry mechanism.


Startup.cs
```csharp
        services.AddHttpClient("Core", (httpClient) => 
        {
            httpClient.DefaultVersionPolicy = HttpVersionPolicy.RequestVersionExact;
            httpClient.DefaultRequestVersion = HttpVersion.Version20;
        }).AddHttpMessageHandler<OAuthHttpHandler>()
          .AddHttpMessageHandler<OpenIDHttpHandler>()
          .AddTransientHttpErrorPolicy(p => p.RetryAsync(3));

```

Proxy code using NSwag code generated.

```csharp
         var client = httpClientFactory.CreateClient("Core");

```

Exemple of an HttpHandler to inject the bearer token.

```csharp
    [Export]
    public class OAuthHttpHandler : JwtHttpHandler
    {
        public OAuthHttpHandler(IHttpContextAccessor accessor) : base(accessor?.HttpContext?.RequestServices?.GetService<IContainerResolve>(), "OAuth")
        {

        }
    }

```

The usage of the HttpContextAccessor is here to have access to the Principal (Scoped information injected).

#### gRPC.

gRPC is natively communicating via http/2. The .Net implementation force the request to be protected via TLS 1.2 (using https) but this is possible to use http.
In the front end applications the communication are always protected and use TLS 1.2. For the non Kubernetes backends, we will also deploy and communicate
between the services via TLS 1.2. But in K8s, we will end the TLS communication from Yarp and communicate in clear inside the nmespace. Services are not exposed (ClusterIP).


## Yarp.ReverseProxy.

Yarp package has been renamed to : Yarp.ReverseProxy -Version 1.0.0-preview.11.21223.5

Two breaking changes have been introduced and of them need the update of the Guidance.

The Routes defined in the reverseProxy.json file is changed from a json array to a dictionnary.

```json
{
  "ReverseProxy": {
    "Routes": {
    },
    "Clusters": {
    }
  }
}
```

It means that the json file manipulation in the Guidance has been adapted to handle this.

The second one is concerning the x-forward funcitonality. Pathbase is changed to Prefix.

It means that if you have to forward a typical configuration will be:


```json
{
  "ReverseProxy": {
    "Routes": {
      "elog": {
        "ClusterId": "elog",
        "AuthorizationPolicy": "Default",
        "Match": {
          "Path": "/elog/{*remainder}"
        },
        "Transforms": [
          {
            "PathPattern": "/{*remainder}"
          },
          {
            "X-Forwarded": "proto,host,for,prefix",
            "Append": "true",
            "Prefix": "X-Forwarded-"
          }
        ]
      }
    },
    "Clusters": {
    }
  }
}
```











