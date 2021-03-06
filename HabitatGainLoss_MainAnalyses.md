Habitat Gain and Loss Analyses
================
Melanie Dickie, Mariana Nagy-Reis, Sophie Gilbert
16/04/2021

This script contains the code used for the analyses in Nagy-Reis et
al. 2021. Habitat loss accelerates for the endangered woodland caribou
in western Canada. Conservation Science and Practice

## Forest Cover

``` r
##load packages
require(dplyr)
library(compositions)
library(devtools)
library(ggpubr)
library(car)
require(ggplot2)
require(cowplot)
require(tidyverse)
library(popbio)
library(fitdistrplus)
library(broom)
require(cowplot)
```

Load and prep data

Data were retrieved from: Global Forest Change:
<https://earthenginepartners.appspot.com/science-2013-global-forest/download_v1.7.html>
CanLaD:
<https://open.canada.ca/data/en/dataset/add1346b-f632-4eb9-a83d-a662b38655ad>

We used provincial maps to delimitate caribou subpopulation range
boundaries (AB:
<https://extranet.gov.ab.ca/srd/geodiscover/srd_pub/LAT/FWDSensitivity/CaribouRange.zip>;
BC:
<https://catalogue.data.gov.bc.ca/dataset/caribou-herd-locations-for-bc>
and updated with data provided by the BC Government) and federal
recovery strategies and management plans (Environment Canada 2012a, b;
Environment Canada 2014) to classify caribou ecotypes and groups.Caribou
range boundaries “CaribouRangesAB\_BC2020\_10TM.zip” are in “data”

The following summaries were run in GIS before being exported as
“habitat\_raw\_data\_final.csv”:

1- forest loss and gain values from GFC were clipped to caribou ranges.

2- the clipped loss product was used to clip the harvest and fire data
(CanLaD). Loss pixels not captured by either harvest or fire were
classified as “other”.

``` r
data_raw<-read.csv(here::here("data", "habitat_raw_data_final.csv"))
options(scipen=999) #to avoid scientific notation

#Estimating percentages
data_raw$Loss_perc<-data_raw$Loss/data_raw$Range_area*100 #dividing metric area by range area *100
data_raw$Loss_perc_2012<-data_raw$Loss_2012/data_raw$Range_area*100
data_raw$Gain_perc<-data_raw$Gain/data_raw$Range_area*100
data_raw$Harv_perc<-data_raw$Harvest/data_raw$Range_area*100
data_raw$Other_perc<-data_raw$Other/data_raw$Range_area*100
data_raw$Fire_perc<-data_raw$Fire/data_raw$Range_area*100

#Estimating annual rates
data_raw$Loss_rate<-data_raw$Loss_perc/18 #dividing metric percentage by number of years of data collection
data_raw$Gain_rate<-data_raw$Gain_perc/12
data_raw$Harv_rate<-data_raw$Harv_perc/15
data_raw$Other_rate<-data_raw$Other_perc/15
data_raw$Fire_rate<-data_raw$Fire_perc/15

#Estimating net change
data_raw$Net_change<-data_raw$Gain_perc-data_raw$Loss_perc_2012

##Plot
ggplot(data=data_raw, aes(x=sort(as.numeric(Range.ID), decreasing = FALSE), y=Net_change, fill=Ecotype)) +
  geom_bar(stat="identity", width=0.8, position = position_dodge(width=0.1))+
  theme_bw()+
  #aes(x=reorder(data_raw,Ecotype, function(x)-length(x)))) +
  ylab("Net Habitat Change (%)")+
  geom_vline(xintercept=55.5, linetype="dashed")+
  geom_hline(yintercept=0)+
  geom_vline(xintercept=0)+
  theme(legend.position = c(0.25, 0.2))+
  scale_fill_manual(values = c("aquamarine2", "cornflowerblue", "deeppink", "lightpink", "violetred"), labels = c("Boreal", "Northern Mountain", "Southern Mountain - CG", "Southern Mountain - NG", "Southern Mountain - SG"))+
  theme(axis.title.x=element_blank(),axis.text.x=element_blank(),axis.ticks.x=element_blank())+
  theme(axis.title.y=element_text(size=14), axis.text.y = element_text(size=12), axis.ticks.y=element_blank()) + 
  theme(panel.grid.minor=element_blank(), panel.grid.major=element_blank(), panel.border = element_blank())
```

![](HabitatGainLoss_MainAnalyses_files/figure-gfm/unnamed-chunk-2-1.png)<!-- -->

``` r
#Log transformation
data_raw$Loss_rate_log<-log(data_raw$Loss+1)
data_raw$Loss_rate_log<-log(data_raw$Loss_rate+1)
data_raw$Gain_log<-log(data_raw$Gain+1)
data_raw$Gain_rate_log<-log(data_raw$Gain_rate+1)
data_raw$Net_log<-log(data_raw$Net_change+1-min(data_raw$Net_change))#to deal with negative values

#Transformation applied to proportional data to break circularity (Centered log ratio transformation)
data_raw$Harvest_tranf<-clr(data_raw$Harv_perc)
data_raw$Fire_tranf<-clr(data_raw$Fire_perc)
data_raw$Other_tranf<-clr(data_raw$Other_perc)
data_raw$Gain_rate_tranf<-clr(data_raw$Gain_rate)
data_raw$Loss_rate_tranf<-clr(data_raw$Loss_rate)
data_raw$Gain_perc_tranf<-clr(data_raw$Gain_perc)
data_raw$Loss_perc_tranf<-clr(data_raw$Loss_perc)
data_raw$Harvest_rate_tranf<-clr(data_raw$Harv_rate)
data_raw$Fire_rate_tranf<-clr(data_raw$Fire_rate)
data_raw$Other_rate_tranf<-clr(data_raw$Other_rate)

data_raw[,c(25:38)]<-apply(data_raw[,c(25:38)],2,as.numeric) #Converting factor columns into numeric
data_raw[,c(13:38)]<-apply(data_raw[,c(13:38)],2,round,2) #Rounding values
```

Summarize data by province and caribou ecotype

