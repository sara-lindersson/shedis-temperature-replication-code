# SHEDIS-Temperature
This dataset links national disaster impact records with subnational information on physical hazards and human exposure. Version 1.0 includes data on 382 heat waves and cold waves from 1979 to 2018, based on records from the EM-DAT international disaster database.

+ Data descriptor article: [upcoming]  
+ Harvard Dataverse for Dataset: https://doi.org/10.7910/DVN/WNOTTC
+ Harvard Dataverse for Replication data: [upcoming]  
+ GitHub Repository: https://github.com/sara-lindersson/shedis-temperature

## Dataset
The dataset is located in the `SHEDIS-Temperature` folder in the [Harvard Dataverse repository](https://doi.org/10.7910/DVN/WNOTTC). The data descriptor describes the content of the dataset files.

## Replication Scripts
This GitHub contains the R notebooks for generating the SHEDIS-Temperature outputs, which are located under in the `scripts` folder.

### Repository structure
The scripts are divided into four main directories:  

+ `a-preprocess-mswx`: Scripts for preprocessing raw meteorological data from the MSWX dataset.  
+ `b-process-data`: Scripts for linking disaster impact records from EM-DAT to subnational administrative units (GADM v3.6), using geocoding from GDIS. These scripts also integrate meteorological data (MSWX) with annual population estimates from GHS-POP.  
+ `c-derive-outputs`: Scripts for generating the final dataset, including identifying heat wave and cold wave events through threshold analysis and summarizing key attributes for each disaster record.
+ `d-figures-data-descriptor`: Scripts for generating the figures for the data descriptor article.

A `master-script` in the root directory orchestrates the execution of these scripts in the correct order. Each script provides further details about its functionality.

### Required Data
To replicate the results, you will need the following datasets:

+ __Processed data__ needed for running the scripts in the `c-derive-outputs` folder can be downloaded from the Harvard Dataverse repository containing the replication data. Place the replication data in a `data-processed` folder in the project’s root directory and unpack the .tar-files before running the scripts. 
+ __Raw data__ for running scripts in the `a-preprocess-mswx` and `b-process-data` should be downloaded from their respective sources. 

## Data sources
+ __EM-DAT__: CRED, UCLouvain, Brussels, Belgium. Downloaded 2023-10-02 from https://www.emdat.be/
+ __GADM v3.6__: Downloaded 2023-10-05 from https://gadm.org/
+ __GDIS__: Rosvold, E. L. and Buhaug, H. (2021) ‘GDIS, a global dataset of geocoded disaster locations’, Scientific Data, 8(1), p. 61. https://doi.org/10.1038/s41597-021-00846-6
+ __GHS-POP__: Schiavina M., Freire S., Carioli A., MacManus K. (2023) GHS-POP R2023A - GHS population grid multitemporal (1975-2030). Resolution 30 arcsec. European Commission, Joint Research Centre (JRC). https://doi.org/10.2905/2FF68A52-5B5B-4A22-8F40-C41DA8332CFE
+ __MSWX__: Beck, H. E. et al. (2022) ‘MSWX: Global 3-Hourly 0.1° Bias-Corrected Meteorological Data Including Near-Real-Time Updates and Forecast Ensembles’, Bulletin of the American Meteorological Society. Boston MA, USA: American Meteorological Society, 103(3), pp. E710–E732. https://doi.org/10.1175/BAMS-D-21-0145.1

## Citation and License
Please cite the dataset as follows:  

+ Lindersson, Sara & Messori, Gabriele (2024). _SHEDIS-Temperature_. https://doi.org/10.7910/DVN/WNOTTC, Harvard Dataverse, version 1.0.

This dataset is licensed under the Creative Commons Attribution License CC BY-NC 4.0.

## Contact
Sara Lindersson, sara.lindersson@geo.uu.se
