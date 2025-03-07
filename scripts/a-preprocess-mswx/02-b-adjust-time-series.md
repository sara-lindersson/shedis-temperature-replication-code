Adjust time series
================
Sara Lindersson
2024-10-14

This script adjusts the time series by:  
+ Removing February 29 to handle leap years  
+ Selecting the last 15 days of 1980 and the first 15 days of 2011,
which are necessary for calculating percentiles using a 30-day moving
window for the period 1981–2010

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
# Define the range of leap years
leap_years <- seq(1980, 2008, by = 4)

# File pathways
ph_tn <- '02/tn'
ph_tx <- '02/tx'
```

### Remove Feb 29 in TX files

``` r
for (year in leap_years) {
  # Define input and output paths for TX
  input_file_tx <- glue('{ph_tx}/tx_{year}.nc')
  output_file_tx <- glue('{ph_tx}/tx_{year}.nc')
  
  # Construct the CDO command for TX
  cdo_command_tx <- glue('cdo -O -z zip_1 -del29feb {input_file_tx} {output_file_tx}')
  
  # Execute the CDO command for TX
  system(cdo_command_tx)
}
```

### Remove Feb 29 in TN files

``` r
for (year in leap_years) {
  # Define input and output paths for TN
  input_file_tn <- glue('{ph_tn}/tn_{year}.nc')
  output_file_tn <- glue('{ph_tn}/tn_{year}.nc')
  
  # Construct the CDO command for TN
  cdo_command_tn <- glue('cdo -O -z zip_1 -del29feb {input_file_tn} {output_file_tn}')
  
  # Execute the CDO command for TN
  system(cdo_command_tn)
}
```

### Select dates of 1980 and 2011

``` r
# Adjust TX for 1980 (timestep 351/365)
input_file_tx_1980 <- glue('{ph_tx}/tx_1980.nc')
output_file_tx_1980 <- glue('{ph_tx}/tx_1980.nc')
cdo_command_tx_1980 <- glue('cdo -O -z zip_1 -seltimestep,351/365 {input_file_tx_1980} {output_file_tx_1980}')
system(cdo_command_tx_1980)

# Adjust TX for 2011 (timestep 1/15)
input_file_tx_2011 <- glue('{ph_tx}/tx_2011.nc')
output_file_tx_2011 <- glue('{ph_tx}/tx_2011.nc')
cdo_command_tx_2011 <- glue('cdo -O -z zip_1 -seltimestep,1/15 {input_file_tx_2011} {output_file_tx_2011}')
system(cdo_command_tx_2011)

# Adjust TN for 1980 (timestep 351/365)
input_file_tn_1980 <- glue('{ph_tn}/tn_1980.nc')
output_file_tn_1980 <- glue('{ph_tn}/tn_1980.nc')
cdo_command_tn_1980 <- glue('cdo -O -z zip_1 -seltimestep,351/365 {input_file_tn_1980} {output_file_tn_1980}')
system(cdo_command_tn_1980)

# Adjust TN for 2011 (timestep 1/15)
input_file_tn_2011 <- glue('{ph_tn}/tn_2011.nc')
output_file_tn_2011 <- glue('{ph_tn}/tn_2011.nc')
cdo_command_tn_2011 <- glue('cdo -O -z zip_1 -seltimestep,1/15 {input_file_tn_2011} {output_file_tn_2011}')
system(cdo_command_tn_2011)
```

End of script.
