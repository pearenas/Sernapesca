SERNAPESCA Chile VMS Fishing Effort in Northern and Southern Regions
================
Esteban Arenas
8/27/2020

Objective: Sernapesca, the Chilean national government agency in charge
of regulating fisheries and aquaculture, is interested in having a
better understanding of the fishing effort of it´s industrial and
artisanal fleet as can be evidenced from the VMS data they have shared
with us. Specifically, Sernapesca is interested in knowing fishing
effort within the 15 regions that make up their EEZ and is interested in
exploring fishing effort for the months of January through July of 2020.
As a result, this analysis calculates fishing hours and total hours
(transit plus fishing hours) for all Chile VMS vessels within the 15
specified EEZ regions and for the specified months of January through
July of 2020.

Main Query Below - Extracting VMS and AIS Fishing Effort. In the end
decided to only use VMS because Chile VMS is pretty frequent and
complete. AIS didn’t seem to complement much and instead was picking up
vessels in Argentina that shared the name of Chilean vessels. Would be a
little complicated interpreting and explaining these results.

AIS was first extracted based on ssvid obtained from two different
tables and, because of how many different svvids were associated to the
same vessels, the AIS extracted dataset was then LEFT JOINED with VMS by
name in order to filter for entries exclusively found in AIS and not
VMS. The idea for this being to complement the VMS dataset with AIS.

``` r
query_string <- glue::glue('
CREATE TEMP FUNCTION hours_diff_abs(timestamp1 TIMESTAMP,
timestamp2 TIMESTAMP) AS (
#
# Return the absolute value of the diff between the two timestamps in hours with microsecond precision
# If either parameter is null, return null
#
ABS(TIMESTAMP_DIFF(timestamp1,
    timestamp2,
    microsecond) / 3600000000.0) );

WITH

# Extracted ssvid and ship names from our vessel databased matched table for Chile

RegistryJoined AS (
SELECT shipname_matched,
mmsi as ssvidRegistry
FROM `world-fishing-827.vessel_database_staging.matched_chl_v*`
),

# Extracted unique vessel names from the Chile VMS messages scored table

UniqueVMSNamesChile AS (
SELECT
DISTINCT n_shipname as n_shipnameVMS
FROM `world-fishing-827.pipe_chile_production_v20200331.messages_scored_*`
),

# Added ssvid to the unique vessels within VMS messages scored, matched by name with
# the matched registry table for Chile

VmsForAisMatching1 AS (
SELECT n_shipnameVMS, ssvidRegistry
  FROM UniqueVMSNamesChile
  LEFT JOIN RegistryJoined
    ON UniqueVMSNamesChile.n_shipnameVMS = RegistryJoined.shipname_matched
),

# Added another ssvid columns to the unique vessels within VMS messages scored, matched by name with
# the AIS IDs table

VmsForAisMatching2 AS (
SELECT n_shipnameVMS, ssvidRegistry,
`world-fishing-827.gfw_research.pipe_v20200203_ids`.ssvid as ssvidAIS
  FROM VmsForAisMatching1
  LEFT JOIN `world-fishing-827.gfw_research.pipe_v20200203_ids`
    ON VmsForAisMatching1.n_shipnameVMS = `world-fishing-827.gfw_research.pipe_v20200203_ids`.shipname
),

# Extracted AIS data with a time and lat lon thresholds of interest

AISChileTmp AS (
SELECT seg_id,ssvid,timestamp,source,nnet_score,
lat as latAIS,
lon as lonAIS,
EXTRACT(DATE from timestamp) as DateAIS,
EXTRACT (Month from timestamp) as Month, hours
FROM `world-fishing-827.gfw_research.pipe_v20200203`
WHERE timestamp BETWEEN TIMESTAMP("2020-01-01")
AND TIMESTAMP("2020-08-01")
AND lat > -69.568329 and lat < -8.502898 and lon > -121.203784 and lon < -46.672534
),

# Only kept AIS data that shared ssvids from either the Chile registry or the AIS vessel 
# name matched ssvid

AISChileTmp2 AS (
SELECT *,
VmsForAisMatching2.n_shipnameVMS as n_shipname_AIS
FROM AISChileTmp
INNER JOIN VmsForAisMatching2
  ON CAST(AISChileTmp.ssvid as INT64) = VmsForAisMatching2.ssvidRegistry
  
UNION DISTINCT

SELECT *,
VmsForAisMatching2.n_shipnameVMS as n_shipname_AIS
FROM AISChileTmp
INNER JOIN VmsForAisMatching2
  ON AISChileTmp.ssvid = VmsForAisMatching2.ssvidAIS
),

# Extracted VMS data with a time threshold of interest

VMSChileTmp AS (
SELECT seg_id,n_shipname,timestamp,source,nnet_score,
lat,lon,
EXTRACT(DATE from timestamp) as Date,
EXTRACT (Month from timestamp) as Month,
FROM `world-fishing-827.pipe_chile_production_v20200331.messages_scored_*`
WHERE timestamp BETWEEN TIMESTAMP("2020-01-01")
AND TIMESTAMP("2020-08-01")
),

# Calculating hours for VMS data, based on previous timestamps sharing the same seg_id

pos AS (
SELECT *,
  LAG(timestamp, 1) OVER (PARTITION BY seg_id  ORDER BY timestamp) prev_timestamp,
FROM VMSChileTmp
),

VMSChileFinal AS (
SELECT *
EXCEPT (prev_timestamp),
IFNULL (hours_diff_abs (timestamp, prev_timestamp), 0) hours
FROM pos
),

# Through a LEFT JOIN, delete all AIS data that shared the same ship name, date,
# lat bin, and lon bin with a VMS data entry. This is to later combine the two and avoid
# double counting

AISNotInVMS AS (
SELECT AISChileTmp2.seg_id,n_shipname_AIS as n_shipname,AISChileTmp2.timestamp,
AISChileTmp2.source,AISChileTmp2.nnet_score,
latAIS as lat,lonAIS as lon, DateAIS as Date, AISChileTmp2.Month, AISChileTmp2.hours,
VMSChileFinal.timestamp as VMStimestamp
FROM AISChileTmp2
LEFT JOIN VMSChileFinal
ON AISChileTmp2.n_shipname_AIS = VMSChileFinal.n_shipname
AND AISChileTmp2.DateAIS = VMSChileFinal.Date
AND  ROUND(AISChileTmp2.latAIS,2) = ROUND(VMSChileFinal.lat,2)
AND  ROUND(AISChileTmp2.lonAIS,2) = ROUND(VMSChileFinal.lon,2)
WHERE VMSChileFinal.timestamp IS NULL
),

# Uniting VMS data and AIS data not found in VMS

FishEffortTmp AS (
SELECT *,
floor(lat * 100) as lat_bin,
floor(lon * 100) as lon_bin,
FROM VMSChileFinal
UNION DISTINCT
SELECT *
EXCEPT (VMStimestamp),
floor(lat * 100) as lat_bin,
floor(lon * 100) as lon_bin,
FROM AISNotInVMS
),

# Summing fishing hours and total hours from the united data set, grouped
# by ship name, month, lat bin, and lon bin

FishEffortTmp2 AS (
SELECT n_shipname, Month,
lat_bin / 100 as lat_bin,
lon_bin / 100 as lon_bin,
SUM(IF(nnet_score > 0.5, hours, 0)) as fishing_hours,
SUM(hours) as total_hours
FROM FishEffortTmp
GROUP BY n_shipname, Month, lat_bin, lon_bin
),

#Transform hours/degrees to hours/km2

FishEffortFinal AS (
SELECT *, 
fishing_hours/(COS(udfs_v20200701.radians(lat_bin)) * (111/100)  * (111/100) ) AS fishing_hours_sq_km,
total_hours/(COS(udfs_v20200701.radians(lat_bin)) * (111/100)  * (111/100) ) AS total_hours_sq_km
FROM FishEffortTmp2
)

SELECT *
FROM FishEffortFinal
')
VMS_AIS_Hours_Final <- DBI::dbGetQuery(con, query_string)
# write.csv(VMS_AIS_Hours_Final, file = "VMS_AIS_Hours_Final.csv")
```

Extracting only Chile VMS Fishing Effort. This was the data set that
ended up being used for the analysis because of what was explained
above.

Worth noting here that around the bottom of this markdown is the code
used to determine the number of individual vessels in: - **The data used
for this analysis**: industrial and artisanal vessels within the Chilean
EEZ and for the months of January through July of 2020 (**754 total
vessels - 115 industrial and 639 artisanal**). Found in
“Resumen\_Embarcaciones\_Del\_Estudio.csv” - **All of the Chile VMS
data**: total number of industrial and artisanal vessels in the entire
Chile VMS data set,Feb 2019 - Aug 2020. (**1,108 total vessels - 141
industrial and 967 artisanal**). Found in
“Resumen\_Embarcaciones\_VMS\_Total.csv”

``` r
query_string <- glue::glue('
CREATE TEMP FUNCTION hours_diff_abs(timestamp1 TIMESTAMP,
timestamp2 TIMESTAMP) AS (
#
# Return the absolute value of the diff between the two timestamps in hours with microsecond precision
# If either parameter is null, return null
#
ABS(TIMESTAMP_DIFF(timestamp1,
    timestamp2,
    microsecond) / 3600000000.0) );

WITH

# Extracted VMS data with a time threshold of interest

VMSChileTmp AS (
SELECT seg_id,n_shipname,timestamp,source,nnet_score,
lat,lon,
EXTRACT(DATE from timestamp) as Date,
EXTRACT (Month from timestamp) as Month,
FROM `world-fishing-827.pipe_chile_production_v20200331.messages_scored_*`
WHERE (source = "chile_vms_industry"
OR source = "chile_vms_small_fisheries")
AND timestamp BETWEEN TIMESTAMP("2020-01-01")
AND TIMESTAMP("2020-08-01")
),

# Calculating hours for VMS data, based on previous timestamps sharing the same seg_id

pos AS (
SELECT *,
  LAG(timestamp, 1) OVER (PARTITION BY seg_id  ORDER BY timestamp) prev_timestamp,
FROM VMSChileTmp
),

VMSChileFinal AS (
SELECT *
EXCEPT (prev_timestamp),
floor(lat * 100) as lat_bin,
floor(lon * 100) as lon_bin,
IFNULL (hours_diff_abs (timestamp, prev_timestamp), 0) hours
FROM pos
),

# Summing fishing hours and total hours from the united data set, grouped
# by ship name, month, lat bin, and lon bin

FishEffortTmp2 AS (
SELECT n_shipname, Month,
lat_bin / 100 as lat_bin,
lon_bin / 100 as lon_bin,
SUM(IF(nnet_score > 0.5, hours, 0)) as fishing_hours,
SUM(hours) as total_hours
FROM VMSChileFinal
GROUP BY n_shipname, Month, lat_bin, lon_bin
),

#Transform hours/degrees to hours/km2

FishEffortFinal AS (
SELECT *, 
fishing_hours/(COS(udfs_v20200701.radians(lat_bin)) * (111/100)  * (111/100) ) AS fishing_hours_sq_km,
total_hours/(COS(udfs_v20200701.radians(lat_bin)) * (111/100)  * (111/100) ) AS total_hours_sq_km
FROM FishEffortTmp2
)

SELECT *
FROM FishEffortFinal
')
VMS_Hours_Final <- DBI::dbGetQuery(con, query_string)
write.csv(VMS_Hours_Final, file = "VMS_Hours_Final.csv")
```

