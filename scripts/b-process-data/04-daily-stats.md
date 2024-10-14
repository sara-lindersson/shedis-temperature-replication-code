Deriving daily statistics from 3-hourly data
================
Sara Lindersson
2024-10-14

This script derives daily statistics from the 3-hourly meteorological
data. The output from this script consists of one R object per event and
subnational unit, containing daily statistics (minimum, maximum, and
mean) at the grid cell level for the following variables:  
+ Air temperature  
+ Wind speed  
+ Relative humidity  
+ Apparent temperature  
+ Wet bulb temperature, following Stull’s method  
+ The simplified wet bulb globe temperature

``` r
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

### Parameters

``` r
ph_in <- here('data-processed', 'temporary', '03')
ph_out <- here('data-processed', 'temporary', '04')

# Check if output-directory exists, otherwise create it
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

Loads output from script *03-gathering-mswx-variables.Rmd*.

``` r
# Loop over the events and subnational units
for (i in 1:nrow(subnat)){
  subnat_i <- subnat[i,]
  disno_i <- subnat_i$disno
  gadm_gid_i <- subnat_i$gadm_gid
  file_i <- paste0(disno_i,'_',gadm_gid_i,'.rds')
  
  # Load data
  x_i <- readRDS(file = paste0(ph_in, '/', file_i)) %>%
    # Round coordinates
    mutate(
      x = round(x, digits = 2),
      y = round(y, digits = 2)
    )
  
  # Derive grid cell indexes
  key_i <- unique(
    x_i %>% select(x, y, cf)
  ) %>%
    mutate(
      cell_id = row_number()
    )
  
  x_i <- x_i %>%
    # Join grid cell index
    left_join(
      key_i,
      by = c('x','y','cf')
    ) %>%
    # Derive date
    mutate(
      date = date(time)
    ) %>%
    # Group by date and cell index
    group_by(cell_id, date) %>%
    # Calculate statistics per group
    dplyr::summarise(
      temp_max = max(temp, na.rm = T),
      temp_min = min(temp, na.rm = T),
      temp_mean = mean(temp, na.rm = T),
      
      wind_max = max(wind, na.rm = T),
      wind_min = min(wind, na.rm = T),
      wind_mean =mean(wind, na.rm = T),
      
      relhum_max = max(relhum, na.rm = T),
      relhum_min = min(relhum, na.rm = T),
      relhum_mean =mean(relhum, na.rm = T),
      
      at_max = max(at, na.rm = T),
      at_min = min(at, na.rm = T),
      at_mean = mean(at, na.rm = T),
      
      wbt_max = max(wbt, na.rm = T),
      wbt_min = min(wbt, na.rm = T),
      wbt_mean = mean(wbt, na.rm = T),
      
      swbgt_max = max(swbgt, na.rm = T),
      swbgt_min = min(swbgt, na.rm = T),
      swbgt_mean = mean(swbgt, na.rm = T)
    ) %>%
    ungroup() %>%
    
    # Join coordinates and coverage fraction of cell
    left_join(
      key_i,
      by = c('cell_id')
    ) %>%
    select(
      -cell_id
    )
  
  # Export R-object
  saveRDS(x_i, file = file.path(ph_out, paste0(disno_i, '_', gadm_gid_i, '.rds')))
}
```

End of script.
