# Applications

Some _significant_ applications are demonstrated in this chapter.

## Salmon Ocean Ecology Data



### Intro

One of the goals of the Hakai Institute and the Canadian Integrated Ocean Observing System (CIOOS) is to facilitate Open Science and FAIR (findable, accessible, interoperable, reusable) ecological and oceanographic data. In a concerted effort to adopt or establish how best to do that, several Hakai and CIOOS staff attended an International Ocean Observing System (IOOS) Code Sprint in Ann Arbour, Michigan between October 7--11, 2019, to discuss how to implement FAIR data principles for biological data collected in the marine environment. 

The [Darwin Core](https://dwc.tdwg.org) is a highly structured data format that standardizes data table relations, vocabularies, and defines field names. The Darwin Core defines three table types: `event`, `occurrence`, and `measurementOrFact`. This intuitively captures the way most ecologists conduct their research. Typically, a survey (event) is conducted and measurements, counts, or observations (collectively measurementOrFacts) are made regarding a specific habitat or species (occurrence). 

In the following script I demonstrate how I go about converting a subset of the data collected from the Hakai Institute Juvenile Salmon Program and discuss challenges, solutions, pros and cons, and when and what's worthwhile to convert to Darwin Core.

The conversion of a dataset to Darwin Core is much easier if your data are already tidy (normalized) in which you represent your data in separate tables that reflect the hierarchical and related nature of your observations. If your data are not already in a consistent and structured format, the conversion would likely be very arduous and not intuitive.

### event 

The first step is to consider what you will define as an event in your data set. I defined the capture of fish using a purse seine net as the `event`. Therefore, each row in the `event` table is one deployment of a seine net and is assigned a unique `eventID`. 

My process for conversion was to make a new table called `event` and map the standard Darwin Core column names to pre-existing columns that serve the same purpose in my original `seine_data` table and populate the other required fields.



```r
#TODO: Include abiotic measurements (YSI temp and salinity from 0 and 1 m) to hang off eventID in the eMoF table

event <- tibble(datasetName = "Hakai Institute Juvenile Salmon Program",
                eventID = survey_seines$seine_id,
                eventDate = date(survey_seines$survey_date),
                eventTime = paste0(survey_seines$set_time, "-0700"),
                eventRemarks = paste3(survey_seines$survey_comments, survey_seines$seine_comments),
                decimalLatitude = survey_seines$lat,
                decimalLongitude = survey_seines$long,
                locationID = survey_seines$site_id,
                coordinatePrecision = 0.00001,
                coordinateUncertaintyInMeters = 10,
                country = "Canada",
                countryCode = "CA",
                stateProvince = "British Columbia",
                habitat = "Nearshore marine",
                geodeticDatum = "EPSG:4326 WGS84",
                minimumDepthInMeters = 0,
                maximumDepthInMeters = 9, # seine depth is 9 m
                samplingProtocol = "http://dx.doi.org/10.21966/1.566666", # This is the DOI for the Hakai Salmon Data Package that contains the smnpling protocol, as well as the complete data package
                language = "en",
                license = "http://creativecommons.org/licenses/by/4.0/legalcode",
                bibliographicCitation = "Johnson, B.T., J.C.L. Gan, S.C. Godwin, M. Krkosek, B.P.V. Hunt. 2020. Hakai Juvenile Salmon Program Time Series. Hakai Institute, Quadra Island Ecological Observatory, Heriot Bay, British Columbia, Canada. v#.#.#, http://dx.doi.org/10.21966/1.566666",
                references = "https://github.com/HakaiInstitute/jsp-data",
                institutionID = "https://www.gbif.org/publisher/55897143-3f69-42f1-810d-ae94b55fde24, https://oceanexpert.org/institution/20121, https://edmo.seadatanet.org/report/5148",
                institutionCode = "Hakai"
               ) 
```


### occurrence

Next you'll want to determine what constitutes an occurrence for your data set. Because each event captures fish, I consider each fish to be an occurrence. Therefore, the unit of observation (each row) in the occurrence table is a fish. To link each occurrence to an event you need to include the `eventID` column for every occurrence so that you know what seine (event) each fish (occurrence) came from. You must also provide a globally unique identifier for each occurrence. I already have a locally unique identifier for each fish in the original `fish_data` table called `ufn`. To make it globally unique I pre-pend the organization and research program metadata to the `ufn` column. 

Not every fish is actually collected and given a Universal Fish Number (UFN) in our fish data tables, so in our field data sheets we record the total number of fish captured and the total number retained. So to get an occurrence row for every fish captured I create a row for every fish caught (minus the number taken) and create a generic numeric id (ie hakai-jsp-1) in one table and then join that to the fish table that includes a row for every fish retained that already has a UFN. 


```r
## make table long first
seines_total_long <- survey_seines %>% 
  select(seine_id, so_total, pi_total, cu_total, co_total, he_total, ck_total) %>% 
  pivot_longer(-seine_id, names_to = "scientificName", values_to = "n")

seines_total_long$scientificName <- recode(seines_total_long$scientificName, so_total = "Oncorhynchus nerka", pi_total = "Oncorhynchus gorbuscha", cu_total = "Oncorhynchus keta", co_total = "Oncorhynchus kisutch", ck_total = "Oncorhynchus tshawytscha", he_total = "Clupea pallasii") 

seines_taken_long <- survey_seines %>%
  select(seine_id, so_taken, pi_taken, cu_taken, co_taken, he_taken, ck_taken) %>% 
  pivot_longer(-seine_id, names_to = "scientificName", values_to = "n_taken") 

seines_taken_long$scientificName <- recode(seines_taken_long$scientificName, so_taken = "Oncorhynchus nerka", pi_taken = "Oncorhynchus gorbuscha", cu_taken = "Oncorhynchus keta", co_taken = "Oncorhynchus kisutch", ck_taken = "Oncorhynchus tshawytscha", he_taken = "Clupea pallasii") 

## remove records that have already been assigned an ID because they were actually retained
seines_long <-  full_join(seines_total_long, seines_taken_long, by = c("seine_id", "scientificName")) %>% 
  drop_na() %>% 
  mutate(n_not_taken = n - n_taken) %>% #so_total includes the number taken so I subtract n_taken to get n_not_taken
  select(-n_taken, -n) %>% 
  filter(n_not_taken > 0)

all_fish_not_retained <-
  seines_long[rep(seq.int(1, nrow(seines_long)), seines_long$n_not_taken), 1:3] %>% 
  select(-n_not_taken) %>% 
  mutate(prefix = "hakai-jsp-",
         suffix = 1:nrow(.),
         occurrenceID = paste0(prefix, suffix)
  ) %>% 
  select(-prefix, -suffix)

#

# Change species names to full Scientific names 
latin <- fct_recode(fish_data$species, "Oncorhynchus nerka" = "SO", "Oncorhynchus gorbuscha" = "PI", "Oncorhynchus keta" = "CU", "Oncorhynchus kisutch" = "CO", "Clupea pallasii" = "HE", "Oncorhynchus tshawytscha" = "CK") %>% 
  as.character()

fish_retained_data <- fish_data %>% 
  mutate(scientificName = latin) %>% 
  select(-species) %>% 
  mutate(prefix = "hakai-jsp-",
         occurrenceID = paste0(prefix, ufn)) %>% 
  select(seine_id, scientificName, occurrenceID)

occurrence <- bind_rows(all_fish_not_retained, fish_retained_data) %>% 
  rename(eventID = seine_id) %>% 
  mutate(`Life stage` = "juvenile")

unique_taxa <- unique(occurrence$scientificName)  
worms_names <- wm_records_names(unique_taxa) 
df_worms_names <- bind_rows(worms_names) %>% 
  select(scientificName = scientificname,
         scientificNameAuthorship = authority,
         taxonRank = rank,
         scientificNameID = lsid
         )

#include bycatch species

unique_bycatch <- unique(bycatch$scientificName) %>%  glimpse()
```

```
##  chr [1:29] "Oncorhynchus nerka" "Oncorhynchus tshawytscha" ...
```

```r
by_worms_names <- wm_records_names(unique_bycatch) %>% 
  bind_rows() %>% 
  select(scientificName = scientificname,
         scientificNameAuthorship = authority,
         taxonRank = rank,
         scientificNameID = lsid
         )

bycatch_occurrence <- bycatch %>% 
  select(eventID = seine_id, occurrenceID, scientificName, `Life stage` = bm_ageclass) %>% 
  filter(scientificName != "unknown")

bycatch_occurrence$`Life stage`[bycatch_occurrence$`Life stage` == "J"] <- "juvenile"
bycatch_occurrence$`Life stage`[bycatch_occurrence$`Life stage` == "A"] <- "adult"
bycatch_occurrence$`Life stage`[bycatch_occurrence$`Life stage` == "Y"] <- "Young of year"

combined_worms_names <- bind_rows(by_worms_names, df_worms_names) %>% 
  distinct(scientificName, .keep_all = TRUE)

occurrence <- bind_rows(bycatch_occurrence, occurrence)

occurrence <- left_join(occurrence, combined_worms_names) %>% 
    mutate(basisOfRecord = "HumanObservation",
        occurrenceStatus = "present")

write_csv(occurrence,here::here("Standardizing Marine Biological Data", "datasets", "hakai_salmon_data", "raw_data",   "occurrence.csv"))

# This removes events that didn't result in any occurrences
event <- dplyr::semi_join(event, occurrence, by = 'eventID') %>% 
  mutate(coordinateUncertaintyInMeters = ifelse(is.na(decimalLatitude), 1852, coordinateUncertaintyInMeters))

simple_sites <- sites %>% 
  select(site_id, ocgy_std_lat, ocgy_std_lon)

event <- dplyr::left_join(event, simple_sites, by = c("locationID" = "site_id")) %>% 
  mutate(decimalLatitude = coalesce(decimalLatitude, ocgy_std_lat),
         decimalLongitude = coalesce(decimalLongitude, ocgy_std_lon)) %>% 
  select(-c(ocgy_std_lat, ocgy_std_lon))

write_csv(event,here::here("Standardizing Marine Biological Data", "datasets", "hakai_salmon_data", "raw_data",   "event.csv"))
```


### measurementOrFact
To convert all your measurements or facts from your normal format to Darwin Core you essentially need to put all your measurements into one column called measurementType and a corresponding column called MeasurementValue. This standardizes the column names are in the `measurementOrFact` table. There are a number of predefined `measurementType`s listed on the [NERC](https://www.bodc.ac.uk/resources/vocabularies/) database that should be used where possible. I found it difficult to navigate this page to find the correct `measurementType`. 

Here I convert length, and weight measurements that relate to an event and an occurrence and call those `measurementTypes` as `length` and `weight`.


```r
mof_types <- read_csv("https://raw.githubusercontent.com/HakaiInstitute/jsp-data/master/OBIS_data/mof_type_units_id.csv")

fish_data$weight <- coalesce(fish_data$weight, fish_data$weight_field)
fish_data$fork_length <- coalesce(fish_data$fork_length, fish_data$fork_length_field)
fish_data$`Life stage` <- "juvenile"



measurementOrFact <- fish_data %>%
  mutate(occurrenceID = paste0("hakai-jsp-", ufn)) %>% 
  select(occurrenceID, eventID = seine_id, "Length (fork length)" = fork_length,
         "Standard length" = standard_length, "Weight" = weight, `Life stage`) %>% 
  pivot_longer(`Length (fork length)`:`Life stage`,
               names_to = "measurementType",
               values_to = "measurementValue",
               values_transform = list(measurementValue = as.character)) %>%
  filter(measurementValue != "NA") %>%
  left_join(mof_types,by = c("measurementType")) %>% 
  mutate(measurementValueID = case_when(measurementValue == "juvenile" ~ "http://vocab.nerc.ac.uk/collection/S11/current/S1127/"),
         measurementID = paste(eventID, measurementType, occurrenceID, sep = "-"))


write_csv(measurementOrFact,here::here("Standardizing Marine Biological Data", "datasets", "hakai_salmon_data", "raw_data",   "extendedMeasurementOrFact.csv"))
```


```r
library(dm)
#Check that every eventID in Occurrence occurs in event table
no_keys <- dm(event, occurrence, measurementOrFact)
only_pk <- no_keys %>% 
  dm_add_pk(event, eventID) %>% 
  dm_add_pk(occurrence, occurrenceID) %>% 
  dm_add_pk(measurementOrFact, measurementID)
dm_examine_constraints(only_pk)

model <- only_pk %>% 
  dm_add_fk(occurrence, eventID, event) %>% 
  dm_add_fk(measurementOrFact, occurrenceID, occurrence)
dm_examine_constraints(model)

# dm_draw(model, view_type = "all") 
```

## Example Two

Add another example here, perhaps zooplankton?
