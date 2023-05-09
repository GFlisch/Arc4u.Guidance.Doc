# Migrating to Arc4u 6.0.14.3
Arc4u version 6.0.14.3 implements breaking changes and a new authentication model for the back-end, which avoids relying on Microsoft-specific libraries.

New versions of the Guidance will automatically generate the correct code. Older back-ends need to be migrated manually. This document explains how.

Part of these instructions also include the necessary changes to support the seamless decryption of encrypted configuration properties, which was introduced in Arc4u 6.0.12.1.


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

... can be deleted and replaced by the following two packages:

~~~xml
    <PackageReference Include="Arc4u.Standard.Configuration.Decryptor" Version="6.0.14.3" />
    <PackageReference Include="Arc4u.Standard.OAuth2.AspNetCore.Authentication" Version="6.0.14.3" />
~~~

The `Arc4u.Standard.Configuration.Decryptor` reference doesn't have anything to do with authentication per se, but is needed 
when decrypting secrets in the configuration files. It's a good idea to include it even if you don't (yet) use it.

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

This update has some implications in the startup code, which will be mentioned in [Step 3.4](#Step-3.4-Update-`Program.cs`).

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
    - add `"Arc4u.Security.Cryptography.X509CertificateLoader, Arc4u.Standard"`
    - add `"Arc4u.OAuth2.Configuration.TokenUserCacheConfiguration, Arc4u.Standard.OAuth2"`
    - add (or make sure it's already there) `""Arc4u.OAuth2.Security.UserObjectIdentifier, Arc4u.Standard.OAuth2.AspNetCore""` (note the change of assembly!)
    - if present, change `"Arc4u.OAuth2.Security.Principal.ClaimsBearerTokenExtractor, Arc4u.Standard.OAuth2.AspNetCore.Adal"` into `"Arc4u.OAuth2.Security.Principal.ClaimsBearerTokenExtractor, Arc4u.Standard.OAuth2"` (note the change of assembly!)
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
    "Default": "Volatile",
    "Principal": {
      "CacheName": "Volatile",
      "Duration": "10:00:00"
    }
  },
~~~

Note that the `"SizeLimit"` property value is usually a small value for developement (like `"10"` here), and a larger value in other environments.
The value will be the same as the `"SizeLimitInMegaBytes"` value in the old `"MemorySize.Settings"` section you just deleted.

#### Authentication configuration for `Yarp` hosts
For the `Yarp` host, the `"OAuth2.Settings"` and `"OpenId.Settings"` sections are replaced by a single section `"Authentication"` that looks like this:

~~~json
 "Authentication": {
    "ClaimsIdentifier": [
      "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn",
      "upn"
    ],
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
       "Authority": "same as the old OAuth2.Settings:Authority",
       "Scopes": "same as the old OAuth2.Settings.:ServiceApplicationId with /user_impersonation appended"
     },
    "OpenId.Settings": {
      "ClientId": "same as the old OpenId.Settings:ClientId",
      "ApplicationKey": "same as the old OpenId.Settings.ApplicationKey"
      "Audiences": "same as the old OpenId.Settings.:ServiceApplicationId",
      "Authority": "same as the old OpenId.Settings:Authority",
      "Scopes": "same as the old OpenId.Settings.:ServiceApplicationId with /user_impersonation appended"
    },
    "DomainsMapping": {
      "belgrid.net": "belgrid",
      "corp.transmission-it.de": "tmit"
    }
  },
~~~

You need to adapt this section for each environment. Most of it should be self-explanatory, but here are a few pointers:
- The name of the certificate (`"Authentication:DataProtection:EncryptionCertificate:CertificateStore:Name"`) depends on the specific environment
- The `"CookieName"` used to be hard-coded in the Yarp host startup code: it is now a per-environment configuration setting. Replace `PROJECT` with the name of your project.
- The `"Audiences"` property value correspond to the content of the old `"ServiceApplicationId"`.
- The `"Scopes`" property correspond to the `"Audiences"` property with `/user_impersonation` appended.
E.g. if `"Audiences"` is `https://project.environment.host`, then `"Scopes"` is `https://project.environment.host/user_impersonation`
- The `"DomainsMapping"` is new: it maps the UPN suffix representing the domain name to the AD domain name. Previously, this 

#### Basic settings
If you have a `"Basic.Settings"` section, the section is written as follows:
~~~json
  "Authentication.Basic": {
    "Settings": {
      "ClientId": "same as the old Basic.Settings:ClientId",
      "Audience": "same as the old Basic.Settings.:ServiceApplicationId",
      "Authority": "same as the old Basic.Settings:Authority"
    }
  }
~~~

#### Authentication settings for non-`Yarp` hosts (standard back-ends)
There is no `"OpenId.Settings"` and the existing `"OAuth2.Settings"` is replaced by an equivalent `"Authentication"` section, but much smaller:
~~~json
  "Authentication": {
    "ClaimsIdentifier": [
      "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn",
      "upn"
    ],
    "MetadataAddress": "same as the old AppSettings:ida_MetadataAddress",
    "OAuth2.Settings": {
      "Authority": "same as the old OAuth2.Settings:Authority",
      "Audiences": "same as the old OAuth2.Settings.:ServiceApplicationId",
       "Scopes": "same as the old OAuth2.Settings.:ServiceApplicationId with /user_impersonation appended"
    }
  },
  "DomainsMapping": {
    "belgrid.net": "belgrid",
    "corp.transmission-it.de": "tmit"
  }
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
... can be deleted and replaced by:
~~~csharp
    var keyValuesOptions = container.Resolve<IOptionsMonitor<SimpleKeyValueSettings>>();
    var OAuth2Settings = keyValuesOptions.Get("OAuth2");
    var OpenIdSettings = keyValuesOptions.Get("OpenId");
~~~

For basic authentication, 

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

Before the call to `UseCreatePrincipal`, add the following statements:
~~~csharp
    var keyValuesOptions = container.Resolve<IOptionsMonitor<SimpleKeyValueSettings>>();
    var OAuth2Settings = keyValuesOptions.Get("OAuth2");
~~~


#### Optional changes
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


## Step 4: Unit Tests
Because `Config` has disappeared, remove the statements in `ContainerFixture`:
~~~csharp
    var config = new Config(configuration);
~~~
and
~~~csharp
    container.RegisterInstance<Config>(config);
~~~
... since `Config` doesn't exist anymore.
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

## Step 5: Blazor WASM
Because `Config` has disappeared, remove the statements in `Program.cs`:
~~~csharp
    var config = new Config(configuration);
~~~
and
~~~csharp
    container.RegisterInstance<Config>(config);
~~~
... since `Config` doesn't exist anymore.
Nothing else needs to change.
