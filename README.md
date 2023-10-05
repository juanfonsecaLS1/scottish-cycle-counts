
<!-- README.md is generated from README.Rmd. Please edit that file -->

# scottish-cycle-counts

<!-- badges: start -->
<!-- badges: end -->

The goal of Scottish-cycle-counts is to read-in a process data on
cycling volumes in Scotland.

``` r
library(tidyverse)
```

First of all, a copy of the original files can be obtained with the code
below. This code is to be run just once.

``` r
dir.create("data_raw")
system("gh release download 1 --dir data_raw")
```

The input dataset is a single .zip file:

``` r
zipped_data = list.files(path = "data_raw",pattern = "\\.zip$",full.names = T)
zipped_data
```

We can unzip it as follows:

``` r
unzip(zipped_data, exdir = "data_raw")
```

``` r
files_csv = list.files("data_raw", pattern = "\\.csv$", full.names = TRUE)
files_csv
#>  [1] "data_raw/Aberdeen City Council.csv"                            
#>  [2] "data_raw/Aberdeenshire Council.csv"                            
#>  [3] "data_raw/City of Edinburgh Council.csv"                        
#>  [4] "data_raw/Comhairle nan Eilean Siar (Western Isles Council).csv"
#>  [5] "data_raw/East Ayrshire Council.csv"                            
#>  [6] "data_raw/East Dunbartonshire Council.csv"                      
#>  [7] "data_raw/Glasgow City Council.csv"                             
#>  [8] "data_raw/John Muir Way.csv"                                    
#>  [9] "data_raw/National Cycle Nework (Scotland) - Sustrans.csv"      
#> [10] "data_raw/National Monitoring Framework - Cycling Scotland.csv" 
#> [11] "data_raw/North Ayrshire Council.csv"                           
#> [12] "data_raw/Perth and Kinross Council.csv"                        
#> [13] "data_raw/Scotland North East Trunk Roads.csv"                  
#> [14] "data_raw/Scotland North West Trunk Roads.csv"                  
#> [15] "data_raw/Scotland South East Trunk Roads.csv"                  
#> [16] "data_raw/Scotland South West Trunk Roads.csv"                  
#> [17] "data_raw/South Ayrshire Council.csv"                           
#> [18] "data_raw/South Lanarkshire Council.csv"                        
#> [19] "data_raw/Stirling Council.csv"                                 
#> [20] "data_raw/The Highland Council.csv"
```

We can read this file in R as follows:

``` r
# library(data.table)
# counts = data.frame(data.table::rbindlist(lapply(files_csv,data.table::fread))) #DT's quick way to read the files 
counts = map_dfr(files_csv, read_csv,show_col_types = FALSE)
dim(counts)
#> [1] 230307     10
counts
#> # A tibble: 230,307 × 10
#>    area    count endTime             latitude location longitude provider siteID
#>    <chr>   <dbl> <dttm>                 <dbl> <chr>        <dbl> <chr>    <chr> 
#>  1 Aberde…    61 2023-06-14 00:00:00     57.1 A956 We…     -2.09 Aberdee… ABE65…
#>  2 Aberde…    26 2023-06-14 00:00:00     57.2 Dyce Dr…     -2.20 Aberdee… ABE918
#>  3 Aberde…    18 2023-06-14 00:00:00     57.2 Dyce Dr…     -2.22 Aberdee… ABE397
#>  4 Aberde…   158 2023-06-14 00:00:00     57.2 Ellon R…     -2.09 Aberdee… ABE920
#>  5 Aberde…   157 2023-06-14 00:00:00     57.1 Shell C…     -2.10 Aberdee… ABE229
#>  6 Aberde…   132 2023-06-14 00:00:00     57.2 Tillydr…     -2.11 Aberdee… ABE899
#>  7 Aberde…   395 2023-06-14 00:00:00     57.1 Deeside…     -2.10 Aberdee… ABE235
#>  8 Aberde…    20 2023-06-14 00:00:00     57.2 Farburn…     -2.17 Aberdee… ABE745
#>  9 Aberde…    91 2023-06-14 00:00:00     57.2 Parkway…     -2.10 Aberdee… ABE894
#> 10 Aberde…    39 2023-06-14 00:00:00     57.2 Site B …     -2.16 Aberdee… ABE288
#> # ℹ 230,297 more rows
#> # ℹ 2 more variables: startTime <dttm>, usmart_id <chr>
```

