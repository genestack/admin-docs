# Installation of generated R API Client

## Requirements

- R

## Instructions

### Install

```R
# Install the latest version
genestackRepo <- "https://public-nexus.devops.gs.team/repository/r-releases"
install.packages("odm-api", repos = genestackRepo)

# OR: Install a specific version starting from 1.53.12, usually the same as ODM version, e.g. 1.54.0
genestackRepo <- "https://public-nexus.devops.gs.team/repository/r-releases"
options("repos" = c(GENESTACK = genestackRepo))

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
install_specific_version_from_nexus("odm-api", version = odmVersion)
```
