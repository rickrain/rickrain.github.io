---
title: "Setting up a Neo4j Server running on Ubuntu in Windows Azure"
date: 2013-05-27 16:06:04 -0500
permalink: /:year/:month/:day/:title/
---

The [VM Depot](http://vmdepot.msopentech.com/List/Index) by Microsoft Open Technologies is a great community resource for finding pre-configured images of various operating systems and software that you can deploy on Azure.  You can also create and share your own images there.

This week I was in need of a graph database running Neo4j.  So, I went to the VM Depot and found this image: **Neo4j Community 1.8 on Ubuntu 12.04 LTS**. I’ve not used any flavor of Unix since college and setting up Neo4j is not something I’ve done before.  So, this was pretty much a complete green field experience for me. This blog captures the steps necessary to set this up based on my experience.

## Import Your Publish-Settings from Azure

Get your **Publish-Settings** data for your subscription imported if you haven’t already. This link will get you going.

https://windows.azure.com/download/publishprofile.aspx

Next, open a **Windows Azure Command Prompt**.

![Azure command prompt](/assets/img/neo4j-01.png)

At the Azure SDK Command Prompt, run the following command to import your publish-settings data.

```
azure account import <publishsettings file>
```

## Create the Virtual Machine

Open a browser and go to the VM Depot.  A search for “neo” returns a couple of options.  I chose this one from Cognosys. 

![Cognosys Neo4j image](/assets/img/neo4j-02.png)

Click the **Deployment Script** link and choose the **region** you want to create the virtual machine in. This will create the script you need to deploy this image. Replacing the **DNS name**, **user**, and **password** with your preferences should be all you need. I added the subscription parameter to my script since I have multiple subscriptions. If you just have one then you don’t need it. This step only took about 30 seconds.

![Command line window](/assets/img/neo4j-03.png)

Go into your Azure portal and you should see your new image there.  It took a minute or two for mine to finish starting up before it reached the “running” status.

![Azure Portal - Virtual machines](/assets/img/neo4j-04.png)

Next, add a public endpoint for port 7474. This is explained in the documentation for this image (although easy to miss if you’re not paying close attention).

![Neo4j on Ubuntu 12.04 LTS](/assets/img/neo4j-05.png)

You can give the endpoint any name you like but the protocol needs to be **TCP** and the port **7474**.

![Azure portal - Virtual machine endpoints](/assets/img/neo4j-06.png)

Now that the virtual machine is up and running, I can connect to it for some final configuration settings.

## Configure the Virtual Machine / Start Neo4j

At the time of this writing, this image does not automatically start the Neo4j server.  Starting the server I found to be a pretty simple task.  However, when Windows Azure decides to recycle your virtual machine (and it will), you will have to connect back to your virtual machine and restart Neo4j.  To mitigate this disruption that my application will experience when this virtual machine is restarted, I configured it to start Neo4j automatically (see below).

**PuTTY** is the tool I used to SSH into the virtual machine.  It can be downloaded from [here](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html).

![PuTTY download link](/assets/img/neo4j-07.png)

Start PuTTY and enter the host name (ie: `<vm-dns-name>.cloudapp.net`) of the virtual machine in **Host Name** field.  Make sure the **Port** is set to **22**, which is the SSH port of the virtual machine.

![PuTTY configuration](/assets/img/neo4j-08.png)

Click **Open** to open a session to the virtual machine. For the login name and password, use the user and password specified in the deployment script (above). After successful login, you land at a prompt as shown here.

![SSH session in VM](/assets/img/neo4j-09.png)

## Start the Neo4j server…

![SSH session - commands to start](/assets/img/neo4j-10.png)

## Configure Virtual Machine to Start Neo4j on Startup…

There is a file named rc.local that my research suggests is the place to put commands to run when a Unix server is started.  This file is located in the **etc** directory.  I used the vi editor to edit this file.

![SSH session - commands to open VI](/assets/img/neo4j-11.png)

In the vi editor, I added the full path to Neo4j and the start command.

![VI updates](/assets/img/neo4j-12.png)

That’s it! Close/Exit the SSH session.

## Test Things Out

To test this setting, I went back to the Windows Azure portal and restarted my virtual machine. Shortly after the machine restarted, I was able to open a browser and access the Neo4j admin portal.

![Neo4j Portal](/assets/img/neo4j-13.png)

## References

http://ss64.com/vi.html

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