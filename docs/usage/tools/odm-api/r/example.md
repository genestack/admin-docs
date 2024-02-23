# Usage of generated R API Client

## Requirements

- Installed odm-api. See [Installation odm-sdk](./installation.md)

### Example of usage

Script to search through samples and display them in table format:

```R
Sys.setenv(PRED_SPOT_HOST = 'GENESTACK_ENDPOINT_ADDR',
           PRED_SPOT_TOKEN = 'GENESTACK_TOKEN',
           PRED_SPOT_VERSION = 'default-released')

library(odm_api)
query <- '"Disease"="Healthy" OR "Disease"="Alzheimer Disease"'
samplesObject <- odm_api::SampleSPoTApiAsCurator_search_samples(filter=query)
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