``` r
LossTable<-group_by(data_raw, Province, Ecotype) %>%
  summarise(
    Count = n(),
    LossMean = round(mean(Loss_rate, na.rm = TRUE),2),
    LossSD = round(sd(Loss_rate, na.rm = TRUE),2))

GainTable<-group_by(data_raw, Province, Ecotype) %>%
  summarise(
    GainMean = round(mean(Gain_rate, na.rm = TRUE),2),
    GainSD = round(sd(Gain_rate, na.rm = TRUE),2))

NetTable<-group_by(data_raw, Province, Ecotype) %>%
  summarise(
    NetMean = round(mean(Net_change, na.rm = TRUE),2),
    NetSD = round(sd(Net_change, na.rm = TRUE),2))

HarvestTable<-group_by(data_raw, Province, Ecotype) %>%
  summarise(
    HarvestMean = round(mean(Harv_rate, na.rm = TRUE),2),
    HarvestSD = round(sd(Harv_rate, na.rm = TRUE),2))

FireTable<-group_by(data_raw, Province, Ecotype) %>%
  summarise(
    FireMean = round(mean(Fire_rate, na.rm = TRUE),2),
    FireSD = round(sd(Fire_rate, na.rm = TRUE),2))

OtherTable<-group_by(data_raw, Province, Ecotype) %>%
  summarise(
    OtherMean = round(mean(Other_rate, na.rm = TRUE),2),
    OtherSD = round(sd(Other_rate, na.rm = TRUE),2))

FullTable<-Reduce(merge, list(LossTable, GainTable, NetTable, HarvestTable, FireTable, OtherTable))
```

Table 1: Annual rate of habitat change (measured as forest cover; mean %
and SD) within the ranges of five caribou ecotypes in Alberta (AB) and
British Columbia (BC). All values refer to annual estimates, except net
change, which is for the entire period of 2000-2012.

``` r
knitr::kable(FullTable)
```

| Province | Ecotype | Count | LossMean | LossSD | GainMean | GainSD | NetMean | NetSD | HarvestMean | HarvestSD | FireMean | FireSD | OtherMean | OtherSD |
| :------- | :------ | ----: | -------: | -----: | -------: | -----: | ------: | ----: | ----------: | --------: | -------: | -----: | --------: | ------: |
| AB       | B       |    12 |     0.69 |   0.37 |     0.08 |   0.10 |  \-7.60 |  7.07 |        0.11 |      0.14 |     0.52 |   0.49 |      0.09 |    0.04 |
| AB       | SM\_CG  |     5 |     0.40 |   0.47 |     0.09 |   0.07 |  \-3.92 |  6.23 |        0.26 |      0.44 |     0.13 |   0.08 |      0.07 |    0.08 |
| BC       | B       |     6 |     0.27 |   0.19 |     0.03 |   0.02 |  \-1.41 |  1.91 |        0.02 |      0.01 |     0.16 |   0.17 |      0.08 |    0.11 |
| BC       | NM      |    17 |     0.12 |   0.13 |     0.03 |   0.03 |  \-1.05 |  1.82 |        0.00 |      0.01 |     0.08 |   0.10 |      0.03 |    0.02 |
| BC       | SM\_CG  |     5 |     0.58 |   0.29 |     0.26 |   0.12 |  \-3.00 |  3.89 |        0.25 |      0.21 |     0.15 |   0.14 |      0.19 |    0.13 |
| BC       | SM\_NG  |    10 |     0.65 |   0.36 |     0.27 |   0.12 |  \-4.28 |  5.22 |        0.16 |      0.11 |     0.16 |   0.18 |      0.31 |    0.26 |
| BC       | SM\_SG  |    17 |     0.26 |   0.18 |     0.35 |   0.29 |    1.61 |  3.95 |        0.09 |      0.10 |     0.06 |   0.06 |      0.07 |    0.07 |

Evaluate loss from 2000 to 2018 by province and ecotype

``` r
res.aov_rate_tranf <- aov(Loss_rate_tranf ~ Province * Ecotype, data = data_raw)
knitr::kable(tidy(res.aov_rate_tranf))
```

| term             | df |     sumsq |    meansq | statistic |   p.value |
| :--------------- | -: | --------: | --------: | --------: | --------: |
| Province         |  1 |  9.750081 | 9.7500806 | 11.316006 | 0.0012924 |
| Ecotype          |  4 | 38.542835 | 9.6357087 | 11.183265 | 0.0000006 |
| Province:Ecotype |  1 |  5.685178 | 5.6851776 |  6.598254 | 0.0125132 |
| Residuals        | 65 | 56.005206 | 0.8616185 |        NA |        NA |

Evaluate gain from 2000 to 2012 by province and ecotype

``` r
res.aov_gain_tranf <- aov(Gain_rate_tranf ~ Province * Ecotype, data = data_raw) 
knitr::kable(tidy(res.aov_gain_tranf))
```

| term             | df |     sumsq |     meansq | statistic |   p.value |
| :--------------- | -: | --------: | ---------: | --------: | --------: |
| Province         |  1 |  2.478185 |  2.4781848 |  2.638892 | 0.1091160 |
| Ecotype          |  4 | 86.517752 | 21.6294379 | 23.032084 | 0.0000000 |
| Province:Ecotype |  1 |  8.840628 |  8.8406277 |  9.413933 | 0.0031387 |
| Residuals        | 65 | 61.041523 |  0.9391004 |        NA |        NA |

Evaluate net change (2000-2012) by province and ecotype

``` r
data_raw_clean_3<-data_raw[-c(64,71,45),]#removing a few outliers
res.aov_net <- aov(Net_change ~ Province * Ecotype, data = data_raw_clean_3)
knitr::kable(tidy(res.aov_net))
```

| term             | df |     sumsq |    meansq | statistic |   p.value |
| :--------------- | -: | --------: | --------: | --------: | --------: |
| Province         |  1 | 141.80569 | 141.80569 | 11.549495 | 0.0011891 |
| Ecotype          |  4 | 202.35663 |  50.58916 |  4.120281 | 0.0050357 |
| Province:Ecotype |  1 |  57.90397 |  57.90397 |  4.716042 | 0.0337169 |
| Residuals        | 62 | 761.24129 |  12.27809 |        NA |        NA |

Evaluate harvest from 2000 to 2015 by province and ecotype

``` r
res.aov_harv_rate <- aov(Harvest_rate_tranf ~ Province * Ecotype, data = data_raw)
knitr::kable(tidy(res.aov_harv_rate))
```

