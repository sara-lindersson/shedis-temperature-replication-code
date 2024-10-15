# SHEDIS-Temperature
This dataset links national disaster impact records with subnational information on physical hazards and human exposure. Version 1.0 includes data on 382 heat waves and cold waves from 1979 to 2018, based on records from the EM-DAT international disaster database.

+ Related publication: [upcoming]  
+ Harvard Dataverse: https://doi.org/10.7910/DVN/WNOTTC  
+ GitHub Repository: https://github.com/sara-lindersson/shedis-temperature

## Ready-to-Use dataset
The ready-to-use data are located in the `data-output` folder in the [Harvard Dataverse repository](https://doi.org/10.7910/DVN/WNOTTC), which contains six files:

+ `data-disno-level.tab`: Key attributes for the EM-DAT record, across all relevant administrative units. The event attributes come from the threshold analysis at disno level, using the 5th percentile as threshold. 
+ `data-disno-level.gpkg`: Same as above but with geometry outlining the adm units.

+ `data-adm-unit-level.tag`: Key attributes for the EM-DAT record, quantified per relevant administrative unit. The event attributes come from the threshold analysis at administrative unit level, using the 5th percentile as threshold.  
+ `data-adm-unit-level.gpkg`: Same as above but with geometry outlining the adm unit.  

+ `data-grid-cell-level.tab`: Key attributes for the EM-DAT record, quantified at grid cell level within each relevant administrative unit. The event attributes come from the threshold analysis at grid cell level, using the 5th percentile as threshold.    
+ `data-grid-cell-level.gpkg`: Same as above but with geometry outlining the adm unit.    

Additionally, the `data-output` folder includes the subfolder `events`, which lists identified heat waves or cold waves per disaster number (disno), quantified at either grid cell-, admin unit- or disno-level. There will only be a file in the subfolder if a heat wave or cold wave could be detected in the relevant area during the analysis period.

## Replication Scripts
The dataset contains R notebooks for generating the SHEDIS-Temperature outputs, which are located under in the `scripts` folder. This is available at both [Harvard Dataverse](https://doi.org/10.7910/DVN/WNOTTC) and[GitHub](https://github.com/sara-lindersson/shedis-temperature).

### Repository structure
The scripts are divided into three main directories:  

+ `a-preprocess-mswx`: Scripts for preprocessing raw meteorological data from the MSWX dataset.  
+ `b-process-data`: Scripts for linking disaster impact records from EM-DAT to subnational administrative units (GADM v3.6), using geocoding from GDIS. These scripts also integrate meteorological data (MSWX) with annual population estimates from GHS-POP.  
+ `c-derive-outputs`: Scripts for generating the final dataset, including identifying heat wave and cold wave events through threshold analysis and summarizing key attributes for each disaster record.

A `master-script` in the root directory orchestrates the execution of these scripts in the correct order. Each script provides further details about its functionality.

### Required Data
To replicate the results, you will need the following datasets:

+ __Processed data__ needed for running the scripts in the `c-derive-outputs` folder can be downloaded from the Harvard Dataverse repository in the `data-processed` folder. This directory contains the file `subnat.rds` and a subfolder with `daily-data`. Place the `data-processed` folder in the project’s root directory before running the scripts. 
+ __Raw data__ for running scripts in the `a-preprocess-mswx` and `b-process-data` should be downloaded from their respective sources. Save the raw data (EM-DAT, GDIS, GADM) in a `data-raw` folder in the project’s root directory. For MSWX and GHS-POP data, update the file paths as needed to match your local setup.

## Data sources
+ __EM-DAT__: CRED, UCLouvain, Brussels, Belgium. https://www.emdat.be/
+ __GADM v3.6__: https://gadm.org/
+ __GDIS__: Rosvold, E. L. and Buhaug, H. (2021) ‘GDIS, a global dataset of geocoded disaster locations’, Scientific Data, 8(1), p. 61. https://doi.org/10.1038/s41597-021-00846-6
+ __GHS-POP__: Schiavina M., Freire S., Carioli A., MacManus K. (2023) GHS-POP R2023A - GHS population grid multitemporal (1975-2030). Resolution 30 arcsec. European Commission, Joint Research Centre (JRC). https://doi.org/10.2905/2FF68A52-5B5B-4A22-8F40-C41DA8332CFE
+ __MSWX__: Beck, H. E. et al. (2022) ‘MSWX: Global 3-Hourly 0.1° Bias-Corrected Meteorological Data Including Near-Real-Time Updates and Forecast Ensembles’, Bulletin of the American Meteorological Society. Boston MA, USA: American Meteorological Society, 103(3), pp. E710–E732. https://doi.org/10.1175/BAMS-D-21-0145.1

## Citation and License
Please cite the dataset as follows:  

+ Lindersson, Sara & Messori, Gabriele (2024). _SHEDIS-Temperature_. https://doi.org/10.7910/DVN/WNOTTC, Harvard Dataverse, version 1.0.

This dataset is licensed under the Creative Commons Attribution License CC BY-NC 4.0.

## Contact
Sara Lindersson, sara.lindersson@geo.uu.se
