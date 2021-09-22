---
title: "Author an Azure Resource Manager Template using Visual Studio"
date: 2016-02-22 16:06:04 -0500
permalink: /:year/:month/:day/:title/
---

**Updated 3/19/2016**

Last month I wrote an [introduction to Azure Resource Manager (ARM)](http://rickrainey.com/2016/01/19/an-introduction-to-the-azure-resource-manager-arm/) that talked about how ARM is different from the traditional Azure service management API’s (ASM) for deploying Azure resources. In that post I discussed some benefits of ARM and mentioned tools you can use to author your own ARM templates. In this post I will discuss how to use Visual Studio to author a very simple ARM template from scratch. If you’re an IT Professional and think that because I just said “Visual Studio” that this won’t apply to you – please do not stop reading. I believe you will find that Visual Studio with the Azure Tools installed delivers the ultimate experience for authoring ARM templates.

For this post I’m using the following software versions:

- Visual Studio 2015 w/Update 1
- Azure Tools SDK v2.8.2

## Create an Azure Resource Group Project ##

When you first create an Azure Resource Group Project in Visual Studio you are presented with a list of common templates you can use to start defining your desired infrastructure. For example, if you wanted a set of Virtual Machines that could be used to run a scalable web server workload, then you could start with the **Windows Server Virtual Machines with Load Balancer** template to describe the initial infrastructure resources such as the Virtual Network, Storage Account, Load Balancer, Availability Set, Network Interfaces, and Virtual Machines. If you have ever provisioned an environment like this on-premises or even in Azure using the old Azure Service Management (ASM) API’s, you will quickly appreciate the power a template like this delivers.

![VM with Load Balancer Tempalte](/assets/img/author-azure-resource manager-template-using-visual-studio-01.png)

While this template is extremely useful for what it can do, it’s probably not a good choice if you’re just starting to learn how to author ARM templates, which is what this post is about. So instead, I’m going to choose the **Blank Template** (shown below) and incrementally add the resources needed to describe a single Virtual Machine.

![ARM Blank Template](/assets/img/author-azure-resource manager-template-using-visual-studio-02.png)

I believe this approach makes it easier to understand the basic concepts of ARM templates. Afterwards, you will be able to parse your way through and edit any template such as the one I referenced above.

## Explore the Project Structure ##

Choosing the **Blank Template** results in a project that contains everything you need to begin authoring and eventually deploy your template. As shown below, the project structure contains three folders; *Scripts, Templates*, and *Tools*.

![ARM Project Structure](/assets/img/author-azure-resource manager-template-using-visual-studio-03.png)

The *Scripts* folder contains the PowerShell script that is used to deploy the environment you describe in the project. This is a fully functional script that essentially requires no editing. It just works! However, if you have a need to modify the script to support some advanced deployment scenarios you can do so.

The *Templates* folder contains the JSON files that describe the environment you want to create. The *azuredeploy.json* file is where you describe the resources you want in your environment, such as the storage account, virtual machine, and other required resources. The *azuredeploy.parameters.json* file is where you can provide parameter values that you want passed into the template during deployment, such as the type of storage account, size of the virtual machine, credentials to use to sign into the virtual machine, and so on.

The *Tools* folder contains AzCopy.exe and is used in scenarios where your deployment template needs to copy files to a storage account used for deploying application and configuration files into a resource. For a virtual machine, this typically would be Desired State Configuration (DSC) and DSC resources that you want pushed into the virtual machine after the virtual machine is provisioned to do things like configure Active Directory or IIS Web Server roles in the virtual machine. The script in the *Scripts* folder uses AzCopy in the *Tools* folder to copy these kinds of artifacts to a storage account so that ARM can then copy them from storage into the virtual machine after the instance is fully provisioned.

## Resource Deployment Template ##

Since I chose the **Blank Template**, the *azuredeploy.json* file contains a skeleton JSON document. Notice the *parameters, variables, resources*, and *outputs* objects. This is where the bulk of your authoring will occur as you start adding resources to the template.

```json
{
 "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
 "contentVersion": "1.0.0.0",
 "parameters": {
 },
 "variables": {
 },
 "resources": [
 ],
 "outputs": {
 }
}
```

The *parameters* object is where you define parameters for your resources to be used during deployment of the template. For example, if you define a Storage Account resource in your template, then you may want to parameterize the type (Standard or Premium) of storage account.

The *variables* object is where you define variables that are referenced in your resource definitions. Using the Storage Account as an example again, you should have a variable for the name of the storage account that you can reference throughout the template rather than hard-coding the same value throughout the template.

The *resources* array is where you describe all the resources for your environment such as a Storage Account Resource, Virtual Network Resource, Virtual Machine Resource, and so on.

The *outputs* object is where you can optionally return values from your deployment. For example, you could return the public IP address of a virtual machine after it has been provisioned.

### JSON Outline Window ###

When you open the *azuredeploy.json* file, the JSON Outline Window will open to the left of your editor window as shown here. If it is not open in your environment, you can open it manually by selecting **View -> Other Windows -> JSON Outline**.

![ARM Project JSON Outline](/assets/img/author-azure-resource manager-template-using-visual-studio-04.png)

The JSON Outline window provides a visual representation of the parameters, variables, resources, and outputs objects defined in *azuredeploy.json*. This is extremely helpful as you add resources to the file and provides a way to easily navigate the JSON file rather than scrolling through the JSON editor. As you are about to see, it also provides a nice UI experience to add resources to your template.

## Add a Storage Account Resource ##

The first resource I will add is a Storage Account resource. The reason I chose this is because it is small and is needed to store the VHD’s (virtual hard disks) of the virtual machine. To add a Storage Account resource, right-click on **resources** in the JSON Outline Windows and select **Add Resource**. In the Add Resource window that opens, click on the **Storage Account** resource, provide a **Name** for the storage account, and click **Add** as shown here.

![ARM Project - Add storage account](/assets/img/author-azure-resource manager-template-using-visual-studio-05.png)

The Storage Account resource is added to the *azuredeploy.json* file in the resources section. Additionally, a parameter for the type of storage account was added, and a variable for the name of the storage account is added. Notice also that the JSON Outline window gets updated. You can click on the parameter, variable, or resource in the JSON Outline window to quickly navigate to that element in the JSON file.

![Azure Deploy - Storage Resource](/assets/img/author-azure-resource manager-template-using-visual-studio-06.png)

The *name* property uses the **variables()** function to reference the value of the variable named MyStorageName. If you click on MyStorageName in the variables section of the JSON Outline window you will see the value of the variable is the name you gave the resource when adding it to the template concatenated with a uniquely generated string as shown here.

```json
“MyStorageName”: “[concat(‘mystorage’, uniqueString(resourceGroup().id))]”
```

> The value of the *MyStorageName* variable is generated using three ARM template functions; **concat()**, **uniqueString()**, and **resourceGroup()**. The **resourceGroup()** function is a function that returns a few properties about the resource group during deployment, such as the id, location, and name of the resource group. The id property is a long path (string) that includes your Azure subscription ID and resource group name. The **uniqueString()** function takes the value from **resourceGroup().id** (a string) and generates a 64-bit hash of it to generate a string that is unique to the resource group. The reason the template does this is to avoid DNS name conflicts for the storage account. Storage Accounts are publicly addressable at a URL unique to the storage account. Therefore, the name of the storage account needs to be unique to avoid DNS name conflicts. Finally, the concat() function takes the results of the **uniqueString()** function and concatenates it with the constant string “mystorage” to generate a string that looks something like “mystoragetj2fcadfgy6jw”.

The *type* property identifies the type of the resource which in this case is a Storage Account. The value for the type follows a naming convention of <resource provider namespace>/<resource type>. When this template is deployed, ARM will use this value to invoke the resource provider at “Microsoft.Storage” to create the resource type “storageAccounts”.

The *location* property tells ARM where to provision this resource. Generally speaking, your resources will be deployed in the same location as the resource group they will be contained in. So, the template uses the **resourceGroup()** function to return the **location** property of the resource group. So, for example, if at deployment time you specify “West US” for the region of your resource group, then your resources will also be deployed in “West US” because they will all be using the **resourceGroup()** function (by default).

The *apiVersion* property tells ARM which version of the resource provider to use. The resource provider version changes as new features and resource types are introduced. Hence the reason for needing to specify this value.

The *properties* object is where resource-specific properties are set. For a Storage Account, the accountType is required by the resource provider to identify the type of storage account to provision. The template takes this value as a parameter and then uses the parameters() function to retrieve the value specified during deployment.

## Add a Virtual Network Resource ##

A Virtual Machine in Azure requires a Virtual Network for the instance to be deployed in. You typically would probably be referencing an existing virtual network. However, for this post we’ll create a new one so we can further explore the basics of ARM templates.

Following the same process as we did to add a Storage Account resource, add a Virtual Network resource as shown here.

![ARM Virtual Network Resource](/assets/img/author-azure-resource manager-template-using-visual-studio-07.png)

The Virtual Network resource is added to the *azuredeploy.json* file in the resources section and some variables are also added that are referenced in the virtual network resource definition. Notice the type property is set to “Microsoft.Network/virtualNetworks”, which is telling ARM which resource provider and resource type to provision. Also, notice that the properties object has virtual network-specific properties defined, such as *addressSpace* (the address space of the virtual network) and a collection of *subnets* describing how to carve up the address space.

![ARM Virtual Network Resource](/assets/img/author-azure-resource manager-template-using-visual-studio-08.png)

## Add a Virtual Machine Resource ##

Now that we have met the minimum resource requirements for a Virtual Machine (ie: Storage Account and Virtual Network), we can add a Virtual Machine resource to the template. Following the same process as before, add a Windows Virtual Machine resource as shown below. The dialog will prompt you to choose a Storage Account and Virtual Network (and subnet) for the Virtual Machine.

![ARM Virtual Network Resource](/assets/img/author-azure-resource manager-template-using-visual-studio-09.png)

The Virtual Machine resource is added to the *azuredeploy.json* file in the resources section. You will also see that a virtual Network Interface Card (NIC) resource is also added. Selecting the NIC resource in the JSON Outline Window will navigate you directly to the resource as shown here.

![ARM Virtual Network Resource](/assets/img/author-azure-resource manager-template-using-visual-studio-10.png)

Notice that the resource type “networkInterfaces” is a type known by the same resource provider responsible for the Virtual Network resource, which is at the namespace “Microsoft.Network”. Also notice that for this resource the dependsOn property has a reference to another resource, which is the Virtual Network resource. When this template is deployed, this tells ARM that before it can create the virtual NIC resource, it needs to first create the Virtual Network resource. The reason this dependency exists is because in the ipConfiguration properties, this NIC binds a resource (the VM – as you will see soon) to a subnet in the Virtual Network resource.

Moving down to the Virtual Machine resource, notice the resource provider and resource type is “Microsoft.Compute/virtualMachines”. Also notice that this resource dependsOn the Storage Account resource (to store it’s VHD’s) and the virtual NIC resource (to bind it to the VNET) as shown here.

![ARM Virtual Network Prefix](/assets/img/author-azure-resource manager-template-using-visual-studio-11.png)

The Virtual Machine resource is added to the *azuredeploy.json* file in the resources section. You will also see that a virtual Network Interface Card (NIC) resource is also added. Selecting the NIC resource in the JSON Outline Window will navigate you directly to the resource as shown here.

![VS Add Resource](/assets/img/author-azure-resource manager-template-using-visual-studio-10.png)

Notice that the resource type “networkInterfaces” is a type known by the same resource provider responsible for the Virtual Network resource, which is at the namespace “Microsoft.Network”. Also notice that for this resource the dependsOn property has a reference to another resource, which is the Virtual Network resource. When this template is deployed, this tells ARM that before it can create the virtual NIC resource, it needs to first create the Virtual Network resource. The reason this dependency exists is because in the *ipConfiguration* properties, this NIC binds a resource (the VM – as you will see soon) to a subnet in the Virtual Network resource.

Moving down to the Virtual Machine resource, notice the resource provider and resource type is “Microsoft.Compute/virtualMachines”. Also notice that this resource *dependsOn* the Storage Account resource (to store it’s VHD’s) and the virtual NIC resource (to bind it to the VNET) as shown here.

![VM DependsOn](/assets/img/author-azure-resource manager-template-using-visual-studio-11.png)

## Add a Public IP Address Resource ##

Technically, we’ve added all the resources necessary to deploy a virtual machine using this template. If you want to connect to the virtual machine after this template is deployed, then you will have to configure a VPN connection to the virtual network so you can communicate to it. That’s actually a best practice for most production scenarios. However, when you’re just learning, sometimes we take shorter paths to our destination. For this post, I’ll add the Public IP Address resource which will enable you to connect to the virtual machine using Remote Desktop Protocol (RDP) over a public IP Address.

Following the same process, add a Public IP Address resource as shown here.

![Public IP Address](/assets/img/author-azure-resource manager-template-using-visual-studio-12.png)

Just like before, the Public IP Address resource is added to the *azuredeploy.json* file in the resources section as shown below.

![Public IP Address Resource Definition](/assets/img/author-azure-resource manager-template-using-visual-studio-13.png)

As you may have expected, the resource type “publicIPAddresses” is a type known by the resource provider at namespace “Microsoft.Network”. Hopefully you are recognizing that a resource provider can (and usually does) support multiple resource types. You probably also expected the virtual NIC to be in the *dependsOn* section for this resource. After all, the Add Resource dialog indicated it was a required resource and had you select the NIC resource when you added it. So, why then is it not there?!?!? The answer is that the Public IP Address resource is added as a dependent resource for the virtual NIC as shown below.

![Public IP Address Resource Dependency](/assets/img/author-azure-resource manager-template-using-visual-studio-14.png)

If you look down a little further in the NIC properties, you will see the Public IP Address resource referenced in the *publicIPAddress* property of the *ipConfigurations* object.

## Add some Outputs ##

When you deploy a template you may want to output values from the deployment. This is optional. For a deployment such as this (a VM) you may want to output the fully qualified DNS name for the virtual machine so you can RDP into it. For the template described in this post the outputs section could look like this.

```json
"outputs": {
  "dnsName": {
    "type": "string",
    "value": "[reference(variables('MyPublicIPName')).dnsSettings.fqdn]"
  }
}
```

For the output I’m using the **reference()** function to reference the Public IP Address resource that gets deployed. When this template is deployed, the **reference()** function can be used to output the fully qualified domain name from the dnsSettings property of the resource.

## Summary ##

In this post I walked through the process of creating a new Azure Resource Group Project using the Blank Template. Next, I added resources individually to support a single virtual machine deployment. Along the way we looked at the components of a Resource Group Project, how ARM templates are structured, and how resource dependencies are defined. We also saw examples of some of the many [ARM Template Functions](https://azure.microsoft.com/en-us/documentation/articles/resource-group-template-functions/) that you can use when authoring templates and how to output information from a deployment when an ARM template is deployed. The completed solution created for this blog post is [available on my GitHub account](https://github.com/rickrain/ARM-Basics).

Next time I’ll show a few ways you can deploy this ARM template.

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