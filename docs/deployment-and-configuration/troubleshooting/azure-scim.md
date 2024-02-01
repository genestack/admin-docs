# Problems with Azure SCIM provisioning

- `SystemForCrossDomainIdentityManagementCredentialValidationUnavailable` - The server is unavailable for Azure to start synchronization (this process is initiated by Azure). You need to make ODM accessible from the Internet (or through internal networks if you are using Azure) so that Azure SCIM provisioning can connect to it.