| term             | df |     sumsq |   meansq | statistic |   p.value |
| :--------------- | -: | --------: | -------: | --------: | --------: |
| Province         |  1 |  10.80365 | 10.80365 |  3.605798 | 0.0620185 |
| Ecotype          |  4 | 229.38655 | 57.34664 | 19.139856 | 0.0000000 |
| Province:Ecotype |  1 |  23.47807 | 23.47807 |  7.835974 | 0.0067350 |
| Residuals        | 65 | 194.75233 |  2.99619 |        NA |        NA |

Evaluate fire from 2000-2015 by province and ecotype

``` r
data_raw_clean_2<-data_raw[-c(8,25),]
res.aov_fire_rate <- aov(Fire_rate_tranf ~ Province * Ecotype, data = data_raw_clean_2)
knitr::kable(tidy(res.aov_fire_rate))
```

| term             | df |      sumsq |    meansq | statistic |   p.value |
| :--------------- | -: | ---------: | --------: | --------: | --------: |
| Province         |  1 |  34.275917 | 34.275917 | 15.413980 | 0.0002166 |
| Ecotype          |  4 |  18.403334 |  4.600834 |  2.069008 | 0.0954252 |
| Province:Ecotype |  1 |   3.269521 |  3.269521 |  1.470313 | 0.2298238 |
| Residuals        | 63 | 140.092485 |  2.223690 |        NA |        NA |

Compare forest loss from in 2000–2008 vs. 2009–2018

``` r
data_t<-read.csv(here::here("data", "T-test.csv"))
t<-t.test(data_t$Loss_km~data_t$Group)
knitr::kable(tidy(t))
```

|   estimate | estimate1 | estimate2 | statistic |   p.value | parameter |   conf.low |  conf.high | method                  | alternative |
| ---------: | --------: | --------: | --------: | --------: | --------: | ---------: | ---------: | :---------------------- | :---------- |
| \-950.2222 |  1365.978 |    2316.2 | \-2.26342 | 0.0432836 |  11.80581 | \-1866.596 | \-33.84798 | Welch Two Sample t-test | two.sided   |

T-test before and after recovery strategies/plans Recovery plans were
implemented in 2014 for SMC and 2012 for Boreal and NM

``` r
data_t_2<-read.csv(here::here("data", "before_after.csv"))
t_SMC<-t.test(data_t_2$SMC~data_t_2$GROUP_2014)
knitr::kable(tidy(t_SMC))
```

|   estimate | estimate1 | estimate2 |   statistic |   p.value | parameter |   conf.low | conf.high | method                  | alternative |
| ---------: | --------: | --------: | ----------: | --------: | --------: | ---------: | --------: | :---------------------- | :---------- |
| \-84.04267 |  702.5207 |  786.5633 | \-0.3722537 | 0.7365449 |  2.746882 | \-841.4809 |  673.3956 | Welch Two Sample t-test | two.sided   |

``` r
data_t_2<-read.csv(here::here("data", "before_after.csv"))
t_NM<-t.test(data_t_2$NM~data_t_2$GROUP_2012)
knitr::kable(tidy(t_NM))
```

|   estimate | estimate1 | estimate2 |  statistic | p.value | parameter |  conf.low | conf.high | method                  | alternative |
| ---------: | --------: | --------: | ---------: | ------: | --------: | --------: | --------: | :---------------------- | :---------- |
| \-226.6254 |  140.1146 |    366.74 | \-1.178218 | 0.29879 |  4.375457 | \-743.067 |  289.8163 | Welch Two Sample t-test | two.sided   |

``` r
data_t_2<-read.csv(here::here("data", "before_after.csv"))
t_Boreal<-t.test(data_t_2$Boreal~data_t_2$GROUP_2012)
knitr::kable(tidy(t_Boreal))
```

|   estimate | estimate1 | estimate2 |   statistic |   p.value | parameter |   conf.low | conf.high | method                  | alternative |
| ---------: | --------: | --------: | ----------: | --------: | --------: | ---------: | --------: | :---------------------- | :---------- |
| \-226.1317 |  854.6623 |  1080.794 | \-0.5038714 | 0.6309027 |  6.528823 | \-1303.072 |  850.8091 | Welch Two Sample t-test | two.sided   |

## Linear features

We quantified the length and density of linear features (LFs) within
caribou ranges in Alberta using the Wall-to-Wall Human Footprint
Inventory (ABMI 2018), available at:
<https://www.abmi.ca/home/data-analytics/da-top/da-product-overview/Human-Footprint-Products/HF-inventory.html>

For British Columbia, we compiled pipeline, seismic line, and road data
from the British Columbia’s Oil and Gas Commission datasets (BCOGC
2020). Additional road data were compiled from the BC Digital Road Atlas
(DRA; 1979–2018; Government of British Columbia 2020) and forestry roads
provincial datasets (1996-2018; Government of British Columbia 2020b).
Road data without year information were only incorporated to estimate
the total length and density of LFs in 2018 but were not used to
estimate yearly values for British Columbia. We augmented this dataset
to incorporate linear features created before 1996 using data from the
Government of British Columbia (TRIM Miscellaneous Lines; Government of
British Columbia 2020c)."

BCOGC. 2020. BC Oil and Gas Commission Open Data Portal
<https://data-bcogc.opendata.arcgis.com/>

Government of British Columbia. 2020a. Digital Road Atlas:
<https://www2.gov.bc.ca/gov/content/data/geographic-data-services/topographic-data/roads>

Government of British Columbia. 2020b. Forest Tenure Road Section Lines:
<https://catalogue.data.gov.bc.ca/dataset/forest-tenure-road-section-lines>

Government of British Columbia. 2020c. TRIM Miscellaneous Lines:
<https://catalogue.data.gov.bc.ca/dataset/trim-miscellaneous-lines>

Calculate linear feature creation rate in BC:

``` r
data_raw_BC<-read.csv(here::here("data","data_LF_BC.csv"))    
ecotype<-read.csv(here::here("data", "herd_ecotype.csv"))

#Standardizing by range size (density: m/km2)  
data_raw_BC$Pipeline_den<-data_raw_BC$cum_pipeline/data_raw_BC$Range_area
data_raw_BC$Roads_den<-data_raw_BC$cum_roads/data_raw_BC$Range_area
data_raw_BC$Seismic_den<-data_raw_BC$cum_seismic/data_raw_BC$Range_area
data_raw_BC$Total_den<-data_raw_BC$cum_total/data_raw_BC$Range_area

colnames_rate<-data_raw_BC[7:ncol(data_raw_BC)] %>% colnames()
rates_list<-list()
#lag function to create annual rates
for(i in colnames_rate){
  selection<-data_raw_BC %>% dplyr::select(i,Year,Range)  #selects columns of interest, year and range 
  selection$variable<-selection[,1] #creates column with same name for all variables
  selection<-selection %>% group_by(Range) %>% #groups ranges to apply lag function
  mutate(rate=(variable-lag(variable,default=first(variable),order_by = Year))) 
  
  rates_list[[i]]<-selection
}

rates_new_list<-lapply(rates_list,FUN=function(x) x %>% ungroup() %>% dplyr::select(rate))
rates<-do.call(cbind,rates_new_list) 
colnames(rates)<-paste(colnames_rate,"rate",sep="_") #creating column names
rates$Range<-rates_list$Roads_den$Range 
rates$Year<-cbind(rates_list[[1]]$Year)
rates_BC_1<- left_join(rates,ecotype, by="Range")#Combining new rates dataset with ecotype dataset

#Selecting years of interest 1996-2018
rates_BC_2<-rates_BC_1%>%
  filter(rates_BC_1$Year>1996)#1996 is used to generate rate of 1997 (i.e., LF change from 1996 to 1997 was assigned "1997". 1996 is "null" due to no data in 1995)
rates_BC<-rates_BC_2%>%
  filter(rates_BC_2$Year<2019)#removing rates from 2019 to 2020 due to the lack of data for some LF types in those years

#Lumping SM_SG, SM_GC and SM_NG into SM
rates_BC$Ecotype2<-rates_BC$Ecotype %>% as.character()
rates_BC$Ecotype2[grepl("SM",rates_BC$Ecotype2)]<-"SM" 

data_raw_BC$Ecotype2<-data_raw_BC$Ecotype %>% as.character()
data_raw_BC$Ecotype2[grepl("SM",data_raw_BC$Ecotype2)]<-"SM"

#Stats (Table 2)
rates_BC$Total_rate_km<-rates_BC$cum_total_rate/1000 #Transforming m data into km
rates_BC$Seismic_rate_km<-rates_BC$cum_seismic_rate/1000 #Transforming m data into km

##Calculate mean
results_TotalBC<-dplyr::select(rates_BC, Ecotype2, Total_den_rate, Seismic_den_rate, Total_rate_km, Seismic_rate_km) %>% 
  group_by(Ecotype2) %>%
  summarise_all(funs(mean, sd), na.rm = TRUE)
```

Calculate linear feature creation rate in AB:

``` r
#Load creation rate by ecotype for Alberta:
YearlyCreation<-read.csv(here::here("data", "LF_length_AB_complete_Nov4v2.csv"))

#Calculate yearly rates for each range, then average the yearly rate per range
data_length_AB<-dplyr::select(YearlyCreation, Ecotype, Range, Year, TOTAL, Seismic_all, Total_den, Seismic_all_den)
colnames_rate<-data_length_AB[4:ncol(data_length_AB)] %>% colnames()

##For Boreal caribou ranges
rates_list<-list()
for(i in colnames_rate){
  selection<-subset(data_length_AB, Ecotype=="B")
  selection<-dplyr::select(selection, i, Year, Range)  #this selects columns of interest, year and range 
  selection$variable<-selection[,1] #create column with same name for all variables
  selection<-selection %>% dplyr::group_by(Range) %>% #group ranges to apply lag function
    mutate(rate=(variable-lag(variable,default=first(variable),order_by = Year))/(Year-lag(Year))) #change divider according to years in study period
  rates_list[[i]]<-selection
}

rates_new_list<-lapply(rates_list,FUN=function(x) x %>% ungroup() %>% dplyr::select(rate))
crearates<-do.call(cbind,rates_new_list)
colnames(crearates)<-paste(colnames_rate,"rate",sep="_") #creating column names
crearates$Year<-cbind(rates_list[[1]]$Year)

##Calculate mean
results_TotalAB_Boreal<-subset(crearates, Year > 2010) %>% 
  summarise_all(funs(mean, sd), na.rm = TRUE)

crearates$Range<-rates_list$TOTAL$Range 
crearatesB<-crearates

##For Southern Mountain caribou ranges
rates_list<-list()
for(i in colnames_rate){
  selection<-subset(data_length_AB, Ecotype=="SM_CG") %>% dplyr::select(i,Year, Range)  #this selects columns of interest, year and range 
  selection$variable<-selection[,1] #create column with same name for all variables
  selection<-selection %>% dplyr::group_by(Range) %>% #group ranges to apply lag function
    mutate(rate=(variable-lag(variable,default=first(variable),order_by = Year))/(Year-lag(Year))) #change divider according to years in study period
  rates_list[[i]]<-selection
}

rates_new_list<-lapply(rates_list,FUN=function(x) x %>% ungroup() %>% dplyr::select(rate))
crearates<-do.call(cbind,rates_new_list)
colnames(crearates)<-paste(colnames_rate,"rate",sep="_") #creating column names
crearates$Year<-cbind(rates_list[[1]]$Year)

##Calculate mean
results_TotalAB_SMC<-subset(crearates, Year >2010) %>% 
  summarise_all(funs(mean, sd), na.rm = TRUE)

crearates$Range<-rates_list$TOTAL$Range 
crearatesSM<-crearates

results_TotalAB_Boreal$Ecotype<-"B"
results_TotalAB_SMC$Ecotype<-"SM"
results_TotalAB<-rbind(results_TotalAB_Boreal, results_TotalAB_SMC)
```

Merge Alberta and BC and plot:

