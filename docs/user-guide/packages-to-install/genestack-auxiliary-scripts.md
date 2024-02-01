# Genestack auxiliary scripts

## How to set up Genestack auxiliary scripts

These scripts provide various auxiliary functions to simplify the work with Omics Data Manager.

## Requirements

- Python 3
- pip

## Instructions

### Start a console/terminal and install Genestack Auxiliary scripts

```shell
# Install the latest version
python3 -m pip install \
      --extra-index-url https://public-nexus.devops.gs.team/repository/pypi-releases/simple \
      genestack-auxiliary-scripts

# OR: Install a specific version starting from 1.53.12, usually the same as ODM version, e.g. 1.54.0
python3 -m pip install \
      --extra-index-url https://public-nexus.devops.gs.team/repository/pypi-releases/simple \
      genestack-auxiliary-scripts==1.53.12
```

### To check the existing version, and view all available console commands, type

```shell
python3 -m pip show --verbose genestack-auxiliary-scripts
```

### You can always remove the package with a help of this command

```shell
python3 -m pip uninstall genestack-auxiliary-scripts
```
