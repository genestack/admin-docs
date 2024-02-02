# Okta setup

This section explains how to integrate ODM with Okta to use Okta as OpenID Connect provider.

[Official documentation](https://help.okta.com/en-us/content/topics/apps/apps_app_integration_wizard_oidc.html)

## Launch the Wizard

1. In the Admin Console, go to **Applications** -> **Applications**.

2. Click Create **App Integration**.

3. To create an OIDC app integration, select **OIDC - OpenID Connect** as the **Sign-in method**.

4. Choose the type of application to integrate with Okta. Select **Web Application**.

5. Click Next.

## Configure initial settings

1. In **General Settings**

    - **App integration name**: Specify a name for your app integration

    - **Grant type**: select **Authorization Code** and **Interaction Code**

    - **Sign-in redirect URIs**: `https://ODM-HOST/frontend/endpoint/okta/back`

2. Next, you can follow the [official instructions](https://help.okta.com/en-us/content/topics/apps/apps_app_integration_wizard_oidc.html).

## Configure OIDC settings

- Copy the **Client ID** and the **Client Secret** from **General** page.

!!! tip ""
    Now you can configure ODM to use Okta as OpenID Connect provider. You can find configuration examples in the Helm chart.
