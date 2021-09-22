---
title: "Auto Scaling Cloud Services on CPU Percentage with the Windows Azure Monitoring Services Management Library"
date: 2013-12-15 16:06:04 -0500
---

The auto scaling feature in Windows Azure is a fantastic feature that enables you to scale your services dynamically based on a set of rules.  For example, in the case of Cloud Cervices, if CPU exceeds a defined threshold in your rule, auto scaling will add additional instances to handle the increased load.  When CPU drops below a defined threshold, auto scaling will remove those extra instances.  It’s this kind of elasticity that makes cloud computing so compelling.

Auto scale (currently in preview) is available for Cloud Services, Websites, and Virtual Machines.  The metric for scaling on these different compute options can be based on CPU or Queue Depth as shown here.

![Autoscale Cloud Services](/assets/img/autoscale-cs-01.png)

Auto scale is also available for Mobile Services with some different metrics (perhaps a topic for a future post).

How to Scale an Application covers some common scenarios for configuring auto scale using the Windows Azure Management Portal.  However, in this post, I’ll show how you can achieve auto scaling using the Windows Azure Monitoring Services Management Library (also in preview) instead.  For the sake of time, I’m only covering Cloud Services and auto scaling based on CPU.  Otherwise, this would be a massively long post.  This is important because some of what I talk about does not apply to the other compute options.

## Overview of my Cloud Service Application

To set the stage for the application I’ll be configuring for auto scale, it is a cloud service consisting of two Web Roles configured for just 1 instance.

- An MVC Web Front End (the UI).  This role defines the single public endpoint for the cloud service that is accessed through the browser.  It simulates, in a very elementary manner, job processing.  You can do two things with the UI – see status of jobs and submit a new job.  When a job is submitted, it looks for available Web API instances running and round-robin posts to them through an internal endpoint.
- An MVC Web API.  This is where the job processing is done and it consumes a lot of CPU cycles.  Therefore, it is this role that I will define auto scaling rules for.

## Introduction to Auto Scale Profiles

When you define auto scale settings, you’re essentially defining what’s called an AutoScaleProfile.  An AutoScaleProfile contains rules, each of which has a MetricTrigger and a ScaleAction.  The MetricTrigger identifies a resource (CPU for example) and some parameters for how the metric is measured and evaluated.  The ScaleAction defines what to do when the conditions for the metric have been met (scale up/down for example).  Basically, you have something like this.

![Autoscale Cloud Services](/assets/img/autoscale-cs-02.png)

This is by no means complete  .  It does, however, show the relationship of some basic constructs that are used in the API’s.

## Benefits of using Windows Azure Monitoring Services Management Library

By using the API’s, you can take advantage of features that may not otherwise be accessible through the portal.  Here’s a list of features I came up with that I found useful.

- The ability to create multiple profiles.  Up to 20.  So, for example, I may define a profile to scale to higher capacity during one time period and another profile that scales to lower capacity during a different time window.  The recent cyber-Monday comes to mind.  Or, perhaps one profile during the week and another for weekends.  The ability to define multiple profiles allows for some interesting possibilities.
- The ability to set the TimeWindow property to a value other than the default of 45 minutes.  The TimeWindow is a property in the MetricTrigger I mentioned above that specifies the range of time in which instance data is collected and evaluated, meaning that, by default, 45 minutes of monitoring data will have to have been collected before a MetricTrigger will result in a ScaleAction being performed.  Now, you can set this value as small as 5 minutes.  However, I would strongly caution against it for reasons I’ll address later.   What I’ve found to be a good value for this is 25 minutes based on my testing.
- The ability to change how the statistics for a metric are evaluated across role instances.  By default, they are averaged, but I could choose to use minimum or maximum values instead.

There are other properties that can be set using the API’s that are not surfaced in the Azure portal.  These are just a few I found interesting.

## How to use the library to Create/Update, Read and Delete Auto Scale Settings

The library is available via Nuget here .  If you’re looking for it in Visual Studio’s Nuget Package Manager, make sure to “Include Prerelease” when searching for “Microsoft.WindowsAzure.Management.Monitoring”.

![Autoscale Cloud Services](/assets/img/autoscale-cs-03.png)

### Instantiate an Instance of AutoscaleClient

The AutoscaleClient is the class you make use to create, read, or delete auto scale settings.  To instantiate it, you need to provide appropriate credentials which can be generated from the usual suspects – Subscription Id and Certificate.  This code uses a helper class to get the management certificate from the certificate store.  It also sets some variables to identify the cloud service and role I’ll apply the auto scale settings to.  The latter is known as the resourceId in subsequent code.

