
---
title: "All Plant iNaturalist Observations Connecticut"
author: "Steven DiFalco"
date: "9/5/2019"
output: html_document
---
###This document shows how iNaturalist data can be used and some simple code to create map outputs. The data here is all plant observations for the state of Connecticut between 2008-2018, obtained from the iNaturalist website (https://www.inaturalist.org/observations/export). Only research grade, non-cultivated, observations were exported. Some of the code in the following example are borrowed from another tutorial using iNaturalist data (https://github.com/micahfreedman/monarch_map_gif)
```{r message=FALSE, warning=FALSE, include=TRUE, paged.print=FALSE}
library(plyr); library(ggplot2); library(tidyverse)
```
```{r setup, include=TRUE}
all_ct <- read.csv("observations-60137.csv")
all_ct[1:5,32:34]
```
<br>

####Looking at the raw data, there are a lot of fields that may not apply depending on your goal. The next section will help clean up the data and keep only the columns I'd like to focus on
```{r echo=TRUE}
columns.keep <- c('id','observed_on','scientific_name', 'user_login','common_name', 'taxon_id','quality_grade','place_guess','latitude','longitude','url')
all_ct <- all_ct[,columns.keep]
head(all_ct)
```

####Get date information into the proper format. This is a bit of a pain to do, but thanks to Micah Freedman's script, I was able to get it done easier. In this example, I will not use day or month, but kept the code to separate out the date just so it was available.
```{r include=TRUE}
#create a new column for Julian day of observation
all_ct$julian.date <- format(as.Date(all_ct$observed_on), "%j") 

#convert this to a continuous numeric variable
all_ct$julian.date <- as.numeric(all_ct$julian.date) 

#create a new column for month of observation
all_ct$month <- format(as.Date(all_ct$observed_on), "%m")

#assign names for months (revalue function requires plyr library)
all_ct$month <- revalue(all_ct$month, c('01' = 'January', '02' = 'February', '03' = 'March', '04' = 'April', '05' = 'May', '06' = 'June', '07' = 'July', '08' = 'August', '09' = 'September', '10' = 'October', '11' = 'November', '12' = 'December')) 

#rearrange so months are in chronological order
all_ct$month <- factor(all_ct$month, levels = c('January','February','March','April','May','June','July','August','September','October','November','December')) 

#adds year of observation to another column 
all_ct$year <- format(as.Date(all_ct$observed_on), "%Y")
```
<br>

####I was interested to identify whether species observed were invasive or native. To do that, I downloaded a vascular plant list with that information, extracted 'origin' category, and combined this with the inat observations.
```{r echo=TRUE}
#vascular plant list for state of CT (https://sites.google.com/a/conncoll.edu/vascular-plants-of-connecticut-checklist/)
CT_Flora_Checklist <- read.csv("CT Flora Checklist 8-1-14.csv")

#keep certain columns from dataset
column_keep <- c('Current.Scienticfic.Name','Common.Name.s.', 'Origin')
CT_Flora_Checklist <- CT_Flora_Checklist[,column_keep]

#get rid of the extraneous things at end of scientific name
CT_Flora_Checklist_sep <- separate(CT_Flora_Checklist, Current.Scienticfic.Name, into = c('genus', 'species'),sep = " ")

#re-combine genus and species in neat form
CT_Flora_Checklist_new <- unite(CT_Flora_Checklist_sep, 'scientific_name', c('genus', 'species'), sep= " ")

#Count inat observations by scientific name
all_ct_scientific <- all_ct %>%
  group_by(scientific_name)%>%
  dplyr::summarise(n=n())%>%  
  filter(n>5) 

##join dataset from all obs and CT Flora to get which ones are invasive/native.
combined_datasets <- dplyr::inner_join(all_ct_scientific, CT_Flora_Checklist_new, by= "scientific_name")
write.csv(combined_datasets, file = "inat_combined.csv")

combined_data_highest <- combined_datasets %>%
  filter(n>100) %>%
  arrange(desc(n))

combined_data_highest$Origin <- revalue(combined_data_highest$Origin, c('E' = 'Invasive', 'I' = 'Invasive', 'N' = 'Native'))

write.csv(combined_data_highest, file = "inat_top30.csv")
```