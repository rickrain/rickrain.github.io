---
title: "Azure Active Directory: An Introduction"
date: 2014-10-16 16:06:04 -0500
permalink: /:year/:month/:day/:title/
---

## What Azure Active Directory is (and is not) ##

Azure Active Directory (aka Azure AD) is a fully managed multi-tenant service from Microsoft that offers identity and access capabilities for applications running in Microsoft Azure *and* for applications running in an on-premises environment. Its name leads some to make incorrect conclusions about what Azure AD really is. Therefore, to avoid any confusion with Windows Server Active Directory that you may already be familiar with in an on-premises environment, understand that Azure AD is not Windows Server Active Directory running on Virtual Machines in Microsoft Azure.

Azure AD is not a replacement for Windows Server Active Directory. If you already have an on-premises directory, it can be extended to the cloud using the directory integration capabilities of Azure AD. In these scenarios, users and groups in the on-premises directory are synced to Azure AD using a tool such as Azure Active Directory Sync (AAD Sync). This has the benefit of users being able to authenticate against Windows Server Active Directory when accessing on-premises applications and resources, and authenticating against Azure AD when accessing cloud applications. The user can authenticate using the same credentials in both scenarios.

Azure AD can also be an organization’s only directory service. For example, many startups today don’t have an on-premises Windows Server Active Directory. In these scenarios, an organization may simply rely on Office 365 and other SaaS applications to conduct its business and manage its user’s identity and access to SaaS applications, all online.

As a developer of cloud applications, you can use Azure AD to accomplish things such as single sign-on (SSO) for your cloud applications, query the directory for user and group information, and even write to the directory provided your application has the permissions to do so. Of course, you can accomplish similar ideas on-premises and build applications using Microsoft proprietary technologies such as Windows NTLM, Kerberos, LDAP and so on. However, as more applications are developed with the intent of running in the cloud, and access to those applications cross organizational boundaries, and are accessed from a growing number of devices across a variety of platforms, organizations need an enterprise ready directory service that can handle the authentication needs of the organization and the applications it depends on. The graphic below illustrates how Azure Active Directory could be used in a company to address some common enterprise needs.

Throughout this series, the focus will be on the developer, with careful attention given to the tools, libraries and features of Azure AD that can be used to build cloud applications that are protected by Azure AD. This post lays the groundwork for some key concepts in Azure AD. Future posts in this series will go deep into the developer experience for different types of cloud applications.

## Users and Groups ##

In Azure AD there are two entities that we are most concerned with during application development; the user and perhaps the group (or groups) the user is a part of. We use the user’s claims (statements about the user from Azure AD) to drive application behavior such as personalizing screens or making authorization decisions about what the user can do. By externalizing the authentication of users to Azure AD we’re able to rely on the claims provided in the authentication token for the user to drive these application experiences.

A user in a directory can be sourced from either a directory in Azure AD, which is referred to as a *Work or Student Account* (previously called an *Organizational Account*). Or, the user can be sourced from a *Microsoft Account*. An example of how this may look in the Azure AD Users page of the Azure Management Portal is shown here.

## Work or Student Accounts ##

![Work Account](/assets/img/introduction-to-azure-ad-01.png)

Users are generally added to a directory in Azure AD as a *Work or Student Account* user (formerly know as *Organizational Accounts*). A user’s user name and email address will take the form of **\<someuser\>@\<someorg\>.onmicrosoft.com** and the account typically exists for as long as the user is part of the organization and until an Administrator removes the account. If you have ever had an account in an on-premises Windows Server Active Directory (for example, at work) then this is essentially the same concept.

A user from a different directory in Azure AD can also be added to a directory (aka external user). This is particularly useful in situations where users in different directories need access to the same cloud applications protected by Azure AD. This is a very powerful scenario and one that is increasing popular as more and more organizations partner with one another. Consider, for example, a user “John Doe” from CompanyA (john.doe@companya.onmicrosoft.com) is added as a user to the directory for CompanyB. This would enable “John Doe” to access CompanyB’s cloud applications protected by CompanyB’s Azure AD. Now, when “John Doe” authenticates, he will do so in the realm of CompanyA’s directory, not CompanyB’s directory. So, his profile data, password, policies, etc. are all managed by the administrators at CompanyA. As it should be – CompanyA knows him best and is the authority that can properly authenticate him. Given that CompanyA and CompanyB have established trust between the two organizations, CompanyB can add “John Doe” to their directory so he can collaborate with users and applications in CompanyB. If CompanyB decides later to remove “John Doe” from the directory, then he will no longer be able to access CompanyB’s resources. However, suppose “John Doe” quits CompanyA or is removed from the organization for other reasons. What is the first thing IT generally does in this case? They delete “John Doe” from the directory (or at least disable the account). They probably don’t call all their partners such as CompanyB to tell them that “John Doe” should be removed. But that’s ok, because as I pointed out earlier, “John Doe” will never authenticate in the realm of CompanyB’s directory. He will authenticate against CompanyA’s directory. And since his account will have been either removed or disabled, he won’t be able to successfully authenticate, and therefore won’t be able to access any CompanyB resources even though his account technically still exists in CompanyB’s directory. If you have been in this industry for at least a few short years you can probably appreciate the immense flexibility and level of security this brings to an organization.

