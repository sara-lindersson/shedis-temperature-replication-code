Attributes at the Administrative Unit Level
================
Sara Lindersson
2024-10-14

This script derives attributes at the administrative unit level for each
disaster number. The outputs from this script include:  
+ *data-adm-unit-level.csv*: This file contains the derived data
attributes.  
+ *data-adm-unit-level.gpkg*: This file includes both the data
attributes and a simplified polygon of the corresponding administrative
unit (sourced from GADM v 3.6).

``` r
library(here)
library(tidyverse)
library(sf)
library(exactextractr)
library(stringr)
library(glue)
library(purrr)
```

``` r
# Execute all notebook chunks in the grandparent folder of the notebook
knitr::opts_knit$set(root.dir = normalizePath(".."))
```

### Parameters

``` r
# File path in hazard and population data
ph_in <- here('data-processed', 'daily-data')

# File path for event data
ph_events <- here('data-output','events','adm-unit-level', 'pct-5-95')

# File path output
ph_out <- here('data-output')
```

### Import subnational disaster records

Loads the output from script *01-emdat-gdis-link.Rmd*.

``` r
subnat <- readRDS(here('data-processed', 'subnat.rds')) %>%
  select(
    iso3c,
    country,
    disno,
    type,
    gadm_gid,
    gadm_level,
    gadm_name,
    area_km2,
    start_ex,
    end_ex
  ) %>%
  rename(
    adm_unit_area_km2 = area_km2,
    analysis_start = start_ex,
    analysis_end = end_ex
  )
```

### Gather general attributes from MSWX and GHS

``` r
# Define a function to process each file based on subnat record
process_attributes <- function(disno, gadm_gid) {
  
  df <- readRDS(file = glue('{ph_in}/{disno}_{gadm_gid}.rds')) %>%
    select(-x, -y, area_km2) %>%
    group_by(date) %>%
    # Calculate zonal values for all columns
    summarize(across(
      everything(),
      ~ case_when(
        # For 'pop', calculate the sum
        cur_column() == 'pop' ~ sum(.x, na.rm = T),
        # For all other columns, calculate the weighted zonal average based on coverage fraction
        T ~ sum(.x * cf, na.rm = T) / sum(cf, na.rm = T)
      )
    )) %>% 
    select(-cf) %>%
    ungroup() %>%
    arrange(date) 
    
  pop <- df %>%
    summarize(
      adm_unit_pop = first(pop)
    )
  
  df <- df %>%
    summarize(
      # Indicators of air temperature 
      mean_t = round(mean(temp_mean), digits = 2), # Average temp
      mean_tx = round(mean(temp_max), digits = 2), # Average daymax
      mean_tn = round(mean(temp_min), digits = 2), # Average daymin
      
      max_tx = round(max(temp_max), digits = 2), # Max daymax with date
      max_tx_date = date[which.max(temp_max)],
      
      min_tn = round(min(temp_min), digits = 2), # Min daymin with date
      min_tn_date = date[which.min(temp_min)],
      
      max_t_mean = round(max(temp_mean), digits = 2), # Max daily average with date
      max_t_mean_date = date[which.max(temp_mean)],
      
      min_t_mean = round(min(temp_mean), digits = 2), # Min daily average with date
      min_t_mean_date = date[which.min(temp_mean)],
      
      # Indicators of apparent temperature 
      mean_at = round(mean(at_mean), digits = 2),
      mean_at_max = round(mean(at_max), digits = 2),
      mean_at_min = round(mean(at_min), digits = 2),
      
      max_at_max = round(max(at_max), digits = 2),
      max_at_max_date = date[which.max(at_max)],
      
      min_at_min = round(min(at_min), digits = 2),
      min_at_min_date = date[which.min(at_min)],
      
      max_at_mean = round(max(at_mean), digits = 2),
      max_at_mean_date = date[which.max(at_mean)],
      
      min_at_mean = round(min(at_mean), digits = 2),
      min_at_mean_date = date[which.min(at_mean)]
    ) 
  
  result <- cbind(pop, df) %>%
    mutate(
      disno = disno,
      gadm_gid = gadm_gid
    )
  return(result)
}

# Iterate function over rows in subnat
general_attributes <- subnat %>%
  as.data.frame() %>%
  mutate(geometry = NULL) %>%
  select(disno, gadm_gid) %>%
  pmap_dfr(function(disno, gadm_gid) {
    process_attributes(disno, gadm_gid)
  })
```

### Gather attributes from event analysis

``` r
# Function to process each file if it exists
process_events <- function(disno, gadm_gid) {
  
  file_path <- glue('{ph_events}/{disno}.csv')
  
  # Check if the file exists
  if (file.exists(file_path)) {
    
    # Read the csv file and filter it to the corresponding admin unit
    df <- read.csv(file_path) %>%
      filter(gadm_gid == !!gadm_gid)
    
    # check if events exist for admin unit
    if (nrow(df) > 0){
      
      event_attr <- df %>%
        summarize(
          event_persondays = sum(persondays, na.rm = T),
          event_max_duration = max(duration, na.rm = T)
        ) 
      
      uniquedays <- df %>%
        mutate(
          start_date = as.Date(start_date),
          end_date = as.Date(end_date)
        ) %>%
        rowwise() %>%
        mutate(days = list(seq(start_date, end_date, by = 'day'))) %>%
        unnest(days) %>%
        ungroup() %>%
        distinct(days)
      
      ndays <- uniquedays %>%
        summarize(
          event_days = n())
      
      result <- cbind(event_attr, ndays) %>%
        mutate(
          disno = disno,
          gadm_gid = gadm_gid
        ) %>%
        mutate(
          event_dates = list(format(uniquedays$days, '%Y-%m-%d'))
        ) %>%
        mutate(
          event_dates = sapply(event_dates, function(x) paste(x, collapse = '; '))
        )
      
      return(result)
      
    } else {
      return(NULL) # Return NULL if no events for adm unit
    }
  } else {
    return(NULL)  # Return NULL if the disno file doesn't exist
  }
}

# Apply the process_file function to each row in subnat
event_attributes <- subnat %>%
  as.data.frame() %>%
  mutate(geometry = NULL) %>%
  select(disno, gadm_gid) %>%
  pmap_dfr(function(disno, gadm_gid) {
    process_events(disno, gadm_gid)
  })
```

### Join results and export outputs

``` r
# Join results to subnat
output <- subnat %>%
  left_join(general_attributes,
            by = c('disno','gadm_gid')) %>%
  relocate(adm_unit_pop, .after = adm_unit_area_km2) %>%
  left_join(event_attributes,
            by = c('disno', 'gadm_gid')) %>%
  mutate(
    adm_unit_area_km2 = round(adm_unit_area_km2, digits = 5)
  )

# Export output with admin unit geometry as geopackage (.gpkg)
st_write(
  output,
  dsn = paste0(ph_out, '/data-adm-unit-level.gpkg'),
  layer = 'data-adm-unit-level',
  delete_dsn = TRUE
)

# Export output without admin unit geometry as csv
output <- output %>%
  as.data.frame() %>%
  mutate(geometry = NULL)

write.csv(
  output,
  paste0(ph_out, '/data-adm-unit-level.csv'),
  row.names = F
)
```

End of script.
