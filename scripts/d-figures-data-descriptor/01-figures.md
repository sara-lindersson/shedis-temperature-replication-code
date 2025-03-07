Figures
================
Sara Lindersson
2025-03-04

This script generates the figures in the data descriptor article of
SHEDIS-Temperature by Lindersson & Messori (2025).

``` r
library(here)
library(tidyverse)
library(readxl)
library(countrycode)
library(sf)
library(rmapshaper)
library(units)
library(nngeo)
library(stringi)
library(leaflet)
library(purrr)
library(ggplot2)
library(tidyquant)
library(bbplot)
library(countrycode)
library(rnaturalearth)
library(rnaturalearthdata)
library(RColorBrewer)
library(ggrepel)
library(viridis)
library(ggsci)
library(cowplot)
library(tibble)
library(ggridges)
library(ggpmisc)
library(ggpubr)
library(mapview)
library(palr)
library(terra)
library(viridis)
library(tidyterra)
library(scales)
```

``` r
# Execute all notebook chunks in the grandparent folder of the notebook
knitr::opts_knit$set(root.dir = normalizePath(".."))
```

## Fig 1 - World map and example maps of France

``` r
## Example maps
# Load polygons
df <- readRDS(here('data-processed', 'subnat.rds')) %>% filter(disno == '2003-0391-FRA')
europe <- ne_countries(scale = "large", returnclass = "sf") %>%  filter(continent == "Europe")
france <- europe %>% filter(admin == "France")

# Create and export map
m <- leaflet() %>%
  setView(lng = 2.5, lat = 46.6, zoom = 5) %>%  # Centered on France
  addPolygons(data = europe, fillColor = "gray", color = "white", weight = 1) %>%
  addPolygons(data = france, fillColor = "blue", color = "white", weight = 1, opacity = 0.3) %>%
  addPolygons(data = df, fillColor = "red", color = "white", weight = 1, opacity = 0.3) %>%
  addScaleBar(position = "bottomleft", options = scaleBarOptions(metric = TRUE, imperial = FALSE))

mapshot(m, file = here('figures','f01_inset_maps.pdf'))
rm(europe, france, m, df)

## World map with sample
# Projection. 
crs <- 8857 # Equal Earth

# Load basemap from Natural Earth
world <- ne_countries(scale = "medium", returnclass = "sf") %>% 
  filter(name != "Antarctica") %>%
  select(adm0_a3, geometry) %>%
  rename(iso3c = adm0_a3)

# Load sample
df <- readRDS(here('data-processed', 'subnat.rds')) %>% 
  as.data.frame() %>%
  select(disno, country, iso3c) %>%
  distinct(disno, .keep_all = TRUE) %>%
  group_by(iso3c) %>%
  summarise(n = n())

sample <- world %>%
  inner_join(df, by = "iso3c")

# Transform projections
world <- st_transform(world, crs = st_crs(crs))
sample <- st_transform(sample, crs = st_crs(crs))

# Plotting parameters
plot_params <- tibble(
  landmass_color = "#e0e0e0",
  stroke_color = "white",
  point_size = 2.5, # Size points on map
  alpha_point_fill = 0.3, # Transparency fill color points
  alpha_point_stroke = 0.3, # Transparency stroke color points
  lwd_point_stroke = 0.01, # Thickness stroke points
  legend_position = "bottom"
)

# Create figure
m <- ggplot() +
  geom_sf(data = world, color = NA, fill = plot_params$landmass_color) +
  geom_sf(data = sample, aes(fill = n), color = plot_params$stroke_color,
          linewidth = plot_params$lwd_point_stroke) +
  scale_fill_binned(
    type = 'viridis',
    breaks = c(1, 2, 5, 10, 15, max(sample$n, na.rm = T))
  ) +
  cowplot::theme_map() +
  theme(
    panel.grid.major = element_line(color = "transparent")
  ) + 
  guides(color = guide_legend(title = NULL, nrow = 1)) +
  cowplot::panel_border(remove = TRUE)

# Export
finalise_plot(plot_name = m,
              save_filepath = here("figures","f01_sample.pdf"),
              source_name = "Projection Equal Earth",
              width_pixels = 800, height_pixels = 500)
```

![](figures_files/figure-gfm/fig-1-1.png)<!-- -->

## Fig 3 - Overview of disaster records

### Panel 3a

``` r
rm(list=ls()); cat('\f') # Clear console
```