Small query to extract unique VMS vessel names to then match and have
the result hour tables have more readable vessel names (including spaces
and characters)

``` r
query_string <- glue::glue('
SELECT
DISTINCT n_shipname,shipname,source
FROM `world-fishing-827.pipe_chile_production_v20200331.messages_scored_*`
')
UniqueVMSNamesChile <- DBI::dbGetQuery(con, query_string)
# write.csv(UniqueVMSNamesChile, file = "UniqueVMSNamesChile.csv")
```

**“VMS\_Hours\_Final.csv”** (generated above) represents all VMS data
filtered to only include observations between January 1st and July 31st,
2020, exclusively belonging to industrial and/or artisanal vessels
(excluding aquaculture and transport classified vessels). Consequently,
due to concerns that the GFW algorithm might be more likely to
incorrectly classify vessel activity as fishing within the first
nautical mile from shore, data within this first nautical mile was
removed from **“VMS\_Hours\_Final.csv”** using QGIS. QGIS was also used
to segment **“VMS\_Hours\_Final.csv”** into the 15 regions of interest.

The tables for the 15 regions are then imported below. These imported
tables are aggregated by vessel and month in order to have one table per
area of interest. The result are tables for each of the 15 regions,
containing total hours and fishing hours for each vessel for each month
(Jan - July, 2020). An example table for the first region is provided
below (ARICA\_Horas). Only the first ten lines are shown.

``` r
#Import lines of interest to be displayed on maps
#Lines of all the areas, north to south
AllPoly <- st_read("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Polygons/All_Poly_Merged_Lines.geojson")
#Fishing hours for first area = ARICA. Method for generating this table is provided below
ARICA_Horas <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/1_ARICA_Horas.csv", header = TRUE)
```

|  X | Embarcacion      | Mes   | Horas\_De\_Pesca\_Km2 | Horas\_Transito\_y\_Pesca\_Km2 |
| -: | :--------------- | :---- | --------------------: | -----------------------------: |
|  1 | ABEL (ART)       | enero |                     7 |                             22 |
|  2 | ABRAHAM (ART)    | enero |                     2 |                             11 |
|  3 | ALERCE (IND)     | enero |                     0 |                             10 |
|  4 | AMADEUS (ART)    | enero |                     7 |                             22 |
|  5 | AMADEUS II (ART) | enero |                     8 |                             21 |
|  6 | ANGAMOS 2 (IND)  | enero |                     0 |                              4 |
|  7 | ANGAMOS 4 (IND)  | enero |                     8 |                             55 |
|  8 | ANGAMOS 9 (IND)  | enero |                     7 |                             50 |
|  9 | ARKHOS I (ART)   | enero |                     6 |                             22 |
| 10 | ARKHOS II (ART)  | enero |                     8 |                             20 |

Methods for generating other tables (as the one shown above) with per
vessel per area per month fishing and total hours.

