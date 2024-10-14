# SHEDIS-Temperature Replication Scripts
This repository contains R notebooks for generating the SHEDIS-Temperature dataset, which links national disaster impact records with subnational information on physical hazards and human exposure. Version 1.0 includes data on 382 heat waves and cold waves from 1979 to 2018, based on records from the EM-DAT international disaster database.  

+ Related publication: [upcoming]  
+ SHEDIS-Temperature dataset: Available at Harvard Dataverse https://doi.org/10.7910/DVN/WNOTTC  

## Repository Structure
The scripts are organized into three main folders:  
+ _a-preprocess-mswx_: Scripts for preprocessing raw meteorological data from the MSWX dataset.  
+ _b-process-data_: Scripts for linking disaster impact records from EM-DAT to subnational administrative units (GADM v3.6) using geocoding from GDIS. These scripts also integrate meteorological data (MSWX) with annual population estimates from GHS-POP.  
+ _c-derive-outputs_: Scripts for generating the final dataset outputs. This includes identifying heat wave and cold wave events through threshold analysis and summarizing key attributes for each disaster record.  

A _master-script_ in the home folder orchestrates the execution of all scripts in the correct order. Each script provides further details about its functionality.

## Required Data
To replicate the results, ensure the following data requirements are met: 
+ __Processed data__ needed for running the scripts in the _c-derive-outputs_ folder can be downloaded from the Harvard Dataverse repository in the _data-processed_ folder. Place this folder in the project’s home directory before executing the scripts.  
+ __Raw data__ for the _a-preprocess-mswx_ and _b-process-data_ folders must be downloaded from their respective sources. Save the raw data (EM-DAT, GDIS, GADM) in a folder named _data-raw_, located in the project's home directory. Additionally, data from MSWX and GHS-POP are currently referenced on an external disk in the scripts. Be sure to update the file paths accordingly when running the scripts on your system.

## Data sources
+ __EM-DAT__: CRED, UCLouvain, Brussels, Belgium. www.emdat.be
+ __GADM v3.6__: https://gadm.org/
+ __GDIS__: Rosvold, E. L. and Buhaug, H. (2021) ‘GDIS, a global dataset of geocoded disaster locations’, Scientific Data, 8(1), p. 61. doi: 10.1038/s41597-021-00846-6
+ __GHS-POP__: Schiavina M., Freire S., Carioli A., MacManus K. (2023) GHS-POP R2023A - GHS population grid multitemporal (1975-2030). Resolution 30 arcsec. European Commission, Joint Research Centre (JRC). doi:10.2905/2FF68A52-5B5B-4A22-8F40-C41DA8332CFE
+ __MSWX__: Beck, H. E. et al. (2022) ‘MSWX: Global 3-Hourly 0.1° Bias-Corrected Meteorological Data Including Near-Real-Time Updates and Forecast Ensembles’, Bulletin of the American Meteorological Society. Boston MA, USA: American Meteorological Society, 103(3), pp. E710–E732. doi: https://doi.org/10.1175/BAMS-D-21-0145.1

## Citation and License
Please cite the dataset as follows:  

Lindersson, Sara & Messori, Gabriele (2024). _SHEDIS-Temperature_. https://doi.org/10.7910/DVN/WNOTTC, Harvard Dataverse, version 1.0.

This dataset is licensed under the Creative Commons Attribution License, which permits re-distribution and re-use, provided that proper credit is given to the creators.

## Contact
Sara Lindersson, sara.lindersson@geo.uu.se
