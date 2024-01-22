# How to share a study programmatically

## Requirements

-   Python 3
-   pip
-   The user should be a member of Curator group and have API token. See [Getting a Genestack API token](https://odm-user-guide.readthedocs.io/en/latest/doc-odm-user-guide/getting-a-genestack-api-token.html#token-label)
-   Genestack Python client installed. See [how to setup Genestack python client](../packages-to-install/genestack-python-client.md)
-   Auxiliary scripts installed. See [how to install Genestack auxiliary scripts](../packages-to-install/genestack-auxiliary-scripts.md)

## Instructions

!!!note "Please note that there are several limitations:"

    1. The script should be launched under the study owner user.
    2. The script will fail if the user attempts to share studies with a group name that is used more than once in groups they are a member of (ie, group names should not be duplicated).

1. Prepare the data for script: you need accessions of the studies you want to share and the names of the user groups with which you want to share they studies.
2. Run the script and follow its login instructions, replacing the host name with the name of the instance the script will apply to. The script will print “Success” or an error stacktrace if there is an error.

```shell
odm-share-study --study_accession GSF013340 --group_name GroupName -H HOST
```

If a group name contains spaces use quotes around group name:

```shell
odm-share-study --study_accession GSF000745 --group_name 'Group name with space' -H HOST
```