``` r
# Load sample
df <- readRDS(here('data-processed', 'subnat.rds')) %>%
  as.data.frame %>%
  select(disno, type, iso3c, start, end) %>%
  mutate(
    year = as.integer(substr(disno, 1, 4)),
    continent = factor(countrycode(iso3c, origin = "iso3c", destination = "continent"))
  ) %>%
  distinct(disno, .keep_all = TRUE)

# Create figure
fig <- ggplot(df, aes(x = year, fill = type)) +
  geom_histogram(colour = NA, bins = 40, alpha = .6, lwd = 0.2) +
  theme_tq() +
  scale_fill_manual(values = c("skyblue", "tomato")) +
  scale_y_continuous(
    limits = c(0, 50), # Adjust these values to set the min and max of the y-axis
    breaks = seq(0, 50, by = 10)
  ) +
  theme(panel.grid.minor = element_blank(),
        panel.grid.major = element_blank(),
        panel.border = element_blank(),
        axis.line = element_line(),
        legend.title = element_blank()) +
  labs(y = NULL, x = 'Year')

# Export
finalise_plot(plot_name = fig,
              save_filepath = here("figures","f03a_disnos_vs_year.pdf"),
              source_name = NULL,
              width_pixels = 400, height_pixels = 300)
```

![](figures_files/figure-gfm/fig-3a-1.png)<!-- -->

### Panel 3b

``` r
rm(list=ls()); cat('\f') # Clear console
```



``` r
# Load sample
df <- readRDS(here('data-processed', 'subnat.rds')) %>%
  as.data.frame %>%
  select(disno, type, iso3c, start) %>%
  distinct(disno, .keep_all = TRUE) %>%
  mutate(
    year = as.integer(substr(disno, 1, 4)),
    continent = factor(countrycode(iso3c, origin = "iso3c", destination = "continent")),
    month = as.integer(month(start))
  )

# Count observations per continent
n_continent <- df %>% 
  group_by(continent) %>%
  summarise(
    n = n()
  )

# Create figure
fig <- ggplot(df, aes(x = month, y = continent)) +
  geom_density_ridges2(aes(fill = type), color = NA, alpha = .6, scale = .9) +
  scale_fill_manual(values = c("skyblue", "tomato")) +
  scale_x_continuous(name = NULL,
                     breaks = seq(1, 12, 1),
                     labels = c('Jan',
                                'Feb',
                                'Mar',
                                'Apr',
                                'May',
                                'Jun',
                                'Jul',
                                'Aug',
                                'Sep',
                                'Oct',
                                'Nov',
                                'Dec'),
                     limits = c(1, 14)) +
  labs(y = NULL) +
  theme_tq() +
  theme(panel.grid.minor = element_blank(),
        panel.grid.major = element_blank(),
        panel.border = element_blank(),
        axis.line = element_line(),
        legend.title = element_blank()) +
  geom_text(
    data = n_continent, 
    aes(x = 12.5, y = continent, label = paste0("n = ", n)), 
    inherit.aes = FALSE, 
    size = 3, hjust = 0
  )

# Export
finalise_plot(plot_name = fig,
              save_filepath = here("figures","f03b_start_vs_continent.pdf"),
              source_name = "",
              width_pixels = 400, height_pixels = 300)
```

    ## Picking joint bandwidth of 1.22

![](figures_files/figure-gfm/fig-3b-1.png)<!-- -->

### Panel 3c

``` r
rm(list=ls()); cat('\f') # Clear console
```



``` r
# Load sample
df <- readRDS(here('data-processed', 'subnat.rds')) %>%
  as.data.frame %>%
  select(disno, type, iso3c) %>%
  mutate(
    year = as.integer(substr(disno, 1, 4)),
    continent = countrycode(iso3c, origin = "iso3c", destination = "continent"),
    cont_group = case_when(
      continent %in% c("Africa","Asia","Oceania") ~ "Africa, Asia and Oceania",
      TRUE ~ continent
    ),
    cont_group = factor(cont_group)
  ) %>% 
  group_by(disno, year, type, cont_group, continent) %>%
  summarise(count = n(), .groups = "drop") %>%
  mutate(
    time_period = case_when(
      year <= 1998 ~ 1,
      year > 1998 ~ 2
    ),
    time_period = factor(time_period, levels = c(1,2), labels = c("1979-1998", "1999-2018"), ordered = T)
  )

# Create figure
fig <- ggplot(df, aes(y = count, x = time_period)) +
  geom_point(
    size = 3,
    alpha = 0.7,
    position = position_jitter(
      seed = 1, width = .1),
    aes(colour = type)) +
  scale_colour_manual(values = c("skyblue", "tomato")) +
  geom_boxplot(
    alpha = 0.5,
    width = .25,
    outlier.shape = NA) +
  scale_y_continuous(
    limits = c(0, 80),
    breaks = seq(0, 80, 10),
    labels = scales::comma
  ) +
  theme_tq() +
  theme(
    panel.grid.minor = element_blank(),
    panel.grid.major.y = element_blank(),
    panel.grid.minor.x = element_blank(),
    axis.line.y = element_line(linewidth = .1),
    axis.line.x = element_line(linewidth = .1),
    panel.grid.major.x = element_blank(),
    panel.border = element_blank(),
    legend.title = element_blank()) + 
  labs(y = "Impacted subdivision per disno",
       x = element_blank())  +
  facet_wrap(~cont_group)

# Export
finalise_plot(plot_name = fig,
              save_filepath = here("figures","f03c_subdivisions_per_disno.pdf"),
              source_name = "",
              width_pixels = 400, height_pixels = 300)
```

