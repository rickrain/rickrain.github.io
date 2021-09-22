---
title: "It works fine in the Windows Azure Emulator but fails when I publish to Windows Azure"
date: 2013-12-11 16:06:04 -0500
permalink: /:year/:month/:day/:title/
---

Today I ran into one of those situations where my cloud service worked just fine running locally in the Windows Azure Emulator but failed when I published to Windows Azure, resulting in this infamous page being rendered in my browser.

![Server Error in Application](/assets/img/debug-cloud-service-01.png)

It’s tempting to jump right into the various techniques for debugging cloud services.  Especially the latest Remote Debugging feature.  While effective, these techniques can be somewhat heavy handed approaches requiring you to publish a debug build instead of a release build or preconfigure some diagnostic settings.  When I am doing basic acceptance testing, I prefer to do so using a release build.  After all, that’s what I will ultimately deploy to production.  I also tend not to enable any diagnostic settings by default.  Again, not something I would do in production unless necessary.

When troubleshooting an issue it’s generally a good idea to start with the least invasive approach and bring out the big debugging tools only if necessary.   This is especially true if you are ever debugging a production environment.

In this post, I’ll walk through the simple process I took to determine root cause.  The issue is not really important.  Well, maybe it is for some MVC developers out there.  For this post though, it’s primarily about the process.

So, I published my cloud service, consisting of an MVC Web Role and a Web API Web Role, and got the Runtime Error above.

## Enable Remote Desktop on the MVC Role

Thanks to the service management extensions in Windows Azure, I went directly to the Windows Azure Portal.  Then, I went to the CONFIGURE tab for my cloud service and clicked the REMOTE button at the bottom of the screen.

![Remote button](/assets/img/debug-cloud-service-02.png)

In the Configure Cloud Service window, I enabled Remote Desktop (RDP) for just my MVC role.

![Configure Remote Desktop](/assets/img/debug-cloud-service-03.png)

## Remote Desktop into the MVC Instance

After about 3-4 minutes RDP was ready to go.  Next, I went to the **INSTANCES** tab for my cloud service and clicked on the **CONNECT** button at the bottom of the screen.

![Connect button](/assets/img/debug-cloud-service-04.png)

I logged in using my user name and password (from above) and went directly to the **Windows Event Viewer** where I quickly found the ASP.NET event I was looking for.

![Windows Event Viewer](/assets/img/debug-cloud-service-05.png)

The details for this event were all I needed to see.

![Event details](/assets/img/debug-cloud-service-06.png)

So that’s it – I’ve determined root cause in a matter of about 5-6 minutes.  Just to re-deploy a debug build would have taken longer.

## The Fix

So, it turns out that `Microsoft.Web.Infrastructure` is in the Global Assembly Cache (GAC) and that’s where it was loading from when I ran it locally.  This was put there during my installation of Visual Studio.

For this to run in Windows Azure, the fix is simple.  Just add the Microsoft.Web.Infrastructure package using the Nuget Package Manager in Visual Studio.

![Microsoft.Web.Infrastructure Nuget Package](/assets/img/debug-cloud-service-07.png)

Build and re-publish the service and I’m done.

## Post-Mortem Analysis

I’ve published many MVC applications to Windows Azure successfully.  Why was this one different?  Well, I did create this application a little differently than past applications.  For this one, I started with the Empty template and checked the option for MVC folders and core references.

![Visual Studio - Select a template](/assets/img/debug-cloud-service-08.png)

Ironically, the Microsoft.Web.Infrastructure reference is not included in the core references in this scenario even though the Microsoft.AspNet.WebPages depends on it.

![Nuget packages without Microsoft.Web.Infrastructure](/assets/img/debug-cloud-service-09.png)

However, if I choose the MVC template (which most people do most of the time), then that package reference is there by default.

![Nuget packages with Microsoft.Web.Infrastructure](/assets/img/debug-cloud-service-10.png)

## Conclusion

The Windows Azure Platform has a rich set of features and tools for troubleshooting applications.  However, start with the least invasive techniques first and build up.  Sometimes you will get lucky and find your problem in just a few minutes like I did here.

Cheers!

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