# How to use AAD Access Token in Connect-MgGraph? | Pankaj Surti's Blog
[How to use AAD Access Token in Connect-MgGraph? | Pankaj Surti's Blog](https://pankajsurti.com/2022/10/07/how-to-use-aad-access-token-in-connect-mggraph/) 

 [How to use AAD Access Token in Connect-MgGraph?](https://pankajsurti.com/2022/10/07/how-to-use-aad-access-token-in-connect-mggraph/)
-------------------------------------------------------------------------------------------------------------------------------------

Summary

The Microsoft Graph PowerShell SDK is a great and simpler ways to get [MS Graph API PowerShell](https://learn.microsoft.com/en-us/powershell/microsoftgraph/?view=graph-powershell-1.0) code working quickly. But what I have found the source code and example to utilize the X509 certificate ways of authentication. For doing a quick demo with the Azure AD security token there a simple way which I will describe here in this post.

Script example

The tip is very simple. Since Connect-MgGraph does not have Client Secret parameter, use the Invoke-RestMethod to get the access token. Once valid token is received pass it to the Connect-MgGraph and make the rest of the other MS Graph SDK calls after that.

See in the following example I have used the Get-MgGroup call after successfully connecting to MS Graph.

```
# The following command only required one time execution
if ( Get-ExecutionPolicy)
{
    Write-Host "RemoteSigned policy exists."
}
else
{
    Write-Host "RemoteSigned policy does not exist."
    **Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser**
}

if (Get-Module -ListAvailable -Name Microsoft.Graph) {
    Write-Host "Microsoft.Graph Module exists"
} 
else {
    Write-Host "Microsoft.Graph Module does not exist"
    **Install-Module Microsoft.Graph -Scope AllUsers**
}

# Populate with the App Registration details and Tenant ID
$ClientId          = "TODO"
$ClientSecret      = "TODO" 
$tenantid          = "TODO" 
$GraphScopes       = "https://graph.microsoft.com/.default"


$headers = @{
    "Content-Type" = "application/x-www-form-urlencoded"
}

$body = "grant_type=client_credentials&client_id=$ClientId&client_secret=$ClientSecret&scope=https%3A%2F%2Fgraph.microsoft.com%2F.default"
$authUri = "https://login.microsoftonline.com/$tenantid/oauth2/v2.0/token"
$response = **Invoke-RestMethod** $authUri  -Method 'POST' -Headers $headers -Body $body
$response | ConvertTo-Json
 
$token = $response.access_token
 
# Authenticate to the Microsoft Graph
**Connect-MgGraph -AccessToken $token**

# If you want to see debugging output of the command just add **"-Debug"** to the call.
**Get-MgGroup -Top 10**
```

Conclusion

I hope this helps you. I use this technique to quickly check / test the calls to the MS Graph.

Note: Please make sure your Azure AD app has required permission applied and consented or else you would get “Insufficient privileges to complete the operation.” error.

Also use the MS Graph explorer as UI ways to test your API and check required permission.

[https://aka.ms/GE](https://aka.ms/GE)

```
PS C:\WINDOWS\system32> **Get-MgUser -Top 10**
Get-MgUser : **Insufficient privileges to complete the operation.**
At line:1 char:1
+ Get-MgUser -Top 10
+ ~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: ({ ConsistencyLe...ndProperty =  }:<>f__AnonymousType59`9) [Get-MgUser_List1], RestException`1
    + FullyQualifiedErrorId : Authorization_RequestDenied,Microsoft.Graph.PowerShell.Cmdlets.GetMgUser_List1

PS C:\WINDOWS\system32> 
```

![](https://1.gravatar.com/avatar/4ca649982306f48fa9fab04edcb5c419?s=60&d=identicon&r=G)

About Pankaj
------------

I am a Developer and my linked profile is https://www.linkedin.com/in/pankajsurti/

This entry was posted in [MS Graph](https://pankajsurti.com/category/technical-stuff/ms-graph/), [Technical Stuff](https://pankajsurti.com/category/technical-stuff/). Bookmark the [permalink](https://pankajsurti.com/2022/10/07/how-to-use-aad-access-token-in-connect-mggraph/ "Permalink to How to use AAD Access Token in Connect-MgGraph?").