``` r
UniqueVMSNamesChile <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/UniqueVMSNamesChile.csv", header = TRUE)

# Aggregate by vessels and by month in order to extract tables with hours per vessel per month per area
# VMS ONLY
#1 ARICA
ARICA_Tmp <- st_read("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/1_ARICA_Tmp.geojson")
ARICA_Tmp2 <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname + Month, ARICA_Tmp, sum))
ARICA_Tmp3 <- data.frame(aggregate(total_hours_sq_km ~ n_shipname + Month, ARICA_Tmp, sum))
ARICA_F <- ARICA_Tmp2
ARICA_F$total_hours_sq_km <- ARICA_Tmp3$total_hours_sq_km
ARICA_F$n_shipname <- UniqueVMSNamesChile$shipname[match(ARICA_F$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Change column names and number months to name months
colnames(ARICA_F)[1] <- "Embarcacion"
colnames(ARICA_F)[2] <- "Mes"
colnames(ARICA_F)[3] <- "Horas_De_Pesca_Km2"
colnames(ARICA_F)[4] <- "Horas_Transito_y_Pesca_Km2"
ARICA_F$Mes[ARICA_F$Mes==1] <- "enero"
ARICA_F$Mes[ARICA_F$Mes==2] <- "febrero"
ARICA_F$Mes[ARICA_F$Mes==3] <- "marzo"
ARICA_F$Mes[ARICA_F$Mes==4] <- "abril"
ARICA_F$Mes[ARICA_F$Mes==5] <- "mayo"
ARICA_F$Mes[ARICA_F$Mes==6] <- "junio"
ARICA_F$Mes[ARICA_F$Mes==7] <- "julio"
#Rounding
ARICA_F$Horas_De_Pesca_Km2 <- round(ARICA_F$Horas_De_Pesca_Km2,0)
ARICA_F$Horas_Transito_y_Pesca_Km2 <- round(ARICA_F$Horas_Transito_y_Pesca_Km2,0)

write.csv(ARICA_F, file = "1_ARICA_Horas.csv")

#2 TARAPACA
TARAPACA_Tmp <- st_read("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/2_Tarapaca_Tmp.geojson")
TARAPACA_Tmp2 <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname + Month, TARAPACA_Tmp, sum))
TARAPACA_Tmp3 <- data.frame(aggregate(total_hours_sq_km ~ n_shipname + Month, TARAPACA_Tmp, sum))
TARAPACA_F <- TARAPACA_Tmp2
TARAPACA_F$total_hours_sq_km <- TARAPACA_Tmp3$total_hours_sq_km
TARAPACA_F$n_shipname <- UniqueVMSNamesChile$shipname[match(TARAPACA_F$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Change column names and number months to name months
colnames(TARAPACA_F)[1] <- "Embarcacion"
colnames(TARAPACA_F)[2] <- "Mes"
colnames(TARAPACA_F)[3] <- "Horas_De_Pesca_Km2"
colnames(TARAPACA_F)[4] <- "Horas_Transito_y_Pesca_Km2"
TARAPACA_F$Mes[TARAPACA_F$Mes==1] <- "enero"
TARAPACA_F$Mes[TARAPACA_F$Mes==2] <- "febrero"
TARAPACA_F$Mes[TARAPACA_F$Mes==3] <- "marzo"
TARAPACA_F$Mes[TARAPACA_F$Mes==4] <- "abril"
TARAPACA_F$Mes[TARAPACA_F$Mes==5] <- "mayo"
TARAPACA_F$Mes[TARAPACA_F$Mes==6] <- "junio"
TARAPACA_F$Mes[TARAPACA_F$Mes==7] <- "julio"
#Rounding
TARAPACA_F$Horas_De_Pesca_Km2 <- round(TARAPACA_F$Horas_De_Pesca_Km2,0)
TARAPACA_F$Horas_Transito_y_Pesca_Km2 <- round(TARAPACA_F$Horas_Transito_y_Pesca_Km2,0)

write.csv(TARAPACA_F, file = "2_Tarapaca_Horas.csv")

#3 ANTOFAGASTA
Antofagasta_Tmp <- st_read("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/3_Antofagasta_Tmp.geojson")
Antofagasta_Tmp2 <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname + Month, Antofagasta_Tmp, sum))
Antofagasta_Tmp3 <- data.frame(aggregate(total_hours_sq_km ~ n_shipname + Month, Antofagasta_Tmp, sum))
Antofagasta_F <- Antofagasta_Tmp2
Antofagasta_F$total_hours_sq_km <- Antofagasta_Tmp3$total_hours_sq_km
Antofagasta_F$n_shipname <- UniqueVMSNamesChile$shipname[match(Antofagasta_F$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Change column names and number months to name months
colnames(Antofagasta_F)[1] <- "Embarcacion"
colnames(Antofagasta_F)[2] <- "Mes"
colnames(Antofagasta_F)[3] <- "Horas_De_Pesca_Km2"
colnames(Antofagasta_F)[4] <- "Horas_Transito_y_Pesca_Km2"
Antofagasta_F$Mes[Antofagasta_F$Mes==1] <- "enero"
Antofagasta_F$Mes[Antofagasta_F$Mes==2] <- "febrero"
Antofagasta_F$Mes[Antofagasta_F$Mes==3] <- "marzo"
Antofagasta_F$Mes[Antofagasta_F$Mes==4] <- "abril"
Antofagasta_F$Mes[Antofagasta_F$Mes==5] <- "mayo"
Antofagasta_F$Mes[Antofagasta_F$Mes==6] <- "junio"
Antofagasta_F$Mes[Antofagasta_F$Mes==7] <- "julio"
#Rounding
Antofagasta_F$Horas_De_Pesca_Km2 <- round(Antofagasta_F$Horas_De_Pesca_Km2,0)
Antofagasta_F$Horas_Transito_y_Pesca_Km2 <- round(Antofagasta_F$Horas_Transito_y_Pesca_Km2,0)

write.csv(Antofagasta_F, file = "3_Antofagasta_Horas.csv")

#4 ATACAMA
Atacama_Tmp <- st_read("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/4_Atacama_Tmp.geojson")
Atacama_Tmp2 <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname + Month, Atacama_Tmp, sum))
Atacama_Tmp3 <- data.frame(aggregate(total_hours_sq_km ~ n_shipname + Month, Atacama_Tmp, sum))
Atacama_F <- Atacama_Tmp2
Atacama_F$total_hours_sq_km <- Atacama_Tmp3$total_hours_sq_km
Atacama_F$n_shipname <- UniqueVMSNamesChile$shipname[match(Atacama_F$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Change column names and number months to name months
colnames(Atacama_F)[1] <- "Embarcacion"
colnames(Atacama_F)[2] <- "Mes"
colnames(Atacama_F)[3] <- "Horas_De_Pesca_Km2"
colnames(Atacama_F)[4] <- "Horas_Transito_y_Pesca_Km2"
Atacama_F$Mes[Atacama_F$Mes==1] <- "enero"
Atacama_F$Mes[Atacama_F$Mes==2] <- "febrero"
Atacama_F$Mes[Atacama_F$Mes==3] <- "marzo"
Atacama_F$Mes[Atacama_F$Mes==4] <- "abril"
Atacama_F$Mes[Atacama_F$Mes==5] <- "mayo"
Atacama_F$Mes[Atacama_F$Mes==6] <- "junio"
Atacama_F$Mes[Atacama_F$Mes==7] <- "julio"
#Rounding
Atacama_F$Horas_De_Pesca_Km2 <- round(Atacama_F$Horas_De_Pesca_Km2,0)
Atacama_F$Horas_Transito_y_Pesca_Km2 <- round(Atacama_F$Horas_Transito_y_Pesca_Km2,0)

write.csv(Atacama_F, file = "4_Atacama_Horas.csv")

#5 COQUIMBO
Coquimbo_Tmp <- st_read("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/5_Coquimbo_Tmp.geojson")
Coquimbo_Tmp2 <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname + Month, Coquimbo_Tmp, sum))
Coquimbo_Tmp3 <- data.frame(aggregate(total_hours_sq_km ~ n_shipname + Month, Coquimbo_Tmp, sum))
Coquimbo_F <- Coquimbo_Tmp2
Coquimbo_F$total_hours_sq_km <- Coquimbo_Tmp3$total_hours_sq_km
Coquimbo_F$n_shipname <- UniqueVMSNamesChile$shipname[match(Coquimbo_F$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Change column names and number months to name months
colnames(Coquimbo_F)[1] <- "Embarcacion"
colnames(Coquimbo_F)[2] <- "Mes"
colnames(Coquimbo_F)[3] <- "Horas_De_Pesca_Km2"
colnames(Coquimbo_F)[4] <- "Horas_Transito_y_Pesca_Km2"
Coquimbo_F$Mes[Coquimbo_F$Mes==1] <- "enero"
Coquimbo_F$Mes[Coquimbo_F$Mes==2] <- "febrero"
Coquimbo_F$Mes[Coquimbo_F$Mes==3] <- "marzo"
Coquimbo_F$Mes[Coquimbo_F$Mes==4] <- "abril"
Coquimbo_F$Mes[Coquimbo_F$Mes==5] <- "mayo"
Coquimbo_F$Mes[Coquimbo_F$Mes==6] <- "junio"
Coquimbo_F$Mes[Coquimbo_F$Mes==7] <- "julio"
#Rounding
Coquimbo_F$Horas_De_Pesca_Km2 <- round(Coquimbo_F$Horas_De_Pesca_Km2,0)
Coquimbo_F$Horas_Transito_y_Pesca_Km2 <- round(Coquimbo_F$Horas_Transito_y_Pesca_Km2,0)

write.csv(Coquimbo_F, file = "5_Coquimbo_Horas.csv")

#6 VALPARAISO
Valparaiso_Tmp <- st_read("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/6_Valparaiso_Tmp.geojson")
Valparaiso_Tmp2 <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname + Month, Valparaiso_Tmp, sum))
Valparaiso_Tmp3 <- data.frame(aggregate(total_hours_sq_km ~ n_shipname + Month, Valparaiso_Tmp, sum))
Valparaiso_F <- Valparaiso_Tmp2
Valparaiso_F$total_hours_sq_km <- Valparaiso_Tmp3$total_hours_sq_km
Valparaiso_F$n_shipname <- UniqueVMSNamesChile$shipname[match(Valparaiso_F$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Change column names and number months to name months
colnames(Valparaiso_F)[1] <- "Embarcacion"
colnames(Valparaiso_F)[2] <- "Mes"
colnames(Valparaiso_F)[3] <- "Horas_De_Pesca_Km2"
colnames(Valparaiso_F)[4] <- "Horas_Transito_y_Pesca_Km2"
Valparaiso_F$Mes[Valparaiso_F$Mes==1] <- "enero"
Valparaiso_F$Mes[Valparaiso_F$Mes==2] <- "febrero"
Valparaiso_F$Mes[Valparaiso_F$Mes==3] <- "marzo"
Valparaiso_F$Mes[Valparaiso_F$Mes==4] <- "abril"
Valparaiso_F$Mes[Valparaiso_F$Mes==5] <- "mayo"
Valparaiso_F$Mes[Valparaiso_F$Mes==6] <- "junio"
Valparaiso_F$Mes[Valparaiso_F$Mes==7] <- "julio"
#Rounding
Valparaiso_F$Horas_De_Pesca_Km2 <- round(Valparaiso_F$Horas_De_Pesca_Km2,0)
Valparaiso_F$Horas_Transito_y_Pesca_Km2 <- round(Valparaiso_F$Horas_Transito_y_Pesca_Km2,0)

write.csv(Valparaiso_F, file = "6_Valparaiso_Horas.csv")

#7 OHIGGINS
Ohiggins_Tmp <- st_read("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/7_Ohiggins_Tmp.geojson")
Ohiggins_Tmp2 <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname + Month, Ohiggins_Tmp, sum))
Ohiggins_Tmp3 <- data.frame(aggregate(total_hours_sq_km ~ n_shipname + Month, Ohiggins_Tmp, sum))
Ohiggins_F <- Ohiggins_Tmp2
Ohiggins_F$total_hours_sq_km <- Ohiggins_Tmp3$total_hours_sq_km
Ohiggins_F$n_shipname <- UniqueVMSNamesChile$shipname[match(Ohiggins_F$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Change column names and number months to name months
colnames(Ohiggins_F)[1] <- "Embarcacion"
colnames(Ohiggins_F)[2] <- "Mes"
colnames(Ohiggins_F)[3] <- "Horas_De_Pesca_Km2"
colnames(Ohiggins_F)[4] <- "Horas_Transito_y_Pesca_Km2"
Ohiggins_F$Mes[Ohiggins_F$Mes==1] <- "enero"
Ohiggins_F$Mes[Ohiggins_F$Mes==2] <- "febrero"
Ohiggins_F$Mes[Ohiggins_F$Mes==3] <- "marzo"
Ohiggins_F$Mes[Ohiggins_F$Mes==4] <- "abril"
Ohiggins_F$Mes[Ohiggins_F$Mes==5] <- "mayo"
Ohiggins_F$Mes[Ohiggins_F$Mes==6] <- "junio"
Ohiggins_F$Mes[Ohiggins_F$Mes==7] <- "julio"
#Rounding
Ohiggins_F$Horas_De_Pesca_Km2 <- round(Ohiggins_F$Horas_De_Pesca_Km2,0)
Ohiggins_F$Horas_Transito_y_Pesca_Km2 <- round(Ohiggins_F$Horas_Transito_y_Pesca_Km2,0)

write.csv(Ohiggins_F, file = "7_Ohiggins_Horas.csv")

#8 MAULE
Maule_Tmp <- st_read("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/8_Maule_Tmp.geojson")
Maule_Tmp2 <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname + Month, Maule_Tmp, sum))
Maule_Tmp3 <- data.frame(aggregate(total_hours_sq_km ~ n_shipname + Month, Maule_Tmp, sum))
Maule_F <- Maule_Tmp2
Maule_F$total_hours_sq_km <- Maule_Tmp3$total_hours_sq_km
Maule_F$n_shipname <- UniqueVMSNamesChile$shipname[match(Maule_F$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Change column names and number months to name months
colnames(Maule_F)[1] <- "Embarcacion"
colnames(Maule_F)[2] <- "Mes"
colnames(Maule_F)[3] <- "Horas_De_Pesca_Km2"
colnames(Maule_F)[4] <- "Horas_Transito_y_Pesca_Km2"
Maule_F$Mes[Maule_F$Mes==1] <- "enero"
Maule_F$Mes[Maule_F$Mes==2] <- "febrero"
Maule_F$Mes[Maule_F$Mes==3] <- "marzo"
Maule_F$Mes[Maule_F$Mes==4] <- "abril"
Maule_F$Mes[Maule_F$Mes==5] <- "mayo"
Maule_F$Mes[Maule_F$Mes==6] <- "junio"
Maule_F$Mes[Maule_F$Mes==7] <- "julio"
#Rounding
Maule_F$Horas_De_Pesca_Km2 <- round(Maule_F$Horas_De_Pesca_Km2,0)
Maule_F$Horas_Transito_y_Pesca_Km2 <- round(Maule_F$Horas_Transito_y_Pesca_Km2,0)

write.csv(Maule_F, file = "8_Maule_Horas.csv")

#9 NUBLE
Nuble_Tmp <- st_read("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/9_Nuble_Tmp.geojson")
Nuble_Tmp2 <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname + Month, Nuble_Tmp, sum))
Nuble_Tmp3 <- data.frame(aggregate(total_hours_sq_km ~ n_shipname + Month, Nuble_Tmp, sum))
Nuble_F <- Nuble_Tmp2
Nuble_F$total_hours_sq_km <- Nuble_Tmp3$total_hours_sq_km
Nuble_F$n_shipname <- UniqueVMSNamesChile$shipname[match(Nuble_F$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Change column names and number months to name months
colnames(Nuble_F)[1] <- "Embarcacion"
colnames(Nuble_F)[2] <- "Mes"
colnames(Nuble_F)[3] <- "Horas_De_Pesca_Km2"
colnames(Nuble_F)[4] <- "Horas_Transito_y_Pesca_Km2"
Nuble_F$Mes[Nuble_F$Mes==1] <- "enero"
Nuble_F$Mes[Nuble_F$Mes==2] <- "febrero"
Nuble_F$Mes[Nuble_F$Mes==3] <- "marzo"
Nuble_F$Mes[Nuble_F$Mes==4] <- "abril"
Nuble_F$Mes[Nuble_F$Mes==5] <- "mayo"
Nuble_F$Mes[Nuble_F$Mes==6] <- "junio"
Nuble_F$Mes[Nuble_F$Mes==7] <- "julio"
#Rounding
Nuble_F$Horas_De_Pesca_Km2 <- round(Nuble_F$Horas_De_Pesca_Km2,0)
Nuble_F$Horas_Transito_y_Pesca_Km2 <- round(Nuble_F$Horas_Transito_y_Pesca_Km2,0)

write.csv(Nuble_F, file = "9_Nuble_Horas.csv")

#10 NUBLE
Biobio_Tmp <- st_read("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/10_Biobio_Tmp.geojson")
Biobio_Tmp2 <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname + Month, Biobio_Tmp, sum))
Biobio_Tmp3 <- data.frame(aggregate(total_hours_sq_km ~ n_shipname + Month, Biobio_Tmp, sum))
Biobio_F <- Biobio_Tmp2
Biobio_F$total_hours_sq_km <- Biobio_Tmp3$total_hours_sq_km
Biobio_F$n_shipname <- UniqueVMSNamesChile$shipname[match(Biobio_F$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Change column names and number months to name months
colnames(Biobio_F)[1] <- "Embarcacion"
colnames(Biobio_F)[2] <- "Mes"
colnames(Biobio_F)[3] <- "Horas_De_Pesca_Km2"
colnames(Biobio_F)[4] <- "Horas_Transito_y_Pesca_Km2"
Biobio_F$Mes[Biobio_F$Mes==1] <- "enero"
Biobio_F$Mes[Biobio_F$Mes==2] <- "febrero"
Biobio_F$Mes[Biobio_F$Mes==3] <- "marzo"
Biobio_F$Mes[Biobio_F$Mes==4] <- "abril"
Biobio_F$Mes[Biobio_F$Mes==5] <- "mayo"
Biobio_F$Mes[Biobio_F$Mes==6] <- "junio"
Biobio_F$Mes[Biobio_F$Mes==7] <- "julio"
#Rounding
Biobio_F$Horas_De_Pesca_Km2 <- round(Biobio_F$Horas_De_Pesca_Km2,0)
Biobio_F$Horas_Transito_y_Pesca_Km2 <- round(Biobio_F$Horas_Transito_y_Pesca_Km2,0)

write.csv(Biobio_F, file = "10_Biobio_Horas.csv")

#11 ARAUCANIA
Araucania_Tmp <- st_read("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/11_Araucania_Tmp.geojson")
Araucania_Tmp2 <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname + Month, Araucania_Tmp, sum))
Araucania_Tmp3 <- data.frame(aggregate(total_hours_sq_km ~ n_shipname + Month, Araucania_Tmp, sum))
Araucania_F <- Araucania_Tmp2
Araucania_F$total_hours_sq_km <- Araucania_Tmp3$total_hours_sq_km
Araucania_F$n_shipname <- UniqueVMSNamesChile$shipname[match(Araucania_F$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Change column names and number months to name months
colnames(Araucania_F)[1] <- "Embarcacion"
colnames(Araucania_F)[2] <- "Mes"
colnames(Araucania_F)[3] <- "Horas_De_Pesca_Km2"
colnames(Araucania_F)[4] <- "Horas_Transito_y_Pesca_Km2"
Araucania_F$Mes[Araucania_F$Mes==1] <- "enero"
Araucania_F$Mes[Araucania_F$Mes==2] <- "febrero"
Araucania_F$Mes[Araucania_F$Mes==3] <- "marzo"
Araucania_F$Mes[Araucania_F$Mes==4] <- "abril"
Araucania_F$Mes[Araucania_F$Mes==5] <- "mayo"
Araucania_F$Mes[Araucania_F$Mes==6] <- "junio"
Araucania_F$Mes[Araucania_F$Mes==7] <- "julio"
#Rounding
Araucania_F$Horas_De_Pesca_Km2 <- round(Araucania_F$Horas_De_Pesca_Km2,0)
Araucania_F$Horas_Transito_y_Pesca_Km2 <- round(Araucania_F$Horas_Transito_y_Pesca_Km2,0)

write.csv(Araucania_F, file = "11_Araucania_Horas.csv")

#12 LOS RIOS
Los_Rios_Tmp <- st_read("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/12_Los_Rios_Tmp.geojson")
Los_Rios_Tmp2 <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname + Month, Los_Rios_Tmp, sum))
Los_Rios_Tmp3 <- data.frame(aggregate(total_hours_sq_km ~ n_shipname + Month, Los_Rios_Tmp, sum))
Los_Rios_F <- Los_Rios_Tmp2
Los_Rios_F$total_hours_sq_km <- Los_Rios_Tmp3$total_hours_sq_km
Los_Rios_F$n_shipname <- UniqueVMSNamesChile$shipname[match(Los_Rios_F$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Change column names and number months to name months
colnames(Los_Rios_F)[1] <- "Embarcacion"
colnames(Los_Rios_F)[2] <- "Mes"
colnames(Los_Rios_F)[3] <- "Horas_De_Pesca_Km2"
colnames(Los_Rios_F)[4] <- "Horas_Transito_y_Pesca_Km2"
Los_Rios_F$Mes[Los_Rios_F$Mes==1] <- "enero"
Los_Rios_F$Mes[Los_Rios_F$Mes==2] <- "febrero"
Los_Rios_F$Mes[Los_Rios_F$Mes==3] <- "marzo"
Los_Rios_F$Mes[Los_Rios_F$Mes==4] <- "abril"
Los_Rios_F$Mes[Los_Rios_F$Mes==5] <- "mayo"
Los_Rios_F$Mes[Los_Rios_F$Mes==6] <- "junio"
Los_Rios_F$Mes[Los_Rios_F$Mes==7] <- "julio"
#Rounding
Los_Rios_F$Horas_De_Pesca_Km2 <- round(Los_Rios_F$Horas_De_Pesca_Km2,0)
Los_Rios_F$Horas_Transito_y_Pesca_Km2 <- round(Los_Rios_F$Horas_Transito_y_Pesca_Km2,0)

write.csv(Los_Rios_F, file = "12_Los_Rios_Horas.csv")

#13 LOS LAGOS
Los_Lagos_Tmp <- st_read("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/13_Los_Lagos_Tmp.geojson")
Los_Lagos_Tmp2 <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname + Month, Los_Lagos_Tmp, sum))
Los_Lagos_Tmp3 <- data.frame(aggregate(total_hours_sq_km ~ n_shipname + Month, Los_Lagos_Tmp, sum))
Los_Lagos_F <- Los_Lagos_Tmp2
Los_Lagos_F$total_hours_sq_km <- Los_Lagos_Tmp3$total_hours_sq_km
Los_Lagos_F$n_shipname <- UniqueVMSNamesChile$shipname[match(Los_Lagos_F$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Change column names and number months to name months
colnames(Los_Lagos_F)[1] <- "Embarcacion"
colnames(Los_Lagos_F)[2] <- "Mes"
colnames(Los_Lagos_F)[3] <- "Horas_De_Pesca_Km2"
colnames(Los_Lagos_F)[4] <- "Horas_Transito_y_Pesca_Km2"
Los_Lagos_F$Mes[Los_Lagos_F$Mes==1] <- "enero"
Los_Lagos_F$Mes[Los_Lagos_F$Mes==2] <- "febrero"
Los_Lagos_F$Mes[Los_Lagos_F$Mes==3] <- "marzo"
Los_Lagos_F$Mes[Los_Lagos_F$Mes==4] <- "abril"
Los_Lagos_F$Mes[Los_Lagos_F$Mes==5] <- "mayo"
Los_Lagos_F$Mes[Los_Lagos_F$Mes==6] <- "junio"
Los_Lagos_F$Mes[Los_Lagos_F$Mes==7] <- "julio"
#Rounding
Los_Lagos_F$Horas_De_Pesca_Km2 <- round(Los_Lagos_F$Horas_De_Pesca_Km2,0)
Los_Lagos_F$Horas_Transito_y_Pesca_Km2 <- round(Los_Lagos_F$Horas_Transito_y_Pesca_Km2,0)

write.csv(Los_Lagos_F, file = "13_Los_Lagos_Horas.csv")

#14 AYSEN
Aysen_Tmp <- st_read("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/14_Aysen_Tmp.geojson")
Aysen_Tmp2 <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname + Month, Aysen_Tmp, sum))
Aysen_Tmp3 <- data.frame(aggregate(total_hours_sq_km ~ n_shipname + Month, Aysen_Tmp, sum))
Aysen_F <- Aysen_Tmp2
Aysen_F$total_hours_sq_km <- Aysen_Tmp3$total_hours_sq_km
Aysen_F$n_shipname <- UniqueVMSNamesChile$shipname[match(Aysen_F$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Change column names and number months to name months
colnames(Aysen_F)[1] <- "Embarcacion"
colnames(Aysen_F)[2] <- "Mes"
colnames(Aysen_F)[3] <- "Horas_De_Pesca_Km2"
colnames(Aysen_F)[4] <- "Horas_Transito_y_Pesca_Km2"
Aysen_F$Mes[Aysen_F$Mes==1] <- "enero"
Aysen_F$Mes[Aysen_F$Mes==2] <- "febrero"
Aysen_F$Mes[Aysen_F$Mes==3] <- "marzo"
Aysen_F$Mes[Aysen_F$Mes==4] <- "abril"
Aysen_F$Mes[Aysen_F$Mes==5] <- "mayo"
Aysen_F$Mes[Aysen_F$Mes==6] <- "junio"
Aysen_F$Mes[Aysen_F$Mes==7] <- "julio"
#Rounding
Aysen_F$Horas_De_Pesca_Km2 <- round(Aysen_F$Horas_De_Pesca_Km2,0)
Aysen_F$Horas_Transito_y_Pesca_Km2 <- round(Aysen_F$Horas_Transito_y_Pesca_Km2,0)

write.csv(Aysen_F, file = "14_Aysen_Horas.csv")

#15 MAGALLANES
Magallanes_Tmp <- st_read("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/15_Magallanes_Tmp.geojson")
Magallanes_Tmp2 <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname + Month, Magallanes_Tmp, sum))
Magallanes_Tmp3 <- data.frame(aggregate(total_hours_sq_km ~ n_shipname + Month, Magallanes_Tmp, sum))
Magallanes_F <- Magallanes_Tmp2
Magallanes_F$total_hours_sq_km <- Magallanes_Tmp3$total_hours_sq_km
Magallanes_F$n_shipname <- UniqueVMSNamesChile$shipname[match(Magallanes_F$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Change column names and number months to name months
colnames(Magallanes_F)[1] <- "Embarcacion"
colnames(Magallanes_F)[2] <- "Mes"
colnames(Magallanes_F)[3] <- "Horas_De_Pesca_Km2"
colnames(Magallanes_F)[4] <- "Horas_Transito_y_Pesca_Km2"
Magallanes_F$Mes[Magallanes_F$Mes==1] <- "enero"
Magallanes_F$Mes[Magallanes_F$Mes==2] <- "febrero"
Magallanes_F$Mes[Magallanes_F$Mes==3] <- "marzo"
Magallanes_F$Mes[Magallanes_F$Mes==4] <- "abril"
Magallanes_F$Mes[Magallanes_F$Mes==5] <- "mayo"
Magallanes_F$Mes[Magallanes_F$Mes==6] <- "junio"
Magallanes_F$Mes[Magallanes_F$Mes==7] <- "julio"
#Rounding
Magallanes_F$Horas_De_Pesca_Km2 <- round(Magallanes_F$Horas_De_Pesca_Km2,0)
Magallanes_F$Horas_Transito_y_Pesca_Km2 <- round(Magallanes_F$Horas_Transito_y_Pesca_Km2,0)

write.csv(Magallanes_F, file = "15_Magallanes_Horas.csv")

#16 TODOS
Todos_Tmp <- read.csv("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Hours/VMS_Hours_Final.csv", header = TRUE)
Todos_Tmp2 <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname + Month, Todos_Tmp, sum))
Todos_Tmp3 <- data.frame(aggregate(total_hours_sq_km ~ n_shipname + Month, Todos_Tmp, sum))
Todos_F <- Todos_Tmp2
Todos_F$total_hours_sq_km <- Todos_Tmp3$total_hours_sq_km
Todos_F$n_shipname <- UniqueVMSNamesChile$shipname[match(Todos_F$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Change column names and number months to name months
colnames(Todos_F)[1] <- "Embarcacion"
colnames(Todos_F)[2] <- "Mes"
colnames(Todos_F)[3] <- "Horas_De_Pesca_Km2"
colnames(Todos_F)[4] <- "Horas_Transito_y_Pesca_Km2"
Todos_F$Mes[Todos_F$Mes==1] <- "enero"
Todos_F$Mes[Todos_F$Mes==2] <- "febrero"
Todos_F$Mes[Todos_F$Mes==3] <- "marzo"
Todos_F$Mes[Todos_F$Mes==4] <- "abril"
Todos_F$Mes[Todos_F$Mes==5] <- "mayo"
Todos_F$Mes[Todos_F$Mes==6] <- "junio"
Todos_F$Mes[Todos_F$Mes==7] <- "julio"
#Rounding
Todos_F$Horas_De_Pesca_Km2 <- round(Todos_F$Horas_De_Pesca_Km2,0)
Todos_F$Horas_Transito_y_Pesca_Km2 <- round(Todos_F$Horas_Transito_y_Pesca_Km2,0)

write.csv(Todos_F, file = "16_Todos_Horas.csv")
```

