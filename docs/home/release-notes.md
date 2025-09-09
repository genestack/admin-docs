# Release notes

## Version 1.60

!!! tip ""
    Helm chart version 1.60.1

### Rclone Migration

The following ODM component called `funcFile` was replaced with `rclone`.

As a result of this migration, the configuration of storages was moved from `funcFile` to `application.yaml` files in `core`, `applications`, and `funcJob`.
You can find configuration examples in the "examples" directory within the Helm chart.
Note that the configuration section `genestack.rclone` in all three of these services should be identical. For this purpose, we recommend using YAML anchors, which are also included in the examples.

Also, Rclone allows to use the AWS IAM role instead of the AWS IAM user. If this is relevant fo your environment, then information on deployment can be found [here](./../clouds/aws.md) in paragraph 4.

### SAML elimitaion

Support of SAML was eliminated.

### Helm configuration changes

- The `credentials` section has been removed, please use the `rclone` configuration instead.

    ```yaml
    credentials:
      awsS3Region:
      awsS3AccessKey:
      awsS3SecretAccessKey:
    ```

- All configuration related to `SAML` has been removed. Example of configuration that can be safely removed:

    ```yaml
    core:
      files:
        "/var/lib/genestack/properties/application.yaml":
          saml:
            entity:
              id: "<SAML_ENTITY_ID>"
            identity:
              provider:
                url: "<SAML_IDENTITY_PROVIDER_URL>"
                certificate:
                  name: "<SAML_IDENTITY_PROVIDER_CERTIFICATE_NAME>"
        "/var/lib/genestack/properties/authentication/saml/idp-certificates/<SAML_IDENTITY_PROVIDER_CERTIFICATE_NAME>": "<IDP_CERTIFICATE>"
        "/var/lib/genestack/properties/authentication/saml/sp-credentials/sp_pkcs8_der.key": "<SP_PKCS8_DER_KEY>"
        "/var/lib/genestack/properties/authentication/saml/sp-credentials/sp_x509_pem.crt": "<SP_X509_PEM_CRT>"
    
    applications:
      files:
        "/var/lib/genestack/properties/application.yaml":
          authentication:
            saml:
              enabled: "true"
    ```

- The `region` parameter has been removed from the `applications` configuration.

    ```yaml
    applications:
      files:
        "/var/lib/genestack/properties/application.yaml":
          frontend:
            aws:
              region:
    ```
