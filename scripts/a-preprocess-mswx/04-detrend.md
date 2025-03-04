Detrend Time Series
================
Sara Lindersson
2024-10-14

This script detrends the time series of daily maximum (TX) and minimum
(TN) temperature values.

``` r
library(knitr)
```

``` r
# Set working directory
knitr::opts_knit$set(root.dir = normalizePath('F:/mswx/data-processed/')) 
```

``` r
# Define the range of leap years
leap_years <- seq(1980, 2008, by = 4)

# File pathways
ph_in_tn <- '03/tn_tiles'
ph_in_tx <- '03/tx_tiles'

ph_out_tn <- '04/tn_tiles_detr/'
ph_out_tx <- '04/tx_tiles_detr/'
```

``` r
# List all files in TX directory
files <- list.files(ph_in_tx)

# Loop over files and detrend
for (file in files) {
  # Input and output file paths
  in_file <- file
  out_file <- paste0(ph_out_tx, basename(file))
  
  # CDO command for detrending
  command <- paste('wsl cdo -O -P 8 -z zip_1 -b 32 -detrend', in_file, out_file)
  
  # Run the command using shell
  shell(cmd = command)
}

# List all files in TN directory
files <- list.files(ph_in_tn)

# Loop over files and detrend
for (file in files) {
  # Input and output file paths
  in_file <- file
  out_file <- paste0(ph_out_tn, basename(file))
  
  # CDO command for detrending
  command <- paste('wsl cdo -O -P 8 -z zip_1 -b 32 -detrend', in_file, out_file)
  
  # Run the command using shell
  shell(cmd = command)
}
```

End of script.