![](figures_files/figure-gfm/fig-3c-1.png)<!-- -->

``` r
# Statistics per group for figure
stats_figure <- df %>%
  group_by(time_period, cont_group) %>%
  summarise(
    md = median(count, na.rm = TRUE),
    n = n(),
    .groups = "drop"
  )

# Statistics per continent
stats <- df %>%
  group_by(continent) %>%
  summarise(
    md = median(count, na.rm = TRUE),
    .groups = "drop"
  )
```

## Fig 5

``` r
rm(list=ls()); cat('\f') # Clear console
```



``` r
# Load polygons
df <- readRDS(here('data-processed', 'subnat.rds')) %>%
  filter(
    disno == '2003-0391-FRA',
    gadm_gid %in% c('FRA_6_1', 'FRA_2_1', 'FRA_1_1')
  )

# Load threshold-exceeding events
events <- read.csv(here('data-output',
                        'heatwaves',
                        'threshold-exceeding-events',
                        'pct95-min3days',
                        '2003-0391-FRA.csv')) %>%
  filter(gadm_gid %in% c('FRA_6_1', 'FRA_2_1', 'FRA_1_1')) %>%
  distinct(x, y, event_start, .keep_all = T) %>% 
  group_by(x, y) %>%
  summarise(duration = max(duration)) %>%
  ungroup() %>%
  st_as_sf(coords = c("x", "y"), crs = 4326)
```

    ## `summarise()` has grouped output by 'x'. You can override using the `.groups`
    ## argument.

``` r
# Load raster
r <- rast('F:/mswx/temp/3hourly/1979001.00.nc')

# Crop raster
cropped_r <- mask(crop(r, vect(df)), vect(df))

# Convert event-points to raster
events_raster <- rasterize(vect(events), cropped_r, field = "duration", fun = "mean")

# Create a palette
temp_palette <- colorNumeric(palette = viridis::viridis(12), domain = c(4, 15))

# Create and export map
m <- leaflet() %>%
  setView(lng = 2.5, lat = 46.6, zoom = 6) %>%  # Centered on France
  addPolygons(data = df, color = "white", weight = 1) %>%
  addRasterImage(events_raster, colors = temp_palette, opacity = 1) %>%
  leaflet::addLegend(
    # position = "bottomright",
    pal = temp_palette,
    values = c(4, 15),
    title = "Days in 95th percentile heat wave",
    opacity = 1
  )

mapview::mapshot(m, file = here("figures","f05c.pdf"))
```

## Fig 7

``` r
rm(list=ls()); cat('\f') # Clear console
```



``` r
# Load EM-DAT 
emdat <- readRDS(here('data-processed', 'emdat.rds')) %>%
  filter(disno == "2003-0391-FRA")

# Load polygons
df <- readRDS(here('data-processed', 'subnat.rds')) %>%
  filter(disno == '2003-0391-FRA')

# Load threshold-exceeding events
events <- read.csv(here('data-output',
                        'heatwaves',
                        'threshold-exceeding-events',
                        'pct95-min3days',
                        '2003-0391-FRA.csv')) 

# Load outputs on subdivision level
subdiv <- st_read((here('data-output',
                        'heatwaves',
                        'shedis_heatwaves_subdivision.gpkg'))) %>% 
  filter(disno == '2003-0391-FRA') %>% 
  mutate(logperson = log10(pct95_persondays + 1))
```

    ## Reading layer `shedis_heatwaves_subdivision' from data source 
    ##   `C:\Users\saran173\Documents\2-Research\2024-shedis-temp\3-analysis\R-shedis-temp\data-output\heatwaves\shedis_heatwaves_subdivision.gpkg' 
    ##   using driver `GPKG'
    ## Simple feature collection with 1039 features and 49 fields
    ## Geometry type: MULTIPOLYGON
    ## Dimension:     XY
    ## Bounding box:  xmin: -124.7628 ymin: -55.11694 xmax: 174.0633 ymax: 63.69792
    ## Geodetic CRS:  WGS 84