``` r
##Convert AB lengths to km to be consistent with BC
results_TotalAB$TOTAL_rate_mean<-results_TotalAB$TOTAL_rate_mean/1000
results_TotalAB$Seismic_all_rate_mean<-results_TotalAB$Seismic_all_rate_mean/1000
results_TotalAB$TOTAL_rate_sd<-results_TotalAB$TOTAL_rate_sd/1000
results_TotalAB$Seismic_all_rate_sd<-results_TotalAB$Seismic_all_rate_sd/1000

##Format for joining
colnames(results_TotalAB)<- c("Total_rate_km_mean","Seismic_rate_km_mean", "Total_den_rate_mean", "Seismic_den_rate_mean","YearMean", "Total_rate_km_sd","Seismic_rate_km_sd", "Total_den_rate_sd", "Seismic_den_rate_sd", "YearSD", "Ecotype")
colnames(results_TotalBC)[which(names(results_TotalBC) == "Ecotype2")] <- "Ecotype"
##Remove year columns from AB. 
results_TotalAB$YearMean<-NULL
results_TotalAB$YearSD<-NULL
results_TotalAB$Province<-"AB"
results_TotalBC$Province<-"BC"

##Join and print
results_TotalABBC<-rbind(results_TotalAB, results_TotalBC)
```

Tab. 2 Annual rate of creation of all linear features (LFs) and seismic
lines within caribou ranges in Alberta (AB) and British Columbia (BC).
Mean rate and standard deviation (in brackets) were measured as annual
increase in LF (values in year t+1 – values in year t).

``` r
knitr::kable(results_TotalABBC)
```

| Total\_rate\_km\_mean | Seismic\_rate\_km\_mean | Total\_den\_rate\_mean | Seismic\_den\_rate\_mean | Total\_rate\_km\_sd | Seismic\_rate\_km\_sd | Total\_den\_rate\_sd | Seismic\_den\_rate\_sd | Ecotype | Province |
| --------------------: | ----------------------: | ---------------------: | -----------------------: | ------------------: | --------------------: | -------------------: | ---------------------: | :------ | :------- |
|             542.52883 |               454.35321 |              62.384497 |               51.0890026 |           1435.2600 |            1409.43176 |           171.863979 |             165.865057 | B       | AB       |
|              39.42150 |                11.17867 |               3.453630 |                0.9056063 |            106.7604 |              29.14393 |             8.633547 |               2.330461 | SM      | AB       |
|            1253.90025 |               956.27744 |             152.538819 |              115.3446923 |           2786.5018 |            2532.61740 |           278.493624 |             255.206948 | B       | BC       |
|              95.64913 |                49.97783 |               8.943364 |                4.1938412 |            384.3873 |             310.92543 |            35.753282 |              28.866869 | NM      | BC       |
|             183.71564 |                68.21198 |              47.530219 |               12.8400306 |            667.5775 |             487.98836 |           148.999201 |              87.529023 | SM      | BC       |

``` r
BC_plot <- data_raw_BC %>%
  filter(Year<2019) %>%
  ggplot(aes(x=Year,y=Total_den/1000,group=Range))+
  geom_line()+ #connects the observations in the order in which they appear in the data
  theme_classic()+ #removes grey background
  theme(text=element_text(size=12))+  #Changes font size
  (ggtitle("British Columbia"))+ #adds title to graph
  facet_wrap(.~Ecotype2,scales = "free_y",labeller=as_labeller(c(`B`="Boreal",`NM`="Northern Mountain",`SM`="Southern Mountain"))) + 
  ylab(expression(paste("LFs (km/km"^2,")")))

crearatesB$Ecotype2<-"B"
crearatesSM$Ecotype2<-"SM"
crearatesBSM<-rbind(crearatesB, crearatesSM)

AB_plot  <-
  YearlyCreation %>%
  ggplot(aes(x=Year,y=Total_den/1000,group=Range))+
  geom_line()+ 
  theme_classic()+ 
  theme(text=element_text(size=12))+
  (ggtitle("Alberta"))+ 
  facet_wrap(.~Ecotype,scales = "free_y",labeller=as_labeller(c(`B`="Boreal",`SM_CG`="Southern Mountain"))) +
  ylab(expression(paste("LFs (km/km"^2,")")))

plot_grid(AB_plot,BC_plot, ncol=1)
```

![](HabitatGainLoss_MainAnalyses_files/figure-gfm/unnamed-chunk-19-1.png)<!-- -->

Then, calculate linear feature recovery rate

Regrowth speed is calculated as the Z-dimension from LiDAR (Veg\_height)
divided by age-since-line-creation (Seismic Age). LiDar data are
proprietary and are not shareable here.

In the “data” folder, “recovery\_data\_Jan18.csv” summarizes the mean
vegetation height (Veg\_height) along each Least Cost Path (LCP)
connecting linear feature segments. Also included is the type of linear
feature and year seismic was created (from ABMI HFI), the caribou range
and subpopulation, Ecosite (Ducks Unlimited Canada 2016), the LCP
length, seismic line segment length (LCP\_Sec\_Length) and the year in
which the LiDar was aqcuired.

``` r
segments = read.csv(here::here("data", "recovery_data_Jan18.csv"))

# First, we need to make any segments with 0 veg height a very small number that is not zero
segments$Veg_height[segments$Veg_height==0] <- 0.0001

# We also want only positive values for Seismic age, calculated as Lidar yr - age of original cut. 
# But there are some negative valuess here (age of orig cut after LiDAR)
segments.old <- segments[segments$Seismic_age>=0,]  # subset to only lines before lidar
segments.old$Seismic_age[segments.old$Seismic_age==0] <- 0.0001  #we need to make age of 0 a very small number that is not zero

# Now re-calculate growth rates, using reduced data set that exclused seismic cut post-LiDar aquisition
segments.old$Regrowth_speed <- segments.old$Veg_height/segments.old$Seismic_age

# Truncate growthrates over 1 m/yr as 1 m
segments.old$Regrowth_speed[segments.old$Regrowth_speed>1.0] <- 1.0 #Note, this isn't needed because of the next step.

# Remove the impossibly high growth rates, defined as 0.50m/year
segments.old2 <- segments.old[segments.old$Regrowth_speed<0.50,]

ggplot(subset(segments.old2, Ecosite != "NA"), aes(x=Ecosite, y=Regrowth_speed, fill=Ecosite)) + 
  geom_violin() + 
  scale_fill_brewer(palette="Dark2")+
  ylab("Regrowth rate (m / year)")+
  theme_bw() +
  theme(panel.grid.minor=element_blank(), panel.grid.major=element_blank())+
  theme(legend.position = "none")
```

![](HabitatGainLoss_MainAnalyses_files/figure-gfm/unnamed-chunk-20-1.png)<!-- -->

