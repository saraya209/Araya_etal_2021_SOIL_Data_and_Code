---
title: "Tidy Up Laboratory Data"
author: "Samuel Araya"
date: 'December 09, 2021'
output:
   html_document:
     toc: true
     keep_md: true
---
Tidy up and reformat laboratory data and export analysis ready tables.



```r
# Set up file directories/paths relative to the working directory
wDir <- getwd()
dataDir <- file.path(wDir,"Data_Raw")
procDataDir <- file.path(wDir,"Data_Processed")
plotDir <- file.path(wDir,"Plots")
#
```

```r
library(tidyverse)
library(data.table)
library(readxl)
## Custom Functions
source("functions_custom.R")
## Import Treatment-Plot looku table
dt.look = read.csv(file.path(dataDir, "Treatment_Plot_Lookup.csv"))

#
```

## Import and Format KSAT Instrument Data

- Import "csv" files exported from KSAT instrument for each sample.
- Loop across each file and extract relevant fields and combine in one table
- Save/export `KSAT_Summary` table 



```r
ksatfile.lst = list.files(path = file.path(dataDir, "KSAT"))
for (i in 1:length(ksatfile.lst)){
  # Import results only from each file
  i.dt <- data.table::fread(file.path(dataDir,"KSAT",ksatfile.lst[i]), 
                header = FALSE,
                sep = "\t",
                skip = 40, # results start at 41 run for 11 lines
                nrows = 11,
                select = c(2),
                fill=TRUE)
  # Set name
  h.name = str_to_upper(str_split(ksatfile.lst[i],"_", simplify = T)[3])
  names(i.dt)[1] = h.name
  # Bind columns
  if(i >1){
    l.dt = cbind(l.dt,i.dt)
  }else{
    l.dt = i.dt
  }
  
}
# Transpose data from long to wide format
w.dt = transpose(l.dt)
names.row = data.table(rownames(t(l.dt)))
w.dt$Sample = names.row

# Append column names
names.col <- c("MaxOffsetAdj_cm", "a_cm","b_ps","c_cm", "r2", 
               "KsTotal_cm_d", "KsTotal_m_s", "KsSoil_cm_d", 
               "KsSoil_m_s", "KsSoil10_cm_d","KsSoil10_m_s",
               "Sample"
)
names(w.dt) = names.col
# Split names to plot number and depth
w.dt <- w.dt%>%
  mutate(Plot_ID = as.numeric(str_extract(Sample, "[0-9]+")),
         Depth_ID = str_extract(Sample, "[aA-zZ]+") )%>%
  arrange(Plot_ID, desc(Depth_ID))
```

### Export KSAT summary table


```r
save_table(w.dt, procDataDir, "KSAT_Summary")

#clean up
rm(i.dt,l.dt, names.row)
```


## Import and Format HYPROP Instrument Data

Import HYPROP software exported "xlsx" files generated for each sample. 
These files include the measured and fitted suction (pF) measurements vs 
water content and conductivity. 

**The HYPROP Excel files also include retention data from WPC instrument and Saturated Conductivity Data from KSAT Instrument**.

Measured values are labeled "Evaluation" in the file and the fitted 
values as "Fitting". 

HYPROP software provides multiple fitting models, we used 
the **bimodal constrained van Genuchten model (Durner)**. 
   
The script loops through each file, extracts relevant variables from relevant 
sheets and combines sample data into tables. 
The formatted data tables are organised into the following tables:

(1) Summary of soils with fit parameters (`HYPROP_Fit_bi_Summary`), 
(2) The measured conductivity and retention curve tables (`HYPROP_Evaluation_K_pF` and `HYPROP_Evaluation_WRC_pF`, respectively) 
(3) The fitted conductivity and retention curve tables.(`HYPROP_Bi_Fit_K_pF` and `HYPROP_Bi_Fit_WRC_pF`, respectively)