Importing tables generated above with per vessel per area per month
fishing and total hours in order to combine them and graph total hours
per area and for all the areas combined

``` r
#Importing tables generated above with per vessel per area per month fishing and total hours
Arica1 <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/1_ARICA_Horas.csv", header = TRUE)
Tarapaca2 <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/2_Tarapaca_Horas.csv", header = TRUE)
Antofagasta3 <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/3_Antofagasta_Horas.csv", header = TRUE)
Atacama4 <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/4_Atacama_Horas.csv", header = TRUE)
Coquimbo5 <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/5_Coquimbo_Horas.csv", header = TRUE)
Valparaiso6 <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/6_Valparaiso_Horas.csv", header = TRUE)
Ohiggins7 <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/7_Ohiggins_Horas.csv", header = TRUE)
Maule8 <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/8_Maule_Horas.csv", header = TRUE)
Nuble9 <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/9_Nuble_Horas.csv", header = TRUE)
Biobio10 <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/10_Biobio_Horas.csv", header = TRUE)
Araucania11 <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/11_Araucania_Horas.csv", header = TRUE)
Los_Rios12 <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/12_Los_Rios_Horas.csv", header = TRUE)
Los_Lagos13 <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/13_Los_Lagos_Horas.csv", header = TRUE)
Aysen14 <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/14_Aysen_Horas.csv", header = TRUE)
Magallanes15 <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/15_Magallanes_Horas.csv", header = TRUE)
Todos16 <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Final_Hrs_x_Area/16_Todos_Horas.csv", header = TRUE)
```