Calculate the rate of recovery per year using each line’s empirical
growth rate if it has one. For newer seismic lines (cut post LiDar), use
the ecosite’s empirical distribution of growth rates

``` r
## Find distrubution of growth rates by ecosite
growth.bog = segments.old2$Regrowth_speed[segments.old2$Ecosite=="Bog"]
growth.poorfen = segments.old2$Regrowth_speed[segments.old2$Ecosite=="Poor fen"]
growth.richfen = segments.old2$Regrowth_speed[segments.old2$Ecosite=="Rich fen"]
growth.upland = segments.old2$Regrowth_speed[segments.old2$Ecosite=="Upland"]
growth.wet = segments.old2$Regrowth_speed[segments.old2$Ecosite=="Wet"]
growth.other = segments.old2$Regrowth_speed[segments.old2$Ecosite=="Other"]
growth.all = segments.old2$Regrowth_speed

tab.growth = data.frame(
Ecosite = c("Bog", "Poor fen", "Rich fen", "Upland", "Wet", "Other", "All"),
Growth.med = c(median(growth.bog, na.rm=T),
            median(growth.poorfen, na.rm=T),
            median(growth.richfen, na.rm=T),
            median(growth.upland, na.rm=T),
            median(growth.wet, na.rm=T),
            median(growth.other, na.rm=T),
            median(growth.all, na.rm=T)),
Growth.mean = c(mean(growth.bog, na.rm=T),
            mean(growth.poorfen, na.rm=T),
            mean(growth.richfen, na.rm=T),
            mean(growth.upland, na.rm=T),
            mean(growth.wet, na.rm=T),
            mean(growth.other, na.rm=T),
            mean(growth.all, na.rm=T)),
sd = c(sd(growth.bog, na.rm=T),
            sd(growth.poorfen, na.rm=T),
            sd(growth.richfen, na.rm=T),
            sd(growth.upland, na.rm=T),
            sd(growth.wet, na.rm=T),
            sd(growth.other, na.rm=T),
            sd(growth.all, na.rm=T)),
lcl = c(quantile(growth.bog, probs = 0.025, na.rm=T),
            quantile(growth.poorfen, probs = 0.025, na.rm=T),
            quantile(growth.richfen, probs = 0.025, na.rm=T),
            quantile(growth.upland, probs = 0.025, na.rm=T),
            quantile(growth.wet, probs = 0.025, na.rm=T),
            quantile(growth.other, probs = 0.025, na.rm=T),
            quantile(growth.all, probs = 0.025, na.rm=T)),
ucl = c(quantile(growth.bog, probs = 0.975, na.rm=T),
            quantile(growth.poorfen, probs = 0.975, na.rm=T),
            quantile(growth.richfen, probs = 0.975, na.rm=T),
            quantile(growth.upland, probs = 0.975, na.rm=T),
            quantile(growth.wet, probs = 0.975, na.rm=T),
            quantile(growth.other, probs =0.975, na.rm=T),
            quantile(growth.all, probs = 0.975, na.rm=T)))
```

Estimate the year each segment should achieve different height
thresholds, based on growth rates

``` r
segments$recov.yr.92 = round(segments$Seismic_year + (0.92/segments$Regrowth_speed)) ## "Partial Recovery"
segments$recov.yr.6.67 = round(segments$Seismic_year + (6.67/segments$Regrowth_speed)) ## "Full Recovery"

### First, a loop to correct growth rates for all LCP segments
for(i in 1:nrow(segments)){             # assign growth rates to "new" features from ecosite median
    ecosite.i = segments$Ecosite[i]     # Ecosite
    if(is.na(ecosite.i)==T){# If no Ecosite, use pooled median growth rate
      ecosite.i = "All"
      }
    growth.rate.i = tab.growth$Growth.med[tab.growth$Ecosite%in%ecosite.i]  # if ecosite, use ecosite growth rate
    # Now assign ecosite median growth rate if no appropriate site-specific growth rate
    # That is, if LCP age is younger than Lidar, or if site growth rate is crazy
    # or if Regrowth_speed is NA
    if(segments$Regrowth_speed[i]<0 || segments$Regrowth_speed[i]>0.5 || is.na(segments$Regrowth_speed[i])){
      segments$Regrowth_speed[i] = growth.rate.i
    } else { #If the above statement are not met, keep the growth rate as-is
      segments$Regrowth_speed[i] = segments$Regrowth_speed[i]
      }
    # Re-calculate year in which recovery will occur, using (maybe) new growth rates
    segments$recov.yr.92[i] = round(segments$Seismic_year[i] + (0.92/segments$Regrowth_speed[i]))
    segments$recov.yr.6.67[i] = round(segments$Seismic_year[i] + (6.67/segments$Regrowth_speed[i]))
}

#Next, loop through all years to calculate percent length recovered in each year in each range

##Range specific recovery rates:
rangeyear<-expand.grid(seq(from=min(segments$Seismic_year), to=max(segments$Seismic_year), by=1), unique(segments$Range))
rangeyear<-unique(paste(rangeyear$Var1,rangeyear$Var2))

for(t in 1:length(rangeyear)){
  y.r = rangeyear[t]
  yr.t = as.numeric(substr(rangeyear[t], start = 1, stop = 4))
  r.t =str_sub(y.r, 6)
  seg.92 = sum(subset(segments,recov.yr.92==yr.t & Range == r.t)$LCP_Sec_Length)        # legth of segments tall enough?
  seg.92.cum = sum(subset(segments,recov.yr.92<=yr.t& Range == r.t)$LCP_Sec_Length)# cumulatively tall enough?
  perc.92 = 100*seg.92/sum(subset(segments,Range == r.t)$LCP_Sec_Length)                            # percentage tall enough in t?
  perc.92.cum = 100*seg.92.cum/sum(subset(segments,Range == r.t)$LCP_Sec_Length)                    # cum. % tall enough by t?
  seg.6.67 = sum(subset(segments,recov.yr.6.67==yr.t& Range == r.t)$LCP_Sec_Length)     # how many segments tall enough?
  seg.6.67.cum = sum(subset(segments,recov.yr.6.67<=yr.t& Range == r.t)$LCP_Sec_Length) # cumulatively tall enough?
  perc.6.67 = 100*seg.6.67/sum(subset(segments,Range == r.t)$LCP_Sec_Length)
  perc.6.67.cum = 100*seg.6.67.cum/sum(subset(segments,Range == r.t)$LCP_Sec_Length)
  tab.rec.t = data.frame(Yearrange = y.r, Year = yr.t, Range = r.t, seg.92=seg.92, seg.92.cum=seg.92.cum, perc.92=perc.92, perc.92.cum = perc.92.cum,seg.6.67=seg.6.67, seg.6.67.cum=seg.6.67.cum, perc.6.67 = perc.6.67, perc.6.67.cum = perc.6.67.cum)
  
  if(t==1){tab.rec = tab.rec.t}else{tab.rec = rbind(tab.rec, tab.rec.t)}
  
}
```