``` r
ev_haz <- events %>% 
  distinct(x, y, event_start, .keep_all = T) %>% 
  group_by(x, y) %>% 
  summarise(
    max_atx = max(max_atx),
    max_tx = max(max_tx),
    duration = max(duration),
    magnitude = max(magnitude)
  ) %>% 
  ungroup() 
```

    ## `summarise()` has grouped output by 'x'. You can override using the `.groups`
    ## argument.

``` r
ev_pop <- events %>% 
  group_by(x, y, event_start) %>%
  summarise(
    persondays = sum(persondays)
  ) %>%
  ungroup() %>% 
  group_by(x, y) %>% 
  summarise(
    persondays = max(persondays)
  ) %>% 
  ungroup()
```

    ## `summarise()` has grouped output by 'x', 'y'. You can override using the
    ## `.groups` argument.
    ## `summarise()` has grouped output by 'x'. You can override using the `.groups`
    ## argument.

``` r
ev <- ev_haz %>% 
  inner_join(ev_pop, by = c('x','y')) %>% 
  st_as_sf(coords = c("x", "y"), crs = 4326)

# Load raster
r <- rast('F:/mswx/temp/3hourly/1979001.00.nc')

# Crop raster
cropped_r <- mask(crop(r, vect(df)), vect(df))

# Convert event-points to raster
max_tx <- rasterize(vect(ev), cropped_r, field = "max_tx", fun = "mean") %>% rename(max_tx = mean)
duration <- rasterize(vect(ev), cropped_r, field = "duration", fun = "mean") %>% rename(duration = mean)
persondays <- rasterize(vect(ev), cropped_r, field = "persondays", fun = "mean") %>% rename(persondays = mean)
logpersondays <- log10(persondays + 1) # Apply log10 transformation for visualization purposes

europe <- ne_countries(scale = "large", returnclass = "sf") %>%  filter(continent == "Europe")
france <- europe %>% filter(admin == "France")

fig_p <- ggplot() +
  geom_sf(data = france,
          color = NA,
          fill = "#e0e0e0") +
  geom_spatraster(data = logpersondays, aes(fill = persondays)) +
  scale_fill_viridis_c(
    name = "log10(pct95 person-days + 1)",
    na.value = "transparent",
    # limits = c(0, 8.2),
    option = "viridis"
  ) +
  geom_sf(data = df, fill = NA, color = "black", size = 1) +
  coord_sf(xlim = c(-5, 11), ylim = c(41, 52), expand = FALSE) +
  cowplot::theme_map() +
  theme(panel.grid.major = element_line(color = "transparent"),
        legend.title = element_text(size = 8),
        legend.text = element_text(size = 8),
        legend.position = "bottom") + 
  guides(color = guide_legend(title = NULL, nrow = 1)) +
  cowplot::panel_border(remove = TRUE)

fig_ps <- ggplot() +
  geom_sf(data = france,
          color = NA,
          fill = "#e0e0e0") +
  geom_sf(data = subdiv, aes(fill = logperson), color = "black", size = 1) +
  scale_fill_viridis_c(
    name = "log10(total pct95 person-days + 1)",
    na.value = "transparent",
    # limits = c(0, 8.2),
    option = "viridis"
  ) +
  coord_sf(xlim = c(-5, 11), ylim = c(41, 52), expand = FALSE) +
  cowplot::theme_map() +
  theme(panel.grid.major = element_line(color = "transparent"),
        legend.title = element_text(size = 8),
        legend.text = element_text(size = 8),
        legend.position = "bottom") + 
  guides(color = guide_legend(title = NULL, nrow = 1)) +
  cowplot::panel_border(remove = TRUE)

fig_d <- ggplot() +
  geom_sf(data = france,
          color = NA,
          fill = "#e0e0e0") +
  geom_spatraster(data = duration, aes(fill = duration)) +
  scale_fill_viridis_c(
    name = "pct95 duration",
    na.value = "transparent",
    # limits = c(3, 31),
    option = "viridis"
  ) +
  geom_sf(data = df, fill = NA, color = "black", size = 1) +
  coord_sf(xlim = c(-5, 11), ylim = c(41, 52), expand = FALSE) +
  cowplot::theme_map() +
  theme(panel.grid.major = element_line(color = "transparent"),
        legend.title = element_text(size = 8),
        legend.text = element_text(size = 8),
        legend.position = "bottom") + 
  guides(color = guide_legend(title = NULL, nrow = 1)) +
  cowplot::panel_border(remove = TRUE)

fig_ds <- ggplot() +
  geom_sf(data = france,
          color = NA,
          fill = "#e0e0e0") +
  geom_sf(data = subdiv, aes(fill = pct95_median_duration), color = "black", size = 1) +
  scale_fill_viridis_c(
    name = "median pct95 duration",
    na.value = "transparent",
    # limits = c(3, 31),
    option = "viridis"
  ) +
  coord_sf(xlim = c(-5, 11), ylim = c(41, 52), expand = FALSE) +
  cowplot::theme_map() +
  theme(panel.grid.major = element_line(color = "transparent"),
        legend.title = element_text(size = 8),
        legend.text = element_text(size = 8),
        legend.position = "bottom") + 
  guides(color = guide_legend(title = NULL, nrow = 1)) +
  cowplot::panel_border(remove = TRUE)

fig_t <- ggplot() +
  geom_sf(data = france,
          color = NA,
          fill = "#e0e0e0") +
  geom_spatraster(data = max_tx, aes(fill = max_tx)) +
  scale_fill_viridis_c(
    name = "Max tx",
    na.value = "transparent",
    # limits = c(18, 41),
    option = "viridis"
  ) +
  geom_sf(data = df, fill = NA, color = "black", size = 1) +
  coord_sf(xlim = c(-5, 11), ylim = c(41, 52), expand = FALSE) +
  cowplot::theme_map() +
  theme(panel.grid.major = element_line(color = "transparent"),
        legend.title = element_text(size = 8),
        legend.text = element_text(size = 8),
        legend.position = "bottom") + 
  guides(color = guide_legend(title = NULL, nrow = 1)) +
  cowplot::panel_border(remove = TRUE)

fig_ts <- ggplot() +
  geom_sf(data = france,
          color = NA,
          fill = "#e0e0e0") +
  geom_sf(data = subdiv, aes(fill = max_tx), color = "black", size = 1) +
  scale_fill_viridis_c(
    name = "Zonal average of max tx",
    na.value = "transparent",
    # limits = c(18, 41),
    option = "viridis"
  ) +
  coord_sf(xlim = c(-5, 11), ylim = c(41, 52), expand = FALSE) +
  cowplot::theme_map() +
  theme(panel.grid.major = element_line(color = "transparent"),
        legend.title = element_text(size = 8),
        legend.text = element_text(size = 8),
        legend.position = "bottom") + 
  guides(color = guide_legend(title = NULL, nrow = 1)) +
  cowplot::panel_border(remove = TRUE)

fig <- ggarrange(fig_t, fig_ts,
                  fig_d, fig_ds,
                  fig_p, fig_ps,
                  ncol = 2, nrow = 3,
                  align = "hv")

# Export
finalise_plot(plot_name = fig,
              save_filepath = here("figures","f07.pdf"),
              source_name = NULL,
              width_pixels = 500, height_pixels = 700)
```