Generating Table with Total Hour Results per Areas

``` r
#Adding area identifying component to each DB
Arica1$Area <- "1_Arica"
Tarapaca2$Area <- "2_Tarapaca"
Antofagasta3$Area <- "3_Antofagasta"
Atacama4$Area <- "4_Atacama"
Coquimbo5$Area <- "5_Coquimbo"
Valparaiso6$Area <- "6_Valparaiso"
Ohiggins7$Area <- "7_Ohiggins"
Maule8$Area <- "8_Maule"
Nuble9$Area <- "9_Nuble"
Biobio10$Area <- "10_Biobio"
Araucania11$Area <- "11_Araucania"
Los_Rios12$Area <- "12_Los_Rios"
Los_Lagos13$Area <- "13_Los_Lagos"
Aysen14$Area <- "14_Aysen"
Magallanes15$Area <- "15_Magallanes"
Todos16$Area <- "Todas"

Arica1$Area_Numero <- 1
Tarapaca2$Area_Numero <- 2
Antofagasta3$Area_Numero <- 3
Atacama4$Area_Numero <- 4
Coquimbo5$Area_Numero <- 5
Valparaiso6$Area_Numero <- 6
Ohiggins7$Area_Numero <- 7
Maule8$Area_Numero <- 8
Nuble9$Area_Numero <- 9
Biobio10$Area_Numero <- 10
Araucania11$Area_Numero <- 11
Los_Rios12$Area_Numero <- 12
Los_Lagos13$Area_Numero <- 13
Aysen14$Area_Numero <- 14
Magallanes15$Area_Numero <- 15
Todos16$Area_Numero <- 16

#Row joining all tables
AllAreas <- rbind(Arica1,Tarapaca2,Antofagasta3,Atacama4,Coquimbo5,Valparaiso6,Ohiggins7,Maule8,Nuble9,Biobio10,Araucania11,Los_Rios12,Los_Lagos13,Aysen14,Magallanes15,Todos16)
#Changing back from name months to number months
AllAreas$Mes[AllAreas$Mes=="enero"] <- 1
AllAreas$Mes[AllAreas$Mes=="febrero"] <- 2
AllAreas$Mes[AllAreas$Mes=="marzo"] <- 3
AllAreas$Mes[AllAreas$Mes=="abril"] <- 4
AllAreas$Mes[AllAreas$Mes=="mayo"] <- 5
AllAreas$Mes[AllAreas$Mes=="junio"] <- 6
AllAreas$Mes[AllAreas$Mes=="julio"] <- 7
#Aggregating by month and area, summing fishing and total hours
AllAreas_Tmp2 <- data.frame(aggregate(Horas_De_Pesca_Km2 ~ Mes + Area_Numero + Area, AllAreas, sum))
AllAreas_Tmp3 <- data.frame(aggregate(Horas_Transito_y_Pesca_Km2 ~ Mes + Area_Numero + Area, AllAreas, sum))
AllAreas_F <- AllAreas_Tmp2
AllAreas_F$Horas_Transito_y_Pesca_Km2 <- AllAreas_Tmp3$Horas_Transito_y_Pesca_Km2
# write.csv(AllAreas_F, file = "Horas_x_Areax_Mes.csv")

Sin_Total <- AllAreas_F[which(AllAreas_F$Area!="Todas"), ]
Total <- AllAreas_F[which(AllAreas_F$Area=="Todas"), ]
```

Graphing Total Fishing Hours per Areas

``` r
Fishing_Total_Hours <- ggplot(Sin_Total, aes(x=Mes, y=Horas_De_Pesca_Km2, group=Area))+
  geom_line(aes(color=Area))+
  geom_point(aes(color=Area))+
  scale_color_discrete(name="Región",
                       breaks=c("1_Arica","2_Tarapaca","3_Antofagasta","4_Atacama","5_Coquimbo","6_Valparaiso",
                            "7_Ohiggins","8_Maule","9_Nuble","10_Biobio","11_Araucania","12_Los_Rios",
                            "13_Los_Lagos","14_Aysen","15_Magallanes"))+
  ggtitle("Horas de Pesca por Región de Acuerdo al Mes, 2020")+
  ylab("Horas de Pesca por Km2")+
  scale_x_discrete(breaks=c("1","2","3","4","5","6","7"),
        labels=c("enero", "febrero", "marzo", "abril", "mayo", "junio", "julio"))+
  scale_y_continuous(label=comma)

Fishing_Total_Hours
```

![](EP_Norte_Sur_files/figure-gfm/unnamed-chunk-10-1.png)<!-- -->

``` r
# ggsave("Horas_Pesca_x_Región_x_Mes.png", dpi=300)
```

Graphing Total transit and Fishing Hours per Areas

``` r
Total_Hours <- ggplot(Sin_Total, aes(x=Mes, y=Horas_Transito_y_Pesca_Km2, group=Area))+
  geom_line(aes(color=Area))+
  geom_point(aes(color=Area))+
  scale_color_discrete(name="Región",
                       breaks=c("1_Arica","2_Tarapaca","3_Antofagasta","4_Atacama","5_Coquimbo","6_Valparaiso",
                            "7_Ohiggins","8_Maule","9_Nuble","10_Biobio","11_Araucania","12_Los_Rios",
                            "13_Los_Lagos","14_Aysen","15_Magallanes"))+
  ggtitle("Horas de Tránsito y Pesca por Región de Acuerdo al Mes, 2020")+
  ylab("Horas de Tránsito y Pesca por Km2")+
  scale_x_discrete(breaks=c("1","2","3","4","5","6","7"),
        labels=c("enero", "febrero", "marzo", "abril", "mayo", "junio", "julio"))+
  scale_y_continuous(label=comma)

Total_Hours
```

