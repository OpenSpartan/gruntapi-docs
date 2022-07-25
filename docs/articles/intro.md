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

First, let's create a new Xbox authentication client, handily available through the [`XboxAuthenticationClient`](xref:OpenSpartan.Grunt.Authentication.XboxAuthenticationClient) class. This class is designed to do the bulk of heavy lifting when it comes to interacting with the Xbox services.

>[!Note]
>We still need to use the Xbox Live authentication services to access the Halo API since _everything_ in the API is tied to a user's Xbox Live identity. Therefore, for us to access the Halo API you need to first be able to authenticate with Xbox Live services.

To create the client, use:

```csharp
XboxAuthenticationClient manager = new();
```

Next, we need to make sure that have a URL which can be used for authentication. To do that, there is another helper method:

```csharp
var url = manager.GenerateAuthUrl(clientConfig.ClientId, clientConfig.RedirectUrl);
```

In the context above, `clientConfig` is an instance of [`ConfigurationReader`](xref:openspartan.grunt.util.configurationreader) that I am using to read the client ID and client secret from a local JSON file:

```csharp
var clientConfig = ConfigurationReader.ReadConfiguration<ClientConfiguration>("client.json");
```

This class is a helper container that allows me to take some JSON data from a file and transform it to a C# object. You don't actually have to use it if you store the client ID and secret in some other way. As you saw above from the `GenerateAuthUrl` function, you can pass those strings directly.