![](figures_files/figure-gfm/fig-7-1.png)<!-- -->

## Fig 6

``` r
rm(list=ls()); cat('\f') # Clear console
```



``` r
# Define parameters
crs <- 8857 # Projection. Alternatives: Mollweide "+proj=moll", Equal Earth 8857, Robinson "+proj=robin"
jitter <- 0.4 # Jitter. To avoid complete overlaps on map

# Load basemap from Natural Earth
world <- ne_countries(scale = "medium", returnclass = "sf") %>%
  filter(name != "Antarctica")

# Load sample
df_h <- st_read(here("data-output", "heatwaves", "shedis_heatwaves_subdivision.gpkg")) %>% 
  st_make_valid() %>%
  st_centroid() %>%
  st_jitter(jitter)
```

    ## Reading layer `shedis_heatwaves_subdivision' from data source 
    ##   `C:\Users\saran173\Documents\2-Research\2024-shedis-temp\3-analysis\R-shedis-temp\data-output\heatwaves\shedis_heatwaves_subdivision.gpkg' 
    ##   using driver `GPKG'
    ## Simple feature collection with 1039 features and 49 fields
    ## Geometry type: MULTIPOLYGON
    ## Dimension:     XY
    ## Bounding box:  xmin: -124.7628 ymin: -55.11694 xmax: 174.0633 ymax: 63.69792
    ## Geodetic CRS:  WGS 84

    ## Warning: st_centroid assumes attributes are constant over geometries