![](EP_Norte_Sur_files/figure-gfm/unnamed-chunk-11-1.png)<!-- -->

``` r
# ggsave("Horas_Totales_x_Región_x_Mes.png", dpi=300)
```

Graphing Total Fishing Hours of all Areas Combined

``` r
Fishing_Total_Hours_1 <- ggplot(Total, aes(x=Mes, y=Horas_De_Pesca_Km2, group=Area))+
  geom_bar(stat="identity", fill="steelblue")+
  theme(legend.position="none")+
  ggtitle("Horas de Pesca Total en Chile de Acuerdo al Mes, 2020")+
  ylab("Horas de Pesca por Km2")+
  scale_x_discrete(breaks=c("1","2","3","4","5","6","7"),
        labels=c("enero", "febrero", "marzo", "abril", "mayo", "junio", "julio"))+
  scale_y_continuous(label=comma)

Fishing_Total_Hours_1
```

![](EP_Norte_Sur_files/figure-gfm/unnamed-chunk-12-1.png)<!-- -->

``` r
# ggsave("Horas_Pesca_Total_x_Mes.png", dpi=300)
```

Graphing Total Transit and Fishing Hours of all Areas Combined

``` r
Total_Hours_1 <- ggplot(Total, aes(x=Mes, y=Horas_Transito_y_Pesca_Km2, group=Area))+
  geom_bar(stat="identity", fill="steelblue")+
  theme(legend.position="none")+
  ggtitle("Horas de Tránsito y Pesca Total en Chile de Acuerdo al Mes, 2020")+
  ylab("Horas de Tránsito y Pesca por Km2")+
  scale_x_discrete(breaks=c("1","2","3","4","5","6","7"),
        labels=c("enero", "febrero", "marzo", "abril", "mayo", "junio", "julio"))+
  scale_y_continuous(label=comma)

Total_Hours_1
```

![](EP_Norte_Sur_files/figure-gfm/unnamed-chunk-13-1.png)<!-- -->

``` r
# ggsave("Horas_Pesca_Transito_Total_x_Mes.png", dpi=300)
```

Aggregate by month, LatBin, and LonBin in order to below extract
individual tables for each month and be able to graph with a raster
fishing effort per month (Jan - July, 2020).

``` r
VMS_Hours_Final <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Hours/VMS_Hours_Final.csv", header = TRUE)

# Aggregate by month and lat lon in order to graph with total fishing hours by lat lon bin per month. One map per month (jan - july)
# VMS ONLY
VMS_FHHoursByMonth <- data.frame(aggregate(fishing_hours_sq_km ~ Month + lat_bin + lon_bin, VMS_Hours_Final, sum))
VMS_FHHoursByMonth$Log_fishing_hours_sq_km <- log10(VMS_FHHoursByMonth$fishing_hours_sq_km)
#For each month
VMS_JanFH <- VMS_FHHoursByMonth[VMS_FHHoursByMonth$Month == 1,]
VMS_FebFH <- VMS_FHHoursByMonth[VMS_FHHoursByMonth$Month == 2,]
VMS_MarFH <- VMS_FHHoursByMonth[VMS_FHHoursByMonth$Month == 3,]
VMS_AprFH <- VMS_FHHoursByMonth[VMS_FHHoursByMonth$Month == 4,]
VMS_MayFH <- VMS_FHHoursByMonth[VMS_FHHoursByMonth$Month == 5,]
VMS_JunFH <- VMS_FHHoursByMonth[VMS_FHHoursByMonth$Month == 6,]
VMS_JulFH <- VMS_FHHoursByMonth[VMS_FHHoursByMonth$Month == 7,]
```

Determining total number of industrial and artisanal vessels in the data
cropped out for this analysis: within the Chilean EEZ and for the months
of January through July of 2020 vs. the total number of industrial and
artisanal vessels in the entire Chile VMS data set (Feb 2019 - Aug
2020). Further categorized vessels according to whether they’re
industrial or artisanal.

**Total Number of Vessels of this analysis (industrial and artisanal):
Jan - July 2020** The count of vessels according to their industrial or
artisanal category is provided below as
**Resumen\_Embarcaciones\_Del\_Estudio.csv**. However, the full list of
vessels is also available as **Lista\_Embarcaciones\_Del\_Estudio.csv**

``` r
#Entire EZZ data from Jan-July 2020
#Unique Vessel Names in order to match and change vessel names in final tables latter on
UniqueVMSNamesChile <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/UniqueVMSNamesChile.csv", header = TRUE)

#Vessels from Jan-July 2020 (this study)
TotalVesselsJanAug <- data.frame(aggregate(fishing_hours_sq_km ~ n_shipname, VMS_Hours_Final, sum))
#Vessel Name
TotalVesselsJanAug$Embarcacion <- UniqueVMSNamesChile$shipname[match(TotalVesselsJanAug$n_shipname, UniqueVMSNamesChile$n_shipname)]
#Vessel Source
UniqueVMSNamesChile <- subset(UniqueVMSNamesChile, source == "chile_vms_industry" | source == "chile_vms_small_fisheries")
TotalVesselsJanAug$Categoria <- UniqueVMSNamesChile$source[match(TotalVesselsJanAug$n_shipname, UniqueVMSNamesChile$n_shipname)]
TotalVesselsJanAug <- TotalVesselsJanAug[-c(1:2)]
#Generate Table with count of industrial and artisanal vessels
CountTotalVesselsJanAug <- count(TotalVesselsJanAug, "Categoria")
#754 total vessels - 115 industrial and 639 artisanal

# write.csv(TotalVesselsJanAug, file="Lista_Embarcaciones_Del_Estudio.csv")
# write.csv(CountTotalVesselsJanAug, file="Resumen_Embarcaciones_Del_Estudio.csv")
```

| Categoria                    | freq |
| :--------------------------- | ---: |
| chile\_vms\_industry         |  115 |
| chile\_vms\_small\_fisheries |  639 |

**Total Number of Vessels of the entire VMS Chile data set (industrial
and artisanal): Feb 2019 - Aug 2020** Query to extract unique vessel
names and their industrial and/or artisanal categories

``` r
query_string <- glue::glue('
WITH

VMSChileTmp AS (
SELECT shipname,source
FROM `world-fishing-827.pipe_chile_production_v20200331.messages_scored_*`
WHERE (source = "chile_vms_industry"
OR source = "chile_vms_small_fisheries")
),

TotalVesselsVMS AS (
SELECT
DISTINCT shipname,
source
FROM VMSChileTmp 
)

SELECT *
FROM TotalVesselsVMS
')
TotalVesselsVMS <- DBI::dbGetQuery(con, query_string)
# write.csv(TotalVesselsVMS, file = "TotalVesselsVMS.csv")
```

Entire Chile VMS data set: The count of vessels according to their
industrial or artisanal category is provided below as
**Resumen\_Embarcaciones\_VMS\_Total.csv**. However, the full list of
vessels is also available as **Lista\_Embarcaciones\_VMS\_Total.csv**

``` r
#Entire VMS Chile vessels data from Feb 2019 - Aug 2020
Lista_Embarcaciones_VMS_Total <- read.csv ("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/Tmp/TotalVesselsVMS.csv", header = TRUE)
#Modify final table before exporting
colnames(Lista_Embarcaciones_VMS_Total)[2] <- "Embarcacion"
colnames(Lista_Embarcaciones_VMS_Total)[3] <- "Categoria"
Lista_Embarcaciones_VMS_Total <- Lista_Embarcaciones_VMS_Total[-1]
#Generate Table with count of industrial and artisanal vessels
Resumen_Embarcaciones_VMS_Total <- count(Lista_Embarcaciones_VMS_Total, "Categoria")

#1,108 total vessels - 141 industrial and 967 artisanal

# write.csv(Lista_Embarcaciones_VMS_Total, file="Lista_Embarcaciones_VMS_Total.csv")
# write.csv(Resumen_Embarcaciones_VMS_Total, file="Resumen_Embarcaciones_VMS_Total.csv")
```

| Categoria                    | freq |
| :--------------------------- | ---: |
| chile\_vms\_industry         |  141 |
| chile\_vms\_small\_fisheries |  967 |

Only the January map will be displayed below. But the code is provided
for the generating of maps from Jan - July, 2020. Zoomed Out January:

``` r
Tmp <- copy(VMS_JanFH[VMS_JanFH$fishing_hours_sq_km > 0,])

# GFW logo
gfw_logo <- png::readPNG("/Users/Esteban/Documents/Jobs/GFW/General/Logo/GFW_logo_primary_White.png")
gfw_logo_rast <- grid::rasterGrob(gfw_logo, interpolate = T)

#Map
land_sf <- rnaturalearth::ne_countries(scale = 10, returnclass = 'sf')
VMS_CH_Jan_2020 <- ggplot() + 
  geom_sf(data = land_sf,
            fill = fishwatchr::gfw_palettes$map_country_dark[1],
            color = fishwatchr::gfw_palettes$map_country_dark[2],
          size=.1) +
    scale_fill_gradientn(colours = fishwatchr::gfw_palettes$map_effort_dark,
                         breaks = c(-3,-2,-1,0,1,2,3), labels = c('.001','0.01', '0.1', '1', '10', '100', '1000'),
                         limits = c(-3,3), oob=scales::squish)+
  fishwatchr::theme_gfw_map(theme = 'dark')+
  geom_tile(data = Tmp, aes(x = lon_bin, y = lat_bin, fill = Log_fishing_hours_sq_km), alpha = 0.5)+
  labs(fill = "Horas", title = "Horas de Pesca por Km2       Chile VMS enero 2020")+
  geom_sf(data=AllPoly,fill=NA, color="#A5AA99", size=.1)+
  coord_sf(xlim = c(-121.20, -46.67), ylim = c(-69.57, -8.50))+
  #Add GFW logo
  annotation_custom(gfw_logo_rast,
                      ymin = -11,
                      ymax = -6,
                      xmin = -59,
                      xmax = -46)
VMS_CH_Jan_2020
```

![](EP_Norte_Sur_files/figure-gfm/unnamed-chunk-20-1.png)<!-- -->