Plot recovery rates over time by range

``` r
ggplot(data=tab.rec, aes(x=as.numeric(Year), y=(seg.92.cum/1000), fill=Range)) +
  geom_line()+
  theme_bw() +
  xlab("Year") +
  ylab("Length > 0.92 m (km)") +
  theme(axis.text.x = element_text(size=12), axis.title = element_text(size=14) ) +
  theme(axis.text.y = element_text(size=12)) +
  theme(panel.grid.minor=element_blank(), panel.grid.major=element_blank()) +
  facet_wrap(~Range)
```

![](HabitatGainLoss_MainAnalyses_files/figure-gfm/unnamed-chunk-23-1.png)<!-- -->

Plot cumulative recovery rates over time by range

``` r
ggplot(data=tab.rec, aes(x=as.numeric(Year), y=(seg.6.67.cum/1000), fill=Range)) +
  geom_line()+
  theme_bw() +
  xlab("Year") +
  ylab("Length > 6.67 m (km)") +
  theme(axis.text.x = element_text(size=12), axis.title = element_text(size=14) ) +
  theme(axis.text.y = element_text(size=12)) +
  theme(panel.grid.minor=element_blank(), panel.grid.major=element_blank()) +
  facet_wrap(~Range)
```

![](HabitatGainLoss_MainAnalyses_files/figure-gfm/unnamed-chunk-24-1.png)<!-- -->

Finally, compare yearly creation rate, regeneration rate, and net change
for both linear feature length and density. Use the annual estimated
percent recovery rates, LF creation lenght(km), and density (km/km2)
from above. Rename annual creation (tab.rec) YearlyRecover above.

``` r
YearlyRecover<-tab.rec
YearlyCreation_AB_Boreal<-subset(YearlyCreation, Ecotype == "B")
##Correct "Chinchaga_AB" to "Chinchaga"
YearlyCreation_AB_Boreal$Range<-as.character(YearlyCreation_AB_Boreal$Range)
YearlyCreation_AB_Boreal$Range[YearlyCreation_AB_Boreal$Range == "Chinchaga_AB"] <- "Chinchaga"
YearlyCreation_AB_Boreal$Range<-as.factor(YearlyCreation_AB_Boreal$Range)
YearlyCreation_AB_Boreal$rangeyear<-paste(YearlyCreation_AB_Boreal$Year,YearlyCreation_AB_Boreal$Range)

##Subset both datasets to when creation and recovery data are available
##2010 to 2018 (limited by creation data)
YearlyRecSub<-subset(YearlyRecover, Year >2009 & Year <2019)

##Assign perc cummulative estimated km recovered to each range/year by year
YearlyCreation_AB_Boreal$perc.92<-YearlyRecSub$perc.92[match(YearlyCreation_AB_Boreal$rangeyear, YearlyRecSub$Yearrange)]
YearlyCreation_AB_Boreal$perc.92.cum<-YearlyRecSub$perc.92.cum[match(YearlyCreation_AB_Boreal$rangeyear, YearlyRecSub$Yearrange)]
YearlyCreation_AB_Boreal$perc.6.67<-YearlyRecSub$perc.6.67[match(YearlyCreation_AB_Boreal$rangeyear, YearlyRecSub$Yearrange)]
YearlyCreation_AB_Boreal$perc.6.67.cum<-YearlyRecSub$perc.6.67.cum[match(YearlyCreation_AB_Boreal$rangeyear, YearlyRecSub$Yearrange)]

##Convert m to Km
YearlyCreation_AB_Boreal$TOTALKM<-YearlyCreation_AB_Boreal$TOTAL/1000
YearlyCreation_AB_Boreal$Seismic_all_KM<-YearlyCreation_AB_Boreal$Seismic_all/1000

##Calculate estimated length regenerated to 0.92m and 6.67m
YearlyCreation_AB_Boreal$RegenLength92<-YearlyCreation_AB_Boreal$Seismic_all_KM*(YearlyCreation_AB_Boreal$perc.92.cum/100)
YearlyCreation_AB_Boreal$RegenLength667<-YearlyCreation_AB_Boreal$Seismic_all_KM*(YearlyCreation_AB_Boreal$perc.6.67.cum/100)

##Calculate estimated net length accounting for each regeneration target
YearlyCreation_AB_Boreal$NetLength92<-(YearlyCreation_AB_Boreal$Seismic_all_KM*(YearlyCreation_AB_Boreal$perc.92.cum/100))-YearlyCreation_AB_Boreal$TOTALKM
YearlyCreation_AB_Boreal$NetLength667<-(YearlyCreation_AB_Boreal$Seismic_all_KM*(YearlyCreation_AB_Boreal$perc.6.67.cum/100))-YearlyCreation_AB_Boreal$TOTALKM
```

