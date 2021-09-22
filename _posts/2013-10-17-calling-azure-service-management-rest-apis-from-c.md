---
title: "Calling Azure Service Management REST API's from C#"
date: 2013-10-17 16:06:04 -0500
permalink: /:year/:month/:day/:title/
---

## Introduction
The Management Portal for Windows Azure provides a nice user experience for many service management operations such as creating cloud services, storage accounts, virtual machines, virtual networks, and so on.  If you want to automate such operations, then the Azure PowerShell Cmdlets are a great choice and you can find a plethora of examples on how to use them on my close friend Michael Washam’s blog.  On the other hand, if PowerShell is not your thing and you would prefer a C# approach, then the Windows Azure Service Management REST API’s are available for you.  Using the REST API’s gives you full control, but at the expense of some added complexity.

In this post, I’m going to show how to use a few of these REST API’s in a way that you can re-use – much like you would a client SDK ( hmmm, that would be nice ).  I’ll cover enough to demonstrate use of common HTTP verbs such as `GET`, `POST` and `DELETE` and show how to serialize complex structures the API’s use so you don’t have to create or parse XML.

## Solution Overview

To demonstrate, I’ve limited the scope of my solution to just three C# libraries (DLL’s), two of which use a small part of the Service Management REST API’s for Cloud Service and Data Center Location operations.

![Visual Studio Projects](/assets/img/service-mgmt-rest-api-01.png)

**ServiceManagement** is the base library for the two other libraries, providing basic requirements for any Service Manage REST API.

![ServiceManagement.dll](/assets/img/service-mgmt-rest-api-02.png)

As you can see, I also created Unit Test projects for each of these libraries so you can see how the different libraries might be used in client code.   In similar fashion, the **ServiceManagement.Tests** project is the base for the other two test projects.  Later, I will cover how to run the tests.

![ServiceManagement.Tests.dll](/assets/img/service-mgmt-rest-api-03.png)

## Base Service Management

