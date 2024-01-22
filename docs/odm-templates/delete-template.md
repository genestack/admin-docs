# Delete template

This article describes how to delete templates in ODM.

## Requirements

-   Python 3
-   pip
-   Genestack Python client installed. See [how to setup Genestack python client](../packages-to-install/genestack-python-client.md)
-   Auxiliary scripts installed. See [how to install Genestack auxiliary scripts](../packages-to-install/genestack-auxiliary-scripts.md)

!!!warning

    1. Currently the Default template CAN be deleted, which may cause issues, so please be careful.
    2. Only users with the “Manage organization” permission can delete templates.
    3. The script doesn’t check that the file with the provided accession actually exists so if nothing is deleted but the script runs correctly it will still output 'Success'.
    4. The script doesn’t check the type of the file so if a study’s accession is provided instead of a template’s accession the study will be deleted. However, for deletion of studies please use the script from [how to delete a study](../working-with-study/deleting-study.md)

## Instructions

1. Before a template deletion all the studies which have this template set should be manually changed: another template which is not going to be deleted should be applied (for example, Default template). Apply template manually via the UI.
2. Run delete template script and follow its login instructions, replacing the host name with the name of the instance the script will apply to. The script will print “Success” or an error stacktrace in case of an error.
    ```shell
    odm-delete-template --accession GSF244345 -H GENESTACK_ENDPOINT_ADDR
    ```
    Where `GENESTACK_ENDPOINT_ADDR` is the URL of the Genestack platform.