```r
hfile.lst = list.files(path = file.path(dataDir, "Hyprop"))
#
for (i in 1:length(hfile.lst)){
  # Import results only from each file
  # Information sheet: two columns
  i.info <- read_excel(file.path(dataDir,"Hyprop",hfile.lst[i]), 
                       sheet = "Information", 
                       col_names = TRUE)
  # subset information columns
  i.info <- i.info%>%
    filter(`Parameter Name` == "Sample name:" |
             `Parameter Name` == "Duration of Measurements:" |
             `Parameter Name` == "Density [g/cm3]:" |
             `Parameter Name` == "Porosity [-]:")
  i.sample <- as.character(i.info[1,2])
  
  
  # Conductivity measurements sheet
  i.k <- read_excel(file.path(dataDir,"Hyprop",hfile.lst[i]),
                    col_names = TRUE,
                    sheet = "Evaluation-Conductivity K(pF)")
  i.k$Sample_ID = i.sample
  
  # Water retention measurements sheet
  i.wrc <- read_excel(file.path(dataDir,"Hyprop",hfile.lst[i]), 
                      col_names = TRUE,
                      sheet = 9) # Evaluation-Retention Θ(pF)
  i.wrc$Sample_ID = i.sample
  
  # Conductivity fit values sheet
  i.fk <- read_excel(file.path(dataDir,"Hyprop",hfile.lst[i]),
                     col_names = TRUE,
                     sheet = "Fitting-Conductivity K(pF)")
  i.fk$Sample_ID = i.sample
  
  # Water retention fit values sheet
  i.fr <- read_excel(file.path(dataDir,"Hyprop",hfile.lst[i]), 
                     col_names = TRUE,
                     sheet = 12) # Fitting-Retention Θ(pF)
  i.fr$Sample_ID = i.sample
  
  # Drying time values sheet
  i.dry <- read_excel(file.path(dataDir,"Hyprop",hfile.lst[i]), 
                      col_names = TRUE,
                      sheet = "Spline Points")
  i.dry$Sample_ID = i.sample
  
  # Fit Parameter sheet
  i.fit <- read_excel(file.path(dataDir,"Hyprop",hfile.lst[i]), 
                      col_names = TRUE,
                      sheet = "Fitting-Parameter value")
  #subset fit parameter columns
  i.fit <- i.fit%>%
    select(Parameter, Value)%>%
    spread(Parameter,Value)
  # # Transpose data from long to wide format
  # i.fit.val = as.data.frame(t(i.fit[,-1]))
  # i.fit.name = as.data.frame(t(i.fit[,-2]))
  # 
  # i.fit = transpose(i.fit)
  # colnames(i.fit.val) = i.fit.name
  # colnames(i.fit) <- i.fit[1, ]
  # i.fit <- i.fit[-1, ]
  i.fit$Sample_ID = i.sample
  
  # Fit statistics sheet
  i.stat <- read_excel(file.path(dataDir,"Hyprop",hfile.lst[i]), 
                       col_names = TRUE,
                       sheet = "Fitting-Statistical analysis")
  # Transpose data from long to wide format
  i.stat = i.stat%>%
    spread(Name,Value)
  # i.stat = transpose(i.stat)
  # colnames(i.stat) <- i.stat[1, ]
  # i.stat <- i.stat[-1, ]
  i.stat$Sample_ID = i.sample
  
  # Set name
  # Bind columns
  if(i >1){
    dt.info = cbind(dt.info, i.info$Value)
    dt.fit = rbind(dt.fit,i.fit)
    dt.k = rbind(dt.k,i.k)
    dt.r = rbind(dt.r,i.wrc)
    dt.fk = rbind(dt.fk,i.fk)
    dt.fr = rbind(dt.fr,i.fr)
    dt.stat = rbind(dt.stat,i.stat)
    dt.dry = rbind(dt.dry,i.dry)
  }else{
    dt.info = i.info
    dt.fit = i.fit
    dt.k = i.k
    dt.r = i.wrc
    dt.fk = i.fk
    dt.fr = i.fr
    dt.stat = i.stat
    dt.dry = i.dry
  }
  
}
```

Rename column headers


```r
colnames(dt.k) = c("pF", "logK_cmd","manual","Sample_ID")
colnames(dt.r) = c("pF", "VWC","manual","Sample_ID")
colnames(dt.fk) = c("pF", "logK_cmd","Sample_ID")
colnames(dt.fr) = c("pF", "VWC","Sample_ID")
# By time
colnames(dt.dry) = c("Time", "pF_B", "pF_T", "Weight","Sample_ID")
#
```

Transpose info data from long to wide format


```r
dt.info = as.data.frame(t(dt.info))
colnames(dt.info) <- c("Sample_ID", "Duration","BD", "Porosity")
dt.info <- dt.info[-1, ]
#
```

Append Treatment-Plot description from look-up table


```r
dt.mk = left_join(dt.k,dt.look)
dt.mr = left_join(dt.r,dt.look)
dt.mfk = left_join(dt.fk,dt.look)
dt.mfr = left_join(dt.fr,dt.look)
dt.minfo = left_join(dt.info, dt.look)
dt.dry = left_join(dt.dry, dt.look)
```

Combine Fitted table information into one table:

(1) fit parameters (`dt.fit`),
(2) Sample info (`dt.minfo`), and
(3) RMSE for theta and K (`dt.stat`)


```r
dt.info_fit <- dt.minfo %>% 
  dplyr::left_join(dt.fit, by = "Sample_ID") %>% 
  dplyr::left_join(dt.stat, by = "Sample_ID")
```

