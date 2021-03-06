# AdobeUMInterface

A PowerShell module for communicating with Adobe's User Management API. **Warning**, if you used prior versions of these functions, they are not compatible with recent changes. Most functions have been updated to use the [non-deprecated API functions.](https://adobe-apiplatform.github.io/umapi-documentation/en/api/DeprecatedApis.html)

## General Request Pattern

Each request sent to Adobe can be split up into 2 parts
1) You have an identity that you are trying to act on. (A user or group for example)
2) You have actions that you want to perform on the identity.

An identity and a list of actions together, make a full request. Keep this in mind when you look at the powershell functions.

In general, you can run the New-\*Action functions, add those to an array, then pass them on to the New-\*Request functions. A list of New-\*Request functions can then be sent to adobe for processing with the Send-UserManagementRequest.
I encourage you to look at the structure of the returns objects from the New-\*Request functions to get a better understanding.

## General Usage Instructions

1) Create a service account/integration and link it to your User Management binding. (Do this at [Adobe's Console](https://console.adobe.io))
2) Create a PKI certificate. You can create a self signed one using the provided New-Cert command
3) Export the PFX and a public certificate from your generated certificate. 
4) Upload the public cert to the account/integration you created in step 1.
5) Using the information adobe gave you in step 1, call New-ClientInformation and provide it the necessary information. (APIKey, ClientSecret, etc)
6) Load the PFX certificate. You can do this with the provided Load-PFXCert function.
7) Call Get-AdobeAuthToken, to request an auth token from adobe, this validates further requests to adobe.
8) Utilize the other functions, and your ClientInformation variable to make further queries against your Adobe users.

A complete example of the calls you should make after step 5 are below. This script below, creates a "add to group" action and then passes that to a "Create User" request. Then that request is sent to adobe for processing.

```powershell
#Load the Auth cert generated with New-Cert
$SignatureCert = Load-PFXCert -Password "MyPassword" -CertPath "C:\Certs\AdobeAuthPrivate.pfx"

#Client info from https://console.adobe.io/
$ClientInformation = New-ClientInformation -APIKey "1234123412341234" -OrganizationID "1234123412341234@AdobeOrg" -ClientSecret "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx" `
    -TechnicalAccountID "12341234123412341234B@techacct.adobe.com" -TechnicalAccountEmail "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx6@techacct.adobe.com"

#Required auth token for further adobe queries. (Is placed in ClientInformation)
Get-AdobeAuthToken -ClientInformation $ClientInformation -SignatureCert $SignatureCert

#Add a new user, and add them to a group in one adobe request
$GroupAddAction = New-GroupAddAction -Groups "All Apps Users"
$Request = New-CreateUserRequest -FirstName "John" -LastName "Doe" -Email "John.Doe@domain.com" -AdditionalActions $GroupAddAction

#Send the generated request to adobe
Send-UserManagementRequest -ClientInformation $ClientInformation -Requests $Request
```

## Sync with an AD Group

The end-goal I had in mind for these was to automatically sync an AD group with Adobe's portal. This allows easy delegation to a service desk. The group used can also be tied in with AppLocker, and automatic deployments could use the same group in SCCM or another application management utility. The general pattern is as follows:

1) Create an active directory group that your Adobe Users will be added to
2) Create another group Adobe-side
3) Get the Adobe group ID by running Get-AdobeGroups
4) Create a script as below, and assign it to a scheduled task to run periodically

```powershell
#Import Module
$ScriptDir = split-path -parent $MyInvocation.MyCommand.Definition
Import-Module "$ScriptDir\AdobeUMInterface.psm1"

#Load cert for auth
$SignatureCert = Import-PFXCert -Password "MyPassword" -CertPath "$ScriptDir\Private.pfx"

#Client info from https://console.adobe.io/
$ClientInformation = New-ClientInformation -APIKey "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" -OrganizationID "xxxxxxxxxxxxxxxxxxxxxxxx@AdobeOrg" -ClientSecret "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" `
    -TechnicalAccountID "xxxxxxxxxxxxxxxxxxxxxxxx@techacct.adobe.com" -TechnicalAccountEmail "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx@techacct.adobe.com"

#Required auth token for further adobe queries. (Is placed in ClientInformation)
Get-AdobeAuthToken -ClientInformation $ClientInformation -SignatureCert $SignatureCert

#Sync AD Group to Adobe
#You can get the groupid by running Get-AdobeGroups
$Request = New-SyncADGroupRequest -ADGroupName "My-AD-Group" -AdobeGroupName "All Apps Users" -ClientInformation $ClientInformation

#ToReview, uncomment
#Write-Host ($Request | ConvertTo-JSON -Depth 10)

#Send the generated request to adobe
Send-UserManagementRequest -ClientInformation $ClientInformation -Requests $Request
```

You can examine the individual steps generated by examining the $Request object first.

## Additional Information

[Adobe.io UM API Documentation](https://adobe-apiplatform.github.io/umapi-documentation/en/RefOverview.html)

[Adobe.io Authentication Docs](https://www.adobe.io/authentication/auth-methods.html)

[JWT.io Java Web Token Documentation](https://jwt.io/)
