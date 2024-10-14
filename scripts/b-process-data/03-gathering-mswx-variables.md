Gathering Meteorological Data
================
Sara Lindersson
2024-10-14

This script gathers the 3-hourly meteorological data variables from
[MSWX](https://doi.org/10.1175/BAMS-D-21-0145.1) into single R-objects.
It then calculates additional variables using the R package
[HeatStress](https://github.com/anacv/HeatStress).

The output from this script is one R object per event and subnational
unit, containing 3-hourly data on the following variables (at the grid
cell level):  
+ Air temperature  
+ Wind speed  
+ Relative humdity  
+ Apparent temperature  
+ Effective temperature  
+ Wet bulb temperature, following Stull’s method  
+ The simplified wet bulb globe temperature

``` r
library(knitr)
library(here)
library(tidyverse)
library(sf)
library(terra)
library(exactextractr)
library(HeatStress)
```

``` r
# Execute all notebook chunks in the grandparent folder of the notebook
knitr::opts_knit$set(root.dir = normalizePath(".."))
```

### Define parameters

``` r
variables <- c('temp', 'wind', 'relhum')
ph_out <- here('data-processed','temporary','03')

# Check if output-directory exists, otherwise creates it
if (!file.exists(ph_out)) {
  dir.create(ph_out, recursive = T)
}
```

### Import subnational disaster records

Loads the output from script *01-emdat-gdis-link.Rmd*.

``` r
subnat <- readRDS(here(
  'data-processed', 'subnat.rds'
  ))
```

### Data processing and export

This section loads the output from the script
*02-extracting-mswx-data.Rmd*. The script iterates over each subnational
unit and subsequently loops over the variables to consolidate this data
into a single dataframe. After calculating the additional variables, the
script exports the final results.

``` r
# Loop over the events and subnational units
for (i in 1:nrow(subnat)){
  subnat_i <- subnat[i,]
  disno_i <- subnat_i$disno
  gadm_gid_i <- subnat_i$gadm_gid
  file_i <- paste0(disno_i,'_',gadm_gid_i,'.rds')
  
  # Loop over each variable
  for (n in 1:length(variables)){
    vr_n <- variables[n]
    ph_n <- here('data-processed', 'temporary', '02', vr_n)
    
    x_i <- readRDS(file = paste0(ph_n, '/', file_i)) %>%
      pivot_longer(
        cols = -c(x, y, coverage_fraction),
        names_to = 'time',
        values_to = vr_n
      ) %>%
      mutate(
        time = as.POSIXct(
          time,
          format = paste0('%Y%j', '.', '%H'),
          tz = 'GMT'
        )
      ) %>%
      rename(cf = coverage_fraction)
    
    if (n == 1){
      data_i <- x_i
    } else {
      data_i <- left_join(
        data_i,
        x_i,
        by = c('x', 'y', 'cf', 'time')
      )
    }
  }
  
  # Calculate additional variables
  data_i <- data_i %>%
    mutate(
      at = HeatStress::apparentTemp(temp, relhum, wind),
      et = HeatStress::effectiveTemp(temp, relhum, wind),
      wbt = HeatStress::wbt.Stull(temp, relhum),
      swbgt = HeatStress::swbgt(temp, relhum)
    )
  
  # Export R-object
  saveRDS(data_i, file = file.path(ph_out, paste0(disno_i, '_', gadm_gid_i, '.rds')))
}
```

End of script.
