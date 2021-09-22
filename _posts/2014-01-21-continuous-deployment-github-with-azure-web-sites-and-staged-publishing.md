---
title: "Continuous Deployment (GitHub) with Azure Web Sites and Staged Publishing"
date: 2014-01-21 16:06:04 -0500
permalink: /:year/:month/:day/:title/
---

With Windows Azure Web Sites you can setup continuous deployment (CD) to publish your site directly from source control as changes are checked in.  This is a fantastic automation feature that can be leveraged from a range of source control tools such as Visual Studio Online and others (GitHub, BitBucket, DropBox, CodePlex, or Mercurial) .

With the recent preview release of Staged Publishing for Windows Azure Web Sites, you can now configure CD to deploy directly to a staging environment where you can do final testing before swapping to your production environment.  If you’re familiar with Cloud Services, then this is similar to the staging and production environments of that execution model.

Together, continuous deployment and staged publishing with Windows Azure Web Sites provides a powerful solution for getting quality changes and new features to market quickly.  In this post I’m going to walk through setting up continuous deployment from a GitHub repository to a Web Site with Staged Publishing enabled.

## Create an Azure Web Site

There are many ways to do this but I will start from Visual Studio’s Server Explorer by right-clicking on Web Sites and selecting Add New Site .

![Server Explorer](/assets/img/cd-github-01.png)

Next, I’ll give my site a name and click Create.

![Create Site](/assets/img/cd-github-02.png)

## Set Web Site Mode to Standard

Before you can enable Staged Publishing for a web site, you first need to set your web site to Standard mode (previously known as Reserved) if it is not already.  Staged Publishing is only available in Standard mode.  This can be set in the Windows Azure Portal on the SCALE tab for the web site.

![Web Site Mode](/assets/img/cd-github-03.png)

## Enable Staged Publishing

To enable Staged Publishing for the web site, click on the Enable staged publishing link in the quick glance section of the DASHBOARD for the site.

![Quick Glance](/assets/img/cd-github-04.png)

When you do this, Windows Azure essentially creates a new web site that will serve as the staging environment.  You will see the staging site under the web site with the same name, but with “(staging)” appended.  Notice also the URL of the staging site.  This is the URL you will use to test your web site prior to swapping it to production.

![Web Site List](/assets/img/cd-github-05.png)

## Setup Publishing from Source Control

For this step, I previously created a new repository in my GitHub account.  The repository is empty (for now), but it has been created.

Next, I went to the DASHBOARD for the staging environment and clicked on the link to Set up deployment from source control .

![Quick Glance](/assets/img/cd-github-06.png)

I selected GitHub from the list of source control providers and then selected my repository I previously created.  Note: Windows Azure has to authenticate you to access your repository so you may be challenged for your GitHub credentials at this step if it cannot authenticate you.

![Choose Repo](/assets/img/cd-github-07.png)

The completes the steps necessary to setup CD to a web site’s staged environment.  In a moment, I’ll create a web site to demonstrate these features working together.  However, before I do, I need to step through a short little work around that at the time of this writing is necessary when working with the preview bits for Staged Publishing.

## Staged Publishing Preview Workaround

This section is a work-around for an issue with continuous deployment to web sites that have Staged Publishing (preview) enabled.  A big THANK YOU to David Ebbo for this work-around.

I went to the DASHBOARD for the production environment and clicked the link to Set up deployment from source control .  Yes, it’s the exact same step as before, but this time for the production environment .

Next, I selected GitHub from the list of source control providers and then selected my repository I had previously created (it’s the exact same repository).

### Remove the GitHub Service Hook Pointing at the Production Environment

For this, I navigated my browser to my GitHub repository and clicked on the **Settings** icon on the lower-right corner of the page.

Next, on the upper-left corner of the page, I clicked on **Service Hooks**.  Notice there are two WebHook URL’s indicated.  That’s because I setup publishing from source control on both the staging and production environments.

Click on the **WebHook URLs (2)** link to see the two URL’s.  Locate the **production** URL (the one that is not the staging URL) and click the **remove** link.

Finally, click the **Update settings** button to save the changes.

You may be asking yourself, why setup publishing from source control to the production environment if all you do is then delete the service hook for it in GitHub?  I wondered that too.  By doing this, there apparently are some settings in Windows Azure Web Sites that gets triggered/applied as a result of initially setting up publishing from source control.  I don’t know the details.  However, I can tell you what happens if you don’t do this.

If you don’t apply this work around, then this is the experience you would have.

1. Assume that you’ve deployed v1.0 to staging and that production is empty.
2. Now, swap the two environments.  After the swap, staging is empty and production has v1.0.
3. Push v1.1 to GitHub.  The CD from GitHub won’t occur.  Why?  Because the service hook is referencing staging and you swapped the staging and production environments.  So, whatever setting is needed for CD to occur is currently in the production environment.
4. Now, swap the two environments again (in other words – back to their original state) so that staging is back to v1.0 and production is empty.
5. Push v1.2 to GitHub.  This time CD from GitHub will occur.

