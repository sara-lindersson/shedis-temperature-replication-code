Attributes at the dinso-level
================
Sara Lindersson
2025-03-04

This script derives attributes aggregated at the disaster number
*disno*-level. The outputs from this script include:  
+ `shedis_heatwaves_disno.csv`: This file contains the derived data
attributes for the heat wave disasters.  
+ `shedis_heatwaves_disno.gpkg`: This file includes both the data
attributes and a simplified polygon of the corresponding administrative
subdivisions that have been linked to the *disno*.  
+ `shedis_coldwaves_disno.csv`: This file contains the derived data
attributes for the cold wave disasters.  
+ `shedis_coldwaves_disno.gpkg`: This file includes both the data
attributes and a simplified polygon of the corresponding administrative
subdivisions that have been linked to the *disno*.

``` r
library(here)
library(tidyverse)
library(sf)
library(exactextractr)
library(stringr)
library(glue)
library(purrr)
library(countrycode)
```

``` r
# Execute all notebook chunks in the grandparent folder of the notebook
knitr::opts_knit$set(root.dir = normalizePath(".."))
```

### Parameters

``` r
# Define how many buffer days to include
extra_days <- 7

# File path in hazard and population data
ph_in <- here('data-processed', 'daily-data')

# File paths for percentile-based threshold analysis results, conducted at grid point level
ph_events_hw_95 <- here('data-output','heatwaves','threshold-exceeding-events', 'pct95-min3days')
ph_events_hw_90 <- here('data-output','heatwaves','threshold-exceeding-events', 'pct90-min3days')
ph_events_cw_05 <- here('data-output','coldwaves','threshold-exceeding-events', 'pct05-min3days')
ph_events_cw_10 <- here('data-output','coldwaves','threshold-exceeding-events', 'pct10-min3days')

# File paths outputs
ph_out_hw <- here('data-output', 'heatwaves')
ph_out_cw <- here('data-output', 'coldwaves')
```

### Import sample of subdivisions for disaster records

Loads `subnat.rds` from script `01-emdat-gdis-link.Rmd`. This dataframe
holds the sample of EM-DAT entries at the subnational level, with one
row per *disno* and administrative *subdivision*, accompanied by
simplified polygons.

``` r
subnat <- readRDS(here('data-processed', 'subnat.rds')) %>%
  mutate(analysis_start = start - extra_days,
         analysis_end = end + extra_days) %>%
  dplyr::select(
    iso3c,
    country,
    disno,
    type,
    gadm_gid,
    gadm_level,
    gadm_name,
    area_km2,
    analysis_start,
    analysis_end
  ) %>%
  rename(adm_unit_area_km2 = area_km2)

# Derive metadata per disno
disnos <- subnat %>%
  group_by(disno) %>%
  summarize(
    iso3c = first(iso3c),
    country = first(country),
    disno = first(disno),
    type = first(type),
    gadm_gid = paste(gadm_gid, collapse = '; '),
    gadm_level = paste(gadm_level, collapse = '; '),
    gadm_name = paste(gadm_name, collapse = '; '),
    adm_unit_area_km2 = sum(adm_unit_area_km2, na.rm = T),
    analysis_start = first(analysis_start),
    analysis_end = first(analysis_end),
    # Combine subdivisions to one multipolygon per disno
    geometry = st_combine(geometry)
  )
```

### Gather general attributes from MSWX and GHS

