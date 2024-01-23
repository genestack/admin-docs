# How to delete a study

## Requirements

-   Python 3
-   pip
-   The user should have the "Manage organisation” permission and have API token. See [Getting a Genestack API token](https://odm-user-guide.readthedocs.io/en/latest/doc-odm-user-guide/getting-a-genestack-api-token.html#token-label)
-   Genestack Python client installed. See [how to setup Genestack python client](../packages-to-install/genestack-python-client.md)
-   Auxiliary scripts installed. See [how to install Genestack auxiliary scripts](../packages-to-install/genestack-auxiliary-scripts.md)

## Instructions

!!!warning Warning

    1. Only users with the “Manage organisation” permission can delete studies.
    2. The script doesn’t check that the file with the provided accession actually exists so if nothing is deleted but the script runs correctly it will still output 'Success'.
    3. The script doesn’t check the type of the file so if a template’s accession is provided instead of a study’s accession the template will be deleted. However, for deletion of templates please use the script from How to delete a template

Run the script and follow its login instructions, replacing the host name with the name of the instance the script will apply to. The script will print “Success” or an error stacktrace if there’s an error.

```shell
odm-delete-study --accession GSF244344 -H HOSTNAME
```
