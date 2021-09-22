---
title: "Developing Native Client Apps for Azure Active Directory"
date: 2014-12-08 16:06:04 -0500
---

In my post titled [Building Web Apps for Azure AD](http://rickrainey.com/2014/11/05/building-web-apps-for-azure-ad/), I discussed developing two types of applications protected by Azure Active Directory: web applications and web API’s. In that post I approached these applications from the perspective of a developer using Visual Studio 2013 and the project templates provided for creating these types of applications. The automation delivered by Visual Studio and the Azure SDK when creating these applications resulted in the two applications being registered in my Azure Active Directory as shown here.

![Azure AD Apps](/assets/img/developing-native-client-apps-for-azure-ad-01.png)

Testing the CloudAlloc.Website was a matter of simply pressing F5 in Visual Studio to launch a browser and access the application. Recall this application is protected using WS-Federation which redirects unauthenticated users to a sign-in page. After successfully authenticating, the user is redirected back to the CloudAlloc.Website URL.

The CloudAlloc.WebAPI is protected using OAuth 2.0 and expects a valid token in the authorization header when its endpoint is accessed. In this post I am going to continue where I left off with the Web API application and discuss how you can develop a native client to acquire an access token that can be used to access the Web API.

A native client application is one that is installed on a user’s computer or device. Examples of such an application include, but is not limited to, WinForms, WPF, Windows Store, Windows Phone, and iOS applications. And while these applications may not necessarily run in Azure as compared to the web application and web API applications discussed in the previous post, native client applications can be protected by Azure Active Directory and access other applications registered with Azure Active Directory.

## Using Claims in the Web API to drive applicaion behavior ##

Before getting into the development of a native client, I want to first point out two changes I made to the Web API to facilitate the topics discussed in this post.

For the Get method, I changed the default implementation provided by the Visual Studio project template to extract the claims from the authenticated user and look for a scope claim with a value of *Read_CloudAlloc_WebAPI* or *Read_Write_CloudAlloc_WebAPI*. If the scope claim is found and contains one of these values, then I return HTTP 200 (OK). Otherwise, I return an HTTP 401 (Unauthorized).You may be wondering where I got the scope claim and value. Azure Active Directory will add this claim and value to the claim set for authenticated users provided that my Web API has been configured to support this claim and my client (native client) has been granted the permissions to include the claim. I’ll come back to this shortly. For now, just understand that with this code, the Web API now requires that this claim and value be present for the user to be able to read these values.The updated Get method implementation is shown here.

```c#
// GET api/values
public HttpResponseMessage Get()
{
      // Look for the scope claim containing the
      //  value 'Read_CloudAlloc_WebAPI' or 'Read_Write_CloudAlloc_WebAPI'
      var principal = ClaimsPrincipal.Current;
      Claim readValuesClaim = principal.Claims.FirstOrDefault(
            c => c.Type == "http://schemas.microsoft.com/identity/claims/scope" &&
            (c.Value.Contains("Read_CloudAlloc_WebAPI") ||
            (c.Value.Contains("Read_Write_CloudAlloc_WebAPI"))));
 
      if (null != readValuesClaim)
      {
          //
          // Code to retrieve values to include in the response goes here.
          //
          return Request.CreateResponse(HttpStatusCode.OK);            }
       else
       {
          return Request.CreateErrorResponse(
              HttpStatusCode.Unauthorized, "You don't have permissions to read values.");
      }
}
```

For the Post method, I made similar changes but instead look for the scope claim containing the value *Read_Write_CloudAlloc_WebAPI*. If the scope claim is found and contains this value, then I return HTTP 201 (Created) to simulate the value being added to the server. Otherwise, I return an HTTP 401 (Unauthorized). The updated Post method implementation is shown here.

```c#
// POST api/values
public HttpResponseMessage Post([FromBody]string value)
{
      // Look for the scope claim containing the value 'Read_Write_CloudAlloc_WebAPI'
      var principal = ClaimsPrincipal.Current;
      Claim writeValuesClaim = principal.Claims.FirstOrDefault(
          c => c.Type == "http://schemas.microsoft.com/identity/claims/scope" &&
              c.Value.Contains("Read_Write_CloudAlloc_WebAPI"));
 
       if (null != writeValuesClaim)
       {
           //
           // Code to add the resource goes here.
           //
           return Request.CreateResponse(HttpStatusCode.Created);
       }
       else
       {
           return Request.CreateErrorResponse(
               HttpStatusCode.Unauthorized, "You don't have permissions to write values.");
       }
}
```

> Note: Generally it is recommended to organize code that tests for the presence of claims with this level of granularity into a custom [AuthorizeAttribute class](http://msdn.microsoft.com/en-us/library/system.web.mvc.authorizeattribute(v=vs.118).aspx). By doing so you isolate the authorization code into one place, reducing the potential for error and improving maintainability of the code. However, in the interest of keeping the focus in this section on how claims can be used to make authorization decisions, I’m including it directly in the method for readability.

## Exposing the Web API to other applications ##

To make the Web API accessible to other applications in Azure Active Directory, permissions must be defined for the Web API that can be assigned to other applications. The Azure Management portal does not provide a user interface to do this. However, it does offer you access to the application’s manifest where you can configure settings for an application, including settings that the portal doesn’t provide a user interface for. Here are the steps to get to an application’s manifest using the Azure Management portal.

1. Go to the **APPLICATIONS** page for the Azure Active Directory the application is registered in.
2. Click on the name of the application whose manifest you’re interested in. For this post, it is the CloudAlloc.WebAPI application.
3. At the bottom of the screen click on the **MANAGE MANIFEST** button and select the option to **Download Manifest** and save it to your local computer, as shown here.

![Download manifest](/assets/img/developing-native-client-apps-for-azure-ad-02.png)

The manifest is a JSON formatted file that contains the Azure Active Directory configuration for an application registered in Azure Active Directory. If you scroll through the file you will see the settings you see on the **CONFIGURE** page for an application and much more. The configuration setting we’re interested in for this post is the oauth2Permissions setting. By default, this is an empty array, as shown here.

![Default oauth2Permissions setting](/assets/img/developing-native-client-apps-for-azure-ad-03.png)

For this application (the CloudAlloc.WebAPI), I am adding two permissions to this array; one that allows read access to the API and one that allows read/write access to the API. Notice the value that I indicated for each permission. This is where the values I’m looking for in the scope claim originate from in the Get and Post methods above.

```json
"oauth2Permissions": [
  {
    "adminConsentDescription": "Allow read access to the CloudAlloc WebAPI on behalf of the signed-in user",
    "adminConsentDisplayName": "Read access to CloudAlloc WebAPI",
    "id": "1835E3A9-C857-4F33-A357-751E620E558D",
    "isEnabled": true,
    "origin": "Application",
    "type": "User",
    "userConsentDescription": "Allow read access to the CloudAlloc WebAPI on your behalf",
    "userConsentDisplayName": "Read access to CloudAlloc WebAPI",
    "value": "Read_CloudAlloc_WebAPI"
  },
  {
    "adminConsentDescription": "Allow read-write access to the CloudAlloc WebAPI on behalf of the signed-in user",
    "adminConsentDisplayName": "Read-Write access to CloudAlloc WebAPI",
    "id": "87A81936-E765-4678-B6DB-8E12197AAA7D",
    "isEnabled": true,
    "origin": "Application",
    "type": "User",
    "userConsentDescription": "Allow read-write access to the CloudAlloc WebAPI on your behalf",
    "userConsentDisplayName": "Read-Write access to CloudAlloc WebAPI",
    "value": "Read_Write_CloudAlloc_WebAPI"
  }],
```

The schema for the oauth2Permissions can be found in the MSDN documentation for [adding, updating, and removing an application in Azure Active Directory](http://msdn.microsoft.com/en-us/library/azure/dn132599.aspx).

After making this update to the manifest file all that is left is to upload it to Azure by clicking the **MANAGE MANIFEST** button and selecting the **Upload Manifest** option.

Now the Web API application can be accessed from other applications using these permissions.

## Add a native client application to Azure Active Directory ##

Regardless of the type of native client application you plan to build, the first step is to register it in Azure Active Directory. This is easily done using the Azure Management portal by clicking the **ADD** button in the **APPLICATIONS** page of the directory and selecting the option to **add an application my organization is developing**. This will open a wizard where you can specify the name and type of application as I’ve done here.

![Add Azure AD application](/assets/img/developing-native-client-apps-for-azure-ad-04.png)

The next page of the wizard will prompt you for the redirect URI associated with the native client application. This is a URI (not a URL), so any value will work as long as it is a valid URI and is unique to your directory as shown here.

![Add Azure AD application URI](/assets/img/developing-native-client-apps-for-azure-ad-05.png)

That is all that is required to register the application with Azure Active Directory. The application will appear in the **APPLICATIONS** page of the directory as a Native client application as shown here.

![Azure AD applications page](/assets/img/developing-native-client-apps-for-azure-ad-06.png)

## Configure permissions to access the Web API ##

Since this native client application is going to be accessing the CloudAlloc.WebAPI I need to configure the permissions for it. In the **CONFIGURE** page of the native client application is where permissions to other applications can be set. And since my Web API application is now accessible as a result of the manifest changes I made earlier, I can easily configure the native client with the permissions I want to give it. For now, I’m going to start by assigning the *Read access to CloudAlloc WebAPI* permission as shown here.

![Azure AD application permissions](/assets/img/developing-native-client-apps-for-azure-ad-07.png)

As users of the native client application authenticate to Azure Active Directory, they will now get the scope claim with a value of *Read_CloudAlloc_WebAPI* in addition to other claims issued by Azure Active Directory.

## Develop the native client application ##

For this post I’m going to build a simple console application for the native client. However, the code in this application would work the same for other types.

Recall that my Web API is protected by Azure AD and expects a security token when accessed from client applications. So, I need a way to authenticate users and acquire a security token (JWT) that can be used when calling the API. Thankfully, Microsoft provides a super-handy library called the Active Directory Authentication Library (ADAL) that simplifies this. Since my client application is a C# console application, I’ll be using [ADAL for .NET](https://github.com/AzureAD/azure-activedirectory-library-for-dotnet) (version 2.x). But, there are other flavors available such as [ADAL for JavaScript](https://github.com/AzureAD/azure-activedirectory-library-for-js), [ADAL for Node.js](https://github.com/AzureAD/azure-activedirectory-library-for-nodejs), [ADAL for Java](https://github.com/AzureAD/azure-activedirectory-library-for-java), [ADAL for Android](https://github.com/AzureAD/azure-activedirectory-library-for-android), and [ADAL for iOS and OSX](https://github.com/AzureAD/azure-activedirectory-library-for-objc) making native client application development for Azure AD a breeze for the most popular languages and platforms used in the enterprise.

To construct the HTTP requests to my Web API, I will use the [Microsoft HTTP Client Libraries](https://www.nuget.org/packages/Microsoft.Net.Http) (version 2.x).

### Define settings used by the native client application ###

The native client application needs the Client ID and Redirect URI of the application registered with Azure AD. The Client ID is generated automatically when the application is registered and the Redirect URI is the URI entered in the new application wizard earlier.

```c#
// Native client application settings
private string clientID = "386985f0-b940-4713-a8e5-f7a49d39f368";
private Uri redirectUri = new Uri("http://cloudalloc.webapi.client");
```

Both of these settings can be found in the properties section of the **CONFIGURE** page for the native client application in the Azure Management portal as shown here.

![Azure AD application configuration](/assets/img/developing-native-client-apps-for-azure-ad-08.png)

When a native client needs to get a token from Azure Active Directory, it needs to specify the resource it wants a token for. In this scenario the client application wants access to the Web API so the APP ID URI for the Web API is used as the resource name. After it has the token it also needs to know the URL where the resource can be accessed, in this case the address of the Web API.

```c#
// Resource settings this application wants to access
private string resource = "https://cloudalloc.com/CloudAlloc.WebAPI";
private Uri WebAPIUri = new Uri("https://localhost:44313");
```

Both of these settings can be found in the single sign-on section of the **CONFIGURE** page for the Web API application in the Azure Management portal as shown here.

![Azure AD application configuration](/assets/img/developing-native-client-apps-for-azure-ad-09.png)

> Note: The reason the reply URL is a localhost address is simply because in the previous post when these applications were created and registered in Azure Active Directory, I didn’t publish them to Azure Websites. If I had, then this reply URL would have a *.azurewebsites.net address instead. To see how publishing a web application and/or web API to Azure applies to applications registered in Azure Active Directory see my post on [Authenticating with Organizational Accounts and Azure Active Directory](http://rickrainey.com/2014/07/28/authenticating-with-organizational-accounts-and-azure-active-directory/).

For ADAL to authenticate the user and acquire a token for the resource, an **AuthenticationContext** must first be instantiated by passing in the URL of your tenant in Azure Active Directory. The URL is of the form *https://login.windows.net/<tenantID>* where <tenantID> can be either the GUID assigned to your tenant in Azure AD or the domain if you have a custom domain configured.

```c#
// Session to Azure AD
private const string authority = "https://login.windows.net/cloudalloc.com";
private AuthenticationContext authContext = new AuthenticationContext(authority);
```

The **AuthenticationContext** is like a connection to your Azure Active Directory and is ultimately used to acquire tokens from your directory.

## Call the Web API to get values ##

The code to issue an HTTP GET request to the Web API essentially breaks down to three tasks:

1. Authenticate the user and get a token from Azure Active Directory. Here is where the **AuthenticationContext** instance is used to call the **AcquireToken** method. If the user is not already authenticated, then the ADAL library will launch a sign-in page for the user to sign-in with. After successfully authenticating, a security tokento access the resource using the native client application is issued.
2. An **HttpClient** instance is instantiated and configured to include the *security token* in the **Authorization** header for HTTP calls to the resource (Web API).
3. An HTTP GET is issued to the resource’s URL to retrieve the values.

The full source code for the native client to call the Get method on the Web API is shown here.

```c#
public async Task<bool> ReadValues()
{
      // Authenticate the user and get a token from Azure AD
      AuthenticationResult authResult = authContext.AcquireToken(resource, clientID, redirectUri);
 
      // Create an HTTP client and add the token to the Authorization header
      HttpClient httpClient = new HttpClient();
      httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue(
          authResult.AccessTokenType, authResult.AccessToken);
 
      // Call the Web API to get the values
      Uri requestURI = new Uri(WebAPIUri, "api/values");
      Console.WriteLine("Reading values from '{0}'.", requestURI);
      HttpResponseMessage httpResponse = await httpClient.GetAsync(requestURI);
      Console.WriteLine("HTTP Status Code: '{0}'", httpResponse.StatusCode.ToString());
      if (httpResponse.IsSuccessStatusCode)
      {
          //
          // Code to do something with the data returned goes here.
          //
      }           
      return (httpResponse.IsSuccessStatusCode);
}
```

## Call the Web API to post a new value ##

The code to issue an HTTP POST request to the Web API is almost identical. Since this method expects a new value in the body of the POST, the content-type header needs to be set and the content needs to be passed along in the body. Otherwise, it’s the same code.

The full source code for the native client to call the Post method on the Web API is shown here.

```c#
public async Task<bool> WriteValue(string value)
{
      // Authenticate the user and get a token from Azure AD
      AuthenticationResult authResult = authContext.AcquireToken(resource, clientID, redirectUri);
 
      // Create an HTTP client and add the token to the Authorization header
      HttpClient httpClient = new HttpClient();
      httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue(
          authResult.AccessTokenType, authResult.AccessToken);
 
 
      // Construct the payload for the HTTP post operation
      var newValue = new StringContent(value);
      newValue.Headers.ContentType = new MediaTypeHeaderValue("application/x-www-form-urlencoded");
 
      // Call the Web API to post the new values
      Uri requestURI = new Uri(WebAPIUri, "api/values");
      Console.WriteLine("Writing value '{0}' to '{1}'.", value, requestURI);
      HttpResponseMessage httpResponse = await httpClient.PostAsync(requestURI, newValue);
      Console.WriteLine("HTTP Status Code: '{0}'", httpResponse.StatusCode.ToString());
 
      return (httpResponse.IsSuccessStatusCode);
}
```

## Running the native client application ##

My primitive console application calls the **ReadValues** method and then the **WriteValue** method shown above. When **AcquireToken** is called in the **ReadValues** method, I am immediately prompted to sign-in since a token doesn’t already exist for me.

![Azure AD Sign-In](/assets/img/developing-native-client-apps-for-azure-ad-10.png)

After successfully authenticating, a token is issued to me and the values are returned from the Get method of the Web API because the security token included the scope claim with the *Read_CloudAlloc_WebAPI* value.

Next, the **WriteValue** method is called. This time I am not prompted to sign-in because ADAL has cached my token and can use it since it hasn’t expired (more on this shortly). However, because the *Read_Write_CloudAlloc_WebAPI* value is not present in the scope claim, the Post method in the Web API returned an HTTP 401 (Unauthorized).

The full output of the console application is shown here.

![Console output](/assets/img/developing-native-client-apps-for-azure-ad-11.png)

Using the Azure Management portal, I can change the permissions for my native client application to grant the read/write permission instead, as shown here.

![Azure AD Application Permissions](/assets/img/developing-native-client-apps-for-azure-ad-12.png)

Running the native client application now results in a successful call to both the Get and Post methods as shown here.

![Console output](/assets/img/developing-native-client-apps-for-azure-ad-13.png)

## ADAL’S Token Cache and Refresh Tokens ###

Previously I mentioned that ADAL cached my token. ADAL does this automatically without you having to write any code, resulting in a positive experience for the end-user. If the token hasn’t expired, ADAL will re-use it in subsequent calls to **AcquireToken**. However, even if the token has expired, the end-user may still avoid being prompted to sign-in if a *refresh token* was issued when the access token was issued.

A refresh token, which may not always be present, can be used to acquire a new access token on behalf of the user if Azure AD allows it. As long as the user’s account in Azure AD hasn’t been deleted, disabled, or some other change in the directory that would invalidate the token, the refresh token can be exchanged for a new access token. Best of all, ADAL will use the refresh token to acquire a new access token if it can and do so without you having to write any code.

The figure below shows an instance of the **AuthenticationResult** returned from the first call to **AcquireToken**. You can see the time the **AccessToken** expires in the **ExpiresOn** property and the **RefreshToken** ADAL may use to acquire a new **AccessToken**.

And it’s not always just for a single resource that a refresh token can be used to acquire an access token. If the **IsMultipleResourceRefreshToken** property is set to true, the **RefreshToken** can also be used to acquire an **AccessToken** for other resources registered in my Azure AD. Currently, I’m only accessing the Web API, but if my native client accessed another resource, then this **RefreshToken** could be used to acquire the **AccessToken** for that resource too without me having to sign-in.

![Refresh Tokens](/assets/img/developing-native-client-apps-for-azure-ad-14.png)

As you can see, ADAL provides a lot of benefits to native client applications with just a couple of lines of code.

## Summary ##

In this post I demonstrated how to use claims in a Web API application protected by Azure Active Directory to make authorization decisions based on the presence of a scope claim in the authenticated user’s claim set.

Next, I discussed how to expose the Web API to other applications in Azure AD by defining permissions in the *oauth2Permissions* array that can be updated in the manifest file for the Web API. By defining permissions for the Web API, other applications can be configured to access the application with selected permissions using the Azure Management portal.

After the Web API was updated and configured, I discussed the steps necessary to register a native client application in Azure Active Directory. Using values in Azure Active Directory for the registered native client application, I then showed how to develop a console application that uses the Active Directory Authentication Library (ADAL) and the Microsoft HTTP Client Library to securely call the Web API. Running the native client, we were able to observe the benefit of the token cache provided by ADAL and the use of a refresh token to avoid unnecessarily prompting the user for credentials.

In the next post, I will show how you can extend this scenario and make authorizations decisions based on a user’s group membership.
