---
title: "Using Service to Service Authentication"
description: Service-to-service authentication enables external services to connect as an application, without impersonating normal users.
author: henrikwh
ms.custom: na
ms.date: 08/23/2022
ms.reviewer: na
ms.suite: na
ms.tgt_pltfrm: na
ms.topic: conceptual
ms.author: jswymer
---
 
# Using Service-to-Service (S2S) Authentication 

[!INCLUDE[azure-ad-to-microsoft-entra-id](~/../shared-content/shared/azure-ad-to-microsoft-entra-id.md)]

Service-to-Service (S2S) authentication is suited for scenarios where integrations are required to run without any user interaction. S2S authentication uses the [Client Credentials OAuth 2.0 Flow](/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow). This flow enables you to access resources by using the identity of an application.

> [!NOTE]
> For more information about OAuth 2.0 flows, see [OAuth 2.0 and OpenID Connect protocols on the Microsoft identity platform](/azure/active-directory/develop/active-directory-v2-protocols) in the Microsoft Entra ID documentation.


In contrast, OAuth delegate flows, like [authorization code](/azure/active-directory/develop/v2-oauth2-auth-code-flow), [implicit grant flow](/azure/active-directory/develop/v2-oauth2-implicit-grant-flow) and [resource owner password credentials](/azure/active-directory/develop/v2-oauth-ropc) can be configured to require multifactor authentication (MFA). This configuration prevents integration from running unattended, because MFA is required to acquire the access token from Microsoft Entra ID. 

## Feature availability

The following table describes in which versions S2S authentication was made available for online or on-premises environments.

|API  |Online |On-premises|
|-----|------|-----------|
|Business Central automation APIs | version 17.0 | versions 18.11 and 19.5 |
|Business Central APIs v1.0| version 18.3 | versions 18.11 and 19.5 |
|Business Central APIs v2.0| version 18.3 | versions 18.11 and 19.5 |
|Business Central custom APIs| version 18.3 | versions 18.11 and 19.5 |
|Business Central web services| version 18.3 | versions 18.11 and 19.5 |

## Main usage scenarios

Two main scenarios are enabled with S2S authentication:

1. Company setup using automation API

    Automation APIs provide capability for automating company setup through APIs. The automation APIs are used to hydrate tenants, that is, to bring them to an initial state.

    The **D365 Automation** entitlements give access to APIs in the `/api/microsoft/automation` route by using the OAuth client credentials flow. An application token with the `Automation.ReadWrite.All` scope is needed for accessing [!INCLUDE[prod_short](../developer/includes/prod_short.md)] Automation APIs.