During authentication, Work or Student Account users may notice the blue badge icon shown above. This icon is a visual hint to the user signing in using a Work or Student Account.

## Microsoft Accounts ##

![Microsoft Account](/assets/img/introduction-to-azure-ad-02.png)

Users can also be added to a directory in Azure AD as a *Microsoft Account* user. In this scenario, the user name and email address will likely take the form of **\<someuser\>@hotmail.com**, **\<someuser\>@outlook.com**, or **\<someuser\>@live.com**. A Microsoft Account is an individual account that a user has created to access consumer services such as Xbox LIVE, Messenger, Outlook/Hotmail, etc. Unlike the Organizational Account, these accounts don’t get deleted when they are removed from a directory in Azure AD. The account belongs to the person in this case, not an organization. However, Azure AD enables organizations to add users to their directory using a person’s Microsoft Account and these users are also considered external users of the directory.

One scenario where you might add a user to a directory in Azure AD using a person’s Microsoft Account is in a “freelance” situation where you need an external user (someone that is not part of the organization) to have access to cloud applications for a particular project or service agreement. This prevents the user from having to keep up with a separate set of user credentials just to access the necessary application resources for the project. When the project term ends the user can simply be removed from the directory.

During authentication Microsoft Account users may notice the Windows icon shown above. This icon is a visual hint to the user signing in using a Microsoft Account.

## Adding users and groups to Azure AD ##

Users and Groups can be added to a directory in a variety of ways. In no particular order here are some common methods:

- By syncing from an on-premises Windows Server Active Directory using AAD Sync. This is how most enterprise customers will get their users added to the directory and requires some additional server configuration on-premises to setup.
- Manually using the Azure Management Portal. The portal experience is very easy and intuitive. Organizations that don’t have an on-premises directory may use this approach for its simplicity, provided the number of users is relatively small. This is also very useful during development and test phases of application development.
- Scripted using PowerShell and the Azure Active Directory cmdlets. PowerShell makes automating this task very useful, particularly for large user bases. This too can be very useful during development and testing.
- Programmatically using the Azure AD Graph API. This is an extremely powerful option that essentially gives you full control of how users are added to the directory. We will see examples of this later in the series.

## Custom Domains ##

Every directory in Azure AD gets a unique DNS name on the shared domain **\*.onmicrosoft.com**. So, as an example, for a directory named “cloudalloc”, the DNS name would be **cloudalloc.onmicrosoft.com**. A user in the directory would therefore have a user name such as **john.doe@cloudalloc.onmicrosoft.com**.

By using a custom domain you are able to associate a domain you own with a directory in Azure AD. This is not required, but often preferred by customers who own their own domain name. Continuing with the same example, if you owned the cloudalloc.com domain a user name would take the form **john.doe@cloudalloc.com** instead of **john.doe@cloudalloc.onmicrosoft.com**.