``` r
# Define a function to process each file based on subnat record
process_attributes <- function(disno, analysis_start, analysis_end) {
  
  # Get all RDS file names that match the pattern
  rds_files <- list.files(path = ph_in, pattern = glue('^{disno}_.*\\.rds$'), full.names = T)
  
  # Read and row-bind all RDS files matching the pattern 
  df <- map_dfr(rds_files, readRDS) %>%
    select(-starts_with("ct"), -area_km2) %>%
    # Filter rows to include only dates within period of analysis
    filter(date >= analysis_start & date <= analysis_end) %>%
    arrange(date)
  
  df_zonal <- df %>%
    select(-x, -y) %>%
    group_by(date) %>%
    # Calculate zonal values for all columns
    summarize(across(
      everything(),
      ~ case_when(
        # For 'pop', calculate the sum
        cur_column() == 'pop' ~ sum(.x, na.rm = T),
        # For all other columns, calculate the weighted average based on coverage fraction
        T ~ sum(.x * cf, na.rm = T) / sum(cf, na.rm = T)
      )
    )) %>% 
    ungroup() %>%
    select(-cf) %>%
    arrange(date)
  
  df_xy <- df %>%
    summarize(
      # Hazard variables of the most extreme daily values
      xy_max_t = round(max(temp_mean), digits = 2),
      xy_max_t_date = date[which.max(temp_mean)],
      xy_max_t_coord = paste0(x[which.max(temp_mean)], '_', y[which.max(temp_mean)]),
      
      xy_min_t = round(min(temp_mean), digits = 2),
      xy_min_t_date = date[which.min(temp_mean)],
      xy_min_t_coord = paste0(x[which.min(temp_mean)], '_', y[which.min(temp_mean)]),
      
      xy_max_at = round(max(at_mean), digits = 2),
      xy_max_at_date = date[which.max(at_mean)],
      xy_max_at_coord = paste0(x[which.max(at_mean)], '_', y[which.max(at_mean)]),
      
      xy_min_at = round(min(at_mean), digits = 2),
      xy_min_at_date = date[which.min(at_mean)],
      xy_min_at_coord = paste0(x[which.min(at_mean)], '_', y[which.min(at_mean)]),
      
      # Hazard variables of the most extreme 3-hourly values
      xy_max_tx = round(max(temp_max), digits = 2),
      xy_max_tx_date = date[which.max(temp_max)],
      xy_max_tx_coord = paste0(x[which.max(temp_max)], '_', y[which.max(temp_max)]),
      
      xy_min_tn = round(min(temp_min), digits = 2),
      xy_min_tn_date = date[which.min(temp_min)],
      xy_min_tn_coord = paste0(x[which.min(temp_min)], '_', y[which.min(temp_min)]),
      
      xy_max_atx = round(max(at_max), digits = 2),
      xy_max_atx_date = date[which.max(at_max)],
      xy_max_atx_coord = paste0(x[which.max(at_max)], '_', y[which.max(at_max)]),
      
      xy_min_atn = round(min(at_min), digits = 2),
      xy_min_atn_date = date[which.min(at_min)],
      xy_min_atn_coord = paste0(x[which.min(at_min)], '_', y[which.min(at_min)])
    )
  
  df_zonal <- df_zonal %>%
    summarize(
      # Use pop sum for start date of analysis
      adm_unit_pop = first(pop),
      
      # Hazard variables averaged across analysis period T
      mean_t = round(mean(temp_mean), digits = 2),
      mean_at = round(mean(at_mean), digits = 2),
      
      mean_tx = round(mean(temp_max), digits = 2),
      mean_tn = round(mean(temp_min), digits = 2),
      
      mean_atx = round(mean(at_max), digits = 2),
      mean_atn = round(mean(at_min), digits = 2),
      
      # Hazard variables of the most extreme daily values
      max_t = round(max(temp_mean), digits = 2),
      max_t_date = date[which.max(temp_mean)],
      
      min_t = round(min(temp_mean), digits = 2),
      min_t_date = date[which.min(temp_mean)],
      
      max_at = round(max(at_mean), digits = 2),
      max_at_date = date[which.max(at_mean)],
      
      min_at = round(min(at_mean), digits = 2),
      min_at_date = date[which.min(at_mean)],
      
      # Hazard variables of the most extreme 3-hourly values
      max_tx = round(max(temp_max), digits = 2),
      max_tx_date = date[which.max(temp_max)],
      
      min_tn = round(min(temp_min), digits = 2),
      min_tn_date = date[which.min(temp_min)],
      
      max_atx = round(max(at_max), digits = 2),
      max_atx_date = date[which.max(at_max)],
      
      min_atn = round(min(at_min), digits = 2),
      min_atn_date = date[which.min(at_min)],
    ) 
  
  result <- cbind(df_zonal, df_xy) %>%
    mutate(disno = disno)
  
  return(result)
}

# Iterate function over rows in disnos
general_attributes <- disnos %>%
  as.data.frame() %>%
  mutate(geometry = NULL) %>%
  select(disno, analysis_start, analysis_end) %>%
  pmap_dfr(function(disno, analysis_start, analysis_end) {
    process_attributes(disno, analysis_start, analysis_end)
  })
```

