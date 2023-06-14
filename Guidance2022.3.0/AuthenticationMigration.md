# Migrating to Arc4u 6.0.14.3
Arc4u version 6.0.14.3 implements breaking changes and a new authentication model for the back-end, which avoids relying on Microsoft-specific libraries.

New versions of the Guidance will automatically generate the correct code. Older back-ends need to be migrated manually. This document explains how (and in some cases why).

Part of these instructions also include the necessary changes to support the seamless decryption of encrypted configuration properties, which was introduced in Arc4u 6.0.12.1.

These instructions are organized as a series of steps. Step 0 is not really a migration step, but contains an overview of the bigger picture.
The remainder roughly represent the types of tasks you need to perform for the migration.


## Step 0. The big(ger) picture
Arc4u version 6.0.14.3 had many goals, but the following are likely to impact any application you will be migrating:
1. Dissociate itself from any Microsoft-specific libraries like Adal and MSAL and rely on generic libraries for authentication scenarios in back-ends
2. Revise the appsettings to provide a clearer and more compact set of configuration settings for authentication. Avoid unnecessary repetition and provide meaningful defaults that can be overridden if necessary.
3. Define more ".NET Core-aligned" ways to:
    - setting up the authentication pipeline
    - define operation-based authorizations
    - handle exceptions in controllers

### Authorization
The most important (and interesting) changes are those pertaining to authorization.

In previous versions this was done with `ServiceAspectAttribute`. This was not ideal since the attribute did 3 unrelated things:
- check if the user is authorized for the specified operation(s)
- set the thread culture information to the culture of the principal
- handle any outgoing exceptions from the controller action.

#### Introducing policy-based authorization
In this version, authorization uses the standard `AuthorizeAttribute`. 
Setting the thread culture and handling outgoing exceptions are now being performed using controller filters, which are defined at application startup:

~~~csharp
    services.AddControllers(opt =>
    {
        opt.Filters.Add<ManageExceptionsFilter>();
        opt.Filters.Add<SetCultureActionFilter>();
    });
~~~

The argument to `AuthorizeAttribute` is a single `string`, representing a [__policy__](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/policies?view=aspnetcore-7.0).  
A policy is a set of requirements. At authorization time, all requirements are evaluated (using their `AuthorizationHandler`) and a user is authorizaed if all requirements are fulfilled.
Since we already have the concept of "Operations", Arc4u has the appropriate handlers to take care of this. You normally don't need to write your own.

At application startup, a new extension method is called:

~~~csharp
    services.AddScopedOperationsPolicy(Operations.Scopes, Operations.Values, options =>
    {
        // Add you custom policy here.
        //options.AddPolicy("Custom", policy =>
        //    policy.Requirements.Add(new ScopedOperationsRequirement((int)Access.AccessApplication, (int)Access.CanSeeSwaggerFacadeApi)));
    });
~~~

This extension methods generates a number of default policies:
- one policy per operation. The name of the policy is the name of the operation. E.g. `"AccessApplication"`
- one policy for each combination of scope and operation. The name of the policy is the name of the scope followed by ':' and the name of the operation. 
In the default case, there are no scopes. If there was one, i.e. `"Toto"`, you'll get a name like `"Toto:AccessApplication"`.

This means that instead of 
~~~csharp
[ServiceAspect(Access.AccessApplication)]
~~~

you now write:
~~~csharp
[Authorize(nameof(Access.AccessApplication))]
~~~

As shown in the code above, you can define your own policies. 
There is a commented-out policy called `"Custom"` which checks if a user has both `Access.AccessApplication` and `Access.CanSeeSwaggerFacadeApi` available.

#### Unlocking new scenarios with Policy based authentication
Moving to policy-based authorization unlocks a number of advantages and unlocks a few new scenarios

##### Setting a policy for an entire controller
You can assign a policy to an entire controller if all the actions of that controller have the same policy. 
No need to repeat it on every method if all methods require the same authorization:

~~~csharp
[Authorize(nameof(Access.AccessApplication))]
[ApiController]
[Route("/Scheduler/facade/[controller]")]
[ApiExplorerSettings(GroupName = "facade")]
public class EnvironmentController : ControllerBase
~~~