Configuring a custom domain and associating it with a directory in Azure AD is a relatively simple process. First, you need to own the domain name you want to use with your directory. Next, you need to go through a domain verification step in Azure AD, which basically involves updating the DNS records with your domain registrar to include a TXT record and value (provided by Azure AD) to prove you own the domain. The details for configuring a custom domain are covered in full detail in this [Verify a domain](http://technet.microsoft.com/en-us/library/jj151788.aspx) article on TechNet.

## Protocols supported by Azure AD ##

As a developer building applications protected by Azure AD you will find that Azure AD provides support for all the common protocols that can be used to secure your applications. Some of these protocols have been around for a really long time and as a result are widely used in the industry today. Others are still emerging as a new (and preferred) way to protect access to cloud applications. The protocols supported are shown here:

- **WS-Federation** – This is arguably one of the most well-known and used protocol today for authenticating users of web applications. Microsoft uses this when authenticating users for some of their own cloud applications, such as the Microsoft Azure Management portal, Office 365, Dynamics CRM Online, and more. There is fantastic tooling support in Visual Studio 2010, 2012, and 2013 for this protocol making it very easy for developers to protect their applications using Azure AD. The token format used in this protocol is SAML.
- **SAML-P** – This is also a widely adopted protocol and follows a very similar authentication pattern to WS-Federation. However, it does not get the same level of tooling support that WS-Federation gets. The token format used in this protocol is also SAML.
- **OAuth 2.0** – This is an authorization protocol that has been quickly and widely adopted in the industry as a way to sign-in users using their credentials at popular web applications such as Facebook, Twitter, and other “social” applications. Some of the benefits of this protocol is its smaller token format, JSON Web Token (JWT), and application scenarios it simplifies such as accessing Web API’s from a native client with an access token. To do the latter with WS-Federation or SAML-P involves a lot of intricate code and configuration.
- **OpenID Connect** – This is a protocol that adds an authentication layer on top of the existing OAuth 2.0 protocol. Because it is layered on OAuth 2.0, it benefits from the highly efficient JWT token format that OAuth 2.0 uses.
 
The OAuth 2.0 and OpenID Connect just recently became Generally Available (GA, or fully supported and out of preview in September of 2014) on Azure AD and there is a great amount of work going into libraries like Active Directory Authentication Library (ADAL) and OWIN middleware components to light up scenarios these protocols enable for developers. This series will cover these libraries in great detail in later posts. The type of application you build and the requirements for your application will largely determine which of these protocols is used to protect the application when registering it with Azure AD. For now, it is enough to know they are supported by Azure AD.

## From protocols to application endpoints ##

The support for these protocols is surfaced in Azure AD through a set of Application Endpoints. These endpoints are unique for each directory (or tenant) in Azure AD. The table below shows application endpoints and URL for each of the supported protocols. It is these endpoints that your application uses to leverage the various protocols when authenticating users. Notice that every endpoint starts with https://login.windows.net/\<tenant\>/, where \<tenant\> is a unique identifier for the directory.

| Application Endpoint        | URL |
| --------------------------- | ------------- |
| Federation Metadata Document | https://login.windows.net/\<tenant\>/federationmetadata/2007-06/federationmetadata.xml |
| WS-Federation | https://login.windows.net/\<tenant\>/wsfed |
| SAML-P | https://login.windows.net/\<tenant\>/saml2 |
| OAuth 2.0 Token | https://login.windows.net/\<tenant\>/oauth2/token |
| OAuth 2.0 Authorization | https://login.windows.net/\<tenant\>/oauth2/authorize |

In each of these endpoints, \<tenant\> can be either the Guid that is assigned to the directory, or the hostname of the directory. In other words, for an Azure Active Directory named “cloudalloc” with a tenant id of “530c3a3b-e508-4826-997a-38fb543bc87f”, the following two URL’s for the WS-Federation endpoint would be equivalent.

- https://login.windows.net/cloudalloc.onmicrosoft.com/wsfed
- https://login.windows.net/530c3a3b-e508-4826-997a-38fb543bc87f/wsfed

If a custom domain was configured for the directory, then the domain name could also be substituted for \<tenant\>.

As this series progresses we will see how these endpoints are leveraged for various types of applications.

## Adding applications to Azure AD ##

Adding applications to an Azure AD tenant (or directory) is necessary when you want users in your organization to be authenticated against your directory before accessing the application. By adding the application to the directory (or registering it), you are adding configuration that Azure AD will need to identify your application as an application so that it can issue authentication tokens when authenticating users.

Applications you develop can be registered with a directory in Azure AD by using tools such as Visual Studio, the Azure Management Portal and other command line tools. The Azure Management Portal provides an easy wizard experience to get the process started as shown here when you click on the add application button in the portal.

![Add application](/assets/img/introduction-to-azure-ad-03.png)

When you choose the option to **add an application my organization is developing**, you must next indicate the type of application you are going to add. The reason the type is important is because it is here where necessary protocol artifacts will be collected leading you eventually to the use of either WS-Federation or OAuth / OpenID Connect. The two choices are:

- **Web Application and/or Web API** – Think of browser-based web applications or services that are accessed using a browser and/or protocols of the web.
- **Native Client Application** – Think of client applications that will run on a desktop computer, laptop, or other smart device.

## Summary ##

In this post I introduced Azure AD and some important concepts regarding users. I talked briefly about custom domains and then wrapped up by covering the protocols supported by Azure AD and how they are surfaced through application endpoints unique to the directory. In the next post, I’ll dive into the developer experience and talk about building on the first of the two choices mentioned in the last section which is Web Applications and/or Web API’s.
