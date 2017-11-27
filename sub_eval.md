  
# Evaluation of subset data{.tabset}

Matched to nearest two sites in each category, +/- five years.

```r
knitr::opts_chunk$set(message = F, warning = F)

library(tidyverse)
library(ggmap)
library(lubridate)
library(geosphere)
library(stringi)
library(tibble)
library(raster)
library(sp)
library(rgdal)
library(foreach)
library(doParallel)

data(restdat)
data(reststat)
data(wqdat)
data(wqstat)
data(tbpoly)
data(allchg)

# source R files
source('R/get_chg.R')
source('R/get_clo.R')
source('R/get_cdt.R')
source('R/get_brk.R')
source('R/get_fin.R')
source('R/get_all.R')
source('R/rnd_dat.R')

# Set parameters, yr half-window for matching, mtch is number of closest matches
yrdf <- 5
mtch <- 2

# base map
ext <- make_bbox(reststat$lon, reststat$lat, f = 0.1)
map <- get_stamenmap(ext, zoom = 10, maptype = "toner-lite")
pbase <- ggmap(map) +
  theme_bw() +
  theme(
    axis.title.x = element_blank(),
    axis.title.y = element_blank()
  )
```

## Complete data


```r
# summarize observed
toplo <- allchg %>% 
  group_by(hab, wtr, salev) %>% 
  summarize(
    chvalmd = mean(chval, na.rm = T)
    )  %>% 
  unite('rest', hab, wtr, sep = ', ') %>% 
  mutate(
    salev = factor(salev, levels = c('lo', 'md', 'hi')) 
  )

# plot
ggplot(toplo, aes(x = rest, y = chvalmd)) + 
  theme_bw() + 
  theme(
    axis.title.y = element_blank()
  ) +
  geom_bar(stat = 'identity') +
  facet_wrap(~ salev, ncol = 1) + 
  coord_flip() +
  scale_y_continuous('chlorophyll', limits = c(0, 15))
```

![](sub_eval_files/figure-html/unnamed-chunk-1-1.png)<!-- -->


```r
# combine restoration locations, date, type
resgrp <- 'top'
restall <- left_join(restdat, reststat, by = 'id')
names(restall)[names(restall) %in% resgrp] <- 'Restoration\ngroup'

# map by restoration type
pbase +
  geom_point(data = restall, aes(x = lon, y = lat, fill = `Restoration\ngroup`), size = 4, pch = 21)
```

![](sub_eval_files/figure-html/unnamed-chunk-2-1.png)<!-- -->

```r
# map by date
pbase +
  geom_point(data = restall, aes(x = lon, y = lat, fill = factor(date)), size = 4, pch = 21)
```

![](sub_eval_files/figure-html/unnamed-chunk-2-2.png)<!-- -->

```r
# barplot of date counts
toplo <- restall %>% 
  group_by(date)
ggplot(restall, aes(x = factor(date))) + 
  geom_bar() + 
  coord_flip() + 
  theme_bw() + 
  theme(
    axis.title.y = element_blank()
  ) +
  scale_y_discrete(expand = c(0, 0))
```

![](sub_eval_files/figure-html/unnamed-chunk-2-3.png)<!-- -->

## Pre-1994 data


```r
# get sub data, restoration sites
restdat_sub <- restdat %>% 
  filter(date < 1994)
reststat_sub <- reststat %>% 
  filter(id %in% restdat_sub$id)

# run all conditional prob functions
allchg_pre <- get_all(restdat_sub, reststat_sub, wqdat, wqstat, mtch = mtch, yrdf = yrdf, resgrp = 'top', 
                      qts = c(0.33, 0.66), lbs = c('lo', 'md', 'hi'))

toplo <- allchg_pre %>% 
  group_by(hab, wtr, salev) %>% 
  summarize(
    chvalmd = mean(cval, na.rm = T)
    ) %>% 
  na.omit %>% 
  unite('rest', hab, wtr, sep = ', ') %>% 
  mutate(
    salev = factor(salev, levels = c('lo', 'md', 'hi')) 
  )


# plot
ggplot(toplo, aes(x = rest, y = chvalmd)) + 
  theme_bw() + 
  theme(
    axis.title.y = element_blank()
  ) +
  geom_bar(stat = 'identity') +
  facet_wrap(~ salev, ncol = 1) + 
  coord_flip() +
  scale_y_continuous('chlorophyll', limits = c(0,15))
```

