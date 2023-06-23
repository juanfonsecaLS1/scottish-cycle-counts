
<!-- README.md is generated from README.Rmd. Please edit that file -->

# scottish-cycle-counts

<!-- badges: start -->
<!-- badges: end -->

The goal of scottish-cycle-counts is to read-in a process data on
cycling volumes in Scotland.

``` r
library(tidyverse)
```

The input dataset is a single .zip file:

``` r
zipped_data = list.files(pattern = ".zip")
zipped_data
#> [1] "June 2022-June 2023.zip"
```

We can unzip it as follows:

``` r
unzip(zipped_data, exdir = "data-raw")
files_csv = list.files("data-raw", pattern = ".csv", full.names = TRUE)
files_csv
#>  [1] "data-raw/Aberdeen City Council.csv"                            
#>  [2] "data-raw/Aberdeenshire Council.csv"                            
#>  [3] "data-raw/City of Edinburgh Council.csv"                        
#>  [4] "data-raw/Comhairle nan Eilean Siar (Western Isles Council).csv"
#>  [5] "data-raw/East Ayrshire Council.csv"                            
#>  [6] "data-raw/East Dunbartonshire Council.csv"                      
#>  [7] "data-raw/Glasgow City Council.csv"                             
#>  [8] "data-raw/John Muir Way.csv"                                    
#>  [9] "data-raw/National Cycle Nework (Scotland) - Sustrans.csv"      
#> [10] "data-raw/National Monitoring Framework - Cycling Scotland.csv" 
#> [11] "data-raw/North Ayrshire Council.csv"                           
#> [12] "data-raw/Perth and Kinross Council.csv"                        
#> [13] "data-raw/Scotland North East Trunk Roads.csv"                  
#> [14] "data-raw/Scotland North West Trunk Roads.csv"                  
#> [15] "data-raw/Scotland South East Trunk Roads.csv"                  
#> [16] "data-raw/Scotland South West Trunk Roads.csv"                  
#> [17] "data-raw/South Ayrshire Council.csv"                           
#> [18] "data-raw/South Lanarkshire Council.csv"                        
#> [19] "data-raw/Stirling Council.csv"                                 
#> [20] "data-raw/The Highland Council.csv"
```

We can read this file in R as follows:

``` r
# counts = arrow::open_dataset("data-raw")
counts = map_dfr(files_csv, read_csv)
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
  ) |>
  arrange(desc(count))
top_5_areas = head(area_counts, n = 5)
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

![](README_files/figure-gfm/unnamed-chunk-8-1.png)<!-- -->

    #> 1/18                   
    #> 2/18 [unnamed-chunk-10]
    #> 3/18                   
    #> 4/18 [unnamed-chunk-11]
    #> 5/18                   
    #> 6/18 [unnamed-chunk-12]
    #> 7/18                   
    #> 8/18 [unnamed-chunk-13]
    #> 9/18                   
    #> 10/18 [unnamed-chunk-14]
    #> 11/18                   
    #> 12/18 [unnamed-chunk-15]
    #> 13/18                   
    #> 14/18 [unnamed-chunk-16]
    #> 15/18                   
    #> 16/18 [unnamed-chunk-17]
    #> 17/18                   
    #> 18/18 [unnamed-chunk-18]
    #> [1] "counts.R"