``` r
df_c <- st_read(here("data-output", "coldwaves", "shedis_coldwaves_subdivision.gpkg")) %>% 
  st_make_valid() %>%
  st_centroid() %>%
  st_jitter(jitter)
```

    ## Reading layer `shedis_coldwaves_subdivision' from data source 
    ##   `C:\Users\saran173\Documents\2-Research\2024-shedis-temp\3-analysis\R-shedis-temp\data-output\coldwaves\shedis_coldwaves_subdivision.gpkg' 
    ##   using driver `GPKG'
    ## Simple feature collection with 1796 features and 49 fields
    ## Geometry type: MULTIPOLYGON
    ## Dimension:     XY
    ## Bounding box:  xmin: -180 ymin: -55.98403 xmax: 180 ymax: 81.275
    ## Geodetic CRS:  WGS 84

    ## Warning: st_centroid assumes attributes are constant over geometries

``` r
# Transform projections
world <- st_transform(world, crs = st_crs(crs))
df_h <- st_transform(df_h, crs = st_crs(crs))
df_c <- st_transform(df_c, crs = st_crs(crs))

# Find even break values of pop numbers close to the tertiles
pop <- combine(df_h %>% as.data.frame() %>% select(geometry_pop), df_c %>% as.data.frame() %>% select(geometry_pop))
```

    ## Warning: `combine()` was deprecated in dplyr 1.0.0.
    ## ℹ Please use `vctrs::vec_c()` instead.
    ## Call `lifecycle::last_lifecycle_warnings()` to see where this warning was
    ## generated.

``` r
print(quantile(pop$geometry_pop, probs = c(1/3, 2/3)))
```

    ## 33.33333% 66.66667% 
    ##    435424   2017411

``` r
df_h <- df_h %>%
  mutate(
    # Define population groups
    pop_group = case_when(
      geometry_pop < 500000 ~ "<500k",
      geometry_pop < 2000000 ~ "<2000k",
      geometry_pop >= 2000000 ~ ">=2000k"
    ),
    pop_group = factor(pop_group, levels = c("<500k", "<2000k", ">=2000k"), ordered = TRUE),
    # Define year
    year = as.integer(substr(disno, 1, 4))
  ) %>%
  # Arrange so smaller dots will be drawn on top of larger dots
  arrange(desc(pop_group)) 

df_c <- df_c %>%
  mutate(
    # Define population groups
    pop_group = case_when(
      geometry_pop < 500000 ~ "<500k",
      geometry_pop < 2000000 ~ "<2000k",
      geometry_pop >= 2000000 ~ ">=2000k"
    ),
    pop_group = factor(pop_group, levels = c("<500k", "<2000k", ">=2000k"), ordered = TRUE),
    # Define year
    year = as.integer(substr(disno, 1, 4))
  ) %>%
  # Arrange so smaller dots will be drawn on top of larger dots
  arrange(desc(pop_group)) 

# Find even break values for heat wave temperatures
print(quantile(df_h$max_atx, probs = c(1/3, 2/3)))
```

    ## 33.33333% 66.66667% 
    ##     34.00     37.28

``` r
df_h <- df_h %>%
  mutate(
    # Define heat wave temperature groups
    atx_group = case_when(
      max_atx < 35 ~ 1,
      max_atx < 40 ~ 2,
      max_atx >= 40 ~ 3
    ),
    atx_group = factor(atx_group,
                       levels = c(1,2,3),
                       labels = c('<35', '<40', '>=40'), ordered = TRUE)
  )

# Find even break values for cold wave temperatures
print(quantile(df_c$min_atn, probs = c(1/3, 2/3)))
```

    ## 33.33333% 66.66667% 
    ##    -21.38     -7.29

