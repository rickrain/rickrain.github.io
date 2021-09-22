---
title: "Using Group Claims in Azure Active Directory"
date: 2015-02-13 16:06:04 -0500
permalink: /:year/:month/:day/:title/
---

In the post titled [Developing Native Client Apps for Azure AD](http://rickrainey.com/2014/12/08/developing-native-client-apps-for-azure-active-directory/) I showed how you can use the Active Directory Authentication Library (ADAL) to build a native client application that calls the CloudAlloc.WebAPI introduced in the post titled [Building Web Apps for Azure AD](http://rickrainey.com/2014/11/05/building-web-apps-for-azure-ad/). As part of that post, I demonstrated how the Web API could be exposed to other applications using *ouath2Permissions* that I defined in the manifest which enabled me to assign permissions to my native client application as shown here.

![Permissions to other apps](/assets/img/using-group-claims-in-azure-ad-01.png)

Being able to assign read and write permissions in this way provided an easy and useful way to control an application’s ability to invoke various operations using the claims that were issued by Azure Active Directory for an authenticated user. However, the level of granularity for which authorization decisions can be made in the application code is pretty coarse with this approach. In my example, it was the simple read and write permissions I defined and if the native client application was configured to allow write permissions, then any authenticated user would be able to invoke such operations. While that may be sufficient for some applications, others may require additional information about the user before making such authorization decisions. For example, you may want to restrict write operations to users in a specific security group. As you can see below, the claims provided by Azure Active Directory for our authenticated user John Doe just don’t provide that level of detail about the user (yet). Other than some various identifier claims about John Doe, we’re currently limited to the values in the *scope* claim to make authorization decisions.

![Default user claims](/assets/img/using-group-claims-in-azure-ad-02.png)

A common need for many applications is the ability to make authorization decisions based on a user’s membership in a specific security group (or groups). In this post I’m going to show how you can take advantage of some recently released features of Azure Active Directory to do just that.

## Using group claims to drive authorization decisions ##
Very recently, the Azure Active Directory team announced the [preview release of two new features: group claims and application roles](http://blogs.technet.com/b/ad/archive/2014/12/18/azure-active-directory-now-with-group-claims-and-application-roles.aspx). Before this, you had to use the [Azure AD Graph API](https://msdn.microsoft.com/en-us/library/azure/hh974478.aspx) to determine a user’s membership in a group. Now, you can just look for it in the *Claims* collection for the authenticated user provided you enable the group claims feature for your application in Azure AD. Let’s see how this would work for the application we’ve been talking about and only allow users to invoke the POST method if they are in the “Dev/Test” security group.

### Enable group claims for the cloudalloc.webapi application ###
Enabling the group claims feature currently requires that you update the manifest for the application in Azure AD. This is exactly the same process I showed you in the last post where I set the *oauth2Permissions* for the application.

In the Azure Management portal I’ll begin by going to the **CONFIGURE** page for the CloudAlloc.WebAPI application in Azure AD. At the bottom of the page is a **MANAGE MANIFEST** button where I can select to download the manifest for the CloudAlloc.WebAPI application as shown here.

![Download manifest](/assets/img/using-group-claims-in-azure-ad-03.png)

The application manifest is just a JSON file that you can edit with the simplest of editors (ie: notepad.exe). By the way, if you’re curious what the GUID in the filename is about when you download the manifest, it is the Client ID that was assigned to the application when it was registered in Azure AD. Scrolling down the manifest file I will find the *groupMembershipClaims* property which will be set to null as shown here.

![Group Membership Claims](/assets/img/using-group-claims-in-azure-ad-04.png)

I’m going to change this value to *SecurityGroup* and then save the changes as shown here.

![Group Membership Claims](/assets/img/using-group-claims-in-azure-ad-05.png)

Your choices for setting the *groupMembershipClaims* property are *null* (the default), *All* or *SecurityGroup*. If you choose *SecurityGroup* you will get group claims in the JWT token for just security groups the user is a member of. If you choose *All* you will get group claims in the JWT token for security groups and distribution lists the user is a member of.

All that remains now is to upload the modified manifest file which I did using the **MANAGE MANIFEST** button.

With that change in place, I will now start getting group claims in the token for users of my application. As mentioned above, I’m interested in checking for the user’s existence in the Dev/Test security group. So, before I show you the code changes needed to do this, let’s review my security groups and where our John Doe user lands in these groups.

### Locate the Objec ID for the Dev/Test Security Group ###

When Azure AD adds applicable group claims to the token it issues for my users, the value for the group claim will be the Object ID of the security group and not the name of the security group. Remember, every entity in Azure AD has a unique Object ID associated with it. You may think it would be more intuitive to refer to the group by name instead of the Object ID. This is true. However, a group’s name can be changed in the directory so it is not a reliable identifier for the group. The Object ID will never change as long as the group exists. So, in this case, I need to find the Object ID for the Dev/Test security group. This can be found in the **CONFIGURE** page of for my Dev/Test security group as shown here.

![Dev/Test Configure Page](/assets/img/using-group-claims-in-azure-ad-06.png)

Now that I have the Object ID for the Dev/Test security group I am ready to make the necessary code changes.

### Update the POST method to look for the Dev/Test Group Claim ###

The only code change needed is to look for a *groups* claim with the value of “244728b5-8b9e-4e2f-8703-9853366cd431” (yours will be different) in the authenticated user’s claims collection as shown here.

```c#
// POST api/values
public HttpResponseMessage Post([FromBody]string value)
{
    // Look for the scope claim containing the value 'Read_Write_CloudAlloc_WebAPI'
    var principal = ClaimsPrincipal.Current;
    Claim writeValuesClaim = principal.Claims.FirstOrDefault(
        c => c.Type == "http://schemas.microsoft.com/identity/claims/scope" &&
            c.Value.Contains("Read_Write_CloudAlloc_WebAPI"));
 
    // Look for the groups claim for the 'Dev/Test' group.
    const string devTestGroup = "244728b5-8b9e-4e2f-8703-9853366cd431";
    Claim groupDevTestClaim = principal.Claims.FirstOrDefault(
        c => c.Type == "groups" &&
            c.Value.Equals(devTestGroup, StringComparison.CurrentCultureIgnoreCase));
 
    // If the app has write permissions and the user is in the Dev/Test group...
    if ((null != writeValuesClaim) && (null != groupDevTestClaim))
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

Note: As I pointed out in my last post, it is a good idea to organize code like this into a custom [AuthorizeAttribute](http://msdn.microsoft.com/en-us/library/system.web.mvc.authorizeattribute(v=vs.118).aspx) class or some other common class so that your application logic and authorization logic are not mixed together. I’m intentionally leaving that as an exercise to you the reader though.

That’s all there is to it! Now, let’s observe the results of these changes by examining the claims collection for the authenticated user (John Doe).

### Examining the Group Claims ###

I ran the application under the debugger and set a breakpoint in the POST method so I could show you the new group claims in the claims collection. As shown below, you can see there are two *groups* claims present for John Doe.

![Groups Claims for John Doe](/assets/img/using-group-claims-in-azure-ad-07.png)

The value for the first groups claim should look familiar – it is the Object ID for the *Dev/Test* group I discussed earlier. The second one happens to be a *Developer* security group in my directory for which John Doe is a member.  The figure here shows the membership for both security groups.

![Security Groups for John Doe](/assets/img/using-group-claims-in-azure-ad-08.png)

Notice that John Doe is explicitly identified as a member of the Developer security group but not the Dev/Test group. Yet, in the token issued to John Doe, a group claim was present for both of these security groups. This is because the group claims feature in Azure AD is transitive. So, since John Doe is a member of the Developer group and the Developer group is a member of the Dev/Test group, John Doe is a member of both security groups.

As you can imagine, in a real world production environment a user is likely to be a member of many different security groups. When you enable the group claims feature the tokens issued for users will contain the group claims for all of these groups which could greatly increase the size of the token. Therefore, there are limits on the number of group claims that Azure AD will place in a token as follows:

- JWT Tokens: Up to 200 group claims
- SAML Tokens: Up to 150 group claims

Currently there is not a way to filter the group claims that Azure AD places in a token. So, if users in your directory could potentially exceed these limits you will need a different solution.

## Working with the Azure AD Group Claims Limit ##

The [Azure AD Graph API](http://msdn.microsoft.com/en-us/library/azure/hh974478.aspx) is a REST API that Azure Active Directory makes available for each tenant. With it you can programmatically access the directory and query about users, groups, contacts, tenant details and more. In addition to querying the directory, the Azure AD Graph API can be used to create, update and even delete entities in the directory.

For the scenario mentioned above, the Azure AD Graph API could be used to look up the security groups a user belongs to or to check if a user is in a specific security group. The latter is particularly applicable in our scenario here and could be achieved using the [IsMemberOf](https://msdn.microsoft.com/en-us/library/azure/dn151601.aspx) function. To execute this query you need only the Object ID of the user and the Object ID of the security group. The query will execute in Azure AD (server side) and simply return true or false. Prior to the release of the group claims feature in Azure AD discussed in this post, this was the only way to check group membership for a user. In situations where the number of groups a user is in could potentially exceed the limits above, it is still available to you. In the next post I’ll talk more about the Azure AD Graph API.

## Summary ##

In this post I showed how to take advantage of the new Azure Active Directory group claims feature and apply it to our scenario to determine a user’s membership in a particular security group. I also discussed the group claims limit as it applies to JWT and SAML tokens issued by Azure AD and how you can fall back on the Azure AD Graph API’s `IsMemberOf` function to query the directory and determine the same.

In the next post, I will introduce you to the Azure AD Graph API and some handy client libraries you can use to access the graph in Azure AD.

