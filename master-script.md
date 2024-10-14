Master Script
================
Sara Lindersson
2024-10-14

This master script orchestrates the execution of all scripts in the
correct order within their respective subfolders. It ensures a
streamlined workflow by coordinating the processes and dependencies
among the various scripts.

``` r
library(here)
```

``` r
# Define folder paths
preprocess_mswx_folder <- here('scripts', 'a-preprocess-mswx/')
process_data_folder <- here('scripts', 'b-process-data/')
derive_outputs_folder <- here('scripts', 'c-derive-outputs/')
```

### Function to source scripts in specific order

Assumes that the scripts are ordered 01-, 02-, 03-, etc.

``` r
source_scripts_in_order <- function(folder) {
  # List script files in the specified folder, sorted by file name
  scripts <- list.files(folder, pattern = '^\\d{2}-.*\\.Rmd$', full.names = TRUE)
  
  # Source each script in order
  for (script in scripts) {
    message('Sourcing script: ', script)
    source(script)
  }
}
```

### Run scripts

Uncomment to run. The subfolders need to be run in the specified order.

``` r
# Run scripts that preprocess the raw MSWX data
# source_scripts_in_order(preprocess_mswx_folder)

# Run scripts to link data from the various data sources and compile into daily statistics
# source_scripts_in_order(process_data_folder)

# Run scripts that create the dataset outputs
# source_scripts_in_order(derive_outputs_folder)
```

End of script.
