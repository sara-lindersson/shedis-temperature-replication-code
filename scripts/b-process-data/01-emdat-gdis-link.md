Linking EM-DAT records to GDIS
================
Sara Lindersson
2025-01-22

This script links [EM-DAT](https://www.emdat.be/) records to subnational
administrative units from
[GADM](https://gadm.org/download_world36.html), using geocoding from
[GDIS](https://www.nature.com/articles/s41597-021-00846-6). It also
identifies the period of interest for further analysis.

The script produces three R objects as outputs: (1) `emdat.rds`, a tidy
dataframe containing EM-DAT entries (i.e. *disno’s*) with impact data,
with one row per national-level disaster. (2) `nat.rds`, one row per
*disno* in the sample, but with columns relevant for the hazard-analysis
instead of impact data. (3) `subnat.rds`, with one row per *disno* and
administrative *subdivision*, accompanied by simplified polygons.

#### Load R-packages

``` r
library(knitr)
library(here)
library(tidyverse)
library(readxl)
library(countrycode)
library(sf)
library(rmapshaper)
library(nngeo)
library(stringi)
library(stringr)
```

#### Set working directory

Execute all chunks in the grandparent folder of the notebook

``` r
knitr::opts_knit$set(root.dir = normalizePath(".."))
```

### Define parameters

This script considers cold waves and heat waves in EM-DAT, categorized
as extreme temperature disasters in GDIS. Additionally, the script
defines the start and end years for the analysis, which are limited to
the start year of
[MSWX](https://journals.ametsoc.org/view/journals/bams/103/3/BAMS-D-21-0145.1.xml)
and end year of
[GDIS](https://www.nature.com/articles/s41597-021-00846-6)).

``` r
# Disaster type(s) of interest
emdat_filename <- 'public_emdat_20231002.xlsx'
gdis_filename <- 'pend-gdis-1960-2018-disasterlocations.csv'

emdat_type <- c('nat-met-ext-col','nat-met-ext-hea')
gdis_type <- 'extreme temperature'

# Period of interest
start <- 1979
end <- 2018

# Output directory
ph_out <- here('data-processed')

# Check if directory exists, otherwise create it
if (!file.exists(ph_out)) {
  dir.create(ph_out, recursive = T)
}
```

### Import and tidy raw data

#### EM-DAT dataframe

``` r
emdat <- 
  # Import raw data
  read_excel(here(
    'data-raw',
    'emdat',
    emdat_filename
  ), skip = 0) %>%
  # Tidy column names
  rename_all(str_to_lower) %>%
  rename(disno = disno.,) %>%
  rename_with(~ str_replace_all(.x, '\\s', '')) %>% # Removes blank spaces
  rename_with(~ str_replace_all(.x, 'no.', '')) %>%
  rename(type = disastersubtype) %>%
  # Filter to specified disaster types and period
  filter(startyear >= start & startyear <= end) %>%
  filter(classificationkey %in% emdat_type) %>%
  # Select columns to keep
  select(
    disno,
    classificationkey,
    disastertype,
    type,
    iso,
    country,
    location,
    magnitude,
    magnitudescale,
    startyear,
    startmonth,
    startday,
    endyear,
    endmonth,
    endday,
    totaldeaths,
    totalaffected
  ) %>%
  # Define factors
  mutate(across(c(
    classificationkey,
    disastertype,
    type,
    iso,
    country,
    magnitudescale
  ), as.factor)) %>%
  # Define integers
  mutate(across(c(
    startyear,
    startmonth,
    startday,
    endyear,
    endmonth,
    endday,
    totaldeaths,
    totalaffected
  ), as.integer))
```

#### GDIS dataframe

**Important note!** The iso3-codes provided by GDIS are inconsistent, as
multiple abbreviations exist for each country. Therefore, the script
drops this column and uses the country name in combination with the
R-package `countrycode` to obtain consistent ISO codes. However, the
country name Kosovo could not be matched unambiguously. The script
assigns the ISO code ‘SRB’ to these entries, as this is the code used in
EM-DAT.

``` r
gdis <- 
  # Import raw data
  read.csv(here(
    'data-raw',
    'gdis',
    gdis_filename
  ), header = TRUE, na.strings = 'NA') %>%
  # Remove leading and trailing whitespaces
  mutate(across(where(is.character), str_trim)) %>%
  # Filter to defined disaster types
  filter(disastertype %in% gdis_type) %>%
  # Select columns to keep
  select(
    disasterno,
    country,
    geo_id,
    level,
    latitude,
    longitude,
    geolocation
  ) %>%
  # Find iso3c-code
  mutate(
    iso3c = countrycode(
      country, origin = 'country.name.en', destination = 'iso3c'
    )) %>%
  # Manually define iso3-code to Kosovo
  mutate(
    iso3c = ifelse(country == 'Kosovo', 'SRB', iso3c)
  ) %>%
  # Define the disno-column as found in EM-DAT
  mutate(
    disno = paste(disasterno, iso3c, sep = '-')
  ) %>%
  # Move disno-column to left
  relocate(disno) %>%
  # Drop now redundant columns
  select(
    -disasterno,
    -country
  ) %>%
  # Tidy column names
  rename(
    gdis_id = geo_id,
    gdis_level = level
  ) %>%
  # Find the disaster subtype (heat wave, cold wave)
  left_join(emdat[c('disno','type')], by = 'disno') %>%
  # Remove unmatched (which belong to EM-DAT subtype 'Severe winter conditions')
  filter(!is.na(type)) %>%
  # Convert to sf object
  st_as_sf(
    # The coordinates are the centroids of GADM polygons
    coords = c('longitude','latitude'),
    # WGS84 projection
    crs = 4326
  )
```

### Link GDIS coordinates to GADM polygons

The script links GDIS records to the corresponding GADM polygons (i.e.,
subnational units) using the nearest neighbor approach. This process is
performed for one GADM level at a time. The resulting dataframe is named
`subnat`, which contains one row for each linked subdivision for each
EM-DAT `disno`.

#### Loop over GADM levels 1, 2 and 3

``` r
for (i in 1:3){
  gadm_i <- st_read(
    dsn = here(
      'data-raw',
      'gadm',
      'v36',
      'shp'
    ), paste0('gadm36_',i)) %>%
    # Convert from polygons to centroid points
    st_centroid() %>%
    mutate(n_gadm = row_number())
  
  # Filter GDIS by admin level
  gdis_i <- gdis %>% filter(gdis_level == i)
  
  # Link GADM to GDIS with nearest neighbor
  nn <- st_nn(
    gdis_i,
    gadm_i,
    sparse = T,
    k = 1,
    returnDist = T,
    progress = T
  )
  nn <- t(
    as.data.frame(
      nn[1]$nn,
      row.names = 'n_gadm'
    )
  )
  gdis_i <- cbind(gdis_i, nn)
  
  # Tidy GADM
  gadm_i <- gadm_i %>%
    # Convert to data frame
    as.data.frame() %>%
    # Drop geometry
    mutate(geometry = NULL) %>%
    # Rename 'GID_i' to 'gadm_gid'
    rename_with(~ ifelse(. == paste0('GID_', i), 'gadm_gid', .)) %>%  
    # Rename 'NAME_i' to 'gadm_name'
    rename_with(~ ifelse(. == paste0('NAME_', i), 'gadm_name', .))
  
  # Join to GDIS
  gdis_i <- gdis_i %>%
    left_join(
      gadm_i[c('gadm_gid','gadm_name','n_gadm')],
      by = 'n_gadm'
    )
  
  # Bind to one dataframe across the GADM-levels
  if (i == 1){
    subnat <- gdis_i
  } else {
    subnat <- rbind(subnat, gdis_i)
  }
}
```

#### Control if linking of GDIS and GADM has been correct

``` r
# Identify differences in names between gdis geolocation and the matched gadm unit
mismatch <- subnat %>%
  mutate(
    gadm_name_translit = stri_trans_general(
      gadm_name, 'Latin-ASCII') %>%
      str_to_lower(),
    geolocation = str_to_lower(geolocation)
  ) %>%
  mutate(
    gadm_name_translit = str_replace_all(gadm_name_translit, '\\s-\\s', '-'),
    name_diff = ifelse(geolocation == gadm_name_translit, 0, 1)
  ) %>%
  filter(name_diff == 1) %>%
  select(gadm_name_translit, geolocation) %>%
  distinct()
```

The script encounters one mismatch from the current sample. The GDIS
geolocation for Chukot was matched to the GADM unit of Sakha. Chukot is
situated on the 180th meridian and appears in both the Western and
Eastern hemispheres, leading to a misplacement of its centroid as
provided by GDIS. In contrast, the centroid calculated here using R is
accurately located within Chukot. Thus, this mismatch arises from
differences in methodologies. The script performs a manual correction
for this case.

#### Manual correction of GDIS and GADM link

``` r
subnat <- subnat %>%
  as.data.frame() %>%
  mutate(
    gadm_name = ifelse(geolocation == 'Chukot', 'Chukot', gadm_name),
    gadm_gid = ifelse(geolocation == 'Chukot', 'RUS.12_1', gadm_gid)
  ) %>%
  select(
    -gadm_name,
    -n_gadm
  ) %>%
  mutate(
    geometry = NULL
  )

# Clear temporary objects from memory
rm(nn, gdis_i, gadm_i, mismatch, i)
```

Some entries are linked to GADM units at level 3. The script now
substitutes these with the corresponding units at level 2. As a final
step, the script removes duplicates that arise from multiple level 3
units being aggregated into the same level 2 unit for the same event.

#### Substitute level 3 with level 2

``` r
# Load GADM level 3
x <- st_read(dsn = here(
  'data-raw',
  'gadm',
  'v36',
  'shp'),
  layer = 'gadm36_3') %>%
  select(GID_2, GID_3) %>%
  rename(
    gadm_gid = GID_3,
    gadm_gid2 = GID_2
  ) %>%
  as.data.frame() %>%
  mutate(geometry = NULL)

# Substitute level 3 with level 2 in subnat
subnat <- subnat %>%
  left_join(x, by = 'gadm_gid') %>%
  mutate(
    gadm_gid = ifelse(is.na(gadm_gid2), gadm_gid, gadm_gid2),
    gadm_level = ifelse(gdis_level == 1, 1, 2)
  ) %>%
  select(
    -gadm_gid2,
    -gdis_id,
    -gdis_level,
    -geolocation
  ) %>%
  # Remove duplicates
  unique()

# Remove temporary object
rm(x)
```

We now have a dataframe called `subnat` that contains one row per EM-DAT
event, along with the identified subnational unit at level 1 or 2. This
dataframe also includes the correct iso3c-codes, the disaster subtype
(heat wave or cold wave), and the identification number `gadm_gid`,
which refers to the GADM database. The script now uses this information
to join the corresponding administrative unit polygons to the dataframe.
Additionally, the script simplifies these polygons using `ms_simplify`
to reduce the file size. Finally, the script calculates the area of
these polygons in km². To perform the area calculations, the script uses
a temporary dataframe, as it needs to apply `st_make_valid` first, which
can distort the polygon shape.

#### Link polygons to subnat-dataframe

``` r
# Load geodata and bind to one dataframe
x <- rbind(
  st_read(dsn = here(
    'data-raw',
    'gadm',
    'v36',
    'shp'),
    'gadm36_1') %>%
    select(
      GID_1,
      NAME_1
    ) %>%
    rename(
      gadm_gid = GID_1,
      gadm_name = NAME_1
    ) %>%
    mutate(gadm_level = 1),
  st_read(dsn = here(
    'data-raw',
    'gadm',
    'v36',
    'shp'),
    'gadm36_2') %>%
    select(
      GID_2,
      NAME_2
      ) %>%
    rename(
      gadm_gid = GID_2,
      gadm_name = NAME_2
      ) %>%
    mutate(gadm_level = 2)
) %>%
  # Filter polygons with id's included in subnat
  filter(gadm_gid %in% unique(subnat$gadm_gid))

# Loop over polygons and simplify
for (i in 1:nrow(x)){
  x$geometry[i] <- ms_simplify(x$geometry[i], keep = .2, keep_shapes = T)
}

subnat <- subnat %>%
  # Join simplified polygons
  left_join(x, by = c('gadm_gid','gadm_level')) %>%
  # Convert to sf object
  st_as_sf() %>%
  # Replace dots with underscores in gadm_gid
  mutate(gadm_gid = str_replace_all(gadm_gid, '\\.', '_'))

rm(x, i)

# Derive area
x <- subnat %>%
  select(gadm_gid) %>%
  unique() %>%
  mutate(
    geometry = st_make_valid(geometry),
    area_km2 = as.numeric(st_area(geometry)) * 1e-6
  ) %>%
  as.data.frame() %>%
  mutate(geometry = NULL)

# Join to subnat
subnat <- subnat %>%
  left_join(x, by = 'gadm_gid')

rm(x)
```

### Create a dataframe on national level

The script now create a dataframe similar to that of `emdat`, with one
row per *disno*, but with columns relevant for meteorological analysis,
and dropping the columns with impact data. First, the script filters
`emdat` to only include the *disno’s* that are included in `subnat`.

#### Filter `emdat`

``` r
emdat <- emdat %>% filter(disno %in% unique(subnat$disno))
```

#### Create `nat`

``` r
nat <- emdat %>%
  # Select relevant columns
  select(
    disno,
    type,
    iso,
    country,
    startyear,
    startmonth,
    startday,
    endyear,
    endmonth,
    endday,
    magnitude
  ) %>%
  # Rename columns
  rename(
    iso3c = iso,
    emdat_magnitude = magnitude
  ) %>%
  # Identify the continent
  mutate(continent = countrycode(iso3c, origin = 'iso3c', destination = 'continent')) %>%
  relocate(continent, .after = country)
```

The script will now explore the data completeness of the temporal
parameters in EM-DAT (now in `nat`).

#### Data completeness of temporal variables

``` r
x_d <- nat %>%
  filter(!is.na(startday) & !is.na(endday)) %>%
  mutate(
    startdate = paste(startyear, startmonth, startday, sep = '-'),
    enddate = paste(endyear, endmonth, endday, sep = '-'),
    coincide = ifelse(startdate == enddate, T, F)
  )

x_d_c <- x_d %>% filter(coincide == T)

x_sm <- nat %>% filter(is.na(startmonth))
x_em <- nat %>% filter(is.na(endmonth))
```

One *disno* has an unspecified end-month: *2007-0673-ROU*. The
start-year and -month is 2007-12 and the end-year is 2008, We therefore
assume that the end-month is January for this event.

``` r
nat$endmonth[nat$disno == '2007-0673-ROU'] <- 1

rm(x_d, x_d_c, x_sm, x_em, n)
```

Now all events in `nat` have specified start- and end-months. We use
these data to define the period of interest *T* with the new columns
*start* and *end*. The script lets the start date be the first day of
the start-month and the end date be the last day of the end-month. The
script also specifies extended period of interests *T<sub>ex</sub>* by
adding one month before and after, with *start_ext* and *end_ext*. The
script also defines the duration of these periods with *dur* and
*dur_ex*.

``` r
nat <- nat %>%
  # Period of interest  
  mutate(
    # Start-date
    start = as.Date(paste(
      startyear,
      startmonth,
      '01',
      sep = '-'
    )),
    # End-date
    end = as.Date(paste(
      endyear,
      endmonth,
      '01',
      sep = '-'
    )) + months(1) - days(1),
    # Duration
    dur = as.integer(interval(start, end) %/% months(1)) + 1,
  ) %>%
  # Extended period of interest
  mutate(
    # Start-date
    start_ex = start - months(1),
    # Duration
    dur_ex = dur + 2,
    # End-date
    end_ex = start_ex + months(dur_ex) - days(1)
  ) %>%
  relocate(dur_ex, .after = end_ex) %>% 
  # Drop columns
  select(
    -startyear,
    -startmonth,
    -startday,
    -endyear,
    -endmonth,
    -endday
  )
```

Linking relevant columns of `nat` to `subnat`, and export outputs.

``` r
subnat <- subnat %>%
  left_join(
    nat %>%
      select(
        disno,
        country,
        continent,
        emdat_magnitude,
        start,
        end,
        dur,
        start_ex,
        end_ex,
        dur_ex
      ),
    by = 'disno'
  )

saveRDS(emdat, file = paste0(ph_out, '/emdat.rds'))
saveRDS(nat, file = paste0(ph_out, '/nat.rds'))
saveRDS(subnat, file = paste0(ph_out, '/subnat.rds'))
```

### Check for hierarchical overlaps

The script now does a final check if there are any hierarchical overlaps
between Level-1 and Level-2 administrative units within the sample.

``` r
subnat <- subnat %>%
  mutate(overlap = F)

gadm2 <- st_read(dsn = here(
    'data-raw',
    'gadm',
    'v36',
    'shp'),
    'gadm36_2') %>%
  as.data.frame() %>%
  mutate(geometry = NULL) %>%
  select(GID_1, GID_2) %>%
  mutate(across(where(is.character), ~ gsub('\\.', '_', .))) %>%
  rename(
    gadm_gid = GID_2
  )

subnat_lev_2 <- subnat %>%
  filter(gadm_level == 2) %>%
  as.data.frame() %>%
  mutate(geometry = NULL) %>%
  left_join(gadm2, by = 'gadm_gid')

subnat_lev_1 <- subnat %>%
  filter(gadm_level == 1) %>%
  as.data.frame() %>%
  mutate(geometry = NULL)

for (i in 1:nrow(subnat_lev_2)){
  x_i <- subnat_lev_2[i,] 
  disno_i <- x_i$disno
  gadm_gid_i <- x_i$gadm_gid
  GID_1_i <- x_i$GID_1
  
  adm_units_1_disno <- subnat_lev_1 %>%
    filter(disno == disno_i) %>%
    select(gadm_gid)
  
  if (nrow(adm_units_1_disno) > 0){
    if(any(adm_units_1_disno$gadm_gid == GID_1_i)){
      subnat <- subnat %>%
        mutate(
          overlap = if_else(disno == disno_i & gadm_gid == gadm_gid_i,
                            T,
                            overlap)
        )
    }
  }
}
overlaps <- subnat %>% filter(overlap == T)
```

No overlaps found.

End of script.