Fix Summary table properties


```r
summary(dt.info_fit)
```

```
##   Sample_ID           Duration              BD              Porosity          Depth_text           Plot_ID        Depth_ID        
##  Length:33          Length:33          Length:33          Length:33          Length:33          Min.   : 5.00   Length:33         
##  Class :character   Class :character   Class :character   Class :character   Class :character   1st Qu.: 8.00   Class :character  
##  Mode  :character   Mode  :character   Mode  :character   Mode  :character   Mode  :character   Median :16.00   Mode  :character  
##                                                                                                 Mean   :18.09                     
##                                                                                                 3rd Qu.:24.00                     
##                                                                                                 Max.   :32.00                     
##  Winter_Cover       Tillage_Type         Depth_Top       Depth_Bottom    Replicate     Distance_From_East_Edge_Approx_m
##  Length:33          Length:33          Min.   : 0.000   Min.   : 5.0   Min.   :2.000   Min.   :30.00                   
##  Class :character   Class :character   1st Qu.: 0.000   1st Qu.: 5.0   1st Qu.:2.000   1st Qu.:40.00                   
##  Mode  :character   Mode  :character   Median : 0.000   Median : 5.0   Median :4.000   Median :40.00                   
##                                        Mean   : 9.697   Mean   :14.7   Mean   :4.909   Mean   :37.58                   
##                                        3rd Qu.:20.000   3rd Qu.:25.0   3rd Qu.:6.000   3rd Qu.:40.00                   
##                                        Max.   :20.000   Max.   :25.0   Max.   :8.000   Max.   :40.00                   
##  Sampled_Date          alpha1             alpha2               Ks                 n1                 n2           
##  Length:33          Length:33          Length:33          Length:33          Length:33          Length:33         
##  Class :character   Class :character   Class :character   Class :character   Class :character   Class :character  
##  Mode  :character   Mode  :character   Mode  :character   Mode  :character   Mode  :character   Mode  :character  
##                                                                                                                   
##                                                                                                                   
##                                                                                                                   
##      tau                th_r               th_s                w2                AICc              RMSE_K         
##  Length:33          Length:33          Length:33          Length:33          Length:33          Length:33         
##  Class :character   Class :character   Class :character   Class :character   Class :character   Class :character  
##  Mode  :character   Mode  :character   Mode  :character   Mode  :character   Mode  :character   Mode  :character  
##                                                                                                                   
##                                                                                                                   
##                                                                                                                   
##    RMSE_TH         
##  Length:33         
##  Class :character  
##  Mode  :character  
##                    
##                    
## 
```

```r
sapply(dt.info_fit, class)
```

```
##                        Sample_ID                         Duration                               BD                         Porosity 
##                      "character"                      "character"                      "character"                      "character" 
##                       Depth_text                          Plot_ID                         Depth_ID                     Winter_Cover 
##                      "character"                        "integer"                      "character"                      "character" 
##                     Tillage_Type                        Depth_Top                     Depth_Bottom                        Replicate 
##                      "character"                        "integer"                        "integer"                        "integer" 
## Distance_From_East_Edge_Approx_m                     Sampled_Date                           alpha1                           alpha2 
##                        "integer"                      "character"                      "character"                      "character" 
##                               Ks                               n1                               n2                              tau 
##                      "character"                      "character"                      "character"                      "character" 
##                             th_r                             th_s                               w2                             AICc 
##                      "character"                      "character"                      "character"                      "character" 
##                           RMSE_K                          RMSE_TH 
##                      "character"                      "character"
```

Assign proper format to columns


```r
## List of columns that should be numeric
cols.num <-c('Duration','BD','Porosity','alpha1','alpha2','Ks','n1','n2','tau',
             'th_r','th_s','w2','AICc','RMSE_K','RMSE_TH')

dt <- dt.info_fit %>%
  mutate(Sampled_Date = parse_date(Sampled_Date, format = "%m/%d/%Y")) %>%
  mutate(across(all_of(cols.num), parse_number))
```

### Export HYPROP Summary Tables 


```r
#Summary fit data
save_table(dt, procDataDir, "HYPROP_Fit_bi_Summary")

## Conductivity
save_table(dt.mk, procDataDir, "HYPROP_Evaluation_K_pF")
save_table(dt.mfk,procDataDir, "HYPROP_Bi_Fit_K_pF")

## Retention
save_table(dt.mr, procDataDir, "HYPROP_Evaluation_WRC_pF")
save_table(dt.mfr,procDataDir, "HYPROP_Bi_Fit_WRC_pF")
## Dry
save_table(dt.dry,procDataDir, "HYPROP_Dry")
```

