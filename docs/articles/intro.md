# Getting Started with Grunt API

## .NET

To get started with Grunt, you will need to install the `Grunt` NuGet package. There are two sources of packages that you can use:

1. [GitHub](https://github.com/OpenSpartan/grunt/packages/). This is the package that is most frequently updated, but that may come at the cost of stability, as experimental changes are baked in with every commit and _things may break_.
2. [NuGet](https://www.nuget.org/packages/OpenSpartan.Grunt). This way, you're getting the most stable version, but you won't have updates as quickly for new or updated APIs, as releases on NuGet must undergo additional checks.

I generally recommend folks stick with the NuGet release, since it's the one that generally has minimal breaking changes (which are talked about in advance) and has better stability. To add it to your .NET application, run the following from your terminal:

```PowerShell
Install-Package OpenSpartan.Grunt -Version 0.1.2
```

If you are using the `dotnet` CLI, you can run:

```bash
dotnet add package OpenSpartan.Grunt --version 0.1.2
```

Once the package is installed, you need to start writing the connection logic to the Halo service, but worry not, since all of it is wrapped inside Grunt. Keep in mind that for the sample below I am using a Console C# application - you can use a _similar_ or somewhat tweaked approach in any .NET application of your choosing (including web apps).

First, let's create a new Xbox authentication client, handily available through the [`XboxAuthenticationClient`](xref:openspartan.grunt.authentication.xboxauthenticationclient) class. This class is designed to do the bulk of heavy lifting when it comes to interacting with the Xbox services.

>[!Note]
>We still need to use the Xbox Live authentication services to access the Halo API since _everything_ in the API is tied to a user's Xbox Live identity. Therefore, for us to access the Halo API you need to first be able to authenticate with Xbox Live services.

## Node.js

_In development_

## Python

_In development_