Calculate Percentiles and Collect Tiles
================
Sara Lindersson
2024-10-14

This script calculates percentiles for daily maximum (TX) and minimum
(TN) temperature values, and subsequently collects the resulting tiles.

``` r
library(glue)
library(knitr)
```

``` r
# Set working directory
knitr::opts_knit$set(root.dir = normalizePath('F:/mswx/data-processed/')) 
```

``` r
ph_in_tn <- '05/tn_tiles_detr_mean'
ph_in_tx <- '05/tx_tiles_detr_mean'

ph_out <- '06/percentiles'

# Create output directory if it doesn't exist
if (!dir.exists(ph_out)) {
  dir.create(ph_out, recursive = TRUE)
}
```

### Percentiles of TN

#### ctn10pct

``` r
# Directory for tiles
ph_out_tiles <- '06/ctn10pct'

# Create output directory if it doesn't exist
if (!dir.exists(ph_out_tiles)) {
  dir.create(ph_out_tiles, recursive = TRUE)
}

# List all the .nc files in the input directory
files <- list.files(ph_in_tn, pattern = "\\.nc$", full.names = TRUE)

for (file in files) {
  # Get the file name
  file_name <- sub('^detr_avg_tn_', '', basename(file))
  # Define the output file path
  out_file <- file.path(ph_out_tiles, paste0("ctn10pct_", file_name))

  # Construct the CDO command to calculate the percentiles
  cmd <- paste('wsl cdo -O -b 32 -z zip_1 -ydrunpctl,10,31 [', file, '-ydrunmin,31', file, '-ydrunmax,31', file, ']', out_file)

  # Execute the CDO command
  shell(cmd = cmd)
}

# Collect tiles
inputs <- paste0(ph_out_tiles, '/ctn10pct_*.nc')
output <- file.path(ph_out, 'ctn10pct.nc')

cmd <- paste('wsl cdo -O -b 32 -z zip_1 -collgrid', inputs, output)
shell(cmd = cmd)
```

#### ctn05pct

``` r
# Directory for tiles
ph_out_tiles <- '06/ctn05pct'

# Create output directory if it doesn't exist
if (!dir.exists(ph_out_tiles)) {
  dir.create(ph_out_tiles, recursive = TRUE)
}

# List all the .nc files in the input directory
files <- list.files(ph_in_tn, pattern = "\\.nc$", full.names = TRUE)

for (file in files) {
  # Get the file name
  file_name <- sub('^detr_avg_tn_', '', basename(file))
  # Define the output file path
  out_file <- file.path(ph_out_tiles, paste0("ctn05pct_", file_name))
  
  # Construct the CDO command to calculate the percentiles
  cmd <- paste('wsl cdo -O -b 32 -z zip_1 -ydrunpctl,5,31 [', file, '-ydrunmin,31', file, '-ydrunmax,31', file, ']', out_file)
  
  # Execute the CDO command
  shell(cmd = cmd)
}

# Collect tiles
inputs <- paste0(ph_out_tiles, '/ctn05pct_*.nc')
output <- file.path(ph_out, 'ctn05pct.nc')

cmd <- paste('wsl cdo -O -b 32 -z zip_1 -collgrid', inputs, output)
shell(cmd = cmd)
```

### Percentiles of TX

#### ctx90pct

``` r
# Directory for tiles
ph_out_tiles <- '06/ctx90pct'

# Create output directory if it doesn't exist
if (!dir.exists(ph_out_tiles)) {
  dir.create(ph_out_tiles, recursive = TRUE)
}

# List all the .nc files in the input directory
files <- list.files(ph_in_tx, pattern = "\\.nc$", full.names = TRUE)

for (file in files) {
  # Get the file name
  file_name <- sub('^detr_avg_tx_', '', basename(file))
  # Define the output file path
  out_file <- file.path(ph_out_tiles, paste0("ctx90pct_", file_name))
  
  # Construct the CDO command to calculate the percentiles
  cmd <- paste('wsl cdo -O -b 32 -z zip_1 -ydrunpctl,90,31 [', file, '-ydrunmin,31', file, '-ydrunmax,31', file, ']', out_file)
  
  # Execute the CDO command
  shell(cmd = cmd)
}

# Collect tiles
inputs <- paste0(ph_out_tiles, '/ctx90pct_*.nc')
output <- file.path(ph_out, 'ctx90pct.nc')

cmd <- paste('wsl cdo -O -b 32 -z zip_1 -collgrid', inputs, output)
shell(cmd = cmd)
```

#### ctx95pct

``` r
# Directory for tiles
ph_out_tiles <- '06/ctx95pct'

# Create output directory if it doesn't exist
if (!dir.exists(ph_out_tiles)) {
  dir.create(ph_out_tiles, recursive = TRUE)
}

# List all the .nc files in the input directory
files <- list.files(ph_in_tx, pattern = "\\.nc$", full.names = TRUE)

for (file in files) {
  # Get the file name
  file_name <- sub('^detr_avg_tx_', '', basename(file))
  # Define the output file path
  out_file <- file.path(ph_out_tiles, paste0("ctx95pct_", file_name))
  
  # Construct the CDO command to calculate the percentiles
  cmd <- paste('wsl cdo -O -b 32 -z zip_1 -ydrunpctl,95,31 [', file, '-ydrunmin,31', file, '-ydrunmax,31', file, ']', out_file)
  
  # Execute the CDO command
  shell(cmd = cmd)
}

# Collect tiles
inputs <- paste0(ph_out_tiles, '/ctx95pct_*.nc')
output <- file.path(ph_out, 'ctx95pct.nc')

cmd <- paste('wsl cdo -O -b 32 -z zip_1 -collgrid', inputs, output)
shell(cmd = cmd)
```

End of script.