>[!Note]
>To be able to authenticate against the Xbox Live service first, you will need to [have an Azure Active Directory application registered](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal#register-an-application-with-azure-ad-and-create-a-service-principal) in the Azure Portal. The application needs to have access to personal Microsoft accounts and can be created on a Free/Trial Azure subscription.
>
>The application enables the user to log in with their own credentials, and then pass the access information to the application that uses the Halo API - in this context, a Grunt sample app.

Now let's add some other items that we need to proceed with the authentication flow:

```csharp
HaloAuthenticationClient haloAuthClient = new();

OAuthToken currentOAuthToken = null;

var ticket = new XboxTicket();
var haloTicket = new XboxTicket();
var extendedTicket = new XboxTicket();
var haloToken = new SpartanToken();
```

In the snippet above we see the following:

- An instance of [`HaloAuthenticationClient`](xref:openspartan.grunt.authentication.haloauthenticationclient) will be used to get the Spartan token, which is the main piece of information we need to be able to get data from the Halo API.
- A placeholder container for [`OAuthToken`](xref:openspartan.grunt.models.oauthtoken), that will store the token we will get from the service.
- Three instances of [`XboxTicket`](xref:openspartan.grunt.models.xboxticket) that will contain authorization information.
- A container for our final [`SpartanToken`](xref:openspartan.grunt.models.spartantoken).

Time for the actual logic. In my sample application, I don't want to go through the dance of getting the auth URL, then getting the code, then pasting the code in and waiting for the tokens to process on every run. Instead, I prefer to store the tokens locally and then work with them and refresh them automatically.

To do the above, I start by reading a `tokens.json` file that I designated locally for the storage of my authentication tokens:

```csharp
if (System.IO.File.Exists("tokens.json"))
{
    Console.WriteLine("Trying to use local tokens...");

    // If a local token file exists, load the file.
    currentOAuthToken = ConfigurationReader.ReadConfiguration<OAuthToken>("tokens.json");
}
else
{
    currentOAuthToken = RequestNewToken(url, manager, clientConfig);
}
```

>[!Warning]
>Anyone that have access to the OAuth tokens for your accounts has _full access_ to said account. Make sure that you store them with caution. Be aware who will be able to see them and when.

If the file exists, I want to read the configuration locally and store the token data in an instance of [`OAuthToken`](xref:openspartan.grunt.models.oauthtoken), as I called out earlier. Otherwise, we're going through the flow of requesting a new token through `RequestNewToken`. This function is something I've created specifically for the sample application, and it wraps some nice functionality exposed in Grunt:

```csharp
private static OAuthToken RequestNewToken(string url, XboxAuthenticationClient manager, ClientConfiguration clientConfig)
{
    Console.WriteLine("Provide account authorization and grab the code from the URL:");
    Console.WriteLine(url);

    Console.WriteLine("Your code:");
    var code = Console.ReadLine();
    var currentOAuthToken = new OAuthToken();

    // If no local token file exists, request a new set of tokens.
    Task.Run(async () =>
    {
        currentOAuthToken = await manager.RequestOAuthToken(clientConfig.ClientId, code, clientConfig.RedirectUrl, clientConfig.ClientSecret);
        if (currentOAuthToken != null)
        {
            var storeTokenResult = StoreTokens(currentOAuthToken, "tokens.json");
            if (storeTokenResult)
            {
                Console.WriteLine("Stored the tokens locally.");
            }
            else
            {
                Console.WriteLine("There was an issue storing tokens locally. A new token will be requested on the next run.");
            }
        }
        else
        {
            Console.WriteLine("No token was obtained. There is no valid token to be used right now.");
        }
    }).GetAwaiter().GetResult();

    return currentOAuthToken;
}
```

What this helper function does is basically ask the user to provide the code from the URL that we created earlier in the very first step. The code is generally appended to the redirect URL for the Azure Active Directory application that you created earlier, and it will show once the user authorizes your application to access Xbox Live through it.

With the code acquired, I then use [`RequestOAuthToken`](xref:openspartan.grunt.authentication.xboxauthenticationclient.requestoauthtoken) through the [`XboxAuthenticationClient`](xref:openspartan.grunt.authentication.xboxauthenticationclient) instance we declared earlier and pass the required application information to it. If the token acquisition process is successful, we store the token to `tokens.json`, otherwise error out.

In the snippet above, `StoreTokens` is yet another helper function I wrote for the sample application. It serializes an [`OAuthToken`](xref:openspartan.grunt.models.oauthtoken) object and stores the output JSON into a file:

```csharp
private static bool StoreTokens(OAuthToken token, string path)
{
    string json = JsonSerializer.Serialize(token);
    try
    {
        System.IO.File.WriteAllText(path, json);
        return true;
    }
    catch
    {
        return false;
    }
}
```

Now, back to continuing the authentication flow. We now need to request the user token, and we do that with the following piece of code:

```csharp
Task.Run(async () =>
{
    ticket = await manager.RequestUserToken(currentOAuthToken.AccessToken);
    if (ticket == null)
    {
        // There was a failure to obtain the user token, so likely we need to refresh.
        currentOAuthToken = await manager.RefreshOAuthToken(
            clientConfig.ClientId,
            currentOAuthToken.RefreshToken,
            clientConfig.RedirectUrl,
            clientConfig.ClientSecret);

        if (currentOAuthToken == null)
        {
            Console.WriteLine("Could not get the token even with the refresh token.");
            currentOAuthToken = RequestNewToken(url, manager, clientConfig);
        }
        ticket = await manager.RequestUserToken(currentOAuthToken.AccessToken);
    }
}).GetAwaiter().GetResult();

```

[`RequestUserToken`](xref:OpenSpartan.Grunt.Authentication.XboxAuthenticationClient.RequestUserToken(System.String)) will helpfully enable us to exchange the OAuth token for something more useful. In the code snippet above, in case the result is `null`, it's very likely that the original OAuth token expired, so we need to refresh it, hence the call to [`RefreshOAuthToken`](xref:OpenSpartan.Grunt.Authentication.XboxAuthenticationClient.RefreshOAuthToken). If the refresh is unsuccessful, we need to run through the `RequestNewToken` flow above, and then go through [`RequestUserToken`](xref:OpenSpartan.Grunt.Authentication.XboxAuthenticationClient.RequestUserToken(System.String)) one more time.

This gets a bit convoluted, but now, assuming that we were successful with the step above, we need to get the [Xbox Live security token (XSTS)](https://docs.microsoft.com/gaming/gdk/_content/gc/live/features/s2s-auth-calls/service-authentication/security-tokens/live-security-tokens). To do that, we use [`RequestXstsToken`](xref:OpenSpartan.Grunt.Authentication.XboxAuthenticationClient.RequestXstsToken(System.String,System.Boolean)) and are exchanging the previous user token for it:

```csharp
Task.Run(async () =>
{
    haloTicket = await manager.RequestXstsToken(ticket.Token);
}).GetAwaiter().GetResult();
```

Notice that I am calling [`RequestXstsToken`](xref:OpenSpartan.Grunt.Authentication.XboxAuthenticationClient.RequestXstsToken(System.String,System.Boolean)) without the optional Boolean argument - this means that by default, I will be using the Halo service relying party. A relying party is a trusted URL that is configured on the developer's side that is used for the creation of a new XSTS token. In the case above, the folks that built Halo already established an appropriate relying party that we can use.

Depending the relying party, the XSTS ticket exchange API will provide some metadata that can be used further in the authorization flow. One piece that is missing from it, however, if I use the Halo relying party, is the Xbox User ID (XUID). To get it, I can tell [`RequestXstsToken`](xref:OpenSpartan.Grunt.Authentication.XboxAuthenticationClient.RequestXstsToken(System.String,System.Boolean)) to _not_ use the Halo relying party and instead use the one for Xbox Live services by setting the optional argument to `false`:

```csharp
Task.Run(async () =>
{
    extendedTicket = await manager.RequestXstsToken(ticket.Token, false);
}).GetAwaiter().GetResult();
```

The extended ticket will have my XUID, which I will use later. Now, I finally get the Spartan token:

```csharp
Task.Run(async () =>
{
    haloToken = await haloAuthClient.GetSpartanToken(haloTicket.Token);
    Console.WriteLine("Your Halo token:");
    Console.WriteLine(haloToken.Token);
}).GetAwaiter().GetResult();
```

[`GetSpartanToken`](xref:OpenSpartan.Grunt.Authentication.HaloAuthenticationClient.GetSpartanToken(System.String)) will take your XSTS ticket and convert it into a reusable Spartan V4 token that can be used with existing Halo API calls. Voila! You are now _almost_ good to go. The last thing is left is for us to configure the [`HaloInfiniteClient`](xref:OpenSpartan.Grunt.Core.HaloInfiniteClient).

Here is how we do it:

```csharp
HaloInfiniteClient client = new(haloToken.Token, extendedTicket.DisplayClaims.Xui[0].Xid);

// Test getting the clearance for local execution.
string localClearance = string.Empty;
Task.Run(async () =>
{
    var clearance = (await client.SettingsGetClearance("RETAIL", "UNUSED", "222249.22.06.08.1730-0")).Result;
    if (clearance != null)
    {
        localClearance = clearance.FlightConfigurationId;
        client.ClearanceToken = localClearance;
        Console.WriteLine($"Your clearance is {localClearance} and it's set in the client.");
    }
    else
    {
        Console.WriteLine("Could not obtain the clearance.");
    }
}).GetAwaiter().GetResult();
```

First, we instantiate the [`HaloInfiniteClient`](xref:OpenSpartan.Grunt.Core.HaloInfiniteClient) class and set the Spartan V4 token and XUID in the constructor. The XUID argument here is optional, but is helpful to have in the future once you start dealing with APIs that do, in fact, require your XUID to move forward.

Next, we use [`SettingsGetClearance`](xref:OpenSpartan.Grunt.Core.HaloInfiniteClient.SettingsGetClearance(System.String,System.String,System.String)) to get the argument that will be further used inside the `343-clearance` header in the API client (don't worry - this is handled automatically). To get the clearance, we are using some built-in parameters that are mimicking the game - `RETAIL` audience, `UNUSED` sandbox, and a `222249.22.06.08.1730-0` build of the executable.

If the clearance acquisition is successful, we can set the clearance in the client as well, otherwise move forward and you will need to set it manually later if you are using a clearance-dependent API call.

You can now get the Halo Infinite data, like this:

```csharp
// Try getting actual Halo Infinite data.
Task.Run(async () =>
{
    var example = await client.StatsGetMatchStats("21416434-4717-4966-9902-af7097469f74");
    Console.WriteLine("You have stats.");
}).GetAwaiter().GetResult();
```

Remember, that you will be getting a <xref:OpenSpartan.Grunt.Models.HaloApiResultContainer`2> as a return value, that contains the type you're asking for, along with any kinds of error messages that might've arisen during the API call.

Congratulations! You are now ready to use Grunt in your .NET application. Make sure to explore the [full set of supported APIs](https://docs.gruntapi.com/api/openspartan.grunt.core/openspartan.grunt.core.haloinfiniteclient), and leave any feedback [on GitHub](https://github.com/OpenSpartan/grunt/issues) if you run into issues. 

## Node.js

_In development_

## Python

_In development_