# Release notes for version 2022.2.1.10

# !!! This is not yet released!!!

- Guidance startup message.
- NSwag and code generation.
- Change Localhost to Development
- Add Csp
  - Yarp and micro-services
  - Blazor => todo.
- Remove Server info in header.
- Fix Blazor Program.cs issue => MSAL config.

## Guidance startup message.

Now when a new Blank solution is created the message in the window will inform that the Guidance x is ready to be used.<br>
Before it was not the case and introduces doubt if the Guidance is activated or not...

## Change Localhost to Development.

.NET by default implements information about the environment based on the IHostEnvironment.

4 extension methods exist.
- IsDevelopment()
- IsStaging()
- IsProduction()
- IsEnvironment("name of the environment").

The Guidance is not fixing the name of the environment but for the local development machine.<br>
To be compliant with the standard and some code generated in .NET examples the Localhost has been replaced by Development!

=> It means that you will have to update your current project
appsettings.Localhost.json to appsettings.Development.json

Change the ASPNETCORE_ENVIRONMENT to Development in launchSettings.json

```json

{
  "$schema": "http://json.schemastore.org/launchsettings.json",
  "profiles": {
    "Kestrel": {
      "commandName": "Project",
      "launchBrowser": true,
      "launchUrl": "swagger/facade",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": ==> "Development" <>==
      },
      "dotnetRunMessages": true,
      "applicationUrl": "https://localhost:44349"
    }
  }
}

```