``` r
# ggsave("1_VMS_CH_Jan_2020.png", dpi=300)
```

Zoomed Out February

``` r
Tmp <- copy(VMS_FebFH[VMS_FebFH$fishing_hours_sq_km > 0,])

#Map
VMS_CH_Feb_2020 <- ggplot() + 
  geom_sf(data = land_sf,
            fill = fishwatchr::gfw_palettes$map_country_dark[1],
            color = fishwatchr::gfw_palettes$map_country_dark[2],
          size=.1) +
    scale_fill_gradientn(colours = fishwatchr::gfw_palettes$map_effort_dark,
                         breaks = c(-3,-2,-1,0,1,2,3), labels = c('.001','0.01', '0.1', '1', '10', '100', '1000'),
                         limits = c(-3,3), oob=scales::squish)+
  fishwatchr::theme_gfw_map(theme = 'dark')+
  geom_tile(data = Tmp, aes(x = lon_bin, y = lat_bin, fill = Log_fishing_hours_sq_km), alpha = 0.5)+
  labs(fill = "Horas", title = "Horas de Pesca por Km2       Chile VMS febrero 2020")+
  geom_sf(data=AllPoly,fill=NA, color="#A5AA99", size=.1)+
  coord_sf(xlim = c(-121.20, -46.67), ylim = c(-69.57, -8.50))+
  #Add GFW logo
  annotation_custom(gfw_logo_rast,
                      ymin = -11,
                      ymax = -6,
                      xmin = -59,
                      xmax = -46)
VMS_CH_Feb_2020
# ggsave("2_VMS_CH_Feb_2020.png", dpi=300)
```

Zoomed Out March

``` r
Tmp <- copy(VMS_MarFH[VMS_MarFH$fishing_hours_sq_km > 0,])

#Map
VMS_CH_Mar_2020 <- ggplot() + 
  geom_sf(data = land_sf,
            fill = fishwatchr::gfw_palettes$map_country_dark[1],
            color = fishwatchr::gfw_palettes$map_country_dark[2],
          size=.1) +
    scale_fill_gradientn(colours = fishwatchr::gfw_palettes$map_effort_dark,
                         breaks = c(-3,-2,-1,0,1,2,3), labels = c('.001','0.01', '0.1', '1', '10', '100', '1000'),
                         limits = c(-3,3), oob=scales::squish)+
  fishwatchr::theme_gfw_map(theme = 'dark')+
  geom_tile(data = Tmp, aes(x = lon_bin, y = lat_bin, fill = Log_fishing_hours_sq_km), alpha = 0.5)+
  labs(fill = "Horas", title = "Horas de Pesca por Km2       Chile VMS marzo 2020")+
  geom_sf(data=AllPoly,fill=NA, color="#A5AA99", size=.1)+
  coord_sf(xlim = c(-121.20, -46.67), ylim = c(-69.57, -8.50))+
  #Add GFW logo
  annotation_custom(gfw_logo_rast,
                      ymin = -11,
                      ymax = -6,
                      xmin = -59,
                      xmax = -46)
VMS_CH_Mar_2020
# ggsave("3_VMS_CH_Mar_2020.png", dpi=300)
```

Zoomed Out April

``` r
Tmp <- copy(VMS_AprFH[VMS_AprFH$fishing_hours_sq_km > 0,])

#Map
VMS_CH_Apr_2020 <- ggplot() + 
  geom_sf(data = land_sf,
            fill = fishwatchr::gfw_palettes$map_country_dark[1],
            color = fishwatchr::gfw_palettes$map_country_dark[2],
          size=.1) +
    scale_fill_gradientn(colours = fishwatchr::gfw_palettes$map_effort_dark,
                         breaks = c(-3,-2,-1,0,1,2,3), labels = c('.001','0.01', '0.1', '1', '10', '100', '1000'),
                         limits = c(-3,3), oob=scales::squish)+
  fishwatchr::theme_gfw_map(theme = 'dark')+
  geom_tile(data = Tmp, aes(x = lon_bin, y = lat_bin, fill = Log_fishing_hours_sq_km), alpha = 0.5)+
  labs(fill = "Horas", title = "Horas de Pesca por Km2       Chile VMS abril 2020")+
  geom_sf(data=AllPoly,fill=NA, color="#A5AA99", size=.1)+
  coord_sf(xlim = c(-121.20, -46.67), ylim = c(-69.57, -8.50))+
  #Add GFW logo
  annotation_custom(gfw_logo_rast,
                      ymin = -11,
                      ymax = -6,
                      xmin = -59,
                      xmax = -46)
VMS_CH_Apr_2020
# ggsave("4_VMS_CH_Apr_2020.png", dpi=300)
```

Zoomed Out May

``` r
Tmp <- copy(VMS_MayFH[VMS_MayFH$fishing_hours_sq_km > 0,])

#Map
VMS_CH_May_2020 <- ggplot() + 
  geom_sf(data = land_sf,
            fill = fishwatchr::gfw_palettes$map_country_dark[1],
            color = fishwatchr::gfw_palettes$map_country_dark[2],
          size=.1) +
    scale_fill_gradientn(colours = fishwatchr::gfw_palettes$map_effort_dark,
                         breaks = c(-3,-2,-1,0,1,2,3), labels = c('.001','0.01', '0.1', '1', '10', '100', '1000'),
                         limits = c(-3,3), oob=scales::squish)+
  fishwatchr::theme_gfw_map(theme = 'dark')+
  geom_tile(data = Tmp, aes(x = lon_bin, y = lat_bin, fill = Log_fishing_hours_sq_km), alpha = 0.5)+
  labs(fill = "Horas", title = "Horas de Pesca por Km2       Chile VMS mayo 2020")+
  geom_sf(data=AllPoly,fill=NA, color="#A5AA99", size=.1)+
  coord_sf(xlim = c(-121.20, -46.67), ylim = c(-69.57, -8.50))+
  #Add GFW logo
  annotation_custom(gfw_logo_rast,
                      ymin = -11,
                      ymax = -6,
                      xmin = -59,
                      xmax = -46)
VMS_CH_May_2020
# ggsave("5_VMS_CH_May_2020.png", dpi=300)
```

Zoomed Out June

``` r
Tmp <- copy(VMS_JunFH[VMS_JunFH$fishing_hours_sq_km > 0,])

#Map
VMS_CH_Jun_2020 <- ggplot() + 
  geom_sf(data = land_sf,
            fill = fishwatchr::gfw_palettes$map_country_dark[1],
            color = fishwatchr::gfw_palettes$map_country_dark[2],
          size=.1) +
    scale_fill_gradientn(colours = fishwatchr::gfw_palettes$map_effort_dark,
                         breaks = c(-3,-2,-1,0,1,2,3), labels = c('.001','0.01', '0.1', '1', '10', '100', '1000'),
                         limits = c(-3,3), oob=scales::squish)+
  fishwatchr::theme_gfw_map(theme = 'dark')+
  geom_tile(data = Tmp, aes(x = lon_bin, y = lat_bin, fill = Log_fishing_hours_sq_km), alpha = 0.5)+
  labs(fill = "Horas", title = "Horas de Pesca por Km2       Chile VMS junio 2020")+
  geom_sf(data=AllPoly,fill=NA, color="#A5AA99", size=.1)+
  coord_sf(xlim = c(-121.20, -46.67), ylim = c(-69.57, -8.50))+
  #Add GFW logo
  annotation_custom(gfw_logo_rast,
                      ymin = -11,
                      ymax = -6,
                      xmin = -59,
                      xmax = -46)
VMS_CH_Jun_2020
# ggsave("6_VMS_CH_Jun_2020.png", dpi=300)
```

Zoomed Out July

``` r
Tmp <- copy(VMS_JulFH[VMS_JulFH$fishing_hours_sq_km > 0,])

#Map
VMS_CH_Jul_2020 <- ggplot() + 
  geom_sf(data = land_sf,
            fill = fishwatchr::gfw_palettes$map_country_dark[1],
            color = fishwatchr::gfw_palettes$map_country_dark[2],
          size=.1) +
    scale_fill_gradientn(colours = fishwatchr::gfw_palettes$map_effort_dark,
                         breaks = c(-3,-2,-1,0,1,2,3), labels = c('.001','0.01', '0.1', '1', '10', '100', '1000'),
                         limits = c(-3,3), oob=scales::squish)+
  fishwatchr::theme_gfw_map(theme = 'dark')+
  geom_tile(data = Tmp, aes(x = lon_bin, y = lat_bin, fill = Log_fishing_hours_sq_km), alpha = 0.5)+
  labs(fill = "Horas", title = "Horas de Pesca por Km2       Chile VMS julio 2020")+
  geom_sf(data=AllPoly,fill=NA, color="#A5AA99", size=.1)+
  coord_sf(xlim = c(-121.20, -46.67), ylim = c(-69.57, -8.50))+
  #Add GFW logo
  annotation_custom(gfw_logo_rast,
                      ymin = -11,
                      ymax = -6,
                      xmin = -59,
                      xmax = -46)
VMS_CH_Jul_2020
# ggsave("7_VMS_CH_Jul_2020.png", dpi=300)
```

Create a GIF from all previous Zoomed Out Images

``` r
#GIF from the images of the maps Zoomed Out
png_files <- list.files("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/ZoomOut",
                        pattern = ".*png$", full.names = TRUE)
gifski(png_files, gif_file = "VMS_CH_Jan_Jul_2020.gif", width = 2100, height = 2100, delay = 1)
```

Only the January map will be displayed below. But the code is provided
for the generating of maps from Jan - July, 2020. Zoomed In January

``` r
Tmp <- copy(VMS_JanFH[VMS_JanFH$fishing_hours_sq_km > 0,])

#Map
VMSz_CH_Jan_2020 <- ggplot() + 
  geom_sf(data = land_sf,
            fill = fishwatchr::gfw_palettes$map_country_dark[1],
            color = fishwatchr::gfw_palettes$map_country_dark[2],
          size=.1) +
    scale_fill_gradientn(colours = fishwatchr::gfw_palettes$map_effort_dark,
                         breaks = c(-3,-2,-1,0,1,2,3), labels = c('.001','0.01', '0.1', '1', '10', '100', '1000'),
                         limits = c(-3,3), oob=scales::squish)+
  fishwatchr::theme_gfw_map(theme = 'dark')+
  geom_tile(data = Tmp, aes(x = lon_bin, y = lat_bin, fill = Log_fishing_hours_sq_km), alpha = 0.5)+
  labs(fill = "Horas", title = "Horas de Pesca por Km2       Chile VMS enero 2020")+
  geom_sf(data=AllPoly,fill=NA, color="#A5AA99", size=.1)+
  coord_sf(xlim = c(-85, -60), ylim = c(-60, -18))+
  #Add GFW logo
  annotation_custom(gfw_logo_rast,
                      ymin = -19,
                      ymax = -16,
                      xmin = -68,
                      xmax = -60)
VMSz_CH_Jan_2020
```

