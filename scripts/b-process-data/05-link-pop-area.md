Extracting data on population and cell area
================
Sara Lindersson
2025-01-29

This script extracts grid point data on yearly population figures cell
areas for each *disno* and *subdivision*. Before this, the script
resamples the original population maps to align with the meteorological
data, and linearly interpolates the 5-year values to annual ones. The
script finnaly joins these estimates with the meteorological daily
statistics.

``` r
library(here)
library(tidyverse)
library(sf)
library(terra)
library(exactextractr)
library(stringr)
library(raster)
library(zoo)
library(purrr)
library(dplyr)
```

``` r
# Execute all notebook chunks in the grandparent folder of the notebook
knitr::opts_knit$set(root.dir = normalizePath(".."))
```

### Parameters

``` r
# Directory to meteorological data
ph_mswx <- 'F:/mswx/temp/3hourly/'

# Directory to population data
ph_ghs <- 'F:/ghs/'

ph_in <- here('data-processed', 'temporary', '04')
ph_out <- here('data-processed', 'temporary', '05')

# Check if output-directory exists, otherwise create it
if (!file.exists(ph_out)) {
  dir.create(ph_out, recursive = T)
}
```

### Import sample of subdivisions for disaster records

Loads `subnat.rds` from script `01-emdat-gdis-link.Rmd`. This dataframe
holds the sample of EM-DAT entries at the subnational level, with one
row per *disno* and administrative *subdivision*, accompanied by
simplified polygons.

``` r
subnat <- readRDS(here('data-processed','subnat.rds'))
```

### Import population maps

``` r
# Years of population data
pop_yrs <- seq(from = 1975, to = 2020, by = 5)

ghs <- terra::rast(paste0(
    ph_ghs,
    'GHS_POP_E', pop_yrs, '_GLOBE_R2023A_4326_30ss_V1_0',
    '/',
    'GHS_POP_E', pop_yrs, '_GLOBE_R2023A_4326_30ss_V1_0.tif'
  )
)
```

### Resample population maps

The script now resamples the population maps from the Global Human
Settlement Layer to align with the grid cells of MSWX (the
meteorological data), using *exactextractr*.

``` r
# Derive area per grid cell of MSWX
aa <- terra::cellSize(
  terra::rast(
    # Select the first file in directory
    list.files(ph_mswx, pattern = '\\.nc$', full.names = T)[1]
  )
)

# Resample population maps
ghs_rs <- map(pop_yrs, function(yr){
  pop_raster <- ghs[[paste0('GHS_POP_E', yr, '_GLOBE_R2023A_4326_30ss_V1_0')]]
  resampled <- exactextractr::exact_resample(pop_raster, aa, fun = 'sum')
  names(resampled) <- paste0('GHS_POP_E', yr, '_GLOBE_R2023A_4326_30ss_V1_0')
  resampled
})

# Combine all the resampled rasters into a single list
ghs_rs <- do.call(c, ghs_rs)

# Combine with cell size raster
ghs_rs <- c(ghs_rs, aa)
```

### Derive area- and population-estimates per event and admin unit

The script iterates over the subnational records in *subnat* to derive
corresponding area and yearly population estimates at the grid cell
level. The resampled maps produce slightly different yearly population
totals for each administrative unit. To address this, the script scales
the resampled totals to ensure they align with the original values (per
administrative unit) before interpolating the yearly population values.