![](sub_eval_files/figure-html/unnamed-chunk-3-1.png)<!-- -->


```r
# combine restoration locations, date, type
resgrp <- 'top'
restall_sub <- left_join(restdat_sub, reststat_sub, by = 'id')
names(restall_sub)[names(restall_sub) %in% resgrp] <- 'Restoration\ngroup'

# map by restoration type
pbase +
  geom_point(data = restall_sub, aes(x = lon, y = lat, fill = `Restoration\ngroup`), size = 4, pch = 21)
```

![](sub_eval_files/figure-html/unnamed-chunk-4-1.png)<!-- -->

```r
# map by date
pbase +
  geom_point(data = restall_sub, aes(x = lon, y = lat, fill = factor(date)), size = 4, pch = 21)
```

![](sub_eval_files/figure-html/unnamed-chunk-4-2.png)<!-- -->

```r
# barplot of date counts
toplo <- restall_sub %>% 
  group_by(date)
ggplot(restall_sub, aes(x = factor(date))) + 
  geom_bar() + 
  coord_flip() + 
  theme_bw() + 
  theme(
    axis.title.y = element_blank()
  ) +
  scale_y_discrete(expand = c(0, 0))
```

![](sub_eval_files/figure-html/unnamed-chunk-4-3.png)<!-- -->

## Post-1994 data


```r
# get sub data, restoration sites
restdat_sub <- restdat %>% 
  filter(date >= 1994)
reststat_sub <- reststat %>% 
  filter(id %in% restdat_sub$id)

# run all conditional prob functions
allchg_pst <- get_all(restdat_sub, reststat_sub, wqdat, wqstat, mtch = mtch, yrdf = yrdf, resgrp = 'top', qts = c(0.33, 0.66), 
                  lbs = c('lo', 'md', 'hi'))

# summarize
toplo <- allchg_pst %>% 
  group_by(hab, wtr, salev) %>% 
  summarize(
    chvalmd = mean(cval, na.rm = T)
    ) %>% 
  unite('rest', hab, wtr, sep = ', ') %>% 
  mutate(
    salev = factor(salev, levels = c('lo', 'md', 'hi')), 
    dat = 'Observed'
  )

# plot
ggplot(toplo, aes(x = rest, y = chvalmd)) + 
  theme_bw() + 
  theme(
    axis.title.y = element_blank()
  ) +
  geom_bar(stat = 'identity') +
  facet_wrap(~ salev, ncol = 1) + 
  coord_flip() +
  scale_y_continuous('chlorophyll', limits = c(0, 15))
```

![](sub_eval_files/figure-html/unnamed-chunk-5-1.png)<!-- -->


```r
# combine restoration locations, date, type
resgrp <- 'top'
restall_sub <- left_join(restdat_sub, reststat_sub, by = 'id')
names(restall_sub)[names(restall_sub) %in% resgrp] <- 'Restoration\ngroup'

# map by restoration type
pbase +
  geom_point(data = restall_sub, aes(x = lon, y = lat, fill = `Restoration\ngroup`), size = 4, pch = 21)
```

![](sub_eval_files/figure-html/unnamed-chunk-6-1.png)<!-- -->

```r
# map by date
pbase +
  geom_point(data = restall_sub, aes(x = lon, y = lat, fill = factor(date)), size = 4, pch = 21)
```

![](sub_eval_files/figure-html/unnamed-chunk-6-2.png)<!-- -->

```r
# barplot of date counts
toplo <- restall_sub %>% 
  group_by(date)
ggplot(restall_sub, aes(x = factor(date))) + 
  geom_bar() + 
  coord_flip() + 
  theme_bw() + 
  theme(
    axis.title.y = element_blank()
  ) +
  scale_y_discrete(expand = c(0, 0))
```

![](sub_eval_files/figure-html/unnamed-chunk-6-3.png)<!-- -->