![](EP_Norte_Sur_files/figure-gfm/unnamed-chunk-28-1.png)<!-- -->

``` r
# ggsave("1_VMSz_CH_Jan_2020.png", dpi=300)
```

Zoomed In February

``` r
Tmp <- copy(VMS_FebFH[VMS_FebFH$fishing_hours_sq_km > 0,])

#Map
VMSz_CH_Feb_2020 <- ggplot() + 
  geom_sf(data = land_sf,
            fill = fishwatchr::gfw_palettes$map_country_dark[1],
            color = fishwatchr::gfw_palettes$map_country_dark[2],
          size=.1) +
    scale_fill_gradientn(colours = fishwatchr::gfw_palettes$map_effort_dark,
                         breaks = c(-3,-2,-1,0,1,2,3), labels = c('.001','0.01', '0.1', '1', '10', '100', '1000'),
                         limits = c(-3,3), oob=scales::squish)+
  fishwatchr::theme_gfw_map(theme = 'dark')+
  geom_tile(data = Tmp, aes(x = lon_bin, y = lat_bin, fill = Log_fishing_hours_sq_km), alpha = 0.5)+
  labs(fill = "Horas", title = "Horas de Pesca por Km2       Chile VMS febrero 2020")+
  geom_sf(data=AllPoly,fill=NA, color="#A5AA99", size=.1)+
  coord_sf(xlim = c(-85, -60), ylim = c(-60, -18))+
  #Add GFW logo
  annotation_custom(gfw_logo_rast,
                      ymin = -19,
                      ymax = -16,
                      xmin = -68,
                      xmax = -60)
VMSz_CH_Feb_2020
# ggsave("2_VMSz_CH_Feb_2020.png", dpi=300)
```

Zoomed In March

``` r
Tmp <- copy(VMS_MarFH[VMS_MarFH$fishing_hours_sq_km > 0,])

#Map
VMSz_CH_Mar_2020 <- ggplot() + 
  geom_sf(data = land_sf,
            fill = fishwatchr::gfw_palettes$map_country_dark[1],
            color = fishwatchr::gfw_palettes$map_country_dark[2],
          size=.1) +
    scale_fill_gradientn(colours = fishwatchr::gfw_palettes$map_effort_dark,
                         breaks = c(-3,-2,-1,0,1,2,3), labels = c('.001','0.01', '0.1', '1', '10', '100', '1000'),
                         limits = c(-3,3), oob=scales::squish)+
  fishwatchr::theme_gfw_map(theme = 'dark')+
  geom_tile(data = Tmp, aes(x = lon_bin, y = lat_bin, fill = Log_fishing_hours_sq_km), alpha = 0.5)+
  labs(fill = "Horas", title = "Horas de Pesca por Km2       Chile VMS marzo 2020")+
  geom_sf(data=AllPoly,fill=NA, color="#A5AA99", size=.1)+
  coord_sf(xlim = c(-85, -60), ylim = c(-60, -18))+
  #Add GFW logo
  annotation_custom(gfw_logo_rast,
                      ymin = -19,
                      ymax = -16,
                      xmin = -68,
                      xmax = -60)
VMSz_CH_Mar_2020
# ggsave("3_VMSz_CH_Mar_2020.png", dpi=300)
```

Zoomed In April

``` r
Tmp <- copy(VMS_AprFH[VMS_AprFH$fishing_hours_sq_km > 0,])

#Map
VMSz_CH_Apr_2020 <- ggplot() + 
  geom_sf(data = land_sf,
            fill = fishwatchr::gfw_palettes$map_country_dark[1],
            color = fishwatchr::gfw_palettes$map_country_dark[2],
          size=.1) +
    scale_fill_gradientn(colours = fishwatchr::gfw_palettes$map_effort_dark,
                         breaks = c(-3,-2,-1,0,1,2,3), labels = c('.001','0.01', '0.1', '1', '10', '100', '1000'),
                         limits = c(-3,3), oob=scales::squish)+
  fishwatchr::theme_gfw_map(theme = 'dark')+
  geom_tile(data = Tmp, aes(x = lon_bin, y = lat_bin, fill = Log_fishing_hours_sq_km), alpha = 0.5)+
  labs(fill = "Horas", title = "Horas de Pesca por Km2       Chile VMS abril 2020")+
  geom_sf(data=AllPoly,fill=NA, color="#A5AA99", size=.1)+
  coord_sf(xlim = c(-85, -60), ylim = c(-60, -18))+
  #Add GFW logo
  annotation_custom(gfw_logo_rast,
                      ymin = -19,
                      ymax = -16,
                      xmin = -68,
                      xmax = -60)
VMSz_CH_Apr_2020
# ggsave("4_VMSz_CH_Apr_2020.png", dpi=300)
```

Zoomed In May

``` r
Tmp <- copy(VMS_MayFH[VMS_MayFH$fishing_hours_sq_km > 0,])

#Map
VMSz_CH_May_2020 <- ggplot() + 
  geom_sf(data = land_sf,
            fill = fishwatchr::gfw_palettes$map_country_dark[1],
            color = fishwatchr::gfw_palettes$map_country_dark[2],
          size=.1) +
    scale_fill_gradientn(colours = fishwatchr::gfw_palettes$map_effort_dark,
                         breaks = c(-3,-2,-1,0,1,2,3), labels = c('.001','0.01', '0.1', '1', '10', '100', '1000'),
                         limits = c(-3,3), oob=scales::squish)+
  fishwatchr::theme_gfw_map(theme = 'dark')+
  geom_tile(data = Tmp, aes(x = lon_bin, y = lat_bin, fill = Log_fishing_hours_sq_km), alpha = 0.5)+
  labs(fill = "Horas", title = "Horas de Pesca por Km2       Chile VMS mayo 2020")+
  geom_sf(data=AllPoly,fill=NA, color="#A5AA99", size=.1)+
  coord_sf(xlim = c(-85, -60), ylim = c(-60, -18))+
  #Add GFW logo
  annotation_custom(gfw_logo_rast,
                      ymin = -19,
                      ymax = -16,
                      xmin = -68,
                      xmax = -60)
VMSz_CH_May_2020
# ggsave("5_VMSz_CH_May_2020.png", dpi=300)
```

Zoomed In June

``` r
Tmp <- copy(VMS_JunFH[VMS_JunFH$fishing_hours_sq_km > 0,])

#Map
VMSz_CH_Jun_2020 <- ggplot() + 
  geom_sf(data = land_sf,
            fill = fishwatchr::gfw_palettes$map_country_dark[1],
            color = fishwatchr::gfw_palettes$map_country_dark[2],
          size=.1) +
    scale_fill_gradientn(colours = fishwatchr::gfw_palettes$map_effort_dark,
                         breaks = c(-3,-2,-1,0,1,2,3), labels = c('.001','0.01', '0.1', '1', '10', '100', '1000'),
                         limits = c(-3,3), oob=scales::squish)+
  fishwatchr::theme_gfw_map(theme = 'dark')+
  geom_tile(data = Tmp, aes(x = lon_bin, y = lat_bin, fill = Log_fishing_hours_sq_km), alpha = 0.5)+
  labs(fill = "Horas", title = "Horas de Pesca por Km2       Chile VMS junio 2020")+
  geom_sf(data=AllPoly,fill=NA, color="#A5AA99", size=.1)+
  coord_sf(xlim = c(-85, -60), ylim = c(-60, -18))+
  #Add GFW logo
  annotation_custom(gfw_logo_rast,
                      ymin = -19,
                      ymax = -16,
                      xmin = -68,
                      xmax = -60)
VMSz_CH_Jun_2020
# ggsave("6_VMSz_CH_Jun_2020.png", dpi=300)
```

Zoomed In July

``` r
Tmp <- copy(VMS_JulFH[VMS_JulFH$fishing_hours_sq_km > 0,])

#Map
VMSz_CH_Jul_2020 <- ggplot() + 
  geom_sf(data = land_sf,
            fill = fishwatchr::gfw_palettes$map_country_dark[1],
            color = fishwatchr::gfw_palettes$map_country_dark[2],
          size=.1) +
    scale_fill_gradientn(colours = fishwatchr::gfw_palettes$map_effort_dark,
                         breaks = c(-3,-2,-1,0,1,2,3), labels = c('.001','0.01', '0.1', '1', '10', '100', '1000'),
                         limits = c(-3,3), oob=scales::squish)+
  fishwatchr::theme_gfw_map(theme = 'dark')+
  geom_tile(data = Tmp, aes(x = lon_bin, y = lat_bin, fill = Log_fishing_hours_sq_km), alpha = 0.5)+
  labs(fill = "Horas", title = "Horas de Pesca por Km2       Chile VMS julio 2020")+
  geom_sf(data=AllPoly,fill=NA, color="#A5AA99", size=.1)+
  coord_sf(xlim = c(-85, -60), ylim = c(-60, -18))+
  #Add GFW logo
  annotation_custom(gfw_logo_rast,
                      ymin = -19,
                      ymax = -16,
                      xmin = -68,
                      xmax = -60)
VMSz_CH_Jul_2020
# ggsave("7_VMSz_CH_Jul_2020.png", dpi=300)
```

Create a GIF from all previous Zoomed In Images

``` r
#GIF from the images of the maps Zoomed In
png_files <- list.files("/Users/Esteban/Documents/Jobs/GFW/Proyectos/Chile/SERNAPESCA/Data/Maps_Tables/ZoomIn",
                        pattern = ".*png$", full.names = TRUE)
gifski(png_files, gif_file = "VMSz_CH_Jan_Jul_2020.gif", width = 2100, height = 2100, delay = 1)
```