``` r
for (i in 1:nrow(subnat)){
  subnat_i <- subnat[i,]
  disno_i <- subnat_i$disno
  gadm_gid_i <- subnat_i$gadm_gid
  
  # Compare yearly population totals to retrieve yearly scaling ratio
  ghs_i <- exact_extract(ghs, subnat_i$geometry, include_xy = T)[[1]] %>%
    rename_with(~ str_replace(.,
                              'GHS_POP_E(\\d+)_GLOBE_R2023A_4326_30ss_V1_0',
                              'p\\1'
    ), 
    starts_with('GHS_POP_E')) %>%
    rename(cf = coverage_fraction) %>%
    pivot_longer(
      cols = -c(x, y, cf),
      names_to = 'year',
      values_to = 'pop'
    ) %>%
    mutate(
      pop = (pop * cf), # Scale population value with coverage fraction
      year = as.POSIXct(
        paste0(as.integer(str_sub(year, 2)), '-01-01'),
        tz = 'GMT'
      )
    ) %>%
    group_by(year) %>%
    summarise(pop_sum_original = sum(pop, na.rm = T))
  
  ghs_rs_i <- exact_extract(ghs_rs, subnat_i$geometry, include_xy = T)[[1]] %>%
    rename_with(~ str_replace(.,
                              'GHS_POP_E(\\d+)_GLOBE_R2023A_4326_30ss_V1_0',
                              'p\\1'), 
                starts_with('GHS_POP_E')) %>%
    rename(cf = coverage_fraction) %>%
    pivot_longer(
      cols = -c(x, y, cf, area),
      names_to = 'year',
      values_to = 'pop'
    ) %>%
    mutate(
      pop = (pop * cf), # Scale population value with coverage fraction
      year = as.POSIXct(
        paste0(as.integer(str_sub(year, 2)), '-01-01'),
        tz = 'GMT'
      )
    ) %>%
    group_by(year) %>%
    summarise(pop_sum_resampl = sum(pop, na.rm = T))
  
  ratios <- left_join(ghs_i, ghs_rs_i, by = 'year') %>%
    mutate(
      ratio = pop_sum_original / pop_sum_resampl
    )
  
  rm(ghs_i, ghs_rs_i)
  
  # Scale the resampled data and interpolate yearly estimates
  x_i <- exact_extract(ghs_rs, subnat_i$geometry, include_xy = T)[[1]] %>%
    rename_with(~ str_replace(.,
                              'GHS_POP_E(\\d+)_GLOBE_R2023A_4326_30ss_V1_0',
                              'p\\1'), 
                starts_with('GHS_POP_E')) %>%
    rename(cf = coverage_fraction) %>%
    mutate(
      y = round(y, 2),
      x = round(x, 2),
      cell_id = row_number(),
      area_km2 = ((area * cf) * 1e-6), # Scale area with cf
      area = NULL
    ) %>%
    # Create columns for every year
    bind_cols(
      purrr::map_dfc(
        c(1976:1979, 1981:1984, 1986:1989, 1991:1994, 1996:1999, 
          2001:2004, 2006:2009, 2011:2014, 2016:2019),
        ~setNames(data.frame(NA),
                  paste0("p", .))
      )
    ) %>%
    pivot_longer(
      cols = -c(cell_id, x, y, cf, area_km2),
      names_to = 'year',
      values_to = 'pop'
    ) %>%
    mutate(
      pop = (pop * cf), # Scale pop with cf
      year = as.POSIXct(
        paste0(as.integer(str_sub(year, 2)), '-01-01'),
        tz = 'GMT'
      )
    ) %>%
    arrange(cell_id, year) %>%
    # Join yearly scaling ratios
    left_join(
      ratios[c('year', 'ratio')],
      by = 'year'
    ) %>%
    # Scale resampled population numbers
    mutate(pop_s = pop * ratio) %>% 
    group_by(cell_id) %>%
    # Interpolate to yearly values
    mutate(
      pop_s_int = na.approx(pop_s, maxgap = 5, na.rm = FALSE)
    ) %>%
    ungroup() %>%
    mutate(year = year(year)) %>%
    # Convert population to integer
    mutate(
      pop = as.integer(pop_s_int)
    ) %>%
    # Drop temporary columns
    dplyr::select(
      -pop_s_int,
      -ratio,
      -pop_s,
      -cell_id,
      -cf
    )
  
  # Load meteorological daily data and link to population and area
  file_i <- paste0(disno_i,'_',gadm_gid_i,'.rds')
  
  data_i <- readRDS(file = paste0(ph_in, '/', file_i)) %>%
    mutate(
      year = year(date)
    ) %>%
    left_join(
      x_i,
      by = c('x', 'y', 'year')
    ) %>%
    mutate(year = NULL)
  
  # Export R-object
  saveRDS(data_i, file = file.path(ph_out, paste0(disno_i, '_', gadm_gid_i, '.rds')))
}
```

End of script.
