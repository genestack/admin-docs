# Okta setup

This section explains how to integrate ODM with Okta to use Okta as SAML service provider.

[Official documentation](https://help.okta.com/en-us/content/topics/apps/apps_app_integration_wizard_saml.htm)

## Create Okta SAML app

1. In the Admin Console, go to **Applications** -> **Applications**

2. Click **Create App Integration**

3. Select **SAML 2.0** as the Sign-in method

    1. **General settings**

        - **App name**: Specify a name for your app

    2. **Configure SAML**

        1. **General**

            - **Single sign-on URL**: `https://ODM-HOST/frontend/endpoint/AssertionConsumer`

            - **Audience URI (SP Entity ID)**: `https://ODM-HOST/frontend/endpoint/SamlSpMetadata`

            - **Name ID format**: `EmailAddress`

            - **Application username**: `Email`

        2. **Attribute Statements (optional)**

            - `urn:mace:dir:attribute-def:mail` -> `user.email`

            - `urn:mace:dir:attribute-def:givenName` -> `user.firstName`

            - `urn:mace:dir:attribute-def:sn` -> `user.lastName`

    3. **Feedback**

        - **App type**: This is an internal app that we have created

    4. Click **Finish**

    5. You’ll be redirected to **Sign On** page.

        - In the **SAML Signing Certificates** section, generate a new one or download an existing certificate. It will be needed for configuring ODM.

## Certificate preparation

You need to create the necessary certificates for ODM.

1. In step [E](#create-okta-saml-app), you downloaded the certificate.

    1. Convert it to `idp.key`.

        ```shell
        openssl x509 -outform der -in okta.cert -out idp.key
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
    Now you can configure ODM to use Okta as SAML service provider. You can find configuration examples in the Helm chart.
