Concatenate 3-hourly files by year
================
Sara Lindersson
2024-10-14

This script processes 3-hourly data files using CDO (Climate Data
Operators) to concatenate individual files into a single netCDF file for
each year. The output consists of one netCDF file per year, containing
3-hourly time steps.

``` r
library(glue)
library(knitr)
```

``` r
# Set working directory
knitr::opts_knit$set(root.dir = normalizePath('F:/mswx/')) 
```

### Parameters

``` r
# Define the range of years
years <- 1979:2023

# File pathways
ph_in <- 'temp/3hourly'
ph_out <- 'data-processed/01'

# Check if output-directory exists, otherwise create it
if (!file.exists(ph_out)) {
  dir.create(ph_out, recursive = T)
}
```

### Loop over years and construct shell command

``` r
for (year in years) {
  # Define input and output paths
  input_files <- glue('{ph_in}/{year}*.nc')
  output_file <- glue('{ph_out}/temp_{year}.nc')
  
  # Construct the CDO command
  # -O to overwrite output if existing, -z to compress output
  cdo_command <- glue('cdo -O -z zip_1 -cat {input_files} {output_file}')
  
  # Execute the CDO command
  system(cdo_command)
}
```

End of script.
