Derive daily Tmax and Tmin
================
Sara Lindersson
2024-10-14

This script calculates the daily minimum (Tmin) and maximum (Tmax) air
temperatures from 3-hourly temperature data. The outputs are two netCDF
files per year, each containing daily time steps:  
+ tn: daily minimum temperature (Tmin)  
+ tx: daily maximum temperature (Tmax)

``` r
library(glue)
library(knitr)
```

``` r
# Set working directory
knitr::opts_knit$set(root.dir = normalizePath('F:/mswx/data-processed/')) 
```

### Parameters

``` r
# Define the range of years
years <- 1980:2011

# File pathways
ph_in <- '01'
ph_out_tn <- '02/tn'
ph_out_tx <- '02/tx'

# Check if output-directories exist, otherwise create
if (!file.exists(ph_out_tn)) {
  dir.create(ph_out_tn, recursive = T)
}

if (!file.exists(ph_out_tx)) {
  dir.create(ph_out_tx, recursive = T)
}
```

### Loop over years and calculate TX

``` r
for (year in years) {
  # Define input and output paths for TX
  input_file <- glue("{ph_in}/temp_{year}.nc")
  output_file_tx <- glue("{ph_out_tx}/tx_{year}.nc")
  
  # Construct the CDO command for TX
  # -O to overwrite output if existing, -z to compress output
  cdo_command_tx <- glue("cdo -O -z zip_1 -daymax {input_file} {output_file_tx}")
  
  # Execute the CDO command for TX
  system(cdo_command_tx)
}
```

### Loop over years and calculate TN

``` r
for (year in years) {
  # Define input and output paths for TN
  input_file <- glue("{ph_in}/temp_{year}.nc")
  output_file_tn <- glue("{ph_out_tn}/tn_{year}.nc")
  
  # Construct the CDO command for TN
  cdo_command_tn <- glue("cdo -O -z zip_1 -daymin {input_file} {output_file_tn}")
  
  # Execute the CDO command for TN
  system(cdo_command_tn)
}
```

End of script.
