# Azure AD setup

This section explains how to enable user and group provisioning from AzureAD into ODM via SCIM protocol.

!!! attention "Attention"
    Before proceeding with SCIM configuration, make sure you have [set up OpenID](../openid/azure.md).

## Configure user and group provisioning

ODM exposes SCIM API endpoints at `https://ODM-HOST/frontend/rs/genestack/scim-integration/default-released/scim`. AzureAD requires access to these API endpoints in order to provision users and groups into ODM.

If your ODM instance is directly accessible from AzureAD servers, you can use the SCIM endpoint above when configuring provisioning in AzureAD. Otherwise, you need to enable access via HTTP protocol from AzureAD servers to the ODM SCIM endpoints, and then use the corresponding external SCIM endpoint URL as configured.

Below are configuration steps for AzureAD portal:

1. Navigate to **Enterprise applications –> All applications –> _{YOUR-APPLICATION}_ –> Provisioning**

2. Press **"Get started"** button and configure provisioning settings:

    1. Set **"Provisioning Mode"** to "Automatic"

    2. Set **"Tenant URL"** to the ODM SCIM endpoint URL mentioned above

    3. Set **"Secret Token"** to a Genestack API token generated for a dedicated ODM service account

    4. Press **"Test Connection"** button

    5. If the test is successful, press **"Save"** button; otherwise ensure your configuration is correct, refer to ODM Administration Guide, or contact ODM support for assistance.

3. Before the first synchronisation we recommend you check the list of groups and users in ODM and add the same users to the same groups in AD.

4. Press **"Start provisioning"** button