``` r
counts_monthly = counts |>
  mutate(
    year = year(endTime),
    month = month(endTime, label = TRUE),
    # date rounded to nearest month in 2020-01-01 format:
    date = lubridate::floor_date(endTime, unit = "month")
  ) |>
  group_by(date, area) |>
  summarise(
    count = sum(count)
  )
# Add column with names for most common areas:
# Most common areas are:
area_counts = counts_monthly |>
  group_by(area) |>
  summarise(
    count = sum(count)
  )

top_5_areas = area_counts |> 
  slice_max(count, n = 5)

# Add column that is area name if in top 5, else "Other":
counts_monthly_top = counts_monthly |>
  mutate(
    Area = ifelse(
      area %in% top_5_areas$area,
      area,
      "Other"
    )
  ) |>
  group_by(date, Area) |>
  summarise(
    count = sum(count)
  )
```

``` r
counts_monthly_top |>
  ggplot(aes(x = date, y = count, colour = Area)) +
  geom_line() +
  # Add log y-axis:
  scale_y_log10()
```

![](README_files/figure-gfm/unnamed-chunk-10-1.png)<!-- -->

A quick look at the data to check if there are sites with missing data
or duplicated records:

``` r
counts |>
  group_by(siteID,location) |> 
  unique() |> 
  count() |> 
  ggplot(aes(n))+geom_histogram(bins = 30)
```

![](README_files/figure-gfm/unnamed-chunk-11-1.png)<!-- -->

``` r
range(counts$startTime)
#> [1] "2022-06-01 UTC" "2023-06-13 UTC"
difftime(range(counts$startTime)[2],range(counts$startTime)[1],units = "days")
#> Time difference of 377 days
```

Each site should have a maximum of 378 days in the dataset. The
following code detects the sites with some type of duplication and the
ones with fewer records.

``` r
repeated_sites = counts |>
  group_by(siteID) |> 
  filter(n()>378) |> 
  select(siteID) |>
  unique()

repeated_sites$siteID  |> length()
#> [1] 22

fewer_sites = counts |>
  group_by(siteID) |> 
  filter(n()<300) |> 
  select(siteID) |>
  unique()

fewer_sites$siteID  |> length()
#> [1] 133
```

A subset of the clean sites is produced, so we can do some AADT
analysis. Records after 2023-06-01 are filtered out to have only one
year of data for each site

``` r

clean_counts = counts |>
  filter(startTime < as.Date("2023-06-01")) |> 
  anti_join(repeated_sites,by =join_by(siteID)) |>
  anti_join(fewer_sites,by =join_by(siteID)) |> 
  filter(n()==365,.by = siteID)

clean_counts |>
  group_by(siteID,location) |> 
  unique() |> 
  count() |> summary()
#>     siteID            location               n      
#>  Length:383         Length:383         Min.   :365  
#>  Class :character   Class :character   1st Qu.:365  
#>  Mode  :character   Mode  :character   Median :365  
#>                                        Mean   :365  
#>                                        3rd Qu.:365  
#>                                        Max.   :365
```

Here we calculate some statistics for the whole year including mean
(Average Annual Daily Flow), median daily flow, minimum and maximum
daily flows,

``` r
AADF_sites = clean_counts |> 
  summarise(across(count,list(mean = mean,
                              median = median,
                              min = min,
                              max = max)),.by = siteID)

AADF_sites
#> # A tibble: 383 × 5
#>    siteID  count_mean count_median count_min count_max
#>    <chr>        <dbl>        <dbl>     <dbl>     <dbl>
#>  1 ABE6535       19.9           18         1        50
#>  2 ABE896        44.1           43         0       195
#>  3 ABE6545       34.3           32         2        84
#>  4 ABE895        86.0           74         0       288
#>  5 ABE918        15.6           15         0        39
#>  6 ABE122        16.6           16         0        57
#>  7 ABE123        13.8           12         0        49
#>  8 ABE400        43.5           33         2       150
#>  9 ABE288        12.5            9         0        81
#> 10 ABE893        43.3           43         0        93
#> # ℹ 373 more rows
```