``` r
##Calculating rate of creation for these ranges and compare to regeneration rates:
data_length_AB<-dplyr::select(YearlyCreation_AB_Boreal, Range, Year, TOTALKM, RegenLength92, RegenLength667, NetLength92, NetLength667)
colnames_rate<-data_length_AB[3:ncol(data_length_AB)] %>% colnames()

rates_list<-list()
for(i in colnames_rate){
  selection<-data_length_AB %>% dplyr::select(i,Year,Range)  #this selects columns of interest, year and range 
  selection$variable<-selection[,1] #create column with same name for all variables
  selection<-selection %>% group_by(Range) %>% #group ranges to apply lag function
    #mutate(rate=(variable-lag(variable,default=first(variable),order_by = Year))) #change divider according to # years in study period
    mutate(rate=(variable-lag(variable,default=first(variable),order_by = Year))/(Year-lag(Year))) #change divider according to # years in study period
  rates_list[[i]]<-selection
}

rates_new_list<-lapply(rates_list,FUN=function(x) x %>% ungroup() %>% dplyr::select(rate))
rates<-do.call(cbind,rates_new_list)
colnames(rates)<-paste(colnames_rate,"rate",sep="_") #crating column names
rates$Year<-cbind(rates_list[[1]]$Year)
rates<-cbind(rates, rates_list[[1]]$Range)
names(rates)[names(rates) == "rates_list[[1]]$Range"] <- "Range"

meanrates<-aggregate(subset(rates, Year >2010)[, 1:5], list(subset(rates, Year >2010)$Range), mean)##2010 doesn't have a rate because there are no data prior (no t-1)
colnames(meanrates)<- c("Range","MeanKmCreateRate","MeanKmPartialRegenRate", "MeanKmFullRegenRate", "MeanNetKmPartialRegenRate", "MeanNetKmFullRegenRate")

sdrates<-aggregate(subset(rates, Year >2010)[, 1:5], list(subset(rates, Year >2010)$Range), sd)##2010 doesn't have a rate because there are no data prior (no t-1)
colnames(sdrates)<- c("Range","SDKmCreateRate","SDKmPartialRegenRate", "SDKmFullRegenRate", "SDNetKmPartialRegenRate", "SDNetKmFullRegenRate")

##Remove ranges with low LiDar coverage - Bischto, Caribou Mountains, Yates:
meanratessub<-subset(meanrates, Range != "Bischto" & Range != "Caribou Mountains" & Range != "Yates")
sdratessub<-subset(sdrates, Range != "Bischto" & Range != "Caribou Mountains" & Range != "Yates")
meanratessub<-meanratessub %>% mutate_if(is.numeric, round)
sdratessub<-sdratessub %>% mutate_if(is.numeric, round)
```

``` r
#Bring in range area to calculate density
meanratessub$RangeArea<-YearlyCreation_AB_Boreal$Range_area[match(meanratessub$Range, YearlyCreation_AB_Boreal$Range)]
meanratessub$MeanDensCreateRate<-meanratessub$MeanKmCreateRate*1000/meanratessub$RangeArea
meanratessub$MeanDensPartialRegenRate<-meanratessub$MeanKmPartialRegenRate*1000/meanratessub$RangeArea
meanratessub$MeanDensFullRegenRate<-meanratessub$MeanKmFullRegenRate*1000/meanratessub$RangeArea
meanratessub$MeanNetDensPartialRegenRate<-meanratessub$MeanNetKmPartialRegenRate*1000/meanratessub$RangeArea
meanratessub$MeanNetDensFullRegenRate<-meanratessub$MeanNetKmFullRegenRate*1000/meanratessub$RangeArea

meanratessub<-meanratessub %>% mutate_if(is.numeric, round,2)

#Add in final regeneration % from 2018
FinalRegen<-subset(YearlyCreation_AB_Boreal, Year == 2018)
meanratessub$PercPartialRegen<-FinalRegen$perc.92.cum[match(meanratessub$Range, FinalRegen$Range)]
meanratessub$PercFullRegen<-FinalRegen$perc.6.67.cum[match(meanratessub$Range, FinalRegen$Range)]
meanratessub<-meanratessub %>% mutate_if(is.numeric, round)
```

Table 3 Mean annual rate of linear feature (LF) creation, regeneration
of LFs, and net LF change in length (km) and density (km/km2) in boreal
caribou ranges in Alberta from 2010–2018. The percent length of seismic
lines that were estimated to have met partial and full regeneration
targets as of 2018 are also provided for each caribou range (Regenerated
(%)). Regeneration was defined using both a partial recovery threshold
(0.92 m) and full recovery threshold (6.67 m).

``` r
knitr::kable(meanratessub)
```

| Range               | MeanKmCreateRate | MeanKmPartialRegenRate | MeanKmFullRegenRate | MeanNetKmPartialRegenRate | MeanNetKmFullRegenRate | RangeArea | MeanDensCreateRate | MeanDensPartialRegenRate | MeanDensFullRegenRate | MeanNetDensPartialRegenRate | MeanNetDensFullRegenRate | PercPartialRegen | PercFullRegen |
| :------------------ | ---------------: | ---------------------: | ------------------: | ------------------------: | ---------------------: | --------: | -----------------: | -----------------------: | --------------------: | --------------------------: | -----------------------: | ---------------: | ------------: |
| Chinchaga           |              321 |                    822 |                 524 |                       502 |                    203 |     17644 |                 18 |                       47 |                    30 |                          28 |                       12 |               80 |             7 |
| Cold Lake           |             2080 |                   1138 |                 104 |                     \-942 |                 \-1976 |      6726 |                309 |                      169 |                    15 |                       \-140 |                    \-294 |               48 |             3 |
| East Side Athabasca |             2234 |                   1270 |                 164 |                     \-963 |                 \-2069 |     13119 |                170 |                       97 |                    12 |                        \-73 |                    \-158 |               36 |             2 |
| Little Smoky        |              217 |                    161 |                 104 |                      \-57 |                  \-113 |      3084 |                 70 |                       52 |                    34 |                        \-18 |                     \-37 |               89 |             9 |
| Nipisi              |               93 |                     83 |                  39 |                      \-10 |                   \-54 |      2104 |                 44 |                       39 |                    19 |                         \-5 |                     \-26 |               66 |            10 |
| Red Earth           |              509 |                    639 |                 392 |                       130 |                  \-117 |     24702 |                 21 |                       26 |                    16 |                           5 |                      \-5 |               52 |             6 |
| Richardson          |              159 |                    142 |                   6 |                      \-17 |                  \-154 |      7074 |                 22 |                       20 |                     1 |                         \-2 |                     \-22 |               52 |             1 |
| Slave Lake          |               59 |                     81 |                  14 |                        23 |                   \-44 |      1516 |                 39 |                       53 |                     9 |                          15 |                     \-29 |               62 |             6 |
| West Side Athabasca |              640 |                    533 |                 135 |                     \-107 |                  \-505 |     15707 |                 41 |                       34 |                     9 |                         \-7 |                     \-32 |               48 |             4 |
