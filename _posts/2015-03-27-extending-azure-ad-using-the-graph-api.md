---
title: "Extending Azure AD using the Graph API"
date: 2015-03-27 16:06:04 -0500
permalink: /:year/:month/:day/:title/
---

The Azure Active Directory Graph API enables some interesting scenarios that you can implement in your applications by enabling you to query and manipulate directory objects in Azure AD. In the last post I presented you with some common scenarios available via the Azure AD Graph API and showed how you can implement them using the Azure Active Directory Graph Client Library. In this post, I’m going to introduce you to another scenario made possible using Azure AD Graph API and then take you on a journey through its implementation.

Applications often have requirements for user data that is beyond what we typically would get from the [Claims](https://msdn.microsoft.com/en-us/library/system.security.claims.claimsprincipal.claims(v=vs.110).aspx) collection for an authenticated user. In such cases, it is very common to store additional user data, for example profile data, in a database and then use it as needed in the application. This is generally the best practice most developers follow. However, if your additional user data requirements are small and your application is protected by Azure AD, then the Azure AD Graph API gives you another technique you can use whereby you can extend the directory schema in Azure AD to include the data your application needs. This feature is called Azure AD Graph API Directory Schema Extensions and can be used to store and retrieve extension properties (ie: custom data) for a variety of object types in Azure AD.

This is really cool, but it does have some limitations so don’t think this should be your go-to solution for all scenarios like this. However, given the right situation, it can be a nifty and cost-effective solution for your application. The limitations of this feature are as follows:

- An extension property must be either a **String** type or **Binary** type.
- The size of the data that can be stored in the extension property **cannot exceed 256 bytes**.
- The directory objects that can be extended using extension properties are [User](https://msdn.microsoft.com/library/azure/ad/graph/api/entity-and-complex-type-reference#UserEntity), [Group](https://msdn.microsoft.com/library/azure/ad/graph/api/entity-and-complex-type-reference#GroupEntity), [TenantDetail](https://msdn.microsoft.com/library/azure/ad/graph/api/entity-and-complex-type-reference#TenantDetailEntity), [Device](https://msdn.microsoft.com/library/azure/ad/graph/api/entity-and-complex-type-reference#DeviceEntity), [Application](https://msdn.microsoft.com/library/azure/ad/graph/api/entity-and-complex-type-reference#ApplicationEntity), and [ServicePrincipal](https://msdn.microsoft.com/library/azure/ad/graph/api/entity-and-complex-type-reference#ServicePrincipalEntity).
- A directory (or Azure AD tenant) can have **up to 100 extension properties registered**.

## A Sample Scenario ##

Now that we know what our limitations are let’s look at a scenario. Assume that I’m building a line-of-business application to manage parking passes for employees. In addition to the employee information I can get from Azure AD I also need some basic vehicle information about the employee’s vehicle so building security knows which vehicles are authorized to be in the parking lot or garage. If this is all I need, then using schema extensions provides a perfect alternative to taking a traditional database approach. Let’s see how this can be done using the Azure AD Graph API Directory Schema Extensions and the Azure AD Graph Client Library.

## Register a New Application in Azure AD ##

For this post I’m using the same application I registered in the previous post to demonstrate the directory schema extensions feature. So, rather than repeat those steps here, see the [Register a New Application in Azure AD](http://rickrainey.com/2015/02/21/introducing-the-azure-ad-graph-api/) section of the previous post.

## Create a New Console Application Using Visual Studio ##

I am also using the same console application I used in the previous post. So, I won’t be repeating the nuances of instantiating an instance of the **ActiveDirectoryClient** in the Azure AD Graph Client Library. I’m assuming that when you see *adClient* in the forthcoming code that you know how it was created. If not, then please go back and review the previous post.

The only change I did make in this version of the application is that I updated the NuGet package for the Active Directory Graph Client Library. It wasn’t necessary for this post, I just updated to get the latest package. So, for the sample code in this post, I’m using these two NuGet packages.

1. [Active Directory Authentication Library (ADAL)](https://www.nuget.org/packages/Microsoft.IdentityModel.Clients.ActiveDirectory/2.14.201151115), v2.14.201151115
2. [Azure Active Directory Graph Client Library](https://www.nuget.org/packages/Microsoft.Azure.ActiveDirectory.GraphClient/2.0.6), v2.06

Perfect! Now with those disclaimers out of the way, let’s dive right into directory schema extensions.

## Define a Model for Vehicle Information ##

In this sample scenario, I added a simple class to hold essential vehicle information that my application will need as shown here. Notice that I’m not using properties but instead just member variables. This helps to reduce the size of the object when it is serialized as binary and therefore intentional in light of limitation #2 above.

```c#
public class VehicleInfo
{
    public int Year;
    public string Make;
    public string Model;
    public string Color;
}
```

You may be thinking that defining a complex type like this violates limitation #1 above. In fact, you can represent an instance of a complex type using String or Binary data and I’ll show you both. If you want to see some examples of simple String extension properties, then take a look at the blog from the Azure AD Graph Team referenced at the bottom of this post.

Now, for instances of **VehicleInfo** to be available as a property for users in my directory, I need to first register anextension property with Azure AD.

## Register an Extension Property in Azure AD ##

Extension properties are defined using the **ExtensionProperty** class from the Azure AD Graph Client Library. To register an extension property you need to create one and set just a few required properties. This is a one-time operation. The extension property for my vehicle information data is shown here.

```c#
// Create the extension property
string extPropertyName = "VehInfo";
ExtensionProperty extensionProperty = new ExtensionProperty()
{
    Name = extPropertyName,
    DataType = "String",
    TargetObjects = { "User" }
};
```

The **Name** is whatever I want it to be. The **DataType** is set to String (for now). Remember, this can be either String or Binary (limitation #1 above). The **TargetObjects** I have set to User because it applies to only users in my directory (limitation #2 above).

An **ExtensionProperty** is always registered for an **Application**    , which means you must first get a reference to the application registered in Azure AD. If you remember from previous posts, Azure AD generates a unique Client ID for an application when you register it in Azure AD. This Client ID can be used to locate the application (via its AppId) using a simple LINQ query as shown here.

```c#
// Get a reference to our application
Microsoft.Azure.ActiveDirectory.GraphClient.Application app =
    (Microsoft.Azure.ActiveDirectory.GraphClient.Application)adClient.Applications.Where(a => a.AppId == ClientID)    .ExecuteSingleAsync().Result;

if (app == null)
{
    throw new ApplicationException("Unable to get a reference to application in Azure AD.");
}
```

After you have a reference to the Application that was registered in Azure AD, you can register the extension for the application as shown here.

```c#
// Register the extension property
app.ExtensionProperties.Add(extensionProperty);
app.UpdateAsync();
Task.WaitAll();
 
// Apply the change to Azure AD
app.GetContext().SaveChanges(); 
```

The **Add** method adds the extension property to the **ExtensionProperties** collection for the application. The **UpdateAsync** method updates the client side Application object while the **SaveChanges** method of the application’s context is used to actually persist the change to Azure AD.

To show what this extension property looks like in Azure AD, I used Postman to call the Azure AD Graph API to get the extension property registered from the code above.  The image here shows how it is actually represented in Azure AD.

![Extension property in Azure AD](/assets/img/extending-azure-ad-using-the-graph-api-01.png)

Notice on line 10 that the name of the extension is not just the name “VehInfo” that I specified earlier. The actual naming convention in Azure AD for an extension property is **extension_[AppID]_[ExtName]**, where [AppID] is the Client ID assigned to your application when you register it using the Azure Management Portal as shown below.  The [ExtName] is the name that was specified in the **ExtensionProperty.Name** above. This naming convention is generally tr   ansparent to you if you are using the Azure AD Graph Client Library. However, if you are using the REST API then this naming convention is important to follow so I’m making this point for those who may choose to use the REST API instead of the client library.

![Extension property in Azure AD](/assets/img/extending-azure-ad-using-the-graph-api-02.png)

Now that Azure AD knows about the extension property for my application, let’s see how you can set the value of the property for a user.

## Lookup a Previously Registered Extension ##

To lookup a previously registered extension you can query the **ExensionProperties** collection of the application as shown here.

```c#
// Locate the vehicle info property extension
IEnumerable<IExtensionProperty> appExtProperties = app.ExtensionProperties;
IExtensionProperty vehInfoExtProperty = appExtProperties.Where(
    prop => prop.Name == extPropertyName).FirstOrDefault();
```

Unfortunately, at the time of writing this post using v2.06 of the client library, this code doesn’t always work. The **ExtensionProperties** collection is always empty if your application instance is not the same instance that registered the extension property. Fortunately, you can work around this by calling the **ActiveDirectoryClient.GetAvailableExtensionPropertiesAsync** method to query for the extension property. This approach requires some knowledge of how the extension property name is actually represented in Azure AD which I showed previously. So, with that understanding of the naming convention, I can query for the single extension property I’m looking for as shown here.

```c#
// Locate the vehicle info property extension (workaround)
string extPropLookupName = string.Format("extension_{0}_{1}", ClientID.Replace("-", ""), extPropertyName);
IEnumerable<IExtensionProperty> allExtProperties = adClient.GetAvailableExtensionPropertiesAsync(false).Result;
IExtensionProperty vehInfoExtProperty = allExtProperties.Where(
    extProp => extProp.Name == extPropLookupName).FirstOrDefault();
```

## Set the Extension Property Value for a User ##

To set the extension property for a user you need to get a **User** reference. I’m going to stick with the “John Doe” user I’ve been using throughout this blog series and invoke a LINQ query against the **ActiveDirectoryClient.Users** property to get a reference to John Doe’s directory object as shown below. Next, I am creating an instance of the VehicleInfo class I defined previously and then used the **User.SetExtendedProperty** method to set the value for my user. Recall that our extension property can be either a String or Binary type – complex types are not supported (limitation #1 from above). To get around this, I’m using the [Json.NET](http://www.newtonsoft.com/json), v6.08 NuGet package to serialize my vehicleInfo object to a JSON string. Finally, I invoked UpdateAsync to update the client side object and then **SaveChange** on the user’s context to save the change in Azure AD.

```c#
// Set the extension property value for a user
var upn = "johndoe@cloudalloc.com";
User user = (User)adClient.Users.Where(u =&amp;amp;amp;gt; u.UserPrincipalName.Equals(
    upn, StringComparison.CurrentCultureIgnoreCase)).ExecuteSingleAsync().Result;
 
if (user != null)
{
    var vehicleInfo = new VehicleInfo()
    {
        Make = "Ford",
        Model = "F150",
        Color = "Silver",
        Year = 2014
    };
 
    user.SetExtendedProperty(vehInfoExtProperty.Name, JsonConvert.SerializeObject(vehicleInfo));
    user.UpdateAsync();
    Task.WaitAll();
 
    // Save the extended property value to Azure AD.
    user.GetContext().SaveChanges();
}
```

The code above results in John Doe’s directory object being updated with the extension property value as shown below    . On line 41 you can see the extension property and the JSON representation of the object.

![Extension property in Azure AD](/assets/img/extending-azure-ad-using-the-graph-api-03.png)

## Retrieve the Extension Property Value for a User ##

To retrieve the value of an extension property for a user you can query the collection returned from the User.GetExtendedProperties method as shown here. And since I’m serializing my complex type to a JSON string when I store it, I am de-serializing it back into a VehicleInfo object when I get it back.

```c#
// Retrieve the extension property value for a user
VehicleInfo vehInfo = null;
var userExtProperty = user.GetExtendedProperties().Where(
    extProp => extProp.Key.Equals(
        vehInfoExtProperty.Name,
        StringComparison.CurrentCultureIgnoreCase)).FirstOrDefault();
if (userExtProperty.Value != null)
{
    vehInfo = JsonConvert.DeserializeObject<VehicleInfo>((string)userExtProperty.Value);
}
```

## Remove the Extension Property Value for a User ##

Removing an extension property for a user is simply a matter of setting the value to null as shown here.

```c#
// Remove the extended property for a user
user.SetExtendedProperty(vehInfoExtProperty.Name, null);
user.UpdateAsync();
Task.WaitAll();
user.GetContext().SaveChanges();
```

The code above removes the extension property (line 41 in the image above) from John Doe’s directory object in Azure AD.

## Unregister an Extension Property in Azure AD ##

If your application no longer needs an extension property you should unregister/remove it to free the space for potentially other extension properties (limitation #4 above). Unregistering an extension property may be accomplished using the **Remove** method of the **Application.ExtensionProperties** collection as shown here. Note: When you unregister an extension property, it also removes it from any directory objects ( ie: users ) that you may have set the extension property for.

```c#
// Unregister the extension property
app.ExtensionProperties.Remove(extensionProperty);
app.UpdateAsync();
Task.WaitAll();
 
// Apply the change to Azure AD
app.GetContext().SaveChanges(); 
```

Unfortunately, at the time of this writing using v2.06 of the client library, this method does not actually remove it from Azure AD. As a workaround, the REST API to unregister an extension property does work so until this is corrected that is the only way to do this. The REST API is included in the References section at the end of this post.

## Storing Extension Properties as Binary Data ##

As I mentioned at the beginning of this post, extension property data can also be stored as binary data. You must be mindful of the 256 byte limit just as you had to be for string data. This can be a bit tricky as the binary representation of a small data type like the one I’m using is significantly larger than its JSON representation. During tests of storing my VehicleInfo object as binary data, I was coming in at about 205 bytes which was much higher than the 70 bytes used to store it as a JSON string. Still, it’s an interesting use case so in this section I’ll show how to create a binary extension property and then store and retrieve the data.

To demonstrate this I’m going to use the [BinaryFormatter](https://msdn.microsoft.com/en-us/library/system.runtime.serialization.formatters.binary.binaryformatter(v=vs.110).aspx) to serialize my object to binary. The BinaryFormatter requires that my class be marked with the Serializable attribute as I’ve shown below. Otherwise, my class remains unchanged.

```c#
[Serializable]
public class VehicleInfo
{
    public int Year;
    public string Make;
    public string Model;
    public string Color;
}
```

## Register a Binary Extension Property in Azure AD ##

To register a binary extension property with Azure AD you need to create a new **ExtensionProperty** and set the **DataType** to Binary as shown in the code below. It is exactly the same as before with just two changes that are highlighted: the name of the property and the data type being set to *Binary*.

```c#
// Create the extension property
string extPropertyName = "VehInfoAsBinary";
ExtensionProperty extensionProperty = new ExtensionProperty()
{
    Name = extPropertyName,
    DataType = "Binary",
    TargetObjects = { "User" }
};
 
// Get a reference to our application.
Microsoft.Azure.ActiveDirectory.GraphClient.Application app =
                (Microsoft.Azure.ActiveDirectory.GraphClient.Application)adClient.Applications.Where(
    a => a.AppId == ClientID).ExecuteSingleAsync().Result;
if (app == null)
{
    throw new ApplicationException("Unable to get a reference to application in Azure AD.");
}
 
// Register the extension property
app.ExtensionProperties.Add(extensionProperty);
app.UpdateAsync();
Task.WaitAll();
 
// Apply the change to Azure AD
app.GetContext().SaveChanges();
```

The code above results in my directory now having two extension properties registered as shown below.

![Extension property in Azure AD](/assets/img/extending-azure-ad-using-the-graph-api-04.png)

## Set the Binary Extension Property Value for a User ##

An example of how this data can be set using the BinaryFormatter is shown below. As you can see, it is essentially the same code with just a few changes highlighted to serialize the data to binary.

```c#
// Set the extension property value for a user
var upn = "johndoe@cloudalloc.com";
User user = (User)adClient.Users.Where(u =&amp;gt; u.UserPrincipalName.Equals(
                upn, StringComparison.CurrentCultureIgnoreCase)).ExecuteSingleAsync().Result;
 
if (user != null)
{
    var vehicleInfo = new VehicleInfo()
    {
        Make = "Ford",
        Model = "F150",
        Color = "Silver",
        Year = 2014
    };
 
    BinaryFormatter binaryFormatter = new BinaryFormatter();
    using (MemoryStream memoryStream = new MemoryStream())
    {
        binaryFormatter.Serialize(memoryStream, vehicleInfo);
        user.SetExtendedProperty(vehInfoExtProperty.Name, memoryStream.GetBuffer());
        user.UpdateAsync();
        Task.WaitAll();
 
        // Save the extended property value to Azure AD.
        user.GetContext().SaveChanges();
    }
}
```

The code above results in John Doe’s directory object being updated with the extension property value as shown below. On line 41 is a property indicating that the extension property is a Binary extension and line 42 shows the actual value of the extension property (in binary).

![Extension property in Azure AD](/assets/img/extending-azure-ad-using-the-graph-api-05.png)

## Retrieve the Binary Extension Property Value for a User ##

The code below is an example of how you could read the binary extension property data and de-serialize it back into a VehicleInfo object. Again, the code that is different has been highlighted.

```c#
// Retrieve the extension property value for a user
VehicleInfo vehInfo = null;
var userExtProperty = user.GetExtendedProperties().Where(
    extProp => extProp.Key.Equals(
        vehInfoExtProperty.Name,
        StringComparison.CurrentCultureIgnoreCase)).FirstOrDefault();
if (userExtProperty.Value != null)
{
    using (MemoryStream memoryStream = new MemoryStream((byte[])userExtProperty.Value))
    {
        vehInfo = (VehicleInfo)binaryFormatter.Deserialize(memoryStream);
    }
}
```

## Summary ##

In this pos t I showed you how you can used Azure AD Graph API Directory Schema Extensions to extend the schema in Azure AD. I discussed constraints that you must accept when using this feature and then showed you how you can use the Azure AD Graph Client Library to register extension properties, store, retrieve data using the property, and then remove the extension property. I showed how you can store complex types as a JSON string and also how it can be stored as binary data.

For additional information on this feature I encourage you to look at the Azure AD Graph API Directory Schema Extensions documentation referenced in the References section below. Also, take a look at the post from the Azure AD Graph Team where they demonstrate using extension properties for simple string types.

## References ##

- [Azure AD Graph API Directory Schema Extensions](https://msdn.microsoft.com/library/azure/ad/graph/howto/azure-ad-graph-api-directory-schema-extensions)
- [Azure AD Graph Team Blog](http://blogs.msdn.com/b/aadgraphteam/archive/2014/12/12/announcing-azure-ad-graph-api-client-library-2-0.aspx)

{% if page.comments %}
<div id="disqus_thread"></div>
<script>

/**
*  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
*  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables*/
/*
var disqus_config = function () {
this.page.url = "{{ site.baseurl }}";  // Replace PAGE_URL with your page's canonical URL variable
this.page.identifier = "{{ page.url }}"; // Replace PAGE_IDENTIFIER with your page's unique identifier variable
};
*/
(function() { // DON'T EDIT BELOW THIS LINE
var d = document, s = d.createElement('script');
s.src = 'https://rickrainey.disqus.com/embed.js';
s.setAttribute('data-timestamp', +new Date());
(d.head || d.body).appendChild(s);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
                            
{% endif %}