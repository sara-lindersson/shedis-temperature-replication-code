Concatenate and Tile Files
================
Sara Lindersson
2024-10-14

This script concatenates and tiles yearly files containing daily maximum
(TX) and minimum (TN) temperature data.

``` r
library(glue)
library(knitr)
```

``` r
# Set working directory
knitr::opts_knit$set(root.dir = normalizePath('F:/mswx/data-processed/')) 
```

``` r
tx_input_files <- '02/tx/tx_*.nc'
tx_obase <- '03/tx_tiles/tx_'
tn_input_files <- '02/tn/tn_*.nc'
tn_obase <- '03/tn_tiles/tn_'
```

``` r
# Create the CDO command for TX files
tx_command <- glue(
  'cdo -O -z zip_1 -b 32 -distgrid,24,12 -cat [ {tx_input_files} ] {tx_obase}'
)

# Create the CDO command for TN files
tn_command <- glue(
  'cdo -O -z zip_1 -b 32 -distgrid,24,12 -cat [ {tn_input_files} ] {tn_obase}'
)

# Run the CDO commands
system(tx_command)
system(tn_command)
```

End of script.
