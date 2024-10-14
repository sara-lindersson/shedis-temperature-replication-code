Extracting Meteorological Data
================
Sara Lindersson
2024-10-14

This script extracts 3-hourly meteorological data from
[MSWX](https://doi.org/10.1175/BAMS-D-21-0145.1) for each subnational
unit and the specified period of interest.

The outputs from this script include three R objects for each event and
subnational unit, containing 3-hourly data on the following variables
(at the grid cell level):  
+ Air temperature  
+ Wind speed  
+ Relative humdity

``` r
library(knitr)
library(here)
library(tidyverse)
library(sf)
library(terra)
library(exactextractr)
```

``` r
# Execute all notebook chunks in the grandparent folder of the notebook
knitr::opts_knit$set(root.dir = normalizePath(".."))
```

### Define parameters

``` r
variables <- c(
  'temp',
  'wind',
  'relhum'
)

ph_in <- c(
  temp = 'F:/mswx/temp/3hourly',
  wind = 'F:/mswx/wind/3hourly',
  relhum = 'F:/mswx/relhum/3hourly'
)
```

### Import subnational disaster records

Loads the output from script *01-emdat-gdis-link.Rmd*.

``` r
subnat <- readRDS(here(
  'data-processed','subnat.rds'
  ))
```

### Extract meteorological data

To extract the meteorological data, the script will first loop over the
specified variables and then iterate through the subnational units to
extract the 3-hourly data within the polygon for the extended period of
interest *T<sub>ex</sub>*.

``` r
# Loop over variables
for (j in 1:length(variables)){
  var_j <- variables[j]
  ph_j <- ph_in[[var_j]] 
  
  # Loop over rows in subnat
  for (i in 1:nrow(subnat)){
    subnat_i <- subnat[i,]
    disno_i <- subnat_i$disno
    gadm_gid_i <- subnat_i$gadm_gid
    
    start_i <- as.POSIXct(paste0(subnat_i$start_ex, '00:00:00'), tz = 'GMT')
    end_i <- as.POSIXct(paste0(subnat_i$end_ex, '21:00:00'), tz = 'GMT')
    time_seq_i <- seq(start_i, end_i, by = 60*60*3)
    
    files_i <- as.list(
      format(time_seq_i, format = paste0(ph_j, '/', '%Y%j', '.', '%H', '.nc'))
    )
    
    # Read the first nc-file to get the coordinates and coverage fraction
    r <- terra::rast(paste0(files_i[1]))
    x_i <- exact_extract(r, subnat_i$geometry, include_xy = T)[[1]] %>%
      select(-value)
    
    # Loop over all time stamps
    for (n in 1:length(files_i)){
      file_n <- paste0(files_i[n])
      time_n <- sub('\\.nc$', '', basename(file_n)) #  timestamp from filename
      r <- terra::rast(paste0(file_n)) # Read raster
      
      x_n <-
        # Extract cell values within geometry
        exact_extract(r, subnat_i$geometry, include_xy = F)[[1]] %>%
        # Drop coverage fraction column
        select(-coverage_fraction) %>%
        # Assign time stamp as column name
        rename(!!time_n := value) # The !! unquotes the variable
      
      x_i <- cbind(x_i, x_n) # Save values
    }
    
    # Export 3-hourly data for variable and polygon
    ph_out <- here('data-processed', 'temporary', '02', var_j)
    
    # Check if directory exists, otherwise create it
    if (!file.exists(ph_out)) {
      dir.create(ph_out, recursive = T)
    }
    # Export extracted data
    saveRDS(x_i, file = file.path(ph_out, paste0(disno_i, '_', gadm_gid_i, '.rds')))
  }
}
```

End of script.
