Extract Percentile Thresholds and Link to Daily Data
================
Sara Lindersson
2025-03-04

This script extracts climatology thresholds for air temperature for each
grid cell and day of the year, joining this information with the daily
statistics of meteorological and population variables. The thresholds of
interest are:  
+ pct10: the 10th percentile of the daily minimum air temperature  
+ pct05: the 5th percentile of the daily minimum air temperature  
+ pct90: the 90th percentile of the daily maximum air temperature  
+ pct95: the 95th percentile of the daily maximum air temperature

The output from this script is one aggregated dataframe per *disno* and
*subdivision*, with daily information of meteorological statistics,
population numbers and pct thresholds. The script also removes the
temporary files produced from scripts 02-05.

``` r
library(here)
library(tidyverse)
library(sf)
library(terra)
library(exactextractr)
library(stringr)
library(glue)
```

``` r
# Execute all notebook chunks in the grandparent folder of the notebook
knitr::opts_knit$set(root.dir = normalizePath(".."))
```

### Parameters

``` r
# Percentiles
tn <- c('10', '05') # For cold waves
tx <- c('90', '95') # For heat waves

# File pathways
ph_in <- here('data-processed', 'temporary', '05')
ph_out <- here('data-processed', 'daily-data')

# Check if output-directory exists, otherwise create it
if (!file.exists(ph_out)) {
  dir.create(ph_out, recursive = T)
}

# File pathway to percentiles
ph_pct <- 'F:/mswx/data-processed/06/percentiles'
```

### Import subnational disaster records

Loads the output from script *01-emdat-gdis-link.Rmd*.

``` r
subnat <- readRDS(here(
  'data-processed', 'subnat.rds'
))
```

### Data processing and final export of daily time series

``` r
setwd(ph_pct)

# Loop over subnational records
for (i in 1:nrow(subnat)){
  subnat_i <- subnat[i,]
  disno_i <- subnat_i$disno
  gadm_gid_i <- subnat_i$gadm_gid
  file_i <- paste0(disno_i,'_',gadm_gid_i,'.rds')
  x_i <- readRDS(file = paste0(ph_in, '/', file_i))
  
  # Adjust day of year if leap year
  x_i <- x_i %>%
    mutate(
      leap = leap_year(date),
      doy = yday(date),
      doy = as.integer(
        ifelse(leap == T & doy >= 60, doy-1, doy)
        )
    )
  
  # Choose corresponding percentiles from disaster type
  if (subnat_i$type == 'Cold wave'){
    pcts <- tn
    stat <- 'tn'
  }
  if (subnat_i$type == 'Heat wave'){
    pcts <- tx
    stat <- 'tx'
  }
  
  # Loop over percentiles
  for (pct in pcts){
    
    # Read raster with percentiles
    r <- terra::rast(
      glue('c{stat}{pct}pct.nc'))
    
    r_dat <- exactextractr::exact_extract(
      r,
      subnat_i$geometry,
      include_xy = T
    )[[1]] %>%
      dplyr::select(
        -coverage_fraction
      ) %>%
      pivot_longer(
        cols = -c(x, y),
        names_to = 'doy',
        values_to = paste0('c', stat, pct)
      ) %>%
      mutate(
        # Remove redundant text
        doy = as.integer(gsub('air_temperature_', '', doy)),
        # Round coordinates
        x = round(x, digits = 2),
        y = round(y, digits = 2)
      )
    
    # Join with daily statistics 
    x_i <- x_i %>%
      left_join(r_dat, by = c('x','y','doy'))
  }
  
  # Drop temporary variables
  x_i <- x_i %>%
    dplyr::select(
      -doy,
      -leap
    )
  
  # Export R object
  saveRDS(x_i, file = file.path(ph_out, paste0(disno_i,'_',gadm_gid_i,'.rds')))
}
```

### Delete temporary files

The script now removes the temporary files generated by the data
processing scripts 02-05.

``` r
# unlink(here('data-processed', 'temporary'), recursive = T)
```

End of script.
