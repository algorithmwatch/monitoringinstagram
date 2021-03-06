# Pre analysis of data

**Goal:** Test the main hypothesis and decide of the opportunity to move
from private beta to public beta.

**Tests:** Is the % of ‘racy’ pictures higher in the feeds of the
donors, compared with the % of racy pictures in the items posted by the
monitored accounts

## Setup

Follow the steps below in a terminal to load the database from a dump
file and save under the name `igdb`.

    pg_restore 02-19_backup > backup.sql
    sudo -i -u postgres
    dropdb igdb
    createdb igdb
    igdb < backup.sql

Load packages needed.

``` r
needs("RPostgreSQL", "tidyverse", "jsonlite")
```

## Read database

Create variables for the tables from the database needed for this
analysis: `data_donors_encounter`, `ig_observer_igimage` and
`ig_observer_gvisionanalyse`.

``` r
# create a connection to the postgres database
con = dbConnect(dbDriver("PostgreSQL"), dbname= "igdb")

#list tables in the database
dbListTables(con)
```

    ##  [1] "data_donors_datadonationerror" "django_migrations"            
    ##  [3] "django_session"                "data_donors_donor_following"  
    ##  [5] "data_donors_datadonation"      "auth_user"                    
    ##  [7] "auth_group"                    "data_donors_encounter"        
    ##  [9] "auth_user_user_permissions"    "data_donors_donor"            
    ## [11] "django_content_type"           "data_donors_donorfollowing"   
    ## [13] "django_admin_log"              "ig_observer_igimage"          
    ## [15] "auth_group_permissions"        "auth_user_groups"             
    ## [17] "auth_permission"               "ig_observer_iguser"           
    ## [19] "ig_observer_gvisionanalyse"    "ig_observer_igpost"

``` r
# data_donors_encounter: Log of encounters with posts from observed accounts
# "id" "position_in_list "data_donation_id"  "ig_post_id"
data_donors_encounter = dbReadTable(con, "data_donors_encounter")

# ig_observer_igimage: List of images in observed posts
# "id" "image_url" "ig_post_id"  Post ID
ig_observer_igimage = dbReadTable(con, "ig_observer_igimage")

# ig_observer_gvisionanalyse: # Result of G Vision analysis
# "id" "analyse" "ig_image_id" "created"  
ig_observer_gvisionanalyse = dbReadTable(con, "ig_observer_gvisionanalyse")
```

## Stats on current encounters

``` r
data_donors_encounter %>% nrow %>% cat("encounters total")
```

    ## 554 encounters total

``` r
(data_donors_encounter$ig_post_id %in% ig_observer_igimage$ig_post_id) %>% ifelse("contains image", "no image") %>% table
```

    ## .
    ## contains image       no image 
    ##            477             77

``` r
data_donors_encounter$ig_post_id %>% unique %>% length %>% cat("unique posts encountered")
```

    ## 79 unique posts encountered

``` r
(unique(data_donors_encounter$ig_post_id) %in% ig_observer_igimage$ig_post_id) %>% ifelse("contains image", "no image") %>% table
```

    ## .
    ## contains image       no image 
    ##             68             11

## Analysis: Is the share of ‘racy’ pictures higher in the feeds of the donors than in the items posted by the monitored accounts?

Extract analysis result from G vision JSON blob and save in new column
called `racy`.

``` r
img = dbReadTable(con, "ig_observer_gvisionanalyse") %>%
  mutate(analyse = sapply(analyse, parse_json),
         racy = sapply(analyse, function(result) result$safeSearchAnnotation$racy) %>% 
           factor(., levels = c("VERY_LIKELY", "LIKELY", "POSSIBLE", "UNLIKELY", "VERY_UNLIKELY"), ordered = TRUE) %>% unname) %>% 
  select(-analyse)
```

Merge sex of IG user to post
data

``` r
sex = dbReadTable(con, "ig_observer_igpost") %>% select(id, ig_user_id) %>%
  left_join(dbReadTable(con, "ig_observer_iguser") %>% select(id, sex), by = c("ig_user_id" = "id"))
```

