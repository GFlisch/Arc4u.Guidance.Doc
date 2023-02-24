# Release notes for version 2022.2.1.12

- Multiple configurations.
- Add Arc4u Decryptor configuration feature. 
- Namespace issue for gRPC.
- OAuth2 config for Core services.
- Add .editorconfig to a project for governance.
- Update to the latest nuget packages.


## Multiple configurations.

One of the most important feature in this Guidance is the capability to configure multiple configuration for a license.

### What does it means?

If your company has more than on IT Department they can have different setup like authorihty for OAuth2 and OpenID connect but also databases or conventions.
This version introduces this concept by integrating a definition with multiple configs.

```json
{
    "Configs": [
        {
            "Id": "ID1",
            "Company": "Arc4u",
            "Description": "Arc4u department configuration."
        },
        {
            "Id": "ID2",
            "Company": "Dev4u",
            "Description": "Dev4u department configuration.",
        },
        ...

    ]
}

```

When a new project is created or an old (created by previous version of the Guidance) a dialog will popup to select which config you want to apply on your project. If only one config is defined in the Guidance config for your company, no popup is displayed.

Once you have selected the configuration, the information is registered in the Guidance.json file. For old applications, no need to do it manually, the Guidance will detect that you have not registered the configuration id and will ask this to you.

The Guidance.json file will change from

```json
{
  "Version": "2022.2.1"
}
```

to

```json
{
  "Version": "2022.2.1",
  "CompanyId": "ID1"
}
```

The impact of the configuration by environment is now multiplied by the number of configurations you create. It means that you also increase the cost and risks regarding the maintenance of your config.

As an example, I will take the OpenTelemetry one.</br>

Before the config was:

    appsettings.Development.json                                   
    appsettings.Production.json                                    
    appsettings.Staging.json

And after,
    appsettings.Development.ID1.json                                   
    appsettings.Production.ID1.json                                    
    appsettings.Staging.ID1.json
    appsettings.Development.ID2.json                                   
    appsettings.Production.ID2.json                                    
    appsettings.Staging.ID2.json

    
## Add Arc4u Decryptor configuration feature.

This new feature is implementing the new capability offers by the framework, check the doc [here](https://github.com/GFlisch/Arc4u/blob/master/Doc/Framework/General/Configuration%20Decryptor.md).

The code is now adding in the program.cs file the sentence:

```csharp
            config.AddJsonFile("configs/appsettings.json", true, true);
            config.AddJsonFile($"configs/appsettings.{env}.json", true, true);
      ==>   config.AddCertificateDecryptorConfiguration();  <==
```

The configuration for the appsettings section regarding the certificate is to define by each customer!


## Namespace issue for gRPC.

Fix the issue when adding a gRPC service, the namespace in the program.cs was not added. 

## OAuth2 config for Core services

When a micro-services is added to the solution, the OAuth2 section from the Yarp project is copied.
There is no need to copy the ApplicationKey which is a secret and this is not used during the validation of a token. This is now fixed.

## Add .editorconfig to a project for governance

No when a solution is created, an .editorconfig file is created. Basicly the file is a copy of the Aspnet core team.
This is a way to give guidance and code rulling.
The file is a per company settings and can be modified like you want.

## Update to the latest nuget packages.

Finally a check has been done to update the nuget packages used to target their final versions.