The **ServiceManagement** project defines a base class, BaseApi, that the other two projects ( and potentially future projects ) derive from.  It encapsulates some basic things that all the REST API’s need such as a valid Azure subscription, a base URI for web requests, and an [HttpClient](https://docs.microsoft.com/en-us/previous-versions/visualstudio/hh193681(v=vs.118)?redirectedfrom=MSDN) that already has the Azure subscription’s management certificate added to the client certificates collection for authentication.  It does a few other things, but these are the main parts.

```csharp
public class BaseApi
{
    private AzureSubscription azureSubscription = null;

    public BaseApi(AzureSubscription Subscription)
    {
        if (Subscription == null)
        {
            throw new ArgumentNullException(
                "Subscription", "Subscription parameter cannot be null.");
        }

        this.azureSubscription = Subscription;
    }

    // Base Uri for all Service Management APIs
    public virtual Uri ServiceManagementUri
    {
        get
        {
            return new Uri(
                string.Format("https://management.core.windows.net/{0}",
                azureSubscription.SubscriptionId));
        }
    }

    // Returns a WebRequestHandler with the AzureSubscriptions
    // management certificate already in the client certificates collection.
    public WebRequestHandler RequestHandler
    {
        get
        {
            var handler = new WebRequestHandler();
            handler.ClientCertificates.Add(
                this.azureSubscription.ManagementCertificate);
            return handler;
        }
    }

    // Returns an HttpClient instance using the RequestHandler for this instance.
    // Sets the base address and common headers for all services APIs.
    // Sets up the Accept header for xml format.
    public HttpClient HttpClientInstance
    {
        get
        {
            var httpClient = new HttpClient(this.RequestHandler);
            httpClient.BaseAddress = ServiceManagementUri;
            httpClient.DefaultRequestHeaders.Add("x-ms-version", "2013-03-01");
            httpClient.DefaultRequestHeaders.Accept.Add(
                new MediaTypeWithQualityHeaderValue("application/xml"));

            return httpClient;
        }
    }
}
```

This project also defines an **AzureSubscription** class that holds the Azure Subscription Id and associated management certificate for the subscription.  An instance of this class is required to instantiate **BaseApi** or any of it’s derived classes.  To instantiate **AzureSubscription** you must provide a Subscription ID and Certificate Thumbprint.

```csharp
public class AzureSubscription
{
    private string subscriptionId = null;
    private X509Certificate2 certificate = null;

    public AzureSubscription(string SubscriptionId, string CertificateThumbprint)
    {
        if (string.IsNullOrEmpty(SubscriptionId))
        {
            throw new ArgumentNullException("SubscriptionId is null or empty.");
        }

        this.subscriptionId = SubscriptionId;
        this.certificate = GetCertificate(CertificateThumbprint);
    }

    public string SubscriptionId { get { return this.subscriptionId; } }

    public X509Certificate2 ManagementCertificate { get { return certificate; } }

    // Looks for the certificate by thumbprint in the "My" certificate store.
    private X509Certificate2 GetCertificate(string thumbprint)
    {
        List<StoreLocation> locations = new List<StoreLocation> { 
            StoreLocation.CurrentUser, 
            StoreLocation.LocalMachine };

        foreach (var location in locations)
        {
            X509Store store = new X509Store("My", location);
            try
            {
                store.Open(OpenFlags.ReadOnly | OpenFlags.OpenExistingOnly);
                X509Certificate2Collection certificates = store.Certificates.Find(
                  X509FindType.FindByThumbprint, thumbprint, false);
                if (certificates.Count == 1)
                {
                    return certificates[0];
                }
            }
            finally
            {
                store.Close();
            }
        }

        throw new ArgumentException(string.Format(
          "A certificate with thumbprint '{0}' could not be located.",
          thumbprint));
    }
}
```

With this base library in place, I’ll transition now to implementing the libraries that will call the various Service Management REST API’s.

## Operations on Data Center Locations

The [Operations on Locations](https://docs.microsoft.com/en-us/previous-versions/azure/reference/gg441299(v=azure.100)?redirectedfrom=MSDN) is very simple with just one operation, [List Locations](https://docs.microsoft.com/en-us/previous-versions/azure/reference/gg441293(v=azure.100)?redirectedfrom=MSDN).  It returns a list of data centers available to your subscription and a list of available services for each data center.  The request is a simple GET request that returns XML that could be serialized into a complex structure.

### Data Center Locations Model

To support serialization into a structure I can code against, I defined three classes according to the REST API documentation for this operation.

![Locations - DataCenterLocation - Services](/assets/img/service-mgmt-rest-api-04.png)

**Locations** contains a collection of **DataCenterLocation**’s and **DataCenterLocation** contains a collection of available **Services** at that data center.  Pretty straight forward.

What you have to be careful about are how the classes are named, the namespace, and the order of properties.  If any of this is off then the data won’t serialize into the structure, even though it was returned in the underlying XML.  This is where the [DataContract](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.serialization.datacontractattribute?redirectedfrom=MSDN&view=net-5.0) and [DataMember](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.serialization.datamemberattribute?redirectedfrom=MSDN&view=net-5.0) attributes come in handy.  I decorated the classes with the necessary attributes according to the documentation so that the response will serialize correctly.

```csharp
[DataContract(
    Namespace = "http://schemas.microsoft.com/windowsazure",
    Name="Location")]
public class DataCenterLocation
{
    [DataMember(Order = 1)]
    public string Name { get; set; }

    [DataMember(Order = 2)]
    public string DisplayName { get; set; }

    [DataMember(Order = 3)]
    public Services AvailableServices { get; set; }
}

[CollectionDataContract(
    Namespace = "http://schemas.microsoft.com/windowsazure")]
public class Locations : Collection<DataCenterLocation>
{
}

[CollectionDataContract(
    Namespace = "http://schemas.microsoft.com/windowsazure",
    ItemName = "AvailableService")]
public class Services : Collection<string>
{
}
```

### Data Center Locations API

Now that the model is defined, the implementation of the **LocationsApi** is quick and to the point.

```csharp
public class LocationsApi : BaseApi
{
    public LocationsApi(AzureSubscription Subscription) 
        : base(Subscription)
    { }

    public override Uri ServiceManagementUri
    {
        get
        {
            return new Uri(string.Format("{0}/locations",
                base.ServiceManagementUri));
        }
    }

    public Locations.Models.Locations List()
    {
        // Invoke REST API
        HttpResponseMessage response = this.HttpClientInstance.GetAsync("").Result;
        response.EnsureSuccessStatusCode();

        List<MediaTypeFormatter> formatters = new List<MediaTypeFormatter>(){
            new XmlMediaTypeFormatter()
        };

        return response.Content.ReadAsAsync<Locations.Models.Locations>(formatters).Result;
    }
}
```
Although there’s not much to it, I’ll explain the main points.

- The implementation of **LocationsApi** derives from **BaseApi** which I covered earlier.  So, it gets basic support needed for Service Management REST API’s.
- The **ServiceManagementUri** is overridden to return the correct Uri for operations specific to locations.
- The one operation defined, which is **List**, uses the HttpClient instance from the base class to invoke the REST API and return back the response as an instance of **Locations** using the **ReadAsAsync** extension method.

### Using the LocationsApi in Client Code

This is an example of client code using the **LocationsApi** to get a list of data center locations.

```csharp
locationsApi = new LocationsApi(this.subscription);
var locations = this.locationsApi.List();

// Print out the locations collection.
foreach (DataCenterLocation loc in locations)
{

    Console.WriteLine("Name: {0}", loc.Name);
    Console.WriteLine("Display Name: {0}", loc.DisplayName);
    Console.WriteLine("Services:");
    foreach (string service in loc.AvailableServices)
    {
        Console.WriteLine("\t{0}", service);
    }
    Console.WriteLine();
}
```

And here is the output for one of the six data centers returned for my subscription.

![Services API output](/assets/img/service-mgmt-rest-api-05.png)

Personally, I like this a lot better than parsing XML.

## Operations on Cloud Services

The [Operations on Cloud Services](https://docs.microsoft.com/en-us/previous-versions/azure/reference/ee460812(v=azure.100)?redirectedfrom=MSDN) is a little more involved and has several operations.  For the sake of this blog, I’m just going to focus on two operations; [Create Cloud Service](https://docs.microsoft.com/en-us/previous-versions/azure/reference/gg441304(v=azure.100)?redirectedfrom=MSDN) and [Delete Cloud Service](https://docs.microsoft.com/en-us/previous-versions/azure/reference/gg441305(v=azure.100)?redirectedfrom=MSDN).  I chose these because they provide small examples of API’s using `POST` and `DELETE` http verbs.

### Create Cloud Service Model

This time, I’m defining a model that I can POST to the REST API’s to create a cloud service.  So, continuing in similar fashion, I defined a class, CreateHostedService, to support the minimum requirements needed to create a cloud service.  Notice that I said “minimum”.  These API’s can get extremely complex quickly.  For the sake of demonstration, I’ve limited my code to handle just these minimum requirements.  Feel free to contribute to my project on github if you want to build this out further.

```csharp
[DataContract(Namespace = "http://schemas.microsoft.com/windowsazure")]
public class CreateHostedService
{
    [DataMember(Order = 1, IsRequired=true)]
    public string ServiceName { get; set; }

    [DataMember(Order = 2, IsRequired = true)]
    public string Label { get; set; }

    [DataMember(Order = 3, IsRequired = true)]
    public string Location { get; set; }
}
```

### Cloud Services API

Here again, a pretty trivial implementation with everything else in place.

```csharp
public class CloudServicesApi : BaseApi
{
    public CloudServicesApi(AzureSubscription Subscription)
        : base(Subscription)
    { }

    public override Uri ServiceManagementUri
    {
        get
        {
            return new Uri(string.Format("{0}/services/hostedservices",
                base.ServiceManagementUri));
        }
    }

    public Uri Create(string ServiceName, string Location)
    {
        // Create the request body
        byte[] serviceNameBytes = System.Text.Encoding.UTF8.GetBytes(ServiceName);
        CreateHostedService requestBody = new CreateHostedService()
        {
            ServiceName = ServiceName,
            Label = Convert.ToBase64String(serviceNameBytes),
            Location = Location
        };

        // Invoke REST API call
        HttpResponseMessage response = 
            this.HttpClientInstance.PostAsXmlAsync<CreateHostedService>("", requestBody).Result;
        response.EnsureSuccessStatusCode();

        return response.Headers.Location;
    }

    public void Delete(string ServiceName)
    {
        // Invoke REST API call
        HttpResponseMessage response = this.HttpClientInstance.DeleteAsync(
            string.Format("{0}/{1}", this.ServiceManagementUri, ServiceName)).Result;
        response.EnsureSuccessStatusCode();
    }
}
```

Like before, here are the main observations in this class library.

- The implementation of **CloudServicesApi** derives from **BaseApi** which I covered earlier.  So, it gets basic support needed for Service Management REST API’s.
- The **ServiceManagementUri** is overridden to return the correct Uri for these operations specific to cloud services.
- The **Create** method constructs a **CreateHostedService** instance, populates it with required data to create the service. Then, it uses the HttpClient instance from the base class to invoke the REST API, passing the **CreateHostedService** instance in the request body using the **PostAsAsync** extension method.
- The **Delete** method is about as simple as it gets. Just use **DeleteAsync** to issue an http `DELETE` to the appropriate Uri for the cloud service to be deleted.

### Using the CloudServicesApi in Client Code

This is an example of client code using the **CloudServicesApi** to create a new cloud service and then delete it.

```csharp
// Set name of new cloud service.
var name = Guid.NewGuid().ToString();

// Set location of new cloud service to first US location 
// found available in the current subscription.
var locationsApi = new Locations.LocationsApi(this.subscription);
var locations = locationsApi.List();
string location = null;
foreach (DataCenterLocation dcLocation in locations)
{
    if (dcLocation.Name.EndsWith(" US"))
    {
        location = dcLocation.Name;
        break;
    }
}
// Create a cloud service.
Console.WriteLine("Creating cloud service '{0}' in data center '{1}'.", 
    name, location);
cloudServicesApi = new CloudServicesApi(this.subscription);
var cloudServiceUri = this.cloudServicesApi.Create(name, location);

Console.WriteLine("Cloud service created.");
Console.WriteLine(cloudServiceUri);

// Delete the same cloud service.
this.cloudServicesApi.Delete(name);

Console.WriteLine("Cloud service deleted.");
```

Notice in this code that I’m also making use of the **LocationsApi** to pick a data center that’s available for my subscription and in the US.

And here is the output.  I know the name is not very user friendly, but at least it’s unique!

![Create serivce output](/assets/img/service-mgmt-rest-api-06.png)

So, there you have it. A few examples using Service Management REST API’s from C# using models to serialize and de-serialize complex structures. Extending this to support more complex API’s is largely going to come down to defining the models. The code to call the REST API’s is pretty short when you have a model to pass to and from the library.

I’m going to wrap things up now with a quick review of the Test Projects.

## Running the Test Projects

If you know me or have read previous posts, you know I don’t like writing client applications just for the sake of testing other code.  So, I’ll try and use test projects when possible to avoid this.  Plus, it’s just good development practice to do so.

### Configure the Unit Tests

All that is needed to configure this for your environment is to replace the subscriptionId and certThumbprint in the BaseApiTest.cs file.

![Initialize method](/assets/img/service-mgmt-rest-api-07.png)

As I indicated earlier, all the test projects derive from this **BaseApiTest** class (just like the REST API’s projects). So, once you set this all the other test projects are good to go.

You can find these values in your Windows Azure Portal in the Settings tab.  The management certificate must also be installed in your personal certificates store in either CurrentUser or LocalMachine.

### Run the Unit Tests

First, build the solution.

Next, to run the Unit Tests, select **TEST | Windows | Test Explorer** from Visual Studio 2013’s menu. This will open the Test Explorer window where you should find all the tests I’ve provided so far. Click the Run All link to run all the tests or right-click on a single test to run just that test.

![Test Explorer](/assets/img/service-mgmt-rest-api-08.png)

### View the Output of the Unit Tests

Most of the tests I’ve written just write data to the Console as you saw previously.  You can view this output by clicking on the test you’re interested in and looking for the **Output** link.

![Link to output](/assets/img/service-mgmt-rest-api-09.png)

If you don’t see an **Output** link for a test, it’s just because I didn’t write anything to the console in the test code.

## Conclusion

In this post I showed how to program against the Service Management REST API’s using C#. I gave examples of how you can build models to pass data to/from the API’s that will make programming against them in client code simpler and more intuitive.

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