``` r
df_c <- df_c %>%
  mutate(
    # Define cold wave temperature groups
    atn_group = case_when(
      min_atn > -10 ~ 1,
      min_atn > -25 ~ 2,
      min_atn <= -25 ~ 3
    ),
    atn_group = factor(atn_group,
                       levels = c(1,2,3),
                       labels = c('>-10', '>-25', '<=-25'), ordered = TRUE)
  )

# Plotting parameters
plot_params <- tibble(
  landmass_color = "#e0e0e0",
  stroke_color = "white",
  point_size = 2.5, # Size points on map
  alpha_point_fill = 0.3, # Transparency fill color points
  alpha_point_stroke = 0.3, # Transparency stroke color points
  lwd_point_stroke = 0.01, # Thickness stroke points
  legend_position = "bottom"
)

fig_atx <- ggplot() +
  geom_sf(data = world, color = NA, fill = plot_params$landmass_color) +
  geom_sf(data = df_h, aes(size = pop_group, color = atx_group),
          alpha = plot_params$alpha_point_fill, shape = 16) +
  scale_size_manual(values = c("<500k" = .75, "<2000k" = 3, ">=2000k" = 6)) +
  cowplot::theme_map() +
  theme(
    panel.grid.major = element_line(color = "transparent"),  
    legend.position = plot_params$legend_position
  ) + 
  guides(color = guide_legend(title = NULL, nrow = 1)) +
  cowplot::panel_border(remove = TRUE)

fig_atn <- ggplot() +
  geom_sf(data = world, color = NA, fill = plot_params$landmass_color) +
  geom_sf(data = df_c, aes(size = pop_group, color = atn_group),
          alpha = plot_params$alpha_point_fill, shape = 16) +
  scale_size_manual(values = c("<500k" = .75, "<2000k" = 3, ">=2000k" = 6)) +
  cowplot::theme_map() +
  theme(
    panel.grid.major = element_line(color = "transparent"),  
    legend.position = plot_params$legend_position
  ) + 
  guides(color = guide_legend(title = NULL)) +
  cowplot::panel_border(remove = TRUE)

fig <- ggarrange(fig_atn, fig_atx, align = "hv")

# Export
finalise_plot(plot_name = fig,
              save_filepath = here("figures","f06_maps.pdf"),
              source_name = "Projection Equal Earth",
              width_pixels = 800, height_pixels = 800)
```

![](figures_files/figure-gfm/fig-6-maps-1.png)<!-- -->

``` r
rm(list=ls()); cat('\f') # Clear console
```



``` r
# Load sample
df_h <- read.csv(here("data-output", "heatwaves", "shedis_heatwaves_subdivision.csv"), header = TRUE) %>%
  mutate(continent = factor(countrycode(iso3c, origin = "iso3c", destination = "continent")),
         year = substr(analysis_start,1,4),
         year = as.integer(year))

df_c <- read.csv(here("data-output", "coldwaves", "shedis_coldwaves_subdivision.csv"), header = TRUE) %>%
  mutate(continent = factor(countrycode(iso3c, origin = "iso3c", destination = "continent")),
         year = substr(analysis_start,1,4),
         year = as.integer(year))

fig_h <- ggplot(df_h, aes(x=geometry_pop, y=max_atx)) +
  geom_point(aes(colour = continent), alpha = 0.5, shape = 16, size = 5) +
  geom_smooth(method = 'lm', se=TRUE, color = 'gray')+
  theme_tq() +
  scale_x_log10() +
  theme(panel.grid.minor = element_blank(),
        panel.grid.major = element_blank(),
        panel.border = element_blank(),
        axis.line = element_line(),
        legend.title = element_blank())

fig_c <- ggplot(df_c, aes(x=geometry_pop, y=min_atn)) +
  geom_point(aes(colour = continent), alpha = 0.5, shape = 16, size = 5) +
  geom_smooth(method = 'lm', se=TRUE, color = 'gray')+
  theme_tq() +
  scale_x_log10() +
  theme(panel.grid.minor = element_blank(),
        panel.grid.major = element_blank(),
        panel.border = element_blank(),
        axis.line = element_line(),
        legend.title = element_blank())

fig <- ggarrange(fig_c, fig_h, ncol = 1, nrow = 2, align = "hv", common.legend = F, legend = 'bottom')
```

    ## `geom_smooth()` using formula = 'y ~ x'
    ## `geom_smooth()` using formula = 'y ~ x'

``` r
finalise_plot(plot_name = fig,
              save_filepath = here("figures","f06_scatter.pdf"),
              source_name = NULL,
              width_pixels = 500,
              height_pixels = 800)
```

![](figures_files/figure-gfm/fig-6-scatterplots-1.png)<!-- -->

## Fig 8 - Compare EM-DAT and our estimates

``` r
rm(list=ls()); cat('\f') # Clear console
```