So, basically, you have to swap the environments back to their original state for CD to work.  Of course, that defeats the purpose of Staged Publishing.  Hence, the reason for the work around.

I look forward to the day when I can delete this section from the blog.  Until then, doing these few extra steps will deliver the CD experience you would expect when your site has Staged Publishing enabled.

## Let the Continuous Deployment Cycle Begin

Now that I’ve got CD all wired up to my GitHub repository and Staged Publishing enabled on my web site, it’s time to take this for a spin.

In Visual Studio, I cloned my GitHub repository so that I have it locally on my machine.  Next, I created an ASP.NET Web Application project using the default project that the Visual Studio template creates and specified the path of my local repository to store the project in.  If this is not something you’re familiar with, check out my post Visual Studio and GitHub: The Basics of Working with Existing Repositories to see essentially the same steps.  The only change I made to the project is a small update to the index.cshtml file to show a version number for the purpose of this blog.

### Push v1.0 to GitHub

In Team Explorer, I added a description (just the version number) and clicked the **Commit** button to commit the change to my local copy of the repository.

![Team Explorer](/assets/img/cd-github-08.png)

Next, I pushed the change to my remote GitHub repository.  It’s this step that kicks off the continuous deployment from my GitHub repository to my Azure Web Site.

![Team Explorer](/assets/img/cd-github-09.png)

In the Windows Azure Portal, I navigated to the DEPLOYMENTS page for the staging environment and I can see that v1.0 was successfully deployed.

![GitHub Staging](/assets/img/cd-github-10.png)

Clicking on the BROWSE button at the bottom of this page, I can see v1.0 of the application running in the browser.  Notice the URL is the staging URL.

![App 1.0](/assets/img/cd-github-11.png)

Navigating to the production URL, I see the standard empty site page.

![App 1.0](/assets/img/cd-github-12.png)

### Swap the Staging and Production Environments

In the Windows Azure Portal, the DASHBOARD for either staging or production now has a SWAP button at the bottom of the screen.  Clicking this raises a dialog to confirm I want to perform the swap and some helpful information about what settings change and what stay the same.

![Swap](/assets/img/cd-github-13.png)

After the swap is complete, which takes just a second or two, I refreshed my browser where my application is running.  Now v1.0 is in the production environment.

![App 1.0](/assets/img/cd-github-14.png)

And as expected, the empty site is now in the staging environment.

![App 1.0](/assets/img/cd-github-15.png)

### Push v1.1 to GitHub

To further illustrate these great features and to point out a couple of observations (which I’ll get to later), I changed the version on the index.cshtml page to 1.1 and committed the change.  Just like before.

![Team Explorer](/assets/img/cd-github-16.png)

And then pushed the change to my remote GitHub repository which again kicks off the continuous deployment from my GitHub repository to my Azure Web Site.

![Team Explorer](/assets/img/cd-github-17.png)

In the Windows Azure Portal, I navigated to the DEPLOYMENTS page for the staging environment and I can see that v1.1 was successfully deployed.  Notice that v1.0 is not in the deployment history.

![Portal - Deployments](/assets/img/cd-github-18.png)

However, if I go over and check the DEPLOYMENTS page for the production environment I see that v1.0 is in the deployment history.

![Portal - Deployments](/assets/img/cd-github-19.png)

So, an observation to note here is that the deployment history follows the environment .  At least for now.  Personally, I would like to see the full deployment history regardless of which environment I’m in.  But, hey, this is preview so maybe that’s coming. Smile

Refreshing my browser where my application is running in the staged environment, I see v1.1 is deployed as expected.

![App 1.1](/assets/img/cd-github-20.png)

And v1.0 is still running in production, just as before.

![App 1.0](/assets/img/cd-github-21.png)

### Swap the Staging and Production Environments

Finally, I performed one last swap and as result, v1.1 is now running in the production environment.

![App 1.1](/assets/img/cd-github-22.png)

And as expected, v1.0 is now running in the staging environment.

![App 1.0](/assets/img/cd-github-23.png)

## Some Final Thoughts

I wanted to briefly mention a previous blog from the Windows Azure Team titled Windows Azure Web Sites: How Application Strings and Connection Strings Work . The reason why is connection strings and application settings are common things to change as you transition from one environment to another.  With Staged Publishing now available, the value of these features, while cool before, just became even more important in my opinion because you can define connection strings and application settings appropriate for each environment.  If you’re not using these features today I would suggest you do – especially if you want to get maximum benefit from continuous deployment and staged publishing.

In this post I demonstrated how to setup continuous deployment from a GitHub repository to a Windows Azure Web Site with Staged Publishing enabled.  I also showed the steps necessary to work around a little issue currently present in the preview bits for Staged Publishing.