### Gather attributes from event analysis

``` r
# First we gather information from the 90th and 10th percentile analysis

# Function to process each file if it exists
process_events10 <- function(disno, type){
  
  file_path <- ifelse(type == "Heat wave",
                      glue("{ph_events_hw_90}/{disno}.csv"),
                      glue("{ph_events_cw_10}/{disno}.csv"))
  
  # Check if the file exists
  if (file.exists(file_path)) {
    
    # Read the csv file
    df <- read.csv(file_path)
    
    duration_attr <- df %>%
      select(x, y, event_start, duration) %>%
      group_by(x, y, event_start) %>%
      summarize(duration = first(duration), .groups = "drop") %>%
      ungroup() %>%
      summarize(
        pct10_max_duration = max(duration, na.rm = T),
        pct10_median_duration = median(duration, na.rm = T),
        .groups = "drop"
      )
    
    event_attr <- df %>%
      summarize(
        pct10_persondays = sum(persondays, na.rm = T),
        .groups = "drop"
      )
    
    uniquepoints <- df %>%
      group_by(x, y, gadm_gid) %>%
      summarize(
        point_pop = first(pop),
        point_area_km2 = first(area_km2),
        .groups = "drop"
      ) %>%
      ungroup() %>%
      summarize(
        pct10_pop = sum(point_pop),
        pct10_area_km2 = round(sum(point_area_km2), digits = 2),
        .groups = "drop"
      )
    
    uniquedays <- df %>%
      mutate(
        event_start = as.Date(event_start),
        event_end = as.Date(event_end)
      ) %>%
      rowwise() %>%
      mutate(days = list(seq(event_start, event_end, by = 'day'))) %>%
      unnest(days) %>%
      ungroup() %>%
      distinct(days) %>% 
      arrange(days)
    
    ndays <- uniquedays %>%
      summarize(
        pct10_days = n(),
        .groups = "drop")
    
    result10 <- cbind(uniquepoints, event_attr, duration_attr, ndays) %>%
      mutate(disno = disno) %>%
      mutate(
        pct10_dates = list(format(uniquedays$days, '%Y-%m-%d'))
      ) %>%
      mutate(
        pct10_dates = sapply(pct10_dates, function(x) paste(x, collapse = '; '))
      ) 
    
    return(result10)
    
  } else {
    return(NULL)  # Return NULL if the disno file doesn't exist
  }
  
}

# Apply the process_file10 function to each row in subnat
event_attributes10 <- disnos %>%
  as.data.frame() %>%
  mutate(geometry = NULL) %>%
  select(disno, type) %>%
  pmap_dfr(function(disno, type) {
    process_events10(disno, type)
  })



# Now do the same thing but for the 5th and 95th percentiles
# Function to process each file if it exists
process_events5 <- function(disno, type){
  
  file_path <- ifelse(type == "Heat wave",
                      glue("{ph_events_hw_95}/{disno}.csv"),
                      glue("{ph_events_cw_05}/{disno}.csv"))
  
  # Check if the file exists
  if (file.exists(file_path)) {
    
    # Read the csv file
    df <- read.csv(file_path)
    
    duration_attr <- df %>%
      select(x, y, event_start, duration) %>%
      group_by(x, y, event_start) %>%
      summarize(duration = first(duration), .groups = "drop") %>%
      ungroup() %>%
      summarize(
        pct5_max_duration = max(duration, na.rm = T),
        pct5_median_duration = median(duration, na.rm = T),
        .groups = "drop"
      )
    
    event_attr <- df %>%
      summarize(
        pct5_persondays = sum(persondays, na.rm = T),
        .groups = "drop"
      )
    
    uniquepoints <- df %>%
      group_by(x, y, gadm_gid) %>%
      summarize(
        point_pop = first(pop),
        point_area_km2 = first(area_km2),
        .groups = "drop"
      ) %>%
      ungroup() %>%
      summarize(
        pct5_pop = sum(point_pop),
        pct5_area_km2 = round(sum(point_area_km2), digits = 2),
        .groups = "drop"
      )
    
    uniquedays <- df %>%
      mutate(
        event_start = as.Date(event_start),
        event_end = as.Date(event_end)
      ) %>%
      rowwise() %>%
      mutate(days = list(seq(event_start, event_end, by = 'day'))) %>%
      unnest(days) %>%
      ungroup() %>%
      distinct(days) %>% 
      arrange(days)
    
    ndays <- uniquedays %>%
      summarize(
        pct5_days = n(),
        .groups = "drop")
    
    result5 <- cbind(uniquepoints, event_attr, duration_attr, ndays) %>%
      mutate(disno = disno) %>%
      mutate(
        pct5_dates = list(format(uniquedays$days, '%Y-%m-%d'))
      ) %>%
      mutate(
        pct5_dates = sapply(pct5_dates, function(x) paste(x, collapse = '; '))
      ) 
    
    return(result5)
    
  } else {
    return(NULL)  # Return NULL if the disno file doesn't exist
  }
  
}

# Apply the process_file10 function to each row in subnat
event_attributes5 <- disnos %>%
  as.data.frame() %>%
  mutate(geometry = NULL) %>%
  select(disno, type) %>%
  pmap_dfr(function(disno, type) {
    process_events5(disno, type)
  })
```