##### Setting a policy for an endpoint at the YARP level
Previously, `reverseproxy.json` entries could only be written with the [`Default`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.authorization.authorizationoptions.defaultpolicy?view=aspnetcore-7.0#Microsoft_AspNetCore_Authorization_AuthorizationOptions_DefaultPolicy) authorization policy.
This policy just required users to be authenticated:

~~~json
{
  "ReverseProxy": {
    "Routes": {
      "referential/environment": {
        "ClusterId": "referential",
        "AuthorizationPolicy": "Default",
        "Match": {
          "Path": "/referential/facade/environment/{**remainder}"
        },
        "Transforms": [
          {
            "PathPattern": "/referential/facade/environment/{**remainder}"
          }
        ]
      }
    }
  }    
}
~~~

With the new Policy-based mechanism, you can now put a known authorization policy in there, and have your endpoint check on that policy:

~~~json
{
  "ReverseProxy": {
    "Routes": {
      "referential/environment": {
        "ClusterId": "referential",
        "AuthorizationPolicy": "AccessApplication",
        "Match": {
          "Path": "/referential/facade/environment/{**remainder}"
        },
        "Transforms": [
          {
            "PathPattern": "/referential/facade/environment/{**remainder}"
          }
        ]
      }
    }
  }    
}
~~~

The `"AuthorizationPolicy"` value can also have scoped operations, if you have them defined.

##### Hangfire dashboard policy-based protection
Adding a Hangfire job adds an `Access.CanSeeJobs` operation. To protect the Hangfire dashboard using previous versions of the Arc4u Framework, 
a custom `IDashboardAuthorizationFilter` implementation needed to be added:

~~~csharp
public class HangfireAuthorization : IDashboardAuthorizationFilter
{
    public bool Authorize(DashboardContext context)
    {
        var httpContext = context.GetHttpContext();

        // Allow all authenticated users to see the Dashboard (potentially dangerous).
        return httpContext.User is AppPrincipal ? ((AppPrincipal)httpContext.User).IsAuthorized(Access.CanSeeJobs) : false;
    }
}
~~~

You would then needed to specify this implementation when setting up Hangfire:

~~~csharp
  app.MapHangfireDashboard("/scheduler/jobs", new DashboardOptions
    {
        Authorization = new[] { new HangfireAuthorization() },
        AppPath = "/healthchecks-ui#/healthchecks"
    });
~~~

With Policy-based authorization, the custom `IDashboardAuthorizationFilter` disappears and the setup call becomes simply:

~~~csharp
    app.MapHangfireDashboardWithAuthorizationPolicy(nameof(Access.CanSeeJobs), "/scheduler/jobs", new DashboardOptions
    {
        AppPath = "/healthchecks-ui#/healthchecks"
    });
~~~

##### SignalR policy based authentication
It becomes possible to associate authorization operations with SignalR hub, simply by specifying the policy, like this:
~~~csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.SignalR;
using PRT.Referential.Domain;

namespace PRT.Referential.Business.Logic;

[Authorize(nameof(Access.ServiceAccessPolicy))]
public class FileProcessingHub : Hub<RunningFileUpload[]>
{
}
~~~

Previously, you could only write `[Authorize]`, which made the hub accessible to any authenticated user, regardless of their assigned operations.
Essentially, all scenarios described [here](https://learn.microsoft.com/en-us/aspnet/core/signalr/authn-and-authz?view=aspnetcore-7.0#authorize-users-to-access-hubs-and-hub-methods) become possible.

##### Blazor WASM
The policy system is available in Blazor WASM applications. You initialize it with exactly the same code and use exactly the same concepts:

~~~csharp
services.AddScopedOperationsPolicy(Operations.Scopes, Operations.Values, options =>
{
    // Add you custom policy here.
    //options.AddPolicy("Custom", policy =>
    //    policy.Requirements.Add(new ScopedOperationsRequirement((int)Access.AccessApplication, (int)Access.CanSeeSwaggerFacadeApi)));
});
~~~

This was already possible before, but now the code is shared.


#### Specifying operations
Each operation consists of a numeric identifier and a name.
In previous versions, the Guidance generated code that mapped the numbers to the names using a dictionary, and methods allowing to get the name of an operation given its number.

It was realized that the variable names used for the operation numbers are in fact the name of these operations.
There is a data type that (somewhat) elegantly represents both aspects: the `enum`.

Current versions of the Guidance now generate the following, more compact code:

~~~csharp
using Arc4u.Security.Principal;

/// <summary>
/// Defined the operations available to secure the application.
/// </summary>
public enum Access : int
{
    AccessApplication = 1,
    CanSeeSwaggerFacadeApi = 2,
    CanSeeJobs = 3,
}

public static class Operations
{
    public static readonly Operation[] Values = Enum.GetValues<Access>().Select(o => new Operation { Name = o.ToString(), ID = (int)o }).ToArray();

    public static readonly string[] Scopes = Array.Empty<string>();
}
~~~

Note that the code now explicitly contains the scopes. By default, this is empty, so no scope is defined (which is the default).

The way to get from `Access` to `int` is just casting: `(int)Access.AccessApplication`.

The way to get from `Access` to `string` is just `nameof` if you have a constant: `nameof(Access.AccessApplication)`.

This explains why you need to write:

~~~csharp
[Authorize(nameof(Access.AccessApplication))]
~~~



## Step 1. Update package references
### Step 1.1 Arc4u package references
In all your projects, update all Arc4u references to the latest version (6.0.14.3 or later).

For example:
~~~xml
		<PackageReference Include="Arc4u.Standard.Data" Version="6.0.14.3" />
		<PackageReference Include="Arc4u.Standard.Diagnostics" Version="6.0.14.3" />
		<PackageReference Include="Arc4u.Standard.FluentValidation" Version="6.0.14.3" />
~~~

In your `Host` projects, the following Arc4u packages (if referenced):

~~~xml
    <PackageReference Include="Arc4u.Standard.OAuth2.AspNetCore.Msal" Version="..." />
~~~
or
~~~xml
    <PackageReference Include="Arc4u.Standard.OAuth2.AspNetCore.Adal" Version=".." />
~~~

... can be deleted and replaced by:

~~~xml
    <PackageReference Include="Arc4u.Standard.OAuth2.AspNetCore.Authentication" Version="6.0.14.3" />
~~~

You need to add a new package containing policy-based security, which is used both in back-ends and front ends:

~~~xml
    <PackageReference Include="Arc4u.Standard.Authorization" Version="6.0.14.3-preview38" />
~~~

Optionally, you can also add:

~~~xml
    <PackageReference Include="Arc4u.Standard.Configuration.Decryptor" Version="6.0.14.3" />
~~~

The `Arc4u.Standard.Configuration.Decryptor` reference doesn't have anything to do with authentication _per se_, but is needed 
when decrypting secrets in the configuration files. It's a good idea to include it even if you don't (yet) use it. See the section on [adding support for seamless decryption](#add-support-for-seamless-decryption)

Finally, there is one package whose NuGet PackageId changed. Instead of:

~~~xml
    <PackageReference Include="Arc4u.Standard.Dependency.ComponentModel.Container" Version="6.0.14.3-preview25" />
~~~
... you need to specify:

~~~xml
    <PackageReference Include="Arc4u.Standard.Dependency.ComponentModel" Version="6.0.14.3-preview27" />
~~~

### Step 1.2 Microsoft package references
It is also a good idea to update the runtime-dependent .NET Core package references to align to the same version as Arc4u.
Otherwise, you may get version mismatch warnings.

For example, Arc4u 6.0.14.x is compiled with runtime 6.0.14. 
This means that references to .NET Core packages should also be updated to the same runtime version. 

For example:

~~~xml
    <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="6.0.14" />
    <PackageReference Include="Microsoft.Extensions.Http.Polly" Version="6.0.14" />
~~~

Be careful: not all Microsoft packages follow the runtime version numbering. For example, `Microsoft.Extensions.Logging` does not!

### Step 1.3 OpenTelemetry package references
You need to update OpenTelemetry packages as well:

~~~xml
    <PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.4.0-rc.4" />
    <PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.4.0-rc.4" />
    <PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.0.0-rc9.13" />
    <PackageReference Include="OpenTelemetry.Instrumentation.Http" Version="1.0.0-rc9.13" />
~~~

This update has some implications in the startup code, which will be mentioned in [Step 3.4](#step-34-update-programcs).

## Step 2. Update the shared `Access.cs`
The way to specify access operations has been specified. In previous versions, the guidance generated a type `Access` that mapped string identifiers to integers, like this:

~~~csharp
/// <summary>
/// Defined the operations available to secure the application.
/// </summary>
public static class Access
{
    public static readonly IReadOnlyDictionary<int, String> Definitions = new Dictionary<int, string>
    {
        {AccessApplication, "AccessApplication"},
        {CanManageServerLoggingLevel, "CanManageServerLoggingLevel"},
        {CanSeeSwaggerFacadeApi, "CanSeeSwaggerFacadeApi"},
    };

    public const int AccessApplication = 1;
    public const int CanManageServerLoggingLevel = 2;
    public const int CanSeeSwaggerFacadeApi = 3;
 

    /// <summary>
    /// Get the name of the operation.
    /// </summary>
    /// <param name="operation">The operation id.</param>
    /// <returns></returns>
    public static string Name(int operation)
    {
        return Definitions[operation];
    }

    /// <summary>
    /// Get an array of operation names for the operations id.
    /// </summary>
    /// <param name="operations">The operations id.</param>
    /// <returns></returns>
    public static string[] Names(params int[] operations)
    {
        var result = new List<string>();

        foreach (int operation in operations)
        {
            if (Definitions.ContainsKey(operation)) result.Add(Name(operation));
        }

        return result.ToArray();
    }
}
~~~

In the newer version, operations are now an `enum`. The default is "no scopes". The code becomes:

~~~csharp
using Arc4u.Security.Principal;

/// <summary>
/// Defined the operations available to secure the application.
/// </summary>
public enum Access : int
{
    AccessApplication = 1,
    CanSeeSwaggerFacadeApi = 2,
    CanSeeJobs = 3,
}

public static class Operations
{
    public static readonly Operation[] Values = Enum.GetValues<Access>().Select(o => new Operation { Name = o.ToString(), ID = (int)o }).ToArray();

    public static readonly string[] Scopes = Array.Empty<string>();
}

~~~

The shared project requires a reference to `Arc4u.Standard` in order for the `Operation` to be known:

~~~xmnl
<ItemGroup>
    <PackageReference Include="Arc4u.Standard" Version="6.0.14.3" />
</ItemGroup>
~~~

The obvious implications are:
- you need to explicitly cast the `Access` enum to `int` to get the operation number. I.e. `(int)Access.AccessApplication`
- you need to explicitly convert to a string or use `nameof` to get the operation name. I.e. `nameof(Access.AccessApplication)` or `access.ToString()` if `access` is of type `Access`.

## Step 2. Update the `.json` configuration files
### Step 2.1 The global `appsettings.json` for hosts
A number of properties and configuration sections disappear because they are either obsolete, or they are moved to the environment specific configuration files.

- The `"loggingLevel"` property in section `"Application.Configuration:Environment"` can be deleted since it's not used anymore.
- The `"Caching"` and `"Memory.Settings"` sections can be deleted. Caching is now specified per environment. If for some reason your caching settings are exactly the same for all environments, you can keep it and modify it as described below.
- The `"Application.TokenCacheConfiguration"` section can be deleted. It is now part of a new section per environment.
- In the section `"Application.Dependency:RegisterTypes"`:
    - delete `"Arc4u.Configuration.ApplicationConfigReader, Arc4u"`
    - delete `"Arc4u.OAuth2.Configuration.OAuthConfig, Arc4u.Standard.OAuth2"`
    - delete `"Arc4u.OAuth2.Security.Principal.ServerPrincipalCache, Arc4u.Standard.OAuth2"`
    - delete `"Arc4u.OAuth2.TokenProvider.MsalTokenProvider, Arc4u.Standard.OAuth2.AspNetCore.Msal"`
    - delete `"Arc4u.OAuth2.TokenProvider.AdfsTokenProvider, Arc4u.Standard.OAuth2.AspNetCore.Adal"`
    - delete `"Arc4u.OAuth2.Security.Principal.ClaimsBearerTokenExtractor, Arc4u.Standard.OAuth2.AspNetCore.Adal"`
    - delete `"Arc4u.OAuth2.Configuration.TokenUserCacheConfiguration, Arc4u.Standard.OAuth2"`
    - add `"Arc4u.Security.Cryptography.X509CertificateLoader, Arc4u.Standard"`
    - add `"Arc4u.OAuth2.AppPrincipalTransform, Arc4u.Standard.OAuth2.AspNetCore"`
    - add `"Arc4u.Authorization.ApplicationAuthorizationPolicy, Arc4u.Standard.Authorization"`
    - add (or make sure it's already there) `"Arc4u.OAuth2.Security.Principal.ClaimsBearerTokenExtractor, Arc4u.Standard.OAuth2"` (note the change of assembly!)
    - add (or make sure it's already there) `""Arc4u.OAuth2.Security.UserObjectIdentifier, Arc4u.Standard.OAuth2.AspNetCore""` (note the change of assembly!)
    - if present, change `"Arc4u.OAuth2.TokenProvider.CredentialTokenCacheTokenProvider, Arc4u.Standard.OAuth2"` into `"Arc4u.OAuth2.TokenProvider.CredentialTokenCacheTokenProvider, Arc4u.Standard.OAuth2.AspNetCore"` (note the change of assembly!)
    - if present, change `"Arc4u.OAuth2.TokenProvider.ClientSecretTokenProvider, Arc4u.Standard.OAuth2"` into `"Arc4u.OAuth2.TokenProvider.RemoteClientSecretTokenProvider, Arc4u.Standard.OAuth2.AspNetCore"` (note the change of type name and assembly!)
    - if present, change `"Arc4u.OAuth2.Security.Principal.KeyGeneratorFromIdentity, Arc4u.Standard.OAuth2"` into `"Arc4u.OAuth2.Security.Principal.KeyGeneratorFromIdentity, Arc4u.Standard.OAuth2.AspNetCore"` (note the change of assembly!)
- Only for the `Yarp` appsettings, in addition to the changes in `"Application.Dependency:RegisterTypes"` described in the previous point:
    - add `"Arc4u.OAuth2.TokenProviders.OidcTokenProvider, Arc4u.Standard.OAuth2.AspNetCore.Authentication"`
    - add `"Arc4u.OAuth2.TokenProviders.BootstrapContextTokenProvider, Arc4u.Standard.OAuth2.AspNetCore.Authentication"`
    - add `"Arc4u.OAuth2.TokenProviders.RefreshTokenProvider, Arc4u.Standard.OAuth2.AspNetCore.Authentication"`
    - add `"Arc4u.OAuth2.TokenProviders.AzureADOboTokenProvider, Arc4u.Standard.OAuth2.AspNetCore.Authentication"`

### Step 2.2 The environment-specific `appsettings.$(env).json`
#### Things to remove
- The property `"ida_MetadataAddress"` in section `"AppSettings"` can be deleted.
- The section `"Memory.Settings"`, if there is one, can be deleted as well.

#### Caching
For all hosts, you need to add a section describing the caching parameters. It should look like this:
~~~json
 "Caching": {
    "Caches": [
      {
        "Name": "Volatile",
        "Kind": "Memory",
        "Settings": {
          "SizeLimit": "10"
        },
        "IsAutoStart": "True"
      }
    ],
    "Default": "Volatile"
  },
~~~

Note that the `"SizeLimit"` property value is usually a small value for developement (like `"10"` here), and a larger value in other environments.
The value will be the same as the `"SizeLimitInMegaBytes"` value in the old `"MemorySize.Settings"` section you just deleted.

#### Authentication configuration for `Yarp` hosts
For the `Yarp` host, the `"OAuth2.Settings"` and `"OpenId.Settings"` sections are replaced by a single section `"Authentication"` that also regroups every other configuration
setting that relates to authentication:

~~~json
 "Authentication": {
    "ResourcesRights": {
      "ResourcesPolicies": {
        "facade": {
          "AuthorizationPolicy": "CanSeeSwaggerFacadeApi",
          "Path": "/swagger/facade/swagger.json"
        },
        "interface": {
          "AuthorizationPolicy": "CanSeeSwaggerFacadeApi",
          "Path": "/swagger/interface/swagger.json"
        }
      }
    },
    "ClaimsMiddleware": {
      "ForceOpenId": {
        "ForceAuthenticationForPaths": [
          "/service/jobs*",
          "/swagger*"
        ]
      },
      "ClaimsFiller": {
        "LoadClaimsFromClaimsFillerProvider": true,
        "AuthenticationSettingsKeys": [
          "OpenId",
          "OAuth2"
        ]
      }
    },
    "ClaimsIdentifier": [
      "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn",
      "upn"
    ],
    "TokenCache": {
      "CacheName": "Volatile",
      "MaxTime": "00:00:20"
    },
    "DefaultAuthority": {
      "Url": "same as the old OAuth2.Settings:Authority or the old OpenId.Settings:Authority",
      "TokenEndpoint": "/oauth2/token"
    },
    "AuthenticationCacheTicketStore": {
      "CacheName": "Volatile"
    },
    "DataProtection": {
      "EncryptionCertificate": {
        "Store": {
          "Name": "environment-dependent encryptor certificate"
        }
      },
      "CacheStore": {
        "CacheKey": "DataProtection-",
        "CacheName": "Volatile"
      }
    },
    "MetadataAddress": "same as the old AppSettings:ida_MetadataAddress",
    "CookieName": "PROJECT.ApiGtw.Cookies",
    "OAuth2.Settings": {
       "Audiences": "same as the old OAuth2.Settings.:ServiceApplicationId",
       "Scopes": "same as the old OAuth2.Settings.:ServiceApplicationId with /user_impersonation appended"
     },
    "OpenId.Settings": {
      "ClientId": "same as the old OpenId.Settings:ClientId",
      "ClientSecret": "same as the old OpenId.Settings.ApplicationKey"
      "Audiences": "same as the old OpenId.Settings.:ServiceApplicationId",
      "Scopes": "same as the old OpenId.Settings.:ServiceApplicationId with /user_impersonation appended"
    },
    "DomainsMapping": {
      "belgrid.net": "belgrid",
      "corp.transmission-it.de": "tmit"
    }
  },
~~~

You need to adapt this section for each environment. Most of it should be self-explanatory, but here are a few pointers:
- The `ResourcesRights` section associates access rights (called "Policies" in the new model) to endpoints that are not handled by `ControllerBase` implementations in your code. 
Currently, these endpoints are associated with the Swagger pages for the facade and the interface. In the example above, these swagger endpoint require the `CanSeeSwaggerFacadeApi` policy.
Only specify the endpoints that are actually present. Yarp probably has no interfaces.
- The `ClaimsMiddleware` section configures the claims middleware:
  - `ForceOpenId` specifies endpoints that are not handled by `ControllerBase` implementations for which unauthenticated requests should be challenged by the OpenId scheme. 
  This will cause a redirect to the STS in order to start authentication, which is useful when the client is a browser. Default endpoint values include `/swagger*` for the Swagger UI, and `/service/jobs*` for the Hangfire UI.
  - `ClaimsFiller` indicates the schemes for which extra claims need to be added to the bearer token by the claims filler provider. This section can be copied verbatim.
- The `ClaimsIdentifier` section can be copied verbatim. It represents the list of claim names identifying the UPN, which can vary.
- The `TokenCache` section can be copied verbatim. Just make sure that the name of the cache correcponds to a cache defined in the `Caching:Caches` section.
- The name of the certificate (`"Authentication:DataProtection:EncryptionCertificate:CertificateStore:Name"`) depends on the specific environment
- The `"CookieName"` used to be hard-coded in the Yarp host startup code: it is now a per-environment configuration setting. Replace `PROJECT` with the name of your project.
- the `"DefaultAuthority"` holds the value of the authority (the STS), since it's usually the same for all authentication schemes. 
Previously, you had an `"Authority"` property for all applicable schemes (OAuth2, OpenId), but they all contained the same value. 
Although this property is still present, it's now optional: specifying a single default authority should be sufficient.
- The `"Audiences"` property value correspond to the content of the old `"ServiceApplicationId"`.
- The `"Scopes`" property correspond to the `"Audiences"` property with `/user_impersonation` appended.
E.g. if `"Audiences"` is `https://project.environment.host`, then `"Scopes"` is `https://project.environment.host/user_impersonation`
- The `"DomainsMapping"` is new: it maps the UPN suffix representing the domain name to the AD domain name. Previously, this was hard-coded in the framework.

#### Basic settings
If you have a `"Basic.Settings"` section, the section is now a `"Basic"` section moved inside the `"Authority"` section (at the same level as the `"OpenId.Settings"` section) written as follows:
~~~json
    "Basic": {
      "Settings": {
      "ClientId": "same as the old Basic.Settings:ClientId",
      "Scope": "same as the old OpenId.Settings.:ServiceApplicationId with /user_impersonation appended"
      },
      "Certificates": {
        "CertificateKey": {
          "Store": {
            "Name": "encryptorSha2[dev|test|acc|prod].belgrid.net"
          }
        }
      }
    }
~~~

#### Authentication settings for non-`Yarp` hosts (standard back-ends)
There is no `"OpenId.Settings"` and the existing `"OAuth2.Settings"` is moved to an `"Authentication"` section very similar to the Yarp case:
~~~json
  "Authentication": {
    "DefaultAuthority": {
      "Url": "same as the old OAuth2.Settings:Authority",
      "TokenEndpoint": "/oauth2/token"
    },
    "ResourcesRights": {
      "ResourcesPolicies": {
        "facade": {
          "AuthorizationPolicy": "CanSeeSwaggerFacadeApi",
          "Path": "/servicename/swagger/facade/swagger.json"
        },
        "interface": {
          "AuthorizationPolicy": "CanSeeSwaggerFacadeApi",
          "Path": "/servicename/swagger/interface/swagger.json"
        }
      }
    },
    "ClaimsIdentifier": [
      "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn",
      "upn"
    ],
    "MetadataAddress": "same as the old AppSettings:ida_MetadataAddress",
    "TokenCache": {
      "CacheName": "Volatile",
      "MaxTime": "00:00:20"
    },
    "OAuth2.Settings": {
      "Audiences": "same as the old OAuth2.Settings.:ServiceApplicationId",
       "Scopes": "same as the old OAuth2.Settings.:ServiceApplicationId with /user_impersonation appended"
    },
    "ClaimsMiddleware": {
      "ClaimsFiller": {
        "LoadClaimsFromClaimsFillerProvider": true,
        "AuthenticationSettingsKeys": []
      }
    },
    "DomainsMapping": {
      "belgrid.net": "belgrid",
      "corp.transmission-it.de": "tmit"
    }
  },
~~~


#### External Service configuration
You may have configuration sections pertaining to external (authenticated) service calls. like this:

~~~json
  "SomeService.Settings": {
    "ProviderId": "ClientSecret",
    "AuthenticationType": "Inject",
    "ClientId": "...",
    "ServiceApplicationId": "...",
    "Authority": "...",
    "BaseUrl": "...",
    "ClientSecret": "...",
    "CertificateName": "...",
    "FindType": "...",
    "StoreName": "...",
    "StoreLocation": "..."
  }
~~~

The `ClientSecretTokenProvider` doesn't exist anymore. You need to **change** the `ProviderId` and **add** a `HeaderKey` and `Audience` property as follows:
~~~json
    "ProviderId": "RemoteSecret",
    "Audience": "same content as SomeService.Settings:ServiceApplicationId",
    "HeaderKey": "CertificateKey",
~~~

Do **not** remove `"ServiceApplicationId"` at this time.


## Step 3 Code changes
### Step 3.1 Removing the `...SettingsReader` classes
Handling configurations has been simplified. Specific "settings reader" types deriving from Arc4u's `KeyValueSettings` can be removed.
This includes types like:
- `OAuth2TokenSettingsReader`
- `VolatileMemorySettingsReader`
- any other `...SettingsReader` deriving from `KeyValueSettings`.

### Step 3.2 Rewrite the handler classes
Handlers (i.e. types inheriting from `JwtHttpHandler`) have slightly different constructors to reflect the new way of getting options.
The standard [Options pattern](https://learn.microsoft.com/en-us/dotnet/core/extensions/options) of .NET core is now used, and `IOptionsMonitor<SimpleKeyValueSettings>` is now injected to get the configuration settings.

Here are two updated examples corresponding to the two most common use cases in the back-ends.

Example 1:

~~~csharp
using Arc4u.Configuration;
using Arc4u.Dependency.Attribute;
using Arc4u.OAuth2.Token;
using Microsoft.Extensions.Options;

[Export]
public class OAuth2HttpHandler : JwtHttpHandler
{
    public OAuth2HttpHandler(IHttpContextAccessor accessor, IOptionsMonitor<SimpleKeyValueSettings> options, ILogger<OAuth2HttpHandler> logger)
        : base(accessor, logger, options, "OAuth2")
    {

    }
}
~~~

Example 2:

~~~csharp
using Arc4u.Configuration;
using Arc4u.Dependency;
using Arc4u.Dependency.Attribute;
using Arc4u.OAuth2.Token;
using Microsoft.Extensions.Options;

[Export]
public class SomeServiceHttpHandler : JwtHttpHandler
{
    public SomeServiceHttpHandler(IContainerResolve containerResolve, IOptionsMonitor<SimpleKeyValueSettings> options, ILogger<SomeServiceHttpHandler> logger)
        : base(containerResolve, logger, options, "SomeService")
    {

    }
}
~~~

The second example is typical of a custom (authenticated) service you back-end is calling. The name `"SomeService"` refers to a configuratin mapping that needs
to be set up, which will be described in Step 3.4.

### Step 3.3 Update the `EnvironmentBL`
Because the `Config` class has disappeared, the `EnvironmentBL` needs to be rewritten.
The configuration settings are now called `ApplicationConfig` and they are now bona fide configuration options:

~~~csharp
public class EnvironmentInfoBL : IEnvironmentInfoBL
{
    public EnvironmentInfoBL(IOptions<ApplicationConfig> config, IAppSettings settings, IConfiguration configuration, ILogger<EnvironmentInfoBL> logger)
    {
        _config = config.Value;
        ...
    }

    private readonly ApplicationConfig _config;
    ...
}
~~~


### Step 3.4 Update `Program.cs`
#### Renames
In order to align with the convention that services are **Add**ed to a service collection and **Use**d in an application, rename
~~~csharp
    services.UseSystemMonitoring();
~~~
to
~~~csharp
    services.AddSystemMonitoring();
~~~

#### Add support for seamless decryption
In `ConfigureAppConfiguration`, you need to add the following statement as the last one in the delegate parameter:

~~~csharp
    config.AddCertificateDecryptorConfiguration();
~~~

This will automatically decrypt all string properties that starts with `Decrypt:`. 

To make this work, you need to specify which certificate to use. This is a separate section to be added to your `appsettings.$(env).json` file, for example:

~~~json
  "EncryptionCertificate": {
    "Store": {
      "Name": "environment-dependent encryptor certificate"
    }
  }
~~~

In `Yarp`, there is already a reference to a certificate in the `Authentication:DataProtection:EncryptionCertificate` section, and it will most likely be the same certificate.
In that case, instead of adding an `"EncryptionCertificate"` to the application settings, you can reference the certificate already present:

~~~csharp
    config.AddCertificateDecryptorConfiguration(options => options.SecretSectionName = "Authentication:DataProtection:EncryptionCertificate");
~~~

#### Policy-based authorization
The following statement:
~~~csharp
    services.AddAuthorizationCheckAttribute();
~~~

... can be deleted and replaced by:

~~~csharp
    services.AddScopedOperationsPolicy(Operations.Scopes, Operations.Values, options =>
    {
        // Add you custom policy here.
        //options.AddPolicy("Custom", policy =>
        //    policy.Requirements.Add(new ScopedOperationsRequirement((int)Access.AccessApplication, (int)Access.CanSeeSwaggerFacadeApi)));
    });
~~~

You may have to add a `using Arc4u.Authorization;` to access the namespace.

The following statements:
~~~csharp
    var keyValuesOptions = container.Resolve<IOptionsMonitor<SimpleKeyValueSettings>>();
    var oauth2Settings = keyValuesOptions.Get("OAuth2");
    var openIdSettings = keyValuesOptions.Get("OpenId");

    app.UseCreatePrincipal(container, new ClaimsPrincipalMiddlewareOption()
    {
        ClaimsFillerOptions = new ClaimsPrincipalMiddlewareOption.ClaimsFillerOption
        {
            LoadClaimsFromClaimsFillerProvider = true,
            Settings = new[] { oauth2Settings, openIdSettings }
        },

        OpenIdOptions = new ClaimsPrincipalMiddlewareOption.OpenIdOption
        {
            ForceAuthenticationForPaths = new String[] { "/swagger*" }
        }
    });
~~~
... can be deleted and replaced by:

~~~csharp
    app.UseForceOfOpenId();
    app.UseResourcesRightValidationFor();
    app.UseAddContextToPrincipal();
    app.UseOpenIdBearerInjector();
~~~

> IMPORTANT: use `app.UseForceOfOpenId()` and `app.UseOpenIdBearerInjector()` only for Yarp, not for other back-ends!


> IMPORTANT: these statements must be located between `app.UseAuthentication();` and `app.UseAuthorization();` for them to work correctly.

### Statements that have been replaced by configuration
The statements:
~~~csharp
    app.AddValidateSwaggerRightFor(new ValidateSwaggerRightMiddlewareOption
    {
        Access = Access.CanSeeSwaggerFacadeApi,
        Path = "/swagger/facade/swagger.json"
    });

    app.AddValidateSwaggerRightFor(new ValidateSwaggerRightMiddlewareOption
    {
        Access = Access.CanSeeSwaggerFacadeApi,
        Path = "/swagger/interface/swagger.json"
    });
~~~

... can be removed. They are replaced by the `Authentication:ResourcesRights` section in the application settings.

### Controller filters
The following statement:
~~~csharp
    services.AddControllers(...);
~~~
...must be extended since exception handling and thread culture information is set through filters:
~~~csharp
	services.AddControllers(opt =>
	{
		opt.Filters.Add<ManageExceptionsFilter>();
		opt.Filters.Add<SetCultureActionFilter>();
	});
~~~

You can also add a filter that allows returning `null` from methods otherwise replying with `204` (No Content):
~~~csharp
		// disable null to 204 => which will throw an exception.
		opt.OutputFormatters.RemoveType<HttpNoContentOutputFormatter>();
~~~

#### Replace configuration statements (Yarp hosts)
The following statements:
~~~csharp
	IKeyValueSettings OAuth2Settings = new OAuth2TokenSettingsReader(Configuration);
	IKeyValueSettings OpenIdSettings = new OpenIdTokenSettingsReader(Configuration);
	IKeyValueSettings AppSettings = new AppSettings(Configuration);
	ICookieManager CookieManager = new ChunkingCookieManager();
~~~
... can be deleted and replaced by:
~~~csharp
    services.ConfigureSettings("AppSettings", Configuration, "AppSettings");
    services.AddBasicAuthenticationSettings(Configuration);
    services.AddApplicationConfig(Configuration);
    services.AddCacheContext(Configuration);
~~~

The following statements:
~~~csharp
	var basicSettings = new BasicTokenSettingsReader(Configuration);
	app.UseBasicAuthentication(container, new BasicAuthenticationContextOption
	{
		Settings = basicSettings
	});
~~~
... can be deleted and replaced by:
~~~csharp
    app.UseBasicAuthentication();
~~~

#### Replace configuration statements (non-Yarp hosts)
The following statement:
~~~csharp
    IKeyValueSettings OAuth2Settings = new OAuth2TokenSettingsReader(Configuration);
~~~
... can be deleted and replaced by:
~~~csharp
    services.ConfigureSettings("AppSettings", Configuration, "AppSettings");
    services.AddCacheContext(Configuration);
    services.AddApplicationContext();
    services.AddAuthorizationCheckAttribute();
~~~


#### Add configuration settings for custom services
If your back-end calls an authenticated external service, Step 2.2 showed you how to adapt the `SomeService.Settings` and Step 3.2 showed how to rewrite the `SomeServiceHttpHandler` handler type.

What is missing from this picture is how to map the section name in the configuration file (`SomeService.Settings`) with the name used in the handler. 
For this, you need to add:

~~~csharp
    services.ConfigureSettings("SomeService", Configuration, "SomeService.Settings");
~~~

#### Adjust OpenTelemetry initialization
The statement:
~~~csharp
    services.AddOpenTelemetryTracing(
~~~
...must be rewritten as:
~~~csharp
    services.AddOpenTelemetry().WithTracing(
~~~


#### Replace authentication configuration (Yarp hosts)
The following statements:
~~~csharp
	services.AddHttpContextAccessor();

	services.AddOpenIdBearerAuthentication(new AuthenticationOptions
	{
		MetadataAddress = AppSettings.Values["ida_MetadataAddress"],
		OAuthSettings = OAuth2Settings,
		OpenIdSettings = OpenIdSettings,
		CookieManager = CookieManager,
		CookieName = CookieName,
		ValidateAuthority = false,
		OnAuthorizationCodeReceived = OpenIdConnectAuthenticationHandlers.AuthorizationCodeReceived
	});
~~~
... can be deleted and replaced by:
~~~csharp
    services.AddOidcAuthentication(Configuration);
~~~

The following statement:
~~~csharp
	app.UseOpenIdCookieValidityCheck(new OpenIdCookieValidityCheckOptions
	{
		CookieManager = CookieManager,
		CookieName = CookieName,
		OpenIdSettings = OpenIdSettings
	});
~~~
... can be deleted.


#### Replace authentication configuration (non-Yarp hosts)
The following statements:
~~~csharp
    services.AddAuthorization();
    services.AddHttpContextAccessor();
    services.AddAuthentication(auth => auth.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(option =>
            {
                option.RequireHttpsMetadata = true;
                option.Authority = OAuth2Settings.Values[TokenKeys.AuthorityKey];
                option.TokenValidationParameters.SaveSigninToken = true;
                option.TokenValidationParameters.AuthenticationType = Arc4u.Standard.OAuth2.Constants.BearerAuthenticationType;
                option.TokenValidationParameters.ValidateIssuer = false;
                option.TokenValidationParameters.ValidateAudience = true;
                option.TokenValidationParameters.ValidAudiences = new[] { OAuth2Settings.Values[TokenKeys.ServiceApplicationIdKey] };
            });

~~~
... can be deleted and replaced by:
~~~csharp
    services.AddJwtAuthentication(Configuration);
~~~


#### Optional changes
##### Container initalization
Previously, the code to get the `container` was:
~~~csharp
    IContainerResolve? container = app.Services.GetService<IContainerResolve>();
    if (null == container)
    {
        Log.Error("No container is registered");
        return;
    }
~~~
This can be simplified to:
~~~csharp
    var container = app.Services.GetRequiredService<IContainerResolve>();
~~~

##### Switch to OpenApiDocument
In code setting up a Swagger documents, like:
~~~csharp
    services.AddSwaggerDocument(option =>
    {
        option.DocumentName = "interface";
        option.ApiGroupNames = new[] { "interface", "deprecated" };
        option.PostProcess = postProcess =>
        {
            postProcess.Info.Title = "Interface description.";
        };
    });
~~~
... you can change `AddSwaggerDocument` to `AddOpenApiDocument` if you want to comply with the latest Guidance code.
However, even though the calls are compatible, they differ in the code they generate:
- in `Swagger`, `[FromBody]` arguments are required by default
- in `OpenApi`, `[FromBody]` arguments are optional by default
This impacts the order of the parameters in the genrated code: optional arguments are moved at the end.
See also https://github.com/RicoSuter/NSwag/issues/4330

##### Kestrel headers
It's good practice not to expose more information than strictly necessary. The statement:
~~~csharp
	// Remove sensitive information that can be exploit by hackers.
	builder.WebHost.ConfigureKestrel(options => options.AddServerHeader = false);
~~~
... can be added right after `var builder = WebApplication.CreateBuilder(args);`.

## Step 3.5 Update all controllers
The way to apply policies to controllers has changed.
Previously the `ServiceAspect` attribute was used:
~~~csharp
    [ServiceAspect(Access.AccessApplication)]
~~~

This has now been replaced by:
~~~csharp
    [Authorize(nameof(Access.AccessApplication))]
~~~

## Step 4: Unit Tests
Because `Config` has disappeared, remove the statements in `ContainerFixture`:
~~~csharp
    var config = new Config(configuration);
~~~
and
~~~csharp
    container.RegisterInstance<Config>(config);
~~~

Add the line ` services.AddApplicationConfig(configuration);` after creating the service collection, like this:

~~~csharp
    var services = new ServiceCollection();
    services.AddApplicationConfig(configuration);
~~~

Because `Config` has disappeared, tests using it need to change:
~~~csharp
    var config = _fixture.Freeze<Config>();
~~~
into:
~~~csharp
   var config = _fixture.Freeze<IOptions<ApplicationConfig>>();
~~~
This is the standard "Options Pattern", the actual value is accessed via the `Value` property.

If you are constructing an `AppPrincipal` as part of your unit tests like this:
~~~csharp
    public virtual AppPrincipal GetPrincipal()
    {
        var authorization = new Authorization
        {
            AllOperations = new(),
            Roles = new List<ScopedRoles> { new ScopedRoles { Scope = "", Roles = new List<string> { "Admin" } } },
            Scopes = new List<string> { "" },
            Operations = new List<ScopedOperations> { new ScopedOperations { Operations = Access.Definitions.Keys.ToList(), Scope = "" } }
        };
        foreach (var definition in Access.Definitions)
            authorization.AllOperations.Add(new Operation { ID = definition.Key, Name = definition.Value });

        return new AppPrincipal(authorization, new GenericIdentity("TestUser"), "S-1-9-5-100") { ActivityID = Guid.NewGuid() };

    }
~~~
... you need to revise this in the light of the change in type of operations (from `int` to `enum`), and the change in type of the `AppPrincipal`'s `ActivityId`:
~~~csharp
    public virtual AppPrincipal GetPrincipal()
    {
        var authorization = new Authorization
        {
            AllOperations = new(),
            Roles = new List<ScopedRoles> { new ScopedRoles { Scope = "", Roles = new List<string> { "Admin" } } },
            Scopes = new List<string> { "" },
            Operations = new List<ScopedOperations> { new ScopedOperations { Operations = Operations.Values.Select(o => o.ID).ToList(), Scope = "" } }
        };
        authorization.AllOperations.AddRange(Operations.Values);

        return new AppPrincipal(authorization, new GenericIdentity("TestUser"), "S-1-9-5-100") { ActivityID = Guid.NewGuid().ToString() };

    }
~~~

If you are using `HostServiceApplicationFactory<TEntryPoint>` for end-to-end testing, this needs to be adapted as follows:
~~~csharp
    class HostServiceApplicationFactory<TEntryPoint> : HostServiceApplicationFactoryBase<TEntryPoint> where TEntryPoint : class
    {
        /// <summary>
        /// The default scope if none is specified
        /// </summary>
        public const string DefaultScope = "";

        /// <summary>
        /// The default role if none is specified
        /// </summary>
        public const string DefaultRole = "Admin";

        private readonly Operation[] _allOperations;
        private List<(string Scope, string Role, int OperationID)>? _rights;

        /// <summary>
        /// Constructor
        /// </summary>
        /// <param name="allOperations">The list of all operations, usually from Access.Definitions in your shared domain assembly</param>
        public HostServiceApplicationFactory(Operation[] allOperations)
        {
            _allOperations = allOperations;
            // don't allow redirection when calling endpoints without proper authentication. This is a test environment and there's no UI.
            ClientOptions.AllowAutoRedirect = false;
        }

        private Authorization CreateAuthorization()
        {
            var authorization = new Authorization { AllOperations = new List<Operation>() };
            authorization.AllOperations.AddRange(_allOperations);
            // If no rights have been explicitly specified, the test user is granted all the operations in the default role and default scope
            if (_rights == null)
            {
                authorization.Scopes.Add(DefaultScope);
                authorization.Roles.Add(new ScopedRoles { Scope = DefaultScope, Roles = new List<string> { DefaultRole } });
                authorization.Operations.Add(new ScopedOperations { Scope = DefaultScope, Operations = _allOperations.Select(o => o.ID).ToList() });
            }
            else
                foreach (var (scope, role, operationID) in _rights)
                {
                    // this looks suspiciously like the code in Arc4u.AttributeStore.ADOGetRightsAuthorizationStore.RunQuery and it's no accident.
                    // the difference is that we are constructing from a list of tuples instead of a result set of a stored procedure.
                    if (!authorization.Scopes.Exists(f => StringComparer.InvariantCultureIgnoreCase.Equals(f, scope)))
                        authorization.Scopes.Add(scope);

                    var scopedRole = authorization.Roles.SingleOrDefault(f => StringComparer.InvariantCultureIgnoreCase.Equals(f.Scope, scope));
                    if (scopedRole == null)
                        authorization.Roles.Add(new ScopedRoles { Scope = scope, Roles = new List<string> { role } });
                    else if (!scopedRole.Roles.Exists(f => StringComparer.InvariantCultureIgnoreCase.Equals(f, role)))
                        scopedRole.Roles.Add(role);

                    var scopedOperation = authorization.Operations.SingleOrDefault(f => StringComparer.InvariantCultureIgnoreCase.Equals(f.Scope, scope));
                    if (scopedOperation == null)
                        authorization.Operations.Add(new ScopedOperations { Scope = scope, Operations = new List<int> { operationID } });
                    else if (!scopedOperation.Operations.Contains(operationID))
                        scopedOperation.Operations.Add(operationID);
                }

            return authorization;
        }

        // remainder of the code here
    }
~~~

Derived classes are trivial to correct. Just pass the `Operations.Values` as argument to the constructor.

## Step 5: Blazor WASM
Because `Config` has disappeared, remove the statements in `Program.cs`:
~~~csharp
    var config = new Config(configuration);
~~~
and
~~~csharp
    container.RegisterInstance<Config>(config);
~~~

You need to replace the following code in `Program.cs`:

~~~csharp
services.RegisterOperationsPolicy(Access.Definitions.AsEnumerable(), options =>
{
    // Add you custom policy here.
    //options.AddPolicy("Custom", policy =>
    //    policy.Requirements.Add(new OperationRequirement(Access.AccessApplication, Access.CanSeeSwaggerFacadeApi)));
});

~~~

by:
~~~csharp
services.AddScopedOperationsPolicy(Operations.Scopes, Operations.Values, options =>
{
    // Add you custom policy here.
    //options.AddPolicy("Custom", policy =>
    //    policy.Requirements.Add(new ScopedOperationsRequirement((int)Access.AccessApplication, (int)Access.CanSeeSwaggerFacadeApi)));
});
~~~

This looks like the same statement that is now in the back-ends `Program.cs` files, and is on purpose.

Nothing else needs to change.

## Step 6: WPF
The disappearence of the `Config` type has two implications:
- references to  `Config` need to be replaced with `ApplicationConfig`
- the initialization changes: in `App.xaml.cs`, the explicit references to `Config` disappears and is replaced by a call to ``:

~~~csharp
    ...
    var container = new ComponentModelContainer(services).InitializeFromConfig(_configuration);

    services.AddApplicationConfig(_configuration);
    container.RegisterInstance<IConfiguration>(_configuration);
    ...
~~~

The WPF applications are otherwise trivially impacted by the Arc4u changes, which won't be described in detail here.
