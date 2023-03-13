# An easy way to get started with Certificate Based Authentication for Azure apps
[An easy way to get started with Certificate Based Authentication for Azure apps](https://msendpointmgr.com/2023/03/11/certificate-based-authentication-aad/) 

 In this blog post I will be showing you how to get started with certificate based authentication for Azure applications.

**Table of Contents**

The story so far
----------------

The world has gone cloud crazy. But this is a good thing! Thinking back to my old admin days, any script I wrote would be deployed using GPO’s, Scheduled Tasks, RMM tools, ConfigMgr, Login Scripts and more. As we now focus our automation efforts on cloud services, the good old days of “dropping a script somewhere and running it” are coming to an end. Whether you are running scripts interactively to get data from Graph API or using Azure Functions to automate that task for you – you would have to authenticate at some point to prove you have the permissions to make that API call.

Thinking back to the old days, I would typically authenticate myself against Active Directory to prove I had the necessary permissions to perform a task. In the modern world, its a similar model and I now authenticate against Azure Active Directory to obtain the token I need to perform a particular function or call to the Graph API.

App Registrations
-----------------

When I think about automation, I instinctively get drawn to Microsoft Graph. The API’s are rich and expose a wealth of information about my Microsoft Services. I can jump over to Graph Explorer [aka.ms/ge](https://aka.ms/ge) and this is a great tool to familiarize yourself with queries to the Microsoft Graph but at some point you are going to want to use other programming logic to build your query that will get posted to the Microsoft Graph. To make calls to the Graph we need need to have an access token and to get that access token we need to authenticate to Azure Active Directory.

Now, typically you are going to use an Azure App Registration (a service principal) to get the token. The service principal is granted the necessary permissions it requires for your automation to do its thing. These permissions are explicitly granted on the service principal (see below) For example, if I wanted to read and write Win32apps in Intune, my app registration must be granted permission **_DeviceManagementApps.ReadWriteAll_**

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-44-1024x470.png)

Client Secrets
--------------

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-54.png)

You will see **A LOT** of blogs and automation examples on the internet where the admin will use a very easy method to authenticate the Service Principal. A basic password – otherwise known as a **Client Secret**. Client secrets are quick and easy to use but they come at a price.

If we look at the Certificates and secrets blade on the app registration, you will see there are 3 ways to authenticate.

*   Certificates
*   Application Secrets
*   Federated Credentials

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-45-1024x598.png)

The client secret is a short/long lived password. When we authenticate to the Microsoft Graph with our service principal, we need to present 3 pieces of information.

*   Tenant ID (which tenant are we authenticating against)
*   Application ID (same concept as a username)
*   Client Secret (same concept as a password)

We don’t have the option of creating a secret that lasts forever, but a secret can be configured to be valid for up to 2 years

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-46-1024x595.png)

2 years is an incredibly long time for basic credential to be hanging around, especially when we don’t have any other protections around it like we do with normal user accounts, like MFA.

Once you create a client secret, it is only visible in the portal temporarily. When you navigate away from the blade the client secret is obfuscated

Authenticating with a client secret is considered basic authentication. Anyone with the Tenant Name/ID, Application ID and Client Secret can authenticate with the service principal in your tenant. Yeah its probably OK for testing with the minimum value of a client secret set to 90 days but will you go in and clear down your client secrets after performing your tests? The likelihood is no.

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-47-1024x781.png)

