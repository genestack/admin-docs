# Google Workspace setup

This section explains how to integrate ODM with Google Workspace to use Google as SAML Identity provider.

[Official documentation](https://apps.google.com/supportwidget/articlehome?article_url=https%3A%2F%2Fsupport.google.com%2Fa%2Fanswer%2F6087519&assistant_id=generic-unu&product_context=6087519&product_name=UnuFlow&trigger_context=a)

## Create Google SAML app

1. In the [GSuite Admin Cosole](https://admin.google.com/ac/home?hl=en), go to **Apps** -> **Web and mobile apps**

2. Click **Add app** -> **Add custom SAML app**

    1. **App details**

        - **App name**: Specify a name for your app

    2. **Google Identity Provider details**

        - Copy **SSO URL** and **Entity ID**

        - Download the **certificate**, it will be useful for [setting up ODM](#certificate-preparation)

    3. **Service provider details**

        - **ACS URL**: `https://ODM-HOST/frontend/endpoint/AssertionConsumer`

        - **Entity ID**: `https://ODM-HOST/frontend/endpoint/SamlSpMetadata`

        - **Start URL**: `https://ODM-HOST/`

        - **Signed Response**: `off` (see below)

        - **Name ID**

            - **Name ID format**: `EMAIL`

            - **Name ID**: `Basic Information / Primary Email`

    4. **Attribute mapping**

        - **Attributes**: Click **Add Mapping** to define the following attributes

            - `Basic Information / Primary Email` -> `urn:mace:dir:attribute-def:mail`

            - `Basic Information / First Name` -> `urn:mace:dir:attribute-def:givenName`

            - `Basic Information / Last Name` -> `urn:mace:dir:attribute-def:sn`

    5. Click **Finish**

    6. You’ll be redirected to “Application settings" page.

        - Click **User access** and then change **Service status** to **ON for everyone**. **SAVE** the change.

!!! warning

    As of Mar 4, 2020 Signed Response checkbox behaves weirdly:

    - when it is `off`, response itself is unsigned, but assertions are signed

    - when the checkbox is `on`, response becomes signed, but assertions for some reason are unsigned

    We do not support the latter combination, hence in Google Suite SAML Signed Response option should be turned off.

## Certificate preparation

You need to create the necessary certificates for ODM.

1. In step [2b](#create-google-saml-app), you downloaded the certificate.

    1. Convert it to `idp.key`.

        ```shell
        openssl x509 -outform der -in downloaded_certificate.pem -out idp.key
        ```

    2. Convert `idp.key` to base64 format

        ```shell
        cat idp.key | base64
        ```

2. Create new certificates

    1. Generate `sp_x509_pem.crt`

        - `ODM-HOST` is the server name without http/https, for example, `odm.example.com`.

        ```shell
        export SAML_SP_HOSTNAME="ODM-HOST"

        openssl req \
        -x509 \
        -nodes \
        -newkey rsa:2048 \
        -keyout sp_pem.key \
        -out sp_x509_pem.crt \
        -subj "/C=UK/ST=England/L=Cambridge/O=Genestack/OU=Genestack/CN=${SAML_SP_HOSTNAME}" \
        -days 3650
        ```

    2. Generate `sp_pkcs8_der.key`

        ```shell
        openssl pkcs8 \
        -topk8 \
        -inform PEM \
        -outform DER \
        -in sp_pem.key \
        -out sp_pkcs8_der.key \
        -nocrypt
        ```

    3. Convert `sp_x509_pem.crt` to base64 format

        ```shell
        cat sp_x509_pem.crt | base64
        ```

    4. Convert `sp_pkcs8_der.key` to base64 format

        ```shell
        cat sp_pkcs8_der.key | base64
        ```

    5. Update the Helm chart with the provided information.

!!! tip ""
    Now you can configure ODM to use Google as SAML Identity provider. You can find configuration examples in the Helm chart.
