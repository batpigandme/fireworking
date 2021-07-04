
<!-- README.md is generated from README.Rmd. Please edit that file -->

# 🎆 fireworking

Playing around with data from U.S. Consumer Products Safety Commission
(CPSC) [2020 Fireworks Annual
Report](https://www.cpsc.gov/s3fs-public/2020-Fireworks-Annual-Report.pdf).[1]

``` r
library(tidyverse)
library(tabulizer)
library(pdftools)
# plus some bonus appearances from other packages,
# e.g. {janitor}, {hrbrthemes}, and {scales}
```

The report is available online as a pdf, so let’s tell R about that.

``` r
pdf_file <- "https://www.cpsc.gov/s3fs-public/2020-Fireworks-Annual-Report.pdf"
```

Now we’ll extract the table on p.14 of the report using
[{tabulizer}](https://github.com/ropensci/tabulizer), and do a little
cleaning up with an assist from
[{janitor}](https://sfirke.github.io/janitor/index.html).

``` r
p14_tables <- extract_tables(pdf_file, pages = 14, output = "data.frame")

# returns a list, so take first item from list
injuries_by_year <- p14_tables[[1]]

injuries_by_year <- injuries_by_year %>%
  as_tibble() %>%
  janitor::clean_names() %>%
  mutate(estimated_injuries = as.numeric(str_remove(estimated_injuries, ","))) %>%
  rename("injuries_per_100000" = "injuries_per_100_000_people")

injuries_by_year
#> # A tibble: 16 x 3
#>     year estimated_injuries injuries_per_100000
#>    <int>              <dbl>               <dbl>
#>  1  2020              15600                 4.7
#>  2  2019              10000                 3  
#>  3  2018               9100                 2.8
#>  4  2017              12900                 4  
#>  5  2016              11100                 3.4
#>  6  2015              11900                 3.7
#>  7  2014              10500                 3.3
#>  8  2013              11400                 3.6
#>  9  2012               8700                 2.8
#> 10  2011               9600                 3.1
#> 11  2010               8600                 2.8
#> 12  2009               8800                 2.9
#> 13  2008               7000                 2.3
#> 14  2007               9800                 3.3
#> 15  2006               9200                 3.1
#> 16  2005              10800                 3.7
```

Now let’s plot it to see the general trend visually.

``` r
ggplot(injuries_by_year, aes(x = year, y = estimated_injuries)) +
  geom_point() +
  scale_y_continuous(label = scales::label_comma()) +
  labs(title = "Estimated fireworks-related injuries 2005-2020",
       caption = "source: NEISS, U.S. CPSC",
       alt = paste("A scatterplot showing an increasing trend in",
                   "estimated-injuries from 2008 to 2020")) +
  ylab("estimated injuries") +
  hrbrthemes::theme_ipsum()
```

<img src="README_files/figure-gfm/injuries-by-year-1.png" title="Estimated fireworks-related injuries from 2005 to 2020: A scatterplot showing an increasing trend in estimated-injuries from 2008 to 2020" alt="Estimated fireworks-related injuries from 2005 to 2020: A scatterplot showing an increasing trend in estimated-injuries from 2008 to 2020"  />

Looks like 2020 was a big year for getting injured with fireworks (which
is probably a surprise to no one).

Let’s see if we can get the data from one of the more complicated (read:
multiple-headered, non-tidy) tables, like the one on page 24 breaking
down 2020 injuries by body region and type.

``` r
p24_tables <- extract_tables(pdf_file, pages = 24, output = "data.frame")

inj_by_body_region <- tibble(p24_tables[[1]]) # tibble for the printing

inj_by_body_region
#> # A tibble: 10 x 5
#>    X             X.1      X.2    Diagnosis                         X.3          
#>    <chr>         <chr>    <chr>  <chr>                             <chr>        
#>  1 "Body Region" "Total"  "Burn… "Contusions/ Lacerations Fractur… "Other Diagn…
#>  2 ""            ""       ""     ""                                ""           
#>  3 "Total"       "10,300" "4,50… "2,000 900"                       "2,900"      
#>  4 ""            ""       ""     ""                                ""           
#>  5 "Arm"         "1,200"  "800"  "100 300"                         "100"        
#>  6 "Eye"         "1,500"  "200"  "500 *"                           "700"        
#>  7 "Head/Face/E… "2,300"  "500"  "900 200"                         "700"        
#>  8 "Hand/Finger" "3,100"  "2,00… "200 400"                         "600"        
#>  9 "Leg"         "1,400"  "700"  "300 *"                           "400"        
#> 10 "Trunk/Other" "800"    "300"  "100 *"                           "400"
```

OK, it’s not the worst thing in the world, but it needs some fixing up.
We’ve got the headers we *actually* want in the first row, and we’ve got
two sets of values in the row currently labelled “Diagnosis”. I’m sure
the latter problem could be dealt with in a nicer way using regular
expressions, or something of the sort, but I’m gonna cheat and just
replace it to make my life easier.

``` r
inj_by_body_region[1,4] <- "Contusions/Lacerations Fractures/Sprains"
```

Since [{janitor}](https://sfirke.github.io/janitor) has a bunch of
functions that are handy for cleaning things like this up, let’s go
ahead and just load the library. I’m also going to use `separate()` on
the empty (-looking) rows to my advantage, so I can filter them out
later with `is.na()`.

``` r
library(janitor)

inj_by_body_region %>%
  separate(Diagnosis, c("diagnosis1", "diagnosis2"), sep = " ") %>%
  filter(!is.na(diagnosis2)) %>%
  row_to_names(1) %>%
  clean_names() -> inj_body_region
#> Warning: Expected 2 pieces. Missing pieces filled with `NA` in 2 rows [2, 4].

inj_body_region
#> # A tibble: 7 x 6
#>   body_region  total  burns contusions_lacerat… fractures_sprai… other_diagnoses
#>   <chr>        <chr>  <chr> <chr>               <chr>            <chr>          
#> 1 Total        10,300 4,500 2,000               900              2,900          
#> 2 Arm          1,200  800   100                 300              100            
#> 3 Eye          1,500  200   500                 *                700            
#> 4 Head/Face/E… 2,300  500   900                 200              700            
#> 5 Hand/Finger  3,100  2,000 200                 400              600            
#> 6 Leg          1,400  700   300                 *                400            
#> 7 Trunk/Other  800    300   100                 *                400
```

Looking pretty good! Even though the `*` denotes “estimates of fewer
than 50 injuries,” I’m going to replace them with zeroes so that we can
treat the column (and all the other number-y ones) as numeric. We’ll
also ditch the row with “Total” listed as a `body_region`, since it’s
not one, as well as the `total` column, since we know how to add.

``` r
inj_body_region %>%
  mutate("fractures_sprains" = replace(fractures_sprains, fractures_sprains == "*", "0")) %>%
  mutate(across(-body_region, ~ as.numeric(str_remove(.x, ",")))) %>%
  filter(body_region != "Total") %>%
  select(-total) -> inj_body_region2

inj_body_region2
#> # A tibble: 6 x 5
#>   body_region   burns contusions_lacerations fractures_sprains other_diagnoses
#>   <chr>         <dbl>                  <dbl>             <dbl>           <dbl>
#> 1 Arm             800                    100               300             100
#> 2 Eye             200                    500                 0             700
#> 3 Head/Face/Ear   500                    900               200             700
#> 4 Hand/Finger    2000                    200               400             600
#> 5 Leg             700                    300                 0             400
#> 6 Trunk/Other     300                    100                 0             400
```

Our data *still* isn’t quite in the tidy form I want (though it does
make for a nice, concise-looking table). Let’s `pivot_longer()` to make
the type of injury a variable/column.

``` r
tidy_inj_body_region <- inj_body_region2 %>%
  pivot_longer(!body_region, names_to = "injury", values_to = "count")

tidy_inj_body_region
#> # A tibble: 24 x 3
#>    body_region   injury                 count
#>    <chr>         <chr>                  <dbl>
#>  1 Arm           burns                    800
#>  2 Arm           contusions_lacerations   100
#>  3 Arm           fractures_sprains        300
#>  4 Arm           other_diagnoses          100
#>  5 Eye           burns                    200
#>  6 Eye           contusions_lacerations   500
#>  7 Eye           fractures_sprains          0
#>  8 Eye           other_diagnoses          700
#>  9 Head/Face/Ear burns                    500
#> 10 Head/Face/Ear contusions_lacerations   900
#> # … with 14 more rows
```

At this point, do I regret making those clean column names? Maybe a
little…but I really hate dealing with backticks. Plus, I could always go
back in and make things pretty now.

------------------------------------------------------------------------

[1] Marier, A., Smith, B., Lee, S. (2021, June). *2020 Fireworks Annual
Report: Fireworks-related deaths, emergency depertment-treated injuries,
and enforcement activities during 2020*. U.S. Consumer Product Safety
Commission.
<https://www.cpsc.gov/s3fs-public/2020-Fireworks-Annual-Report.pdf>