```csharp
// Cloud Service and Role to be auto scaled.
var cloudServiceName = "[YOUR CLOUD SERVICE NAME]";
var isProduction = true;
var roleName = "[THE ROLE NAME TO SCALE]";

// Azure Subscription ID and Management Certificate Thumbprint.
string subscriptionId = "[YOUR SUBSCRIPTION ID]";
string certThumbprint = "‎[YOUR CERTIFICATE THUMBPRINT]";

// Get the certificate from the local store.
X509Certificate2 cert = CertificateHelper.GetCertificate(
    StoreName.My, StoreLocation.CurrentUser, certThumbprint);

// Genereate a resource Id for the cloud service and role.
var resourceId = AutoscaleResourceIdBuilder.BuildCloudServiceResourceId(
    cloudServiceName, roleName, isProduction);

// Create the autoscale client.
AutoscaleClient autoscaleClient =
   new AutoscaleClient(new CertificateCloudCredentials(subscriptionId, cert));
```

### Create an Autoscale Profile

Using the psuedo-code above as reference, the code here shows how to create a profile.  Notice there are some other settings here like ScaleCapacity where you can define minimum, maximum, and default capacity limits for scaling.

```csharp
AutoscaleSettingCreateOrUpdateParameters createParams =
    new AutoscaleSettingCreateOrUpdateParameters()
    {
        Setting = new AutoscaleSetting()
        {
            Enabled = true,
            Profiles = new List<AutoscaleProfile>() 
            {
                new AutoscaleProfile()
                {
                    Name = "Scale.WebApi.Cpu",
                    Capacity = new ScaleCapacity()
                    {
                        Default = "1",
                        Minimum = "1",
                        Maximum = "4"
                    },
                    Rules = new List<ScaleRule>()
                }
            }
        }
    };
```

### Create a ScaleRule to Scale Up

Again, using the psuedo-code above as a reference, this code creates the first of two ScaleRule instances.  The rule contains a MetricTrigger and a ScaleAction to scale up the instances (WebApi in this case) when the average CPU exceeds 40% across all instances.

A few things to point out about this code are…

- The MetricName and MetricNamespace are not values I just made up.  These have to be precise.  You can get these values from the MetricsClient API and there is some sample code in this link to show how to get the values.
- The TimeWindow is set to 25 minutes.  As I indicated earlier, the default for this is 45 minutes.  My experience has been intermittent with a value of 20 minutes or less.  So, I recommend at least 25 minutes for this value.  I’ll come back to this later.

```csharp
var cpuScaleUpRule = new ScaleRule()
{
    // Define the MetricTrigger Properties
    MetricTrigger = new MetricTrigger()
    {
        MetricName = "Percentage CPU",
        MetricNamespace = "",
        MetricSource = AutoscaleMetricSourceBuilder.BuildCloudServiceMetricSource(cloudServiceName, roleName, isProduction),
        TimeGrain = TimeSpan.FromMinutes(5),
        TimeWindow = TimeSpan.FromMinutes(25),
        TimeAggregation = TimeAggregationType.Average,
        Statistic = MetricStatisticType.Average,
        Operator = ComparisonOperationType.GreaterThan,
        Threshold = 40.0
    },
    // Define the ScaleAction Properties
    ScaleAction = new ScaleAction()
    {
        Direction = ScaleDirection.Increase,
        Type = ScaleType.ChangeCount,
        Value = "1",
        Cooldown = TimeSpan.FromMinutes(10)
    }
};
```

### Create a ScaleRule to Scale Down

Similar to the previous rule, this rule contains a MetricTrigger and ScaleAction to scale down the instances when the average CPU falls below 25% across all instances.

```csharp
var cpuScaleDownRule = new ScaleRule()
{
    // Define the MetricTrigger Properties
    MetricTrigger = new MetricTrigger()
    {
        MetricName = "Percentage CPU",
        MetricNamespace = "",
        MetricSource = AutoscaleMetricSourceBuilder.BuildCloudServiceMetricSource(cloudServiceName, roleName, isProduction),
        TimeGrain = TimeSpan.FromMinutes(5),
        TimeWindow = TimeSpan.FromMinutes(25),
        TimeAggregation = TimeAggregationType.Average,
        Statistic = MetricStatisticType.Average,
        Operator = ComparisonOperationType.LessThan,
        Threshold = 25.0
    },
    // Define the ScaleAction Properties
    ScaleAction = new ScaleAction()
    {
        Direction = ScaleDirection.Decrease,
        Type = ScaleType.ChangeCount,
        Value = "1",
        Cooldown = TimeSpan.FromMinutes(10)
    }
};
```

### Create or Update Auto Scale Settings

Finally, to create (or update existing settings), all that is left is to add the rules to the profile and call the CreateOrUpdate method to apply the settings for the resource.

