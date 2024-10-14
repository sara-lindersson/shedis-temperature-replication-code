Threshold Analysis at the Disaster Record Level
================
Sara Lindersson
2024-10-14

This script identifies periods of heat waves and cold waves at the
disaster record (disno) level through threshold analysis. The final
output consists of one file per disaster number, detailing identified
events according to the selected thresholds. Daily zonal values are
calculated across all grid cells within all the impacted subnational
units before performing the threshold analysis.

**Available percentiles for cold spells:**  
+ ctn05: 5th percentile of daily minimum air temperature  
+ ctn10: 10th percentile of daily minimum air temperature

**Available percentiles for heat waves:**  
+ ctx90: 90th percentile of daily maximum air temperature  
+ ctx95: 95th percentile of daily maximum air temperature

**Available variables for fixed thresholds:**  
+ temp_max: daily maximum air temperature  
+ temp_min: daily minimum air temperature  
+ temp_mean: daily mean air temperature  
+ at_max: daily maximum apparent temperature  
+ at_min: daily minimum apparent temperature  
+ at_mean: daily mean apparent temperature  
+ wbt_max: daily maximum wet bulb temperature, following Stull’s
method  
+ wbt_min: daily minimum wet bulb temperature, following stull’s
method  
+ wbt_mean: daily mean wet bulb temperature, following Stull’s method  
+ swbgt_max: daily maximum simplified wet bulb globe temperature  
+ swbgt_min: daily minimum simplified wet bulb globe temperature  
+ swbgt_mean: daily mean simplified wet bulb globe temperature