Where do I see Client secrets used badly? Hard coded in PowerShell scripts ![](https://s.w.org/images/core/emoji/14.0.0/svg/1f641.svg)
 Please, for the sake of all things great and small, do not do this! If you insist on using Client Secrets, at least store them in a key vault but never hard code them in your scripts.

Jan wrote a really good article on using Client Secrets in conjunction with Azure Key vault at [Securing Intune Enhanced Inventory with Azure Function – MSEndpointMgr](https://msendpointmgr.com/2022/01/17/securing-intune-enhanced-inventory-with-azure-function/)

The IntuneManagementExtension.log will display the script body of a script pushed from Intune. If you hard coded a client secret in your script it will be visible to anyone who opens that log file.

You could mitigate some malicious abuse of your app registration if these credentials are leaked, perhaps by using Conditional Access, but I would argue its better to use a stronger form of authentication if you can. Think about what this Service Principal is allowed to do – create Win32 apps in your tenant! A malicious actor would love to create and deploy a Win32 app that dropped malware on your devices if they got hold of those credentials.

Certificates
------------

Remember back to your basic training around authentication. Passwords are bad, strong authentication methods are good. When we think about Multi Factor Authentication we **_have_** something that others do not. A Certificate is something that I have, it is not a credential that can be used by anyone, which effectively makes it a stronger form of authentication than a client secret.

When I authenticate my Service Principal to get a token, I can use a certificate – but how does this work? A high level view of certificate based authentication:-

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-48-1024x809.png)

1.  Create a certificate or obtain one from a trusted public authority
2.  Upload the public key to the Azure app registration
3.  The private key is used from the local device or uploaded and used from Azure automation
4.  Authentication is successful
5.  A token is returned which can be used to make calls to Graph

### Create a Self Signed Certificate

Authentication with a certificate, from the perspective of automation at least, is quite simple. We often only need the private key on the local device performing the calls to Graph, normally the admin or developer, or it needs to be in Azure automation.

#### Certificate Requirements

There are many flavours of certificate that we can create. What should the cryptographic algorithm, validity period, key length be? Microsoft have a great [article](https://learn.microsoft.com/en-us/azure/active-directory/develop/howto-create-self-signed-certificate) calling these out and I’ll list them below.

*   A 2048-bit key length. While longer values are supported, the 2048-bit size is highly recommended for the best combination of security and performance.
*   Uses the RSA cryptographic algorithm. Azure AD currently supports only RSA.
*   The certificate is signed with the SHA256 hash algorithm. Azure AD also supports certificates signed with SHA384 and SHA512 hash algorithms.
*   The certificate is valid for only one year.
*   The certificate is supported for use for both client and server authentication

[https://learn.microsoft.com/en-us/azure/active-directory/develop/howto-create-self-signed-certificate](https://learn.microsoft.com/en-us/azure/active-directory/develop/howto-create-self-signed-certificate)

So that begs the question, “How do we create a self signed certificate”?

#### The “New-SelfSignedCertificate” PowerShell Cmdlet

The **New-SelfSignedCertificate** PowerShell cmdlet allows us to create a self signed certificate for the purpose of certificate based authentication to Azure AD.

If we just run the cmdlet and pass the **Subject** parameter, lets see what happens

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-49-1024x186.png)

What happened then. It looks like a certificate was created somewhere. Let’s take a peek.

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-51-1024x172.png)

**certlm.msc** show us some magic.

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-50-1024x225.png)

We can see the default properties of the certificate that was created. Information like the private key is exportable (useful if we need to export it for Azure Automation) and the default validity period is 12 months.

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-52-1024x623.png)

The default characteristics of the certificate, without passing any other parameters are:-

*   A 2048-bit key length *
*   Uses the RSA cryptographic algorithm (Azure AD currently supports only RSA)
*   The certificate is signed with the SHA256 hash algorithm **
*   The certificate is valid for only one year
*   The certificate is supported for use for both client and server authentication

\* While longer values are supported, the 2048-bit size is highly recommended for the best combination of security and performance.  
\*\* Azure AD also supports certificates signed with SHA384 and SHA512 hash algorithms.

Can we have a little more control? This snippet is useful to help you quickly create a self-signed certificate by splatting some params to the cmdlet.

**Important:** This is just one example of the params you can use when creating a certificate. I mark the key as exportable because later on in this post I need it. If you intend to authenticate only using the certificate store on the device where you generated the certificate from or by calling the certificate resource in Azure Automation, you probably don’t want to mark the private key as exportable.

More information on creating a self signed certificate can be found at [https://learn.microsoft.com/en-us/powershell/module/pki/new-selfsignedcertificate?view=windowsserver2022-ps](https://learn.microsoft.com/en-us/powershell/module/pki/new-selfsignedcertificate?view=windowsserver2022-ps)

#### Export the Public Portion for use in Azure

Use the following snippet to export the public portion of the certificate as a **.cer** file which you can upload and use for your Service Principal (App Registration) authentication.

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-53-1024x252.png)

#### Export the Private Key

