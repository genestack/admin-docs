# Google Workspace setup

This section explains how to integrate ODM with Google Workspace to use Google as OpenID Connect provider.

[Official documentation](https://developers.google.com/identity/openid-connect/openid-connect#appsetup)

## Create Google OAuth keys

1. Go to [Google Console](https://console.developers.google.com/apis/credentials)

2. Click **Create Credentials**, then click **OAuth Client ID** in the drop-down menu

3. Enter the following:

    - **Application Type**: Web Application

    - **Name**: Specify a name for your app

    - **Authorized JavaScript Origins**: `https://ODM-HOST`

    - **Authorized Redirect URLs**: `https://ODM-HOST/frontend/endpoint/google/back`

4. Click **Create**

5. Copy the `Client ID` and `Client Secret` from the **OAuth Client** modal

!!! tip ""
    Now you can configure ODM to use Google as OpenID Connect provider. You can find configuration examples in the Helm chart.