**Step 1:** Get share of *created* posts containing racy pictures
(basis: collected posts by monitored accounts).

``` r
# take list of posted images (ig_observer_igimage)
x = dbReadTable(con, "ig_observer_igimage") %>%
  # join sex to post ID
  left_join(sex, by = c("ig_post_id" = "id")) %>% 
  # join analysis result to image ID with ig_observer_igimage
  left_join(img, by = c("id" = "ig_image_id"), suffix = c("_ig_img", "_analysis")) %>%
  # summarize by raciness: calculate number and shares of posts
  group_by(sex, racy) %>%
  summarise(n = ig_post_id %>% unique %>% length) %>%
  mutate(share =  n/sum(n))
x
```

    ## # A tibble: 10 x 4
    ## # Groups:   sex [2]
    ##    sex   racy              n  share
    ##    <chr> <ord>         <int>  <dbl>
    ##  1 F     VERY_LIKELY      61 0.170 
    ##  2 F     LIKELY           51 0.142 
    ##  3 F     POSSIBLE         60 0.167 
    ##  4 F     UNLIKELY         79 0.220 
    ##  5 F     VERY_UNLIKELY   108 0.301 
    ##  6 M     VERY_LIKELY       9 0.0833
    ##  7 M     LIKELY           15 0.139 
    ##  8 M     POSSIBLE         21 0.194 
    ##  9 M     UNLIKELY         21 0.194 
    ## 10 M     VERY_UNLIKELY    42 0.389

  - **17%** of posts by women are “very likely” racy, **14%** are
    “likely” racy.
  - **8%** of posts by men are “very likely” racy, **14%** are “likely”
    racy.

To support our main hypothesis, the shares would need to be higher than
this in the encounters logged by data donors.

**Step 2:** Get share of *encountered* posts containing racy pictures
(basis: all posts from monitored accounts encountered by donors).

``` r
# take encounters (data_donors_encounter)
x = dbReadTable(con, "data_donors_encounter") %>% 
# does the post contain a racy image?
  # join sex to post ID
  left_join(sex, by = c("ig_post_id" = "id")) %>% 
  # join image ID to post ID with ig_observer_igimage
  left_join(dbReadTable(con, "ig_observer_igimage") %>% select(-image_url), by = c("ig_post_id"), suffix = c("_encounter", "_ig_img")) %>% 
  # join analysis result to image ID with ig_observer_igimage
  left_join(img, by = c("id_ig_img" = "ig_image_id")) %>% 
  # filter out videos, which have no analysis result
  filter(!is.na(id_ig_img)) %>% 
# how likely is that?
  # summarize by post ID: get highest level of raciness
  group_by(sex, id_encounter, ig_post_id) %>% summarise(racy = min(racy, na.rm = T)) %>% 
  # summarize by raciness: calculate shares
  group_by(sex, racy) %>% summarise(n = n()) %>% mutate(share =  n/sum(n))
x
```

    ## # A tibble: 10 x 4
    ## # Groups:   sex [2]
    ##    sex   racy              n  share
    ##    <chr> <ord>         <int>  <dbl>
    ##  1 F     VERY_LIKELY      65 0.199 
    ##  2 F     LIKELY           39 0.119 
    ##  3 F     POSSIBLE         27 0.0826
    ##  4 F     UNLIKELY        123 0.376 
    ##  5 F     VERY_UNLIKELY    73 0.223 
    ##  6 M     VERY_LIKELY      19 0.127 
    ##  7 M     LIKELY           10 0.0667
    ##  8 M     POSSIBLE         27 0.18  
    ##  9 M     UNLIKELY         27 0.18  
    ## 10 M     VERY_UNLIKELY    67 0.447

  - **20%** of encountered posts by women are “very likely” racy,
    **12%** are “likely” racy.
  - **13%** of encountered posts by men are “very likely” racy, **7%**
    are “likely” racy.

This does not show a clear picture yet.