``` r
# Load emdat
emdat <- readRDS(here("data-processed", "emdat.rds")) %>%
  filter(!is.na(magnitude)) %>%
  mutate(continent = factor(countrycode(iso, origin = "iso3c", destination = "continent"))) %>% 
  rename(emdat_magnitude = magnitude)

df_h <- read.csv(here("data-output", "heatwaves", "shedis_heatwaves_disno.csv"), header = TRUE) %>% 
  filter(disno %in% emdat$disno) %>% 
  left_join(emdat, by = 'disno')

df_c <- read.csv(here("data-output", "coldwaves", "shedis_coldwaves_disno.csv"), header = TRUE) %>% 
  filter(disno %in% emdat$disno) %>% 
  left_join(emdat, by = 'disno')

limheat <- c(20,63)
limcold <- c(-63,20)

npg_pal <- pal_npg('nrc')(10)
cont_pal <- c(npg_pal[9],npg_pal[3:5],npg_pal[7]) # Palette for continents

fig_h <- ggplot(df_h, aes(y = emdat_magnitude, x = xy_max_tx)) +
  geom_point(aes(colour = continent), alpha = 0.7, shape = 16, size = 5) +
  # geom_text(aes(label = disno)) +
  scale_color_manual(values = cont_pal) +
  stat_poly_line(col = 'grey', se = T, linewidth = 1) +
  stat_poly_eq() +
  scale_y_continuous(limits = limheat,
                     breaks = seq(from = 20, to = 60, by = 10)) +
  scale_x_continuous(limits = limheat,
                     breaks = seq(from = 20, to = 60, by = 10)) +
  labs(x = "xy max tx (\u00B0C), MSWX",
       y = "Magnitude (\u00B0C), EM-DAT",
       title = 'Heat waves') +
  theme_tq() +
  geom_abline(slope = 1, intercept = 0, col = 'grey', linetype = 'dashed') +
  theme(panel.grid.minor = element_blank(),
        panel.grid.major = element_blank(),
        panel.border = element_blank(),
        axis.line = element_line(),
        legend.title = element_blank(),
        aspect.ratio = 1)

fig_c <- ggplot(df_c, aes(y = emdat_magnitude, x = xy_min_tn)) +
  geom_point(aes(colour = continent), alpha = 0.7, shape = 16, size = 5) +
  # geom_text(aes(label = disno)) +
  scale_color_manual(values = cont_pal) +
  stat_poly_line(col = 'grey', se = T, linewidth = 1) +
  stat_poly_eq() +
  scale_y_continuous(limits = limcold,
                     breaks = seq(from = -60, to = 20, by = 10)) +
  scale_x_continuous(limits = limcold,
                     breaks = seq(from = -60, to = 20, by = 10)) +
  labs(x = "xy min tn (\u00B0C), MSWX",
       y = "Magnitude (\u00B0C), EM-DAT",
       title = 'Cold waves') +
  theme_tq() +
  geom_abline(slope = 1, intercept = 0, col = 'grey', linetype = 'dashed') +
  theme(panel.grid.minor = element_blank(),
        panel.grid.major = element_blank(),
        panel.border = element_blank(),
        axis.line = element_line(),
        legend.title = element_blank(),
        aspect.ratio = 1)

fig <- ggarrange(fig_c, fig_h, ncol = 2, nrow = 1, align = "hv", legend = 'bottom')
```

    ## Warning: The `scale_name` argument of `continuous_scale()` is deprecated as of ggplot2
    ## 3.5.0.
    ## This warning is displayed once every 8 hours.
    ## Call `lifecycle::last_lifecycle_warnings()` to see where this warning was
    ## generated.

    ## Warning: The `trans` argument of `continuous_scale()` is deprecated as of ggplot2 3.5.0.
    ## ℹ Please use the `transform` argument instead.
    ## This warning is displayed once every 8 hours.
    ## Call `lifecycle::last_lifecycle_warnings()` to see where this warning was
    ## generated.

``` r
finalise_plot(plot_name = fig,
              save_filepath = here("figures","f08_mswx_emdat.pdf"),
              source_name = NULL,
              width_pixels = 600,
              height_pixels = 700)
```

![](figures_files/figure-gfm/fig-8-1.png)<!-- -->

``` r
# Mean Absolute Error, the average absolute difference between the vectors
mae_h <- round(Metrics::mae(df_h$emdat_magnitude, df_h$xy_max_tx), digits = 2)
mae_c <- round(Metrics::mae(df_c$emdat_magnitude, df_c$xy_min_tn), digits = 2)

mae_h_daily <- round(Metrics::mae(df_h$emdat_magnitude, df_h$xy_max_t), digits = 2)
mae_c_daily <- round(Metrics::mae(df_c$emdat_magnitude, df_c$xy_min_t), digits = 2)

bias_h <- round(Metrics::bias(df_h$emdat_magnitude, df_h$xy_max_tx), digits = 2) # the average amount which EM-DAT is greater than MSWX
bias_c <- round(Metrics::bias(df_c$emdat_magnitude, df_c$xy_min_tn), digits = 2) # the average amount which EM-DAT is greater than MSWX
```

End of script.