``` r
counts_per_area = counts |> select(siteID,area) |> unique()

AADF_sites |>
  left_join(counts_per_area,by = join_by(siteID)) |> 
  mutate(
    area = ifelse(
      area %in% top_5_areas$area,
      area,
      "Other"
    )) |> 
  ggplot(aes(x=fct_reorder(area,count_mean,.desc = T),
             y=count_mean))+
  geom_boxplot(outlier.shape = NA)+
  geom_jitter(aes(col = area),alpha = 0.2,shape = 19,show.legend = F)+
  coord_cartesian(ylim = c(0,1000))+
  theme_minimal()+
  labs(x = "Area",
       y = "AADF")
```

![](README_files/figure-gfm/unnamed-chunk-16-1.png)<!-- -->

We create a vector to store the bank holidays in Scotland extracted from
the [mygov.scot web](https://www.mygov.scot/scotland-bank-holidays)

``` r
scot_bank_holidays = as.Date(c("2022/06/03",
                               "2022/06/02",
                               "2022/08/1",
                               "2022/11/30",
                               "2022/12/25",
                               "2022/12/26",
                               "2022/12/27",
                               "2023/01/02",
                               "2023/01/03",
                               "2023/04/07",
                               "2023/05/01",
                               "2023/05/08",
                               "2023/05/29"))
```

We can calculate the same summary statistics by type of day: bank
holidays, weekends and weekdays (AAWDF).

``` r
ADF_dtype = clean_counts |> 
  mutate(d.type = case_when(as.Date(startTime) %in% scot_bank_holidays~"Bank Holiday",
                            wday(startTime,week_start = 1)<6~"Weekday",
                            TRUE~"Weekend")) |> 
  summarise(across(count,list(mean = mean,
                              median = median,
                              min = min,
                              max = max)),
            .by = c(siteID,d.type))

ADF_dtype
#> # A tibble: 1,149 × 6
#>    siteID  d.type  count_mean count_median count_min count_max
#>    <chr>   <chr>        <dbl>        <dbl>     <dbl>     <dbl>
#>  1 ABE6535 Weekday       23.6           22         6        50
#>  2 ABE896  Weekday       46.4           46         2       101
#>  3 ABE6545 Weekday       40.3           40         2        84
#>  4 ABE895  Weekday       78.1           65         1       247
#>  5 ABE918  Weekday       19.5           20         0        39
#>  6 ABE122  Weekday       17             16         0        50
#>  7 ABE123  Weekday       12.7           11         0        45
#>  8 ABE400  Weekday       40.1           31         2       110
#>  9 ABE288  Weekday       11.8            8         0        62
#> 10 ABE893  Weekday       52.6           52         6        93
#> # ℹ 1,139 more rows
```

``` r
counts_per_area = counts |> select(siteID,area) |> unique()

ADF_dtype |>
  left_join(counts_per_area,by = join_by(siteID)) |> 
  mutate(
    area = ifelse(
      area %in% top_5_areas$area,
      area,
      "Other"
    )) |> 
  ggplot(aes(x=fct_reorder(area,count_mean,.desc = T),
             y=count_mean))+
  geom_boxplot(outlier.shape = NA)+
  geom_jitter(aes(col = area),alpha = 0.2,shape = 19,show.legend = F)+
  coord_cartesian(ylim = c(0,1000))+
  facet_wrap(d.type~.,ncol = 2)+
  theme_minimal()+
  labs(x = "Area",
       y = "ADF")
```

![](README_files/figure-gfm/unnamed-chunk-19-1.png)<!-- -->

## Spatial Analysis

``` r
library(sf)
library(tmap)
```

The following code reads the network with the estimated commute trips
from the npt repository. Each edge/link of the network has four
attributes. We will focus on `bicycle` which is the estimated number of
daily commute trips in both directions.

``` r
rnet_commute = read_rds("../npt/outputs/rnet_commute.Rds")
rnet_commute
#> Simple feature collection with 1709 features and 4 fields
#> Geometry type: LINESTRING
#> Dimension:     XY
#> Bounding box:  xmin: -3.33653 ymin: 55.89548 xmax: -3.12415 ymax: 55.98413
#> Geodetic CRS:  WGS 84
#> First 10 features:
#>    bicycle bicycle_go_dutch Gradient Quietness                       geometry
#> 1        0                1        0        40 LINESTRING (-3.1547 55.9749...
#> 2        0                1        0        60 LINESTRING (-3.27797 55.911...
#> 3        0                1        2        70 LINESTRING (-3.27672 55.910...
#> 4        0                1        4        70 LINESTRING (-3.14531 55.969...
#> 5        0                1        0        80 LINESTRING (-3.27741 55.911...
#> 6        0                1        0        80 LINESTRING (-3.15282 55.976...
#> 7        0                1        1        80 LINESTRING (-3.17113 55.961...
#> 8        0                1        0        90 LINESTRING (-3.1547 55.9749...
#> 9        0                1        1        90 LINESTRING (-3.16781 55.961...
#> 10       0                1        1        90 LINESTRING (-3.16555 55.962...
```

An `sf` object is created from the `clean_counts` data frame. AADF for
each counts are joined using the `siteID`

``` r
sf_counts = clean_counts |>
  select(siteID,latitude,longitude,provider,location) |>
  unique() |>
  left_join(AADF_sites,by = "siteID") |>
  filter(count_mean > 0) |> 
  st_as_sf(coords = c("longitude","latitude"),crs = 4326)
sf_counts
#> Simple feature collection with 338 features and 7 fields
#> Geometry type: POINT
#> Dimension:     XY
#> Bounding box:  xmin: -7.307251 ymin: 54.91291 xmax: -1.15021 ymax: 60.1511
#> Geodetic CRS:  WGS 84
#> # A tibble: 338 × 8
#>    siteID  provider         location count_mean count_median count_min count_max
#>  * <chr>   <chr>            <chr>         <dbl>        <dbl>     <dbl>     <dbl>
#>  1 ABE6535 Aberdeen City C… A96 Auc…       19.9           18         1        50
#>  2 ABE896  Aberdeen City C… F&B Way…       44.1           43         0       195
#>  3 ABE6545 Aberdeen City C… A9119 Q…       34.3           32         2        84
#>  4 ABE895  Aberdeen City C… Beach E…       86.0           74         0       288
#>  5 ABE918  Aberdeen City C… Dyce Dr…       15.6           15         0        39
#>  6 ABE122  Aberdeen City C… Maidenc…       16.6           16         0        57
#>  7 ABE123  Aberdeen City C… Maidenc…       13.8           12         0        49
#>  8 ABE400  Aberdeen City C… Seaton …       43.5           33         2       150
#>  9 ABE288  Aberdeen City C… Site B …       12.5            9         0        81
#> 10 ABE893  Aberdeen City C… Welling…       43.3           43         0        93
#> # ℹ 328 more rows
#> # ℹ 1 more variable: geometry <POINT [°]>
```

A subset of the counts are taken based on a buffer of the `rnet_commute`
object.

``` r
rnet_buffer20 = rnet_commute |> st_union() |>  st_buffer(dist = 20)

sf_counts_selected = sf_counts[rnet_buffer20,]
```

### Approach A

The nearest feature is joined to each point location

``` r
val_app1 = st_join(sf_counts_selected,rnet_commute,join = st_nearest_feature)
val_app1
#> Simple feature collection with 21 features and 11 fields
#> Geometry type: POINT
#> Dimension:     XY
#> Bounding box:  xmin: -3.24015 ymin: 55.92337 xmax: -3.12779 ymax: 55.9566
#> Geodetic CRS:  WGS 84
#> # A tibble: 21 × 12
#>    siteID  provider         location count_mean count_median count_min count_max
#>  * <chr>   <chr>            <chr>         <dbl>        <dbl>     <dbl>     <dbl>
#>  1 EDH0030 City of Edinbur… Dalry R…      1009.          331        40     27670
#>  2 EDH0037 City of Edinbur… Whiteho…       437.          457        54       753
#>  3 EDH0045 City of Edinbur… Melvill…       128.          128        13       266
#>  4 EDH0022 City of Edinbur… North M…       885.          888        99      1839
#>  5 EDH0035 City of Edinbur… A90 Dea…       415.          361        36       857
#>  6 EDH0039 City of Edinbur… Bruntsf…       218.          236         0       481
#>  7 EDH0043 City of Edinbur… Melvill…      1099.         1109       242      1813
#>  8 EDH0041 City of Edinbur… Mayfiel…       152.            0         0       683
#>  9 EDH0029 City of Edinbur… A8 Cors…       100.            0         0       829
#> 10 EDH0044 City of Edinbur… Melvill…       251.          257        32       446
#> # ℹ 11 more rows
#> # ℹ 5 more variables: geometry <POINT [°]>, bicycle <dbl>,
#> #   bicycle_go_dutch <dbl>, Gradient <dbl>, Quietness <dbl>
```

``` r
tmap_mode("view")
tm_shape(rnet_commute)+
  tm_lines(col = "bicycle",lwd = "bicycle",lwd.legend = tm_legend_combine("col"))+
  tm_shape(val_app1)+
  tm_dots(col = "count_mean")
```

The following plot compares the observed counts and the paired estimate
flows.

``` r
val_app1 |> 
  st_drop_geometry() |>
  mutate(ratio = bicycle/count_mean) |> 
  ggplot(aes(ratio))+
  geom_histogram()
```

![](README_files/figure-gfm/unnamed-chunk-26-1.png)<!-- -->

``` r
val_app1 |>
  st_drop_geometry() |>
  ggplot(aes(x = count_mean,
             y = bicycle)) +
  geom_point() +
  geom_smooth(method = "lm",
              formula = 'y ~ x',
              se = F) +
  coord_fixed(xlim = c(0, max(
    c(val_app1$count_mean, val_app1$bicycle)
  )),
  ylim = c(0, max(
    c(val_app1$count_mean, val_app1$bicycle)
  )))
```

![](README_files/figure-gfm/unnamed-chunk-27-1.png)<!-- -->

A linear regression is used to evaluate the fit of the estimates, it is
assumed that the proportion of commute trips is constant across counts
(intercept = `0`).

``` r
lm_app1 = lm(bicycle ~ count_mean+0,data = val_app1)
summary(lm_app1)
#> 
#> Call:
#> lm(formula = bicycle ~ count_mean + 0, data = val_app1)
#> 
#> Residuals:
#>     Min      1Q  Median      3Q     Max 
#> -4.8149 -0.5083  0.6479  4.4314  8.0276 
#> 
#> Coefficients:
#>            Estimate Std. Error t value Pr(>|t|)    
#> count_mean 0.006192   0.001606   3.855 0.000988 ***
#> ---
#> Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
#> 
#> Residual standard error: 4.054 on 20 degrees of freedom
#> Multiple R-squared:  0.4263, Adjusted R-squared:  0.3976 
#> F-statistic: 14.86 on 1 and 20 DF,  p-value: 0.0009876
```

### Approach B

The previous approach assigned a count site to each road link based. So
far, it has not been addressed the fact that counts might be reporting
uni-directional flows along specific links.

Using the `sf_counts_selected` object, we produce a buffer of 30 metres
from each count site.

``` r
sel_counts_buf30 = st_buffer(sf_counts_selected,dist = 30)
```

All overlaps within the `sel_counts_buf30` are identified using the
following code.

``` r
counts_overlap = st_intersects(sel_counts_buf30, sel_counts_buf30)
```

Some overlaps might reveal count sites reporting flows of different
directions on the same edge/link. For example, the sites EDH0040 and
EDH0041 which are located on Mayfield Road.

``` r
tmap_mode("view")
tm_shape(sel_counts_buf30[sel_counts_buf30$siteID %in% c("EDH0040","EDH0041"),])+
  tm_polygons(alpha = 0.5)+
  tm_shape(sf_counts_selected)+
  tm_dots()
  
```

A more complex instance is the overlap of sensors on Melville Dr, two of
the sensors report flows on the main road and the other has data of the
adjacend path.

``` r
tmap_mode("view")
tm_shape(sel_counts_buf30[sel_counts_buf30$siteID %in% c("EDH0042","EDH0043","EDH0044","EDH0045"),])+
  tm_polygons(alpha=0.3)+
  tm_shape(sf_counts_selected)+
  tm_dots()
```

The following code aggregates some of the overlapping counts using the
`location` attribute as aggregation criteria.

``` r
grouped_counts =
  do.call(rbind,
          lapply(unique(counts_overlap),
                 function(x) {
                   tmp_group = sf_counts_selected[x,]
                   
                   # Count aggreagation
                   simp_data = tmp_group |>
                     # Removing the direction from the location string
                     mutate(location = str_remove(location,
                                                  "\\s\\w*bound")) |>
                     st_drop_geometry() |>
                     # Extracting the first value for the siteID,
                     # provider and adds up the counts for sites with
                     # the same 'location'
                     summarise(across(c("siteID", "provider"),
                                      \(x) head(x, n = 1)),
                               across(starts_with("count_"), sum),
                               .by =  "location")
                   
                   simp_group = tmp_group |>
                     select(siteID) |>
                     filter(siteID %in% simp_data$siteID)
                   
                   simp_counts = simp_group |>
                     left_join(simp_data, by = "siteID") |>
                     relocate(location, .after = provider) |>
                     relocate(geometry, .after = count_max)
                   
                   return(simp_counts)
                 }))
```

As in the previous approach, the count sites are joined to the nearest
feature in the `rnet_commute` network.

``` r
grouped_counts$nearest_edge = st_nearest_feature(grouped_counts,
                                                 rnet_commute,
                                                 check_crs = T)


val_app2 = cbind(grouped_counts,st_drop_geometry(rnet_commute)[grouped_counts$nearest_edge,])
```

The following code shows the counts with the corresponding network
edges/links.

``` r
tm_shape(grouped_counts)+
  tm_dots()+
  tm_shape(rnet_commute[grouped_counts$nearest_edge,])+
  tm_lines()
```

The figure below compares the counts and the estimated flows for the
current approach

``` r
val_app2 |>
  st_drop_geometry() |>
  ggplot(aes(x = count_mean,
             y = bicycle)) +
  geom_point() +
  geom_smooth(method = "lm",
              formula = 'y ~ x',
              se = F) +
  coord_fixed(xlim = c(0, max(
    c(val_app2$count_mean, val_app2$bicycle)
  )),
  ylim = c(0, max(
    c(val_app2$count_mean, val_app2$bicycle)
  )))
```

![](README_files/figure-gfm/unnamed-chunk-36-1.png)<!-- -->

As in the previous approach, a linear regression is used to have a
high-level assessment of the estimations. With this approach, although
there is not a significant change in the estimate, there is a slight
improvement in the R<sup>2</sup>.

``` r
lm_app2 = lm(bicycle ~ count_mean+0,data = val_app2)
summary(lm_app2)
#> 
#> Call:
#> lm(formula = bicycle ~ count_mean + 0, data = val_app2)
#> 
#> Residuals:
#>     Min      1Q  Median      3Q     Max 
#> -4.6021 -0.7120  0.2573  4.3372  8.0632 
#> 
#> Coefficients:
#>            Estimate Std. Error t value Pr(>|t|)   
#> count_mean 0.005965   0.001523   3.917  0.00111 **
#> ---
#> Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
#> 
#> Residual standard error: 3.916 on 17 degrees of freedom
#> Multiple R-squared:  0.4744, Adjusted R-squared:  0.4435 
#> F-statistic: 15.35 on 1 and 17 DF,  p-value: 0.001109
```

## New networks

Downloading the data from latest builds

``` r
setwd("../npt/outputdata")
system("gh release download v2023-09-10-17-43-21.109279_commit_86ae338b12f523c27fcc290f48105f2e5dbdcab7")
system("gh release download v2023-08-18-10-42-44_commit_cbb84b024550d638dbca066c5850d1b03d55fc66 --skip-existing")
```

From all RDS files, we select the ones that contain the networks as a
list

``` r
networks_files = list.files(path = "../npt/outputdata/",pattern = "^rnet.*_list",full.names = TRUE)
networks_files              
#> [1] "../npt/outputdata/rnet_commute_list.Rds"
#> [2] "../npt/outputdata/rnet_school_list.Rds"
```

The following function is prepared to run the analysis as per the second
approach

``` r
library(purrr)

rnet_val = function(rnet_path,counts){
  
  # Loading the network RDS file
  rnet_nested_list = read_rds(rnet_path)
  
  # Detecting the name of the main purpose
  main_rnet_name = str_extract(rnet_path,"(commute|school)")
  
  # Flattening the list
  rnet_flat_list = rnet_nested_list |> list_flatten()
  
  
  lst_names = paste(main_rnet_name,names(rnet_flat_list),sep = ".")
  
  # Assigning names to the lists
  names(rnet_flat_list) = lst_names
  
  lm_rnet = 
    lapply(lst_names,function(rnet_name){
    rnet = rnet_flat_list[[rnet_name]]
    
  # Creating buffer
  rnet_buffer20 = rnet |>
    st_union() |>
    st_buffer(dist = 20)
  
  # Subsetting counts based on buffer
  counts_selected = counts[rnet_buffer20,]
  
  # Creating buffer around counts 30 m
  counts_buf30 = st_buffer(counts_selected,dist = 30)
  
  # Finding overlapping counts
  counts_overlap_30 = st_intersects(counts_buf30, counts_buf30)
  
  # Processing overlaps and aggregating if possible
  aggregated_counts =
    do.call(rbind,
            lapply(unique(counts_overlap_30),
                   function(x) {
                     tmp_group = counts_selected[x, ]
                     
                     # Count aggreagation
                     simp_data = tmp_group |>
                       # Removing the direction from the location string
                       mutate(location = str_remove(location,
                                                    "\\s\\w*bound")) |>
                       st_drop_geometry() |>
                       # Extracting the first value for the siteID,
                       # provider and adds up the counts for sites with
                       # the same 'location'
                       summarise(across(c("siteID", "provider"),
                                        \(x) head(x, n = 1)),
                                 across(starts_with("count_"), sum),
                                 .by =  "location")
                     
                     simp_group = tmp_group |>
                       select(siteID) |>
                       filter(siteID %in% simp_data$siteID)
                     
                     simp_counts = simp_group |>
                       left_join(simp_data, by = "siteID") |>
                       relocate(location, .after = provider) |>
                       relocate(geometry, .after = count_max)
                     
                     return(simp_counts)
                   }))
  
  aggregated_counts$nearest_edge = st_nearest_feature(aggregated_counts,
                                                      rnet,
                                                      check_crs = T)
  
  val_counts = cbind(aggregated_counts,
                     st_drop_geometry(rnet)[aggregated_counts$nearest_edge,])
  
  
  
  
  lm_counts = lm(bicycle ~ count_mean+0,data = val_counts)
  
  return(lm_counts)
    
  })
  names(lm_rnet) = lst_names
  
  return(lm_rnet)
}
```

The following code runs the validation for each network

``` r
val_results = lapply(networks_files,
                     rnet_val,
                     counts = sf_counts
                     )
```

Summarising the high-level results for all networks:

``` r
library(broom)
val_results_flat = val_results |> list_flatten(name_spec = "{inner}")

bind_cols(tibble(network = names(val_results_flat)),
          do.call(bind_rows,lapply(val_results_flat,function(tlm){
            bind_cols(fit = glance(tlm) |> select(adj.r.squared),
                      tidy(tlm) |> select(estimate,std.error))
            }
            )
            )
          )
#> # A tibble: 12 × 4
#>    network                   adj.r.squared estimate std.error
#>    <chr>                             <dbl>    <dbl>     <dbl>
#>  1 commute.fastest                  0.0912  0.0626   0.0106  
#>  2 commute.balanced                 0.0954  0.0895   0.0148  
#>  3 commute.quietest                 0.0751  0.0953   0.0179  
#>  4 commute.ebike                    0.0814  0.0553   0.00997 
#>  5 school.Primary_fastest           0.0643  0.00229  0.000469
#>  6 school.Primary_balanced          0.0410  0.00217  0.000556
#>  7 school.Primary_quietest          0.0459  0.00270  0.000656
#>  8 school.Primary_ebike             0.0468  0.00185  0.000443
#>  9 school.Secondary_fastest         0.0274  0.00220  0.000690
#> 10 school.Secondary_balanced        0.0321  0.00263  0.000769
#> 11 school.Secondary_quietest        0.0227  0.00250  0.000857
#> 12 school.Secondary_ebike           0.0227  0.00202  0.000692
```

    #>   |                                                           |                                                   |   0%  |                                                           |.                                                  |   1%                     |                                                           |.                                                  |   2% [unnamed-chunk-46]  |                                                           |..                                                 |   3%                     |                                                           |..                                                 |   4% [unnamed-chunk-47]  |                                                           |...                                                |   5%                     |                                                           |...                                                |   7% [unnamed-chunk-48]  |                                                           |....                                               |   8%                     |                                                           |....                                               |   9% [unnamed-chunk-49]  |                                                           |.....                                              |  10%                     |                                                           |......                                             |  11% [unnamed-chunk-50]  |                                                           |......                                             |  12%                     |                                                           |.......                                            |  13% [unnamed-chunk-51]  |                                                           |.......                                            |  14%                     |                                                           |........                                           |  15% [unnamed-chunk-52]  |                                                           |........                                           |  16%                     |                                                           |.........                                          |  18% [unnamed-chunk-53]  |                                                           |..........                                         |  19%                     |                                                           |..........                                         |  20% [unnamed-chunk-54]  |                                                           |...........                                        |  21%                     |                                                           |...........                                        |  22% [unnamed-chunk-55]  |                                                           |............                                       |  23%                     |                                                           |............                                       |  24% [unnamed-chunk-56]  |                                                           |.............                                      |  25%                     |                                                           |.............                                      |  26% [unnamed-chunk-57]  |                                                           |..............                                     |  27%                     |                                                           |...............                                    |  29% [unnamed-chunk-58]  |                                                           |...............                                    |  30%                     |                                                           |................                                   |  31% [unnamed-chunk-59]  |                                                           |................                                   |  32%                     |                                                           |.................                                  |  33% [unnamed-chunk-60]  |                                                           |.................                                  |  34%                     |                                                           |..................                                 |  35% [unnamed-chunk-61]  |                                                           |..................                                 |  36%                     |                                                           |...................                                |  37% [unnamed-chunk-62]  |                                                           |....................                               |  38%                     |                                                           |....................                               |  40% [unnamed-chunk-63]  |                                                           |.....................                              |  41%                     |                                                           |.....................                              |  42% [unnamed-chunk-64]  |                                                           |......................                             |  43%                     |                                                           |......................                             |  44% [unnamed-chunk-65]  |                                                           |.......................                            |  45%                     |                                                           |........................                           |  46% [unnamed-chunk-66]  |                                                           |........................                           |  47%                     |                                                           |.........................                          |  48% [unnamed-chunk-67]  |                                                           |.........................                          |  49%                     |                                                           |..........................                         |  51% [unnamed-chunk-68]  |                                                           |..........................                         |  52%                     |                                                           |...........................                        |  53% [unnamed-chunk-69]  |                                                           |...........................                        |  54%                     |                                                           |............................                       |  55% [unnamed-chunk-70]  |                                                           |.............................                      |  56%                     |                                                           |.............................                      |  57% [unnamed-chunk-71]  |                                                           |..............................                     |  58%                     |                                                           |..............................                     |  59% [unnamed-chunk-72]  |                                                           |...............................                    |  60%                     |                                                           |...............................                    |  62% [unnamed-chunk-73]  |                                                           |................................                   |  63%                     |                                                           |.................................                  |  64% [unnamed-chunk-74]  |                                                           |.................................                  |  65%                     |                                                           |..................................                 |  66% [unnamed-chunk-75]  |                                                           |..................................                 |  67%                     |                                                           |...................................                |  68% [unnamed-chunk-76]  |                                                           |...................................                |  69%                     |                                                           |....................................               |  70% [unnamed-chunk-77]  |                                                           |....................................               |  71%                     |                                                           |.....................................              |  73% [unnamed-chunk-78]  |                                                           |......................................             |  74%                     |                                                           |......................................             |  75% [unnamed-chunk-79]  |                                                           |.......................................            |  76%                     |                                                           |.......................................            |  77% [unnamed-chunk-80]  |                                                           |........................................           |  78%                     |                                                           |........................................           |  79% [unnamed-chunk-81]  |                                                           |.........................................          |  80%                     |                                                           |.........................................          |  81% [unnamed-chunk-82]  |                                                           |..........................................         |  82%                     |                                                           |...........................................        |  84% [unnamed-chunk-83]  |                                                           |...........................................        |  85%                     |                                                           |............................................       |  86% [unnamed-chunk-84]  |                                                           |............................................       |  87%                     |                                                           |.............................................      |  88% [unnamed-chunk-85]  |                                                           |.............................................      |  89%                     |                                                           |..............................................     |  90% [unnamed-chunk-86]  |                                                           |...............................................    |  91%                     |                                                           |...............................................    |  92% [unnamed-chunk-87]  |                                                           |................................................   |  93%                     |                                                           |................................................   |  95% [unnamed-chunk-88]  |                                                           |.................................................  |  96%                     |                                                           |.................................................  |  97% [unnamed-chunk-89]  |                                                           |.................................................. |  98%                     |                                                           |.................................................. |  99% [unnamed-chunk-90]  |                                                           |...................................................| 100%                   
    #> [1] "counts.R"
