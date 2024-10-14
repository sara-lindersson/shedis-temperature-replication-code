# SHEDIS-Temperature Replication Scripts
This repository contains R notebooks used to generate the SHEDIS-Temperature dataset. The dataset links national disaster impact records with subnational information on physical hazards and human exposure. Version 1.0 includes data on 382 heat waves and cold waves from 1979 to 2018, as recorded in the EM-DAT international disaster database.

SHEDIS-Temperature is available at the Harvard Dataverse: https://doi.org/10.7910/DVN/WNOTTC

## Repository Structure
The scripts are organized into three main folders:  
+ _a-preprocess-mswx_: Scripts for preprocessing raw meteorological data from the MSWX dataset.  
+ _b-process-data_: Scripts for linking disaster impact records from EM-DAT to subnational administrative units (GADM v3.6) using geocoding from GDIS. These scripts also integrate meteorological data (MSWX) with annual population estimates from GHS-POP.  
+ _c-derive-outputs_: Scripts for generating the final dataset outputs. This includes identifying heat wave and cold wave events through threshold analysis and summarizing key attributes for each disaster record.  

The home folder also contains a _master-script_ that orchestrates the execution of all scripts in the correct order within their respective subfolders.

## Required Data
To replicate the results, follow these data requirements:  
+ __Processed data__ needed for running the scripts in the _c-derive-outputs_ folder can be downloaded from the Harvard Dataverse repository in the _data-processed_ folder. Place this data in the project’s home directory before executing the scripts.  
+ __Raw data__ for the _a-preprocess-mswx_ and _b-process-data_ folders must be downloaded from their respective sources. Save the raw data (EM-DAT, GDIS, GADM) in a folder named _data-raw_, located in the project's home directory. Additionally, data from MSWX and GHS-POP are currently referenced on an external disk in the scripts. Be sure to update the file paths accordingly when running the scripts on your system.