2. External user and non-interactive user access to APIs and web services.

    S2S authentication enables both external user and non-interactive user access to Business Central online. Refer to [license guide](https://www.microsoft.com/licensing/product-licensing/dynamics365) for scenarios and usage. An application token with the `API.ReadWrite.All` scope is needed for accessing [!INCLUDE[prod_short](../developer/includes/prod_short.md)] APIs and web services.  

> [!NOTE]
> When you use S2S authentication, you can now use the integration session to create scheduled tasks. This option is available only on versionw 21.2 and later and only online.

## Business Central on-premises prerequisite

Business Central on-premises must be configured for Microsoft Entra authentication with OpenID Connect.

> [!IMPORTANT]
> The `ValidAudiences` parameter of the [!INCLUDE [prod_short](../developer/includes/prod_short.md)] must include the endpoint `https://api.businesscentral.dynamics.com`. If it doesn't, you'll get the error `Authentication_InvalidCredentials` on API requests, or the error `securitytokeninvalidaudienceexception` in the application log when you try to download symbols from Visual Studio.

For more information, go to [Configure Microsoft Entra authentication with OpenID Connect](authenticating-users-with-azure-ad-openid-connect.md).

## Set up service-to-service authentication

To set up service-to-service authentication, you'll have to do two things:

- Register an application in your Microsoft Entra tenant for authenticating API calls against [!INCLUDE[prod_short](../developer/includes/prod_short.md)].

- Grant access for that application in [!INCLUDE[prod_short](../developer/includes/prod_short.md)].

These tasks are described in the sections that follow. 

## Task 1: Register a Microsoft Entra application for authentication to Business Central

Complete these steps to register an application in your Microsoft Entra tenant for service-to-service authentication.

1. Sign in to the [Azure portal](https://portal.azure.com).

2. Register an application for [!INCLUDE [prod_short](../developer/includes/prod_short.md)] in Microsoft Entra tenant.

    Follow the general guidelines at [Register your application with your Microsoft Entra tenant](/azure/active-directory/active-directory-app-registration).

    When you add an application to a Microsoft Entra tenant, you must specify the following information:

    |Setting|Description|
    |-------|-----------|
    |Name|Specify a unique name for your application. |
    |Supported account types| Select either <strong>Accounts in this organizational directory only (Microsoft only - Single tenant)</strong> or <strong>Accounts in any organizational directory (Any Microsoft Entra ID directory - Multitenant)</strong>.|
    |Redirect URI|(optional) This setting is only required if you want to use the Business Central web client to grant consent to the API (see task 2). Otherwise, you can grant consent using the Azure portal.<br><br>To specify the redirect URL, set the first box to **Web** to specify a web application. Then, enter the URL for your Business Central on-premises browser client, followed by *OAuthLanding.htm*, for example: `https://MyServer/BC210/OAuthLanding.htm` or `https://cronus.onmicrosoft.com/BC210/OAuthLanding.htm`. This file is used to manage the exchange of data between Business Central and other services through Microsoft Entra ID.<br> <br>**Important:** The URL must match the URL of Web client, as it appears in the browser address. For example, even though the actual URL might be `https://MyServer:443/BC210/OAuthLanding.htm`, the browser typically removes the port number `:443`.|

    When completed, an **Overview** displays in the portal for the new application.

    > [!NOTE]
    > Copy the **Application (client) ID** of the registered application. You'll need this later. You can get this value from the **Overview** page.

3. Create a client secret for the registered application as follows:

    1. Select **Certificates & secrets** > **New client secret**.
    2. Add a description, select a duration, and select **Add**.

    > [!NOTE]
    > Copy the secret's value for use in your client application code. This secret value is never displayed again after you leave this page.

    For the latest guidelines about adding client secrets in Microsoft Entra ID, see [Add credentials ](/azure/active-directory/develop/quickstart-register-app#add-credentials) in the Azure documentation.

4. Grant the registered application **API.ReadWrite.All** and **Automation.ReadWrite.All** permission to the **Dynamics 365 [!INCLUDE [prod_short](../developer/includes/prod_short.md)]** API as follows:

    1. Select **API permissions** > **Add a permission** > **Microsoft APIs**.
    2. Select **Dynamics 365 [!INCLUDE [prod_short](../developer/includes/prod_short.md)]**.
    3. Select **Application permissions**, select **API.ReadWrite.All** and **Automation.ReadWrite.All**, then select **Add permissions**.

        The **API permissions** page will include one of the following entries:

        |API / Permission name|Type|Description|
        |---------------------|----|-----------|
        |Dynamics 365 Business Central / Automation.ReadWrite.All|Application|Full access to automation|
        |Dynamics 365 Business Central / API.ReadWrite.All|Application|Access to APIs and webservices|

        For the latest guidelines about adding permissions in Microsoft Entra ID, see [Add permissions to access your APIs](/azure/active-directory/develop/quickstart-configure-app-access-web-apis#add-permissions-to-access-your-web-api) in the Azure documentation.

    4. (optional) Grant admin consent on each permission by selecting it in the list, then selecting **Grant admin consent for \<tenant name\>**.

        This step isn't required if you'll be granting consent from the Business Central web client in task 2.

## Task 2: Set up the Microsoft Entra application in [!INCLUDE[prod_short](../developer/includes/prod_short.md)]

Complete these steps to set up the Microsoft Entra application for service-to-service authentication in [!INCLUDE[prod_short](../developer/includes/prod_short.md)]. 

1. In the [!INCLUDE[prod_short](../developer/includes/prod_short.md)] client, search for **Microsoft Entra applications**  and open the page.

2. Select **New**.

    The **Microsoft Entra application Card** opens.

3. In the **Client ID** field, enter the **Application (Client) ID**  for the registered application in Microsoft Entra ID from task 1. 

4. Fill in the **Description** field. If this application is set up by a partner, please enter sufficient partner-identifying information, so all applications set up by this partner can be tracked in the future if necessary.

5. Set the **State** to **Enabled**.

6. Assign permissions to objects as needed.

   For more information, [Assign Permissions to Users and Groups](/dynamics365/business-central/ui-define-granular-permissions).

   > [!IMPORTANT]
   > Applications can't be assigned the **SUPER** permission set. Make sure that applications follows least-privilege principle and only assign permissions required for the integration to work.

   > [!NOTE]
   > The system permission sets and user groups called **D365 AUTOMATION** and **EXTEND. MGT. - ADMIN** provide access to most typical objects used with automation.
   >
   > The **EXTEND. MGT. - ADMIN** permission set was introduced in Business Central 2021 release wave 1 as a replacement for the **D365 EXTENSION MGT** permission set in earlier versions.

7. (optional) Select **Grant Consent** and follow the wizard. 

    This step will grant consent to the API. This step is only required if you haven't granted consent from the Azure portal in task 1. You can only complete this step if you've configured a redirect URL in the registered Microsoft Entra app.

   > [!TIP]
   > Pre-consent can be done by adding the Microsoft Entra application to the **Adminagents** group in the partner tenant.  For more information, see [Pre-consent your app for all your customers](/graph/auth-cloudsolutionprovider#pre-consent-your-app-for-all-your-customers) in the Graph documentation.

## Calling API and web services OAuth2Flows

After the Microsoft Entra application has been set up and access has been granted, you're ready to make API and web service calls to [!INCLUDE[prod_short](../developer/includes/prod_short.md)].

For most cases, use the `AcquireTokenByAuthorizationCode` method from the OAuth 2.0 module. For more information, see [Microsoft identity platform and OAuth 2.0 authorization code flow](/azure/active-directory/develop/v2-oauth2-auth-code-flow). To explore an example, see [OAuth2Flows](https://github.com/microsoft/BCTech/blob/master/samples/OAuth2Flows/TestOAuth2Flows.Page.al).

> [!IMPORTANT]
> When getting access tokens, it's important to keep security in mind. For example, ensure that you don't expose the tokens. You can do that in two ways:
>
>* Make the method you are using non-debuggable. Here's an example of how to use the non-debuggable property for protecting access tokens:
>
>```
>    [NonDebuggable]
>    procedure GetAccessToken(var AccessToken: Text): Boolean
>    begin
>        ...
>    end;
>
>```
>
>* Set the `showMyCode` property to false for the extension.

> [!TIP]
> You can also see this sample in the [BCTech Github repo](https://github.com/microsoft/BCTech/tree/master/samples/VSCRestClientOAuthBCAccess).

The following sample uses the [Rest Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) for Visual Studio Code using the Client Credentials OAuth 2.0 flow. Using the Rest Client makes it easy to see which HTTP calls are made both against [!INCLUDE[prod_short](../developer/includes/prod_short.md)] and Azure Active Directory. Any HTTP client can be used to create the requests below. Or you can choose any library, like MSAL. Remember to specify the scope as `https://api.businesscentral.dynamics.com/.default` and the Token URL as `https://login.microsoftonline.com/<tenantId>/oauth2/v2.0/token`.

```http
@tenantId = <tenant id>
@clientId = <client id>
@clientSecret = <client secret>
@baseUri = https://api.businesscentral.dynamics.com
@scope = {{baseUri}}/.default
@bcEnvironmentName = production
@url = {{baseUri}}/v2.0/{{bcEnvironmentName}}/api/v2.0

### Define entity, like customers, items, or vendors
@entityName =customers 

# @name auth
POST https://login.microsoftonline.com/{{tenantId}}/oauth2/v2.0/token HTTP/1.1
Content-type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id={{clientId}}
&client_secret={{clientSecret}}
&scope={{scope}}

### Variable Response
@accessHeader = Bearer {{auth.response.body.$.access_token}}

# @name GetCompanies
GET {{url}}/companies HTTP/1.1
Authorization: {{accessHeader}}

### Variable Response
@companyId = {{GetCompanies.response.body.value.[0].id}}
@companyUrl = {{url}}/companies({{companyId}})
@displayName = MyItemDisplayName2

### Get entities

# @name GetEntities
GET {{companyUrl}}/{{entityName}} HTTP/1.1
Authorization: {{accessHeader}}

### Create entity

# @name CreateEntity
POST {{companyUrl}}/{{entityName}} HTTP/1.1
Content-Type:  application/json
Authorization: {{accessHeader}}

{
 "displayName" : "{{displayName}}"
}

### Get created entity

# @name GetCreatedEntity
GET {{companyUrl}}/{{entityName}}/?$filter=displayName eq '{{displayName}}' HTTP/1.1
Authorization: {{accessHeader}}

#### Variable Response
@entityId = {{GetCreatedEntity.response.body.value.[0].id}}

### Modify entity

# @name ModifyEntity
PATCH {{companyUrl}}/{{entityName}}({{entityId}})
Authorization: {{accessHeader}}
Content-Type:  application/json
If-Match: *

{
    "displayName" : "DeleteMe-{{displayName}}"
}

### Delete entity

# @name DeleteItem
Delete {{companyUrl}}/{{entityName}}({{entityId}})
Authorization: {{accessHeader}}
```

## See Also
[OAuth2 and Microsoft Entra ID](/azure/active-directory/develop/active-directory-v2-protocols)  
[Client Credentials flow/S2S using MSAL library](/azure/active-directory/develop/scenario-daemon-overview)  
[C# samples using Client Credentials flow](https://github.com/Azure-Samples/active-directory-dotnetcore-daemon-v2)  
[OAuth 2.0 client credentials flow on the Microsoft identity platform](/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow)  
[Samples and libraries for OAuth: Microsoft identity platform authentication libraries](/azure/active-directory/develop/reference-v2-libraries)  
[Business Central Repository on GitHub - PowerShell samples using MSAL](https://github.com/microsoft/BCTech/tree/master/samples/PSOAuthBCAccess)  
[Business Central API v2.0](/dynamics365/business-central/dev-itpro/api-reference/v2.0/)