```csharp
// Add the rules to the profile
createParams.Setting.Profiles[0].Rules.Add(cpuScaleUpRule);
createParams.Setting.Profiles[0].Rules.Add(cpuScaleDownRule);

// Apply the settings in Azure to this resource
autoscaleClient.Settings.CreateOrUpdate(resourceId, createParams);
```

### Get Auto Scale Settings

To retrieve existing auto scale settings for a particular resource, you just need to call Get, passing in the resourceId.  The AutoscaleSettingGetResponse will have all the goodness in it that we just went through to create the the settings.

```csharp
AutoscaleSettingGetResponse setting = autoscaleClient.Settings.Get(resourceId);
```

### Delete Auto Scale Settings

To delete existing auto scale settings just call Delete, passing in the resourceId.

```csharp
var deleteResponse = autoscaleClient.Settings.Delete(resourceId);
```
 
## Observations of a Running Application Configured for Auto Scale

With these auto scale settings in place, I decided to take my MVC and CPU consuming Web Api for a test drive to see this in action.

I started by going to my MVC page and submitting a couple of “jobs” over the span of about 10 minutes.  And as you can see here, my CPU is certainly climbing and quickly I’m above my 40% threshold.

![Autoscale Cloud Services](/assets/img/autoscale-cs-04.png)

As expected, after 25 minutes of average CPU above 40%, I saw my WebApi role spin up a 2nd instance.

![Autoscale Cloud Services](/assets/img/autoscale-cs-05.png)


Once the instance was ready, I threw a couple more jobs at it.  The home page for my MVC front end displays the results of all the jobs and also which instance serviced (or is servicing them).  My MVC front end is dispatching these jobs in round-robin fashion to the available WebApi instances.

![Autoscale Cloud Services](/assets/img/autoscale-cs-06.jpg)

Continuing with this pattern, I saw another instance get created about 15 minutes later.

![Autoscale Cloud Services](/assets/img/autoscale-cs-07.png)

And I continued to throw a few more jobs at the system.  So, now I have 5 jobs being worked across 3 instances.

![Autoscale Cloud Services](/assets/img/autoscale-cs-08.jpg)

The timeline for the scale up actions is below and shows when my instances were created and the effect they had on the average CPU for WebApi while I continued to throw jobs at the system.

![Autoscale Cloud Services](/assets/img/autoscale-cs-09.png)

What I didn’t show is the scale down behavior that occurred after I stopped loading the system up with work.  Over the course of another 20 minutes or so, the WebApi instances dropped back down to the single instance I started with.

## Caution About Setting the Time Window

When I first learned about the TimeWindow property and the default value of 45 minutes, I immediately elected to use the API’s to set this to as small a number as allowed by the API (which is 5 minutes).  After all, if my service is getting hammered, I would prefer that it scale right away instead of wait for 45 minutes to do so.  Unfortunately, when I did this I started seeing messages like this in the Azure Portal with no scale actions being invoked.

![Autoscale Cloud Services](/assets/img/autoscale-cs-10.png)

What I’ve learned is that the gap between the current time and the last data point that you can see in the MONITORING page is part of that TimeWindow.  So, consider for example the scenario below where it may be 8:43 but the last data point available was at 8:30.  That’s a 13 minute gap in time.  Defining a TimeWindow of 5 minutes would result in the warning above because there was no monitoring data points in the last 5 minutes ( between 8:38 and 8:43 ) when auto scale was trying to evaluate the MetricTrigger.

![Autoscale Cloud Services](/assets/img/autoscale-cs-11.png)

This lag in monitoring data and current time is common and the lag time is unpredictable.  It might be 10 minutes during part of the day and 20 minutes at another.  In my testing, I’ve found that 25 minutes is a safe value for TimeWindow.

So, experiment with this setting to find the value that works for you if you choose to deviate from the default of 45 minutes.

## Examining the Operating Logs

Anytime you change auto scale settings an AutoscaleAction is invoked.  You can see the details of such changes in the Azure Portal.  Click on the MANAGEMENT SERVICES section.  Next, click on a particular log and click on the DETAILS link at the bottom to see the changes.  I’ve found this to be very helpful when testing out various profile settings.

![Autoscale Cloud Services](/assets/img/autoscale-cs-12.png)

## Conclusion

There is so much more to auto scaling that I’ve not even mentioned.  As long as this post is, it’s hard to believe I’ve just barely scratched the surface.  Hopefully this has helped you understand a little more about auto scale for cloud services, the Windows Azure Monitoring Services Management Library, and the benefits of using it to automate your auto scale settings.

The Service Management REST API for Autoscaling provides documentation that at the time of this writing wasn’t available in the .NET API’s that I wrote about.  Even if you don’t use the REST API’s, referring to them for documentation is a good idea.