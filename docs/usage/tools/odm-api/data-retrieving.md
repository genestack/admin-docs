# Usage of Python and R API-SDK

ODM APIs empower you to do large-scale, cross-study, and cross-omics analysis on-the-fly.
The ODM OpenAPI Specification can be reviewed at `GENESTACK_ENDPOINT_ADDR/swagger/helper/`,
where `GENESTACK_ENDPOINT_ADDR` is the URL of the Genestack platform.

In addition to that you can also use Python and R SDKs, so you can write your own script, notebook or application.

## Requirements
- Installed odm-sdk. See [Installation odm-sdk](./installation.md)

### Example of usage

Script to search through samples and display them in table format:

```python
    import os
    import pandas as pd
    import sample_curator

    os.environ['PRED_SPOT_HOST'] = 'GENESTACK_ENDPOINT_ADDR'
    os.environ['PRED_SPOT_TOKEN'] = 'GENESTACK_TOKEN'
    os.environ['PRED_SPOT_VERSION'] = 'default-released'

    api = sample_curator.SampleSPoTApi()
    query = '"Disease"="Healthy" OR "Disease"="Alzheimer Disease"'
    samples = api.search_samples(filter=query, page_offset=0)
    print(pd.DataFrame.from_dict(samples.data[:5]))
```

The output:

```text
  genestack:accession Sample Source ID Sample Name Organism     Sex  Disease   Age Age Unit Tissue Cell Type  ... Family.ID Donor Treatment/Treatment Name Specimen Type Tissue or Cell Type Sample Source Continental Group Code Sample Type Population Code Sample ID Continental Group
0           GSF136813          HG00096        None     None    Male  Healthy  None     None   None      None  ...   HG00096                    Not treated           LCL               Blood         NHGRI                    EUR         DNA             GBR   HG00096       "European "
1           GSF136814          HG00097        None     None  Female  Healthy  None     None   None      None  ...   HG00097                    Not treated           LCL               Blood         NHGRI                    EUR         DNA             GBR   HG00097       "European "
2           GSF136815          HG00099        None     None  Female  Healthy  None     None   None      None  ...   HG00099                    Not treated           LCL               Blood         NHGRI                    EUR         DNA             GBR   HG00099       "European "
3           GSF136816          HG00100        None     None  Female  Healthy  None     None   None      None  ...   HG00100                    Not treated           LCL               Blood         NHGRI                    EUR         DNA             GBR   HG00100       "European "
4           GSF136817          HG00101        None     None    Male  Healthy  None     None   None      None  ...   HG00101                    Not treated           LCL               Blood         NHGRI                    EUR         DNA             GBR   HG00101       "European "
```

## R SDK

### How to install

Similar to python, you can install small libraries for each ODM endpoint.

Current libraries list: - studyCurator - studyUser - sampleCurator - sampleUser - libraryCurator - libraryUser - preparationCurator - preparationUser - variantCurator - variantUser - expressionCurator - expressionUser - flow-cytometryCurator - flow-cytometryUser - integrationCurator - integrationUser - tasks

Example of library installation:

```R
# Install the latest version
genestackRepo <- "https://public-nexus.devops.gs.team/repository/r-releases"
install.packages("sampleCurator", repos = genestackRepo)

# OR: Install a specific version starting from 1.53.12, usually the same as ODM version, e.g. 1.54.0
genestackRepo <- "https://public-nexus.devops.gs.team/repository/r-releases"
options("repos" = c(
  GENESTACK = genestackRepo)
  )

install_specific_version_from_nexus <- function(pkgs, version = NULL, ...) {
  # Build the package name
  pkg_name <- paste0(pkgs, "_", version, ".tar.gz")
  # build the url knowing it should be in root /src/contrib
  for (repo in getOption("repos")) {
    url <- paste(repo, "src/contrib", pkg_name, sep = "/")
    # try to download
    try <- tryCatch({
      path <- file.path(tempdir(), pkg_name)
      suppressWarnings(download.file(url, path, mode = "wb"))},
      # catch the error
      error = function(e) 1L
    )
  }

  # error result means that specific version is not available
  if (try == 1L) stop("\nError: ", pkgs, " not available in version ", version, call. = FALSE)
  on.exit(unlink(path))

  # if no error, install the package using local path
  install.packages(path, repos=NULL, ...)
}

odmVersion <- "1.53.12"
install_specific_version_from_nexus("sampleCurator", version = odmVersion)
```

### Example of usage

Script to search through samples and display them in table format:

```R
Sys.setenv(PRED_SPOT_HOST = 'GENESTACK_ENDPOINT_ADDR',
           PRED_SPOT_TOKEN = 'GENESTACK_TOKEN',
           PRED_SPOT_VERSION = 'default-released')

library(sampleCurator)
query <- '"Disease"="Healthy" OR "Disease"="Alzheimer Disease"'
samplesObject <- sampleCurator::SampleSPoTApi_search_samples(filter=query)
samplesObject$content$data[1:5]
```

The output (shorten):

```text
  genestack:accession Sample Source ID Sample Name Organism     Sex
1           GSF136813          HG00096        None     None    Male
2           GSF136814          HG00097        None     None  Female
3           GSF136815          HG00099        None     None  Female
4           GSF136816          HG00100        None     None  Female
5           GSF136817          HG00101        None     None    Male
```

**Note:** To successfully run the example you might need to install additional packages such as
`R6` and `httr`. You can do it with a commands like this:

```R
install.packages('R6', repos = "http://cran.us.r-project.org")
install.packages('httr', repos = "http://cran.us.r-project.org")
```
