# Enable the MCP server

The ODM MCP server lets AI assistants that support the [Model Context Protocol](https://modelcontextprotocol.io) (MCP) search and read ODM data for the user who is signed in to them. It ships with the ODM Helm chart from release 1.65 and is off by default.

## How it works

- The MCP server runs next to ODM and is reached through ODM's own address: MCP clients connect to `https://ODM-HOST/mcp`.

- Users sign in with Microsoft Entra ID (Azure AD), through the application ODM already uses for Microsoft sign-in. For every request, the MCP server exchanges the user's token for one scoped to ODM (On-Behalf-Of), so the assistant sees exactly what the user can see in ODM, no more. MCP clients never receive the Microsoft tokens.

- Its tools only search and read: an assistant cannot change data in ODM through it.

## Requirements

- ODM 1.65 or later, deployed with the ODM Helm chart.

- Microsoft sign-in configured for ODM as [Azure AD setup](../single-sign-on/openid/azure.md) describes, including [Configure OAuth resource server](../single-sign-on/openid/azure.md#configure-oauth-resource-server): ODM accepts access tokens for the `api://{clientId}/default` scope. The MCP server expects the default Application ID URI, `api://{clientId}`. If ODM's scope has another name than `default`, set `mcp.envVars.ODM_SCOPE` to it in [Configure the Helm chart](#configure-the-helm-chart).

## Configure the Azure AD application

The MCP server uses the application ODM uses. Navigate to **Azure Active Directory –> App registrations –> _{YOUR-APPLICATION}_** and do the following:

1. Under **"Authentication"** menu item, in the **"Web"** platform configuration ODM already has:

    1. Press **"Add URI"** and specify the MCP server redirect URI, `https://ODM-HOST/auth/callback`

    2. Press **"Save"** button

2. Under **"Expose an API"** menu item, add a second scope next to `default`:

    1. Press **"Add a scope"** button under "Scopes defined by this API" section

    2. Set **"Scope name"** to `user_impersonation`

    3. Set **"Who can consent?"** to "Admins and users"

    4. Set **"Admin consent display name"** to "Access ODM through an AI assistant"

    5. Set **"Admin consent description"** to "Allows an AI assistant to read the signed-in user's data in ODM"

    6. Set **"User consent display name"** to "Access ODM through an AI assistant"

    7. Set **"User consent description"** to "Allows an AI assistant to read your data in ODM"

    8. Set **"State"** to "Enabled"

    9. Press **"Add scope"** button

3. Under **"API permissions"** menu item:

    1. Press **"Add a permission"** button under "Configured permissions" section

        1. Select **"My APIs"**, then _{YOUR-APPLICATION}_. If you are not an owner of the application, find it by its name under **"APIs my organization uses"** instead

        2. Select **"Delegated permissions"**

        3. Select `default` and `user_impersonation`

        4. Press **"Add permissions"** button

    2. Press **"Grant admin consent for _{YOUR ORGANIZATION}_"** button under "Configured permissions" section

    With admin consent, users do not have to consent to these scopes themselves, which many organizations do not allow, and the MCP server can exchange their tokens for ODM ones.

4. The MCP server needs a client secret of this application. The `clientSecret` ODM uses will do, or add a new one under **"Certificates and secrets"** menu item.

## Configure the Helm chart

1. Add the tenant ID, application (client) ID and client secret of that application to your values:

    ```yaml
    mcp:
      enabled: true
      envVars:
        AZURE_TENANT_ID: "<TENANT_ID>"
        AZURE_CLIENT_ID: "<CLIENT_ID>"
        AZURE_CLIENT_SECRET: "<CLIENT_SECRET>"
    ```

    To keep the secret out of your values, create a Kubernetes Secret with those three keys in the ODM namespace and name it instead:

    ```yaml
    mcp:
      enabled: true
      envVarsFromSecrets:
        - odm-mcp-creds
    ```

    You can find both examples, `mcp.yaml` and `mcp-external-secret.yaml`, in the `examples` directory of the Helm chart.

    !!! warning
        With `envVars`, the chart refuses to render if any of the three is missing. It cannot read a Secret, so with `envVarsFromSecrets`, make sure the Secret holds all three keys: without them the MCP server does not start, or no user can sign in. Do not put `ODM_API_TOKEN` or `API_TOKEN` in the Secret either: with one, the MCP server lets anyone who reaches it act as that token's user, and the chart cannot check the Secret for it.

2. Upgrade ODM with the values file you deploy it with, now holding the `mcp` section, for example:

    ```bash
    helm upgrade <RELEASE> <ODM_CHART> \
      --version <VERSION> \
      --namespace <NAMESPACE> \
      --values custom-values.yaml
    ```

    This adds an `mcp` Deployment and Service, and routes these paths on `https://ODM-HOST` to it: `/mcp`, and the OAuth endpoints MCP clients sign in through, `/.well-known/oauth-authorization-server`, `/.well-known/oauth-protected-resource`, `/authorize`, `/token`, `/register`, `/consent` and `/auth/callback`.

3. Check that it answers:

    ```bash
    curl https://ODM-HOST/.well-known/oauth-protected-resource/mcp
    ```

    The response names `https://ODM-HOST/mcp` as the resource and `user_impersonation` as its scope. This shows that ODM routes to the MCP server, not that users can sign in.

4. Connect an AI assistant yourself, as the ODM user documentation at `https://ODM-HOST/user-docs/explore/connect-an-ai-assistant/` describes, and ask it one question about ODM data. Sign-in and its answer show that the Azure AD application and the client secret are right.

!!! tip ""
    Now give your users the address `https://ODM-HOST/mcp` and the user documentation page above.

## Good to know

- **Network access.** An MCP client that runs on the user's machine, such as Claude Code or VS Code, reaches ODM as the browser does. An assistant hosted by an AI provider connects from the provider's servers, so those servers must reach `https://ODM-HOST` too.

- **Where the data goes.** What an assistant reads from ODM is sent to the language model behind it, and so to that model's provider.

- **Sign-ins do not survive a pod replacement.** The MCP server keeps registered clients and sign-ins in its pod. When the pod is replaced, during an upgrade for example, users sign in again.

- **No API tokens.** The MCP server only runs with user sign-in here: the chart refuses `ODM_API_TOKEN` in `mcp.envVars`, since with a token everyone would act as that one user.