### Join results and export outputs

``` r
# Join results to disnos
output <- disnos %>%
  left_join(general_attributes,
            by = c('disno')) %>%
  relocate(adm_unit_pop, .after = adm_unit_area_km2) %>%
  left_join(event_attributes10,
            by = c('disno')) %>%
  left_join(event_attributes5,
            by = c('disno')) %>%
  mutate(
    adm_unit_area_km2 = round(adm_unit_area_km2, digits = 2)
  )

shedis_coldwaves_disno <- output %>%
  filter(type == "Cold wave") %>%
  rename(
    geometry_area_km2 = adm_unit_area_km2,
    geometry_pop = adm_unit_pop
  ) %>%
  mutate(
    pct5_area_share = ifelse(!is.na(pct5_area_km2), round(pct5_area_km2 / geometry_area_km2, digits = 4), NA),
    pct10_area_share = ifelse(!is.na(pct10_area_km2), round(pct10_area_km2 / geometry_area_km2, digits = 4), NA),
  ) %>%
  mutate(
    pct5_area_share = ifelse(pct5_area_share>1, 1, pct5_area_share),
    pct10_area_share = ifelse(pct10_area_share>1, 1, pct10_area_share),
  ) %>%
  mutate(
    pct5_pop_share = ifelse(!is.na(pct5_pop), round(pct5_pop / geometry_pop, digits = 4), NA),
    pct10_pop_share = ifelse(!is.na(pct10_pop), round(pct10_pop / geometry_pop, digits = 4), NA),
  ) %>%
  select(
    iso3c,
    country,
    disno,
    type,
    gadm_gid,
    gadm_level,
    gadm_name,
    geometry_area_km2,
    geometry_pop,
    analysis_start,
    analysis_end,
    
    mean_t,
    mean_at,
    mean_tn,
    mean_atn,
    
    min_t, min_t_date,
    xy_min_t, xy_min_t_date, xy_min_t_coord,
    min_at, min_at_date,
    xy_min_at, xy_min_at_date, xy_min_at_coord,
    
    min_tn, min_tn_date,
    xy_min_tn, xy_min_tn_date, xy_min_tn_coord,
    min_atn, min_atn_date,
    xy_min_atn, xy_min_atn_date, xy_min_atn_coord,
    
    pct10_area_share,
    pct10_pop,
    pct10_persondays,
    pct10_median_duration, pct10_max_duration,
    pct10_days,
    pct10_dates,
    
    pct5_area_share,
    pct5_pop,
    pct5_persondays,
    pct5_median_duration, pct5_max_duration,
    pct5_days,
    pct5_dates
  )

shedis_heatwaves_disno <- output %>%
  filter(type == "Heat wave") %>%
  rename(
    geometry_area_km2 = adm_unit_area_km2,
    geometry_pop = adm_unit_pop
  ) %>%
  rename_with(~ str_replace(., "^pct5_", "pct95_"), starts_with("pct5_")) %>%
  rename_with(~ str_replace(., "^pct10_", "pct90_"), starts_with("pct10_")) %>% 
  mutate(
    pct95_area_share = ifelse(!is.na(pct95_area_km2), round(pct95_area_km2 / geometry_area_km2, digits = 4), NA),
    pct90_area_share = ifelse(!is.na(pct90_area_km2), round(pct90_area_km2 / geometry_area_km2, digits = 4), NA),
  ) %>%
  mutate(
    pct95_area_share = ifelse(pct95_area_share>1, 1, pct95_area_share),
    pct90_area_share = ifelse(pct90_area_share>1, 1, pct90_area_share),
  ) %>%
  mutate(
    pct95_pop_share = ifelse(!is.na(pct95_pop), round(pct95_pop / geometry_pop, digits = 4), NA),
    pct90_pop_share = ifelse(!is.na(pct90_pop), round(pct90_pop / geometry_pop, digits = 4), NA),
  ) %>%
  select(
    iso3c,
    country,
    disno,
    type,
    gadm_gid,
    gadm_level,
    gadm_name,
    geometry_area_km2,
    geometry_pop,
    analysis_start,
    analysis_end,
    
    mean_t,
    mean_at,
    mean_tx,
    mean_atx,
    
    max_t, max_t_date,
    xy_max_t, xy_max_t_date, xy_max_t_coord,
    max_at, max_at_date,
    xy_max_at, xy_max_at_date, xy_max_at_coord,
    
    max_tx, max_tx_date,
    xy_max_tx, xy_max_tx_date, xy_max_tx_coord,
    max_atx, max_atx_date,
    xy_max_atx, xy_max_atx_date, xy_max_atx_coord,
    
    pct90_area_share,
    pct90_pop,
    pct90_persondays,
    pct90_median_duration, pct90_max_duration,
    pct90_days,
    pct90_dates,
    
    pct95_area_share,
    pct95_pop,
    pct95_persondays,
    pct95_median_duration, pct95_max_duration,
    pct95_days,
    pct95_dates
  )

# Make some final adjustments to the output tables
shedis_coldwaves_disno <- shedis_coldwaves_disno %>%
  mutate(country = countrycode(country, origin = "country.name", destination = "country.name.en")) %>%
  arrange(iso3c, disno) %>% 
  rename(subtype = type,
         pct5_area_percentage = pct5_area_share,
         pct10_area_percentage = pct10_area_share) %>% 
  mutate(pct5_area_percentage = pct5_area_percentage * 100,
         pct10_area_percentage = pct10_area_percentage * 100) %>% 
  rename_with(~ str_replace(.x, "^pct5_", "pct05_"))

shedis_heatwaves_disno <- shedis_heatwaves_disno %>%
  mutate(country = countrycode(country, origin = "country.name", destination = "country.name.en")) %>%
  arrange(iso3c, disno) %>% 
  rename(subtype = type,
         pct95_area_percentage = pct95_area_share,
         pct90_area_percentage = pct90_area_share) %>% 
  mutate(pct95_area_percentage = pct95_area_percentage * 100,
         pct90_area_percentage = pct90_area_percentage * 100)

# Export outputs with geometries (in GPKG format)
st_write(
  shedis_coldwaves_disno,
  dsn = paste0(ph_out_cw, '/shedis_coldwaves_disno.gpkg'),
  layer = 'shedis_coldwaves_disno',
  delete_dsn = TRUE
)

st_write(
  shedis_heatwaves_disno,
  dsn = paste0(ph_out_hw, '/shedis_heatwaves_disno.gpkg'),
  layer = 'shedis_heatwaves_disno',
  delete_dsn = TRUE
)

# Export outputs as CSV (no geometries)
shedis_coldwaves_disno_csv <- shedis_coldwaves_disno %>%
  as.data.frame() %>%
  mutate(geometry = NULL)

write.csv(
  shedis_coldwaves_disno_csv,
  paste0(ph_out_cw, '/shedis_coldwaves_disno.csv'),
  row.names = F
)

shedis_heatwaves_disno_csv <- shedis_heatwaves_disno %>%
  as.data.frame() %>%
  mutate(geometry = NULL)

write.csv(
  shedis_heatwaves_disno_csv,
  paste0(ph_out_hw, '/shedis_heatwaves_disno.csv'),
  row.names = F
)
```

End of script.