**Note**: The variable denoted as *temp* refers to the 2-m air
temperature as given by
[MSWX](https://doi.org/10.1175/BAMS-D-21-0145.1). The heat stress
indices denoted as *at*, *wbt* and *swbgt* have been calculated from
MSWX data using the R package
[HeatStress](https://rdrr.io/github/anacv/HeatStress/).

``` r
library(here)
library(tidyverse)
library(sf)
library(terra)
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
# Define minimum duration [days]
min_dur <- 3

# Define threshold type of interest
t_type <- 'pct' # Options: 'pct' or 'fixed'

# For percentiles, define their values
# First entry for cold waves and second for heat waves
pct_levels <- c(5, 95) # Options: c(5,95) or c(10,90)

# For fixed thresholds, define their variables and levels
# First entry for cold waves and second for heat waves
fixed_variables <- c('at_min', 'at_max') 
fixed_levels <- c(-30, 30)

# File path in hazard and population data
ph_in <- here('data-processed', 'daily-data')

# File path out
if (t_type == 'pct'){
  ph_out <- here('data-output',
                 'events','disno-level',
                 paste0('pct-', pct_levels[1], '-', pct_levels[2]))
}
if (t_type == 'fixed'){
  ph_out <- here('data-output',
                 'events','disno-level',
                 paste0('fixed-', gsub('_.*', '', fixed_variables[1]), '-', fixed_levels[1], '-', fixed_levels[2]))
}

# Check if output-directory exists, otherwise create it
if (!file.exists(ph_out)) {dir.create(ph_out, recursive = T)}
```

### Import subnational disaster records

Loads the output from script *01-emdat-gdis-link.Rmd*.

``` r
subnat <- readRDS(here('data-processed','subnat.rds')) %>%
  as.data.frame() %>%
  mutate(geometry = NULL) %>%
  select(disno, type) %>%
  distinct()
```

### Create a key tibble of parameters

``` r
param <- tibble(
  thres_type = c('pct', 'pct', 'fixed', 'fixed'),
  dis_type = c('Cold wave', 'Heat wave', 'Cold wave', 'Heat wave'),
  min_dur = min_dur
) %>%
  mutate(
    data_var = case_when(
      thres_type == 'pct' & dis_type == 'Cold wave' ~ 'temp_min',
      thres_type == 'pct' & dis_type == 'Heat wave' ~ 'temp_max',
      thres_type == 'fixed' & dis_type == 'Cold wave' ~ fixed_variables[1],
      thres_type == 'fixed' & dis_type == 'Heat wave' ~ fixed_variables[2],
      T ~ NA_character_
    ),
    
    thres_crit = case_when(
      dis_type == 'Cold wave' ~ '<',
      dis_type == 'Heat wave' ~ '>',
      T ~ NA_character_
    ),
    
    thres_var = case_when(
      thres_type == 'pct' & dis_type == 'Cold wave' ~ 
        glue('ctn{sprintf("%02d", pct_levels)[1]}'),
      thres_type == 'pct' & dis_type == 'Heat wave' ~ 
        glue('ctx{sprintf("%02d", pct_levels)[2]}'),
      thres_type == 'fixed' & dis_type == 'Cold wave' ~ fixed_variables[1],
      thres_type == 'fixed' & dis_type == 'Heat wave' ~ fixed_variables[2],
      T ~ NA_character_
    ),
    
    thres_lev = case_when(
      thres_type == 'fixed' & dis_type == 'Cold wave' ~ fixed_levels[1],
      thres_type == 'fixed' & dis_type == 'Heat wave' ~ fixed_levels[2]
    )
  )
```

### Event analysis

``` r
# Define a function to process each file based on a disaster number
process_file <- function(disno, type) {

key <- param %>%
    filter(thres_type == t_type & dis_type == type)
  
# Get all RDS file names that match the pattern
  rds_files <- list.files(path = ph_in, pattern = glue('^{disno}_.*\\.rds$'), full.names = T)
  
  # Read and row-bind all RDS files 
result <- map_dfr(rds_files, readRDS) %>%
    select(-x, -y, -cf) %>%
    group_by(date) %>%
    # Calculate zonal values for all columns
    summarize(across(
      everything(),
      ~ case_when(
        # For 'pop' and 'area_km2', calculate the sum
        cur_column() == 'pop' ~ sum(.x, na.rm = T),
        cur_column() == 'area_km2' ~ sum(.x, na.rm = T),
        # For all other columns, calculate the weighted average based on area of grid cell in admin unit
        T ~ sum(.x * area_km2, na.rm = T) / sum(area_km2, na.rm = T)
      )
    )) %>% 
    ungroup() %>%
    arrange(date) %>%
    # Filter rows with missing values
    filter(!is.na(get(key$thres_var))) %>%
    filter(!is.na(get(key$data_var))) %>%
    # Identify extreme temp days
    mutate(ext = case_when(
      t_type == 'pct' &
        eval(parse(text = glue('{key$data_var} {key$thres_crit} {key$thres_var}'))) ~ 1,
      t_type == 'fixed' &
        eval(parse(text = glue('{key$data_var} {key$thres_crit} {key$thres_lev}'))) ~ 1,
      T ~ 0
    )) %>%
    # Reclassify single non-extreme days as extreme 
    mutate(ext_recl = case_when(
      ext == 0 & lag(ext) == 1 & lead(ext) == 1 ~ 1,
      TRUE ~ ext
    )) %>%
    mutate(
      sequence_id = cumsum(ext_recl != lag(ext_recl, default = 0)), # Create a sequence ID for consecutive 1s
      sequence_id = if_else(ext_recl == 0, NA_integer_, sequence_id) # Set sequence ID to NA for 0s
    ) %>%
    group_by(sequence_id) %>%
    mutate(duration = if_else(!is.na(sequence_id), n(), 0)) %>% # Count the length of each sequence of 1s
    # Only consider sequences of at least the minimum duration
    filter(duration >= key$min_dur) %>%
    ungroup() %>%
    # Quantify the severity of each sequence
    mutate(magnitude = case_when(
      t_type == 'pct' & ext == 1 ~ abs(eval(parse(text = glue('{key$data_var} - {key$thres_var}')))),
      t_type == 'fixed' & ext == 1 ~ abs(eval(parse(text = glue('{key$data_var} - {key$thres_lev}')))),
      T ~ 0
    )) %>%
    # Summarize data to get one row per sequence_id
    group_by(sequence_id) %>%
    summarize(
      
      area_km2 = round(first(area_km2), digits = 4),
      pop = first(pop),
      
      start_date = min(date),
      end_date = max(date),
      
      mean_pct_10 = round(mean(pull(pick(matches('^ct.*0$')))), digits = 2),
      mean_pct_05 = round(mean(pull(pick(matches('^ct.*5$')))), digits = 2),
      
      duration = first(duration),
      persondays = first(pop) * first(duration),
      
      magnitude = round(sum(magnitude), digits = 2),
      
      # Indicators of air temperature 
      mean_t = round(mean(temp_mean), digits = 2), # Average temp
      mean_tx = round(mean(temp_max), digits = 2), # Average daymax
      mean_tn = round(mean(temp_min), digits = 2), # Average daymin
      
      max_tx = round(max(temp_max), digits = 2), # Max daymax and date
      max_tx_date = date[which.max(temp_max)],
      
      min_tn = round(min(temp_min), digits = 2), # Min daymin and date
      min_tn_date = date[which.min(temp_min)], 
      
      max_t_mean = round(max(temp_mean), digits = 2), # Max daily average and date
      max_t_mean_date = date[which.max(temp_mean)], 
      
      min_t_mean = round(min(temp_mean), digits = 2), # Min daily average and date
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
    ) %>%
    ungroup() %>%
    select(-sequence_id) %>%
    arrange(start_date) %>%
    mutate(
      disno = disno,
      subtype = type
    ) %>%
    relocate(
      disno,
      subtype
    )
  
  # Export the result to a .csv-file only if the result dataframe is not empty
  if (nrow(result) > 0) {
    write.csv(result, file = glue('{ph_out}/{disno}.csv'), row.names = F)
  }
}

# Iterate function over rows in subnat
subnat %>%
  pmap(function(disno, type) {
    process_file(disno, type)
  })
```

End of script.