If you previously marked the private key as exportable when you created the certificate, and you have a valid reason to use the .pfx for authentication, you can use the following snippet to export the private portion of the certificate as a **.pfx** file which you can upload and use when authenticating with the Service Principal (App Registration).

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-55-1024x240.png)

#### Create and Export the Public and Private Keys

We can take the examples above and run the following script block to generate a certificate and export the public and private keys.

We should end up with 2 files in our output directory as illustrated below.

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-56-1024x265.png)

The certificate, including the private key, will also be in the certificate store of the local machine as we saw earlier.

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-57-1024x246.png)

Configure Certificate Authentication for an App Registration
------------------------------------------------------------

At the beginning of this post, we viewed the “Certificate & secret” blade on our app registrations. I will revisit that again now and show you how to add the certificate in readiness for certificate based authentication.

Navigate to **Certificates & secrets** blade and select **Upload Certificate**

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-58-1024x526.png)

Browse the the **.cer** file we created earlier and click **Add**

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-59-1024x878.png)

Great, the public portion of the certificate is right where we need it to be.

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-60-1024x293.png)

Easy: Authenticate to Azure AD when the private key is in the local certificate store
-------------------------------------------------------------------------------------

If you didn’t previously mark the private key as exportable and/or it is in the local certificate store on the device where you generated the certificate, there is a really simple way to authenticate using the [MSAL.PS](https://www.powershellgallery.com/packages/MSAL.PS) module.

Lets try it out.

We have a token!

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-66-1024x477.png)

Sign-in used the certificate that we had selected.

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-67-777x1024.png)

### Example: Authenticate with a certificate and make a Graph call to Intune

Lets use the access token to make a call to Microsoft Graph.

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-68-1024x157.png)

There are a couple of ways to use the certificate for authentication in Azure Automation. I am reserving one method for part 2 where we will store the certificate with the private key in Azure Key Vault. In the following example, we will simply upload the private key to the automation account.

From the Automation account, navigate to **Certificates** and upload the .pfx you generated previously when you created the self-signed certificate.

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-70-1024x1011.png)

Add the certificate from the Runbook assets and assign it to a variable

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-71-1024x531.png)

Once we have the certificate, we can use the MSAL.PS module to authenticate to Graph. Make sure you have also added the MSAL.PS module from the gallery to your Automation Account.

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-72-1024x607.png)

### Example: Authenticate with a certificate and make a Graph call to Intune

Lets use the access token to make a call to Microsoft Graph.

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-69-1024x306.png)

Advanced: Authenticate to Azure AD when the private key is not in a certificate store
-------------------------------------------------------------------------------------

If you are connecting from a Windows device, you may already have the certificate in your certificate store if you generated the self signed certificate from the same computer. If the certificate is not in a certificate store or Azure Automation, you can still generate a JSON Web Token (JWT) for authorization. A great thread and code example can be found [here](https://learn.microsoft.com/en-us/answers/questions/346048/how-to-get-access-token-from-client-certificate-ca).

**Caution** Using this method could significantly increase the risk of the private key being obtained maliciously. Ideally, the certificate with the private key should never leave the certificate store

When we execute the script, we get a token!

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-61-1024x135.png)

And the Azure AD sign-in log shows the success authorization

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-63-796x1024.png)

### Example: Authenticate with a certificate and make a Graph call to Intune

We can simplify things now. Lets use the access token to make a call to Microsoft Graph

![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-64-1024x267.png)
![](https://msendpointmgr.com/wp-content/uploads/2023/03/image-65-1024x430.png)

Summary
-------

In this post we looked at creating a self signed certificate for the purpose of authentication to Azure AD using a Service Principal. Certificates are a much better way to authenticate than client secrets. You can use MSAL.PS module to very easily authenticate when the certificate is in the “LocalMachine” or “CurrentUser” certificate store or Azure Automation. If the certificate is not in a certificate store, we looked at how you cold create a JWT for authentication using the private key and make a simple call to Graph to get the token (although we did caveat proceed with caution if you start exporting private keys).

I feel part 2 might be in the works. We spoke about using Azure Automation with Certificate Based Authentication and I would like to give some examples of how that can be achieved. We may also explore how to create the certificate directly in Azure so there is no need to export/import private keys (which always adds a degree of security concern).

(3160)