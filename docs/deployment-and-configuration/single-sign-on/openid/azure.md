# Azure AD setup

This section explains how to integrate ODM with AzureAD to use AzureAD tenant as OpenID Connect provider, OAuth 2.0 server.

## Create ODM application configuration in AzureAD

1. Go to [Azure Portal App Registration](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredApps)

2. Navigate to **Enterprise applications –> All applications**

3. Press **"New application"** button, then press **"Create your own application"** button

4. Assign the application name, select **"Non-gallery"** application type, and press **"Create"** button

!!! warning
    Do not create the application via **Azure Active Directory –> App registrations**, because applications created that way do not support SCIM user provisioning.

## Configure OpenID Connect client

1. Navigate to **Azure Active Directory –> App registrations –> _{YOUR-APPLICATION}_**

2. Under **"Overview"** menu item:

    1. Copy the **"Application (client) ID"** value. You will need to save this alue as `clientId` ODM configuration parameter

    2. Press **Endpoints** button in toolbar, and copy **OpenID Connect metadata document** URL. It should look like `https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration`.
        You will need to save it as `discoveryDocumentUri` ODM configuration parameter. Refer to [Fetch the OpenID Connect metadata document](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-protocols-oidc#fetch-the-openid-connect-metadata-document) for details about OpenID Connect discovery.

3. Under **"Branding and properties"** menu item, configure **"Home page URL"** (e.g. as `https://ODM-HOST`) and press **"Save"** button

4. Under **"Authentication"** menu item:

    1. Press **"Add a platform"** button under "Platform configurations" section

        1. Select **"Web"** application type

        2. Specify ODM redirect URI, e.g. `https://ODM-HOST/frontend/endpoint/microsoft/back`

        3. Press **"Configure"** button

    2. Disable all tokens for "Implicit grant and hybrid flows"

    3. Press **"Save"** button

5. Under **"Certificates and secrets"** menu item:

    1. Press **"New client secret"** button under "Client secrets" section, specify the secret description, choose the expiration time, and press **"Add"** button

    2. Find the newly added client secret in the table and copy the client secret value (from **"Value"** column). You will need to save this value as `clientSecret` ODM configuration parameter

6. Under **"API permissions"** menu item:

    1. Press **"Add a permission"** button under "Configured permissions" section

        1. Select **"Microsoft Graph"** application type

        2. Select **"Delegated permissions"**

        3. On the menu **"OpenId permissions"** select: `email`, `openid`, `profile`, `offline_access`

        4. Press **"Add permission"** button

    2. Press **"Grant admin consent for _{YOUR ORGANIZATION}_"** button under "Configured permissions" section

!!! attention "Attention"
    You may want to review settings under "Supported account types" and update
    them as appropriate for your use case. Do not forget to press the **"Save"** button.

!!! tip ""
    Now you can configure ODM to use AzureID as OpenID Connect provider. You can find configuration examples in the Helm chart.

## Configure OAuth resource server

ODM exposes REST APIs which can only be called by other applications. These client applications can use OAuth 2.0 access tokens, issued by Authorization Server, to authorize REST API requests to ODM.

To make it work, you need to establish trust relationships between ODM and AzureAD (it will play the Authorization Server role). You can do this by configuring OpenID Connect integration (see [Configure OpenID Connect client](#configure-openid-connect-client) section above).

After that you need to provide AzureAD with necessary information about ODM. Navigate to **Azure Active Directory –> App registrations –> _{YOUR-APPLICATION}_** and do the following:

1. Set up OAuth scopes for ODM:

    1. Open **"Expose an API"** menu item

    2. Set **"Application ID URI"** if it is not set already:

        1. Press the **"Set"** hyperlink to the right of "Application ID URI" label

        2. Leave the default "Application ID URI" value which is `api://{clientId}`

        3. Press **"Save"** button

    3. Define a single default scope for ODM REST API clients:

        1. Press **"Add a scope"** button under "Scopes defined by this API" section

        2. Set **"Scope name"** to whatever you like, e.g. `default`

        3. Set **"Who can consent?"** to "Admins and users"

        4. Set **"Admin consent display name"** to "Access user data in ODM"

        5. Set **"Admin consent description"** to "Allows the app to read signed-in user's data in ODM"

        6. Set **"User consent display name"** to "Access your data in ODM"

        7. Set **"User consent description"** to "Allows the app to read your data in ODM"

        8. Set **"State"** to "Enabled"

        9. Press **"Add scope"** button

    4. Tell AzureAD which applications should be able to retrieve access tokens from AzureAD to access ODM REST API endpoints:

        1. Press **"Add a client application"** button under "Authorized client applications" section

        2. Set **"Client ID"** to the `clientId` value of the application that will retrieve access tokens from AzureAD to access ODM REST APIs. This client application must have its own App registration in AzureAD where it receives its own `clientId`, different from the `clientId` value used to configure ODM

        3. Select the `api://{clientId}/default` scope from the list of scopes under **"Authorized scopes"**

        4. Press **"Add application"** button

    5. Repeat the previous step for every client application in case you have many of them

2. Configure AzureAD to issue to ODM clients Access Tokens version 2 (otherwise access tokens will not be compatible with configuration from OpenID Connect discovery document; see [Note about access token version](#note-about-access-token-version) section below):

    1. Open **"Manifest"** menu item

    2. Find `accessTokenAcceptedVersion` parameter in the manifest JSON

    3. Set the value of `accessTokenAcceptedVersion` to `2`

    4. Press **"Save"** button

!!! tip ""
    Now AzureAD is configured to issue access tokens with the `api://{clientId}/default` scope to applications that want to access ODM REST APIs using OAuth 2.0 access tokens.

### Note about access token version

ODM is configured to use version 2 of OpenID Connect protocol implementation in AzureAD, as defined by the discovery document URL: `https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration?appId={clientId}`

However, for access tokens AzureAD uses the version 1 format by default, which is incompatible with configuration from v2 OpenID Connect discovery document: `iss` claim in v1 access token format has values like `https://sts.windows.net/{uuid}/` while the expected `iss` claim format in v2 configuration is `https://login.microsoftonline.com/{uuid}/v2.0`.

Therefore, we need to tell AzureAD explicitly to use v2 format of access tokens, which is currently possible only by changing `accessTokenAcceptedVersion` parameter in application manifest manually. Refer to [manifest reference](https://docs.microsoft.com/en-us/azure/active-directory/develop/reference-app-manifest#accesstokenacceptedversion-attribute) for details.

## Configure user and group provisioning

An article about SCIM can be found here [SCIM provisioning](../scim/azure.md)
