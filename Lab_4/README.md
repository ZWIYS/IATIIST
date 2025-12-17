# Исследование метаданных DNS трафика
p.pluvkov@yandex.ru

## Цель работы

1.  Зекрепить практические навыки использования языка программирования R
    для обработки данных
2.  Закрепить знания основных функций обработки данных экосистемы
    tidyverse языка R
3.  Закрепить навыки исследования метаданных DNS трафика

## Исходные данные

1.  Програмное обеспеченье MacOS
2.  RStudio
3.  Интерпретатор языка R 4.5.1
4.  Файл dns.log

## План

Используя программный пакет dplyr, проанализировать DNS лог с помощью
языка программирования R.

## Подготовка данных

1.  Импорт библиотек

``` r
library(dplyr)
```


    Attaching package: 'dplyr'

    The following objects are masked from 'package:stats':

        filter, lag

    The following objects are masked from 'package:base':

        intersect, setdiff, setequal, union

``` r
library(readr)
```

    Warning: package 'readr' was built under R version 4.5.2

1.  Добавьте пропущенные данные о структуре данных (назначении столбцов)

``` r
col_names <- c( "ts", "uid", "id.orig_h", "id.orig_p", "id.resp_h", "id.resp_p", "proto", "trans_id", "query", "unknown1", "qclass", "qclass_name", "qtype", "qtype_name", "rcode", "rcode_name", "AA", "TC", "RD", "RA", "Z", "answers", "TTLs")
```

1.  Преобразуйте данные в столбцах в нужный формат

``` r
dns_log <- read_tsv( "dns.log", col_names = col_names, col_types = cols(.default = col_character(), ts = col_double(), id.orig_p = col_integer(), id.resp_p = col_integer(), trans_id = col_integer(), qclass_name = col_integer(), qtype_name = col_integer(), AA = col_logical(), TC = col_logical(), RD = col_logical(), RA = col_integer()), na = c("-", "(empty)", ""), comment = "#", show_col_types = FALSE)
```

1.  Просмотрите общую структуру данных с помощью функции glimpse()

``` r
glimpse(dns_log)
```

    Rows: 427,935
    Columns: 23
    $ ts          <dbl> 1331901006, 1331901015, 1331901016, 1331901017, 1331901006…
    $ uid         <chr> "CWGtK431H9XuaTN4fi", "C36a282Jljz7BsbGH", "C36a282Jljz7Bs…
    $ id.orig_h   <chr> "192.168.202.100", "192.168.202.76", "192.168.202.76", "19…
    $ id.orig_p   <int> 45658, 137, 137, 137, 137, 137, 137, 137, 137, 137, 137, 1…
    $ id.resp_h   <chr> "192.168.27.203", "192.168.202.255", "192.168.202.255", "1…
    $ id.resp_p   <int> 137, 137, 137, 137, 137, 137, 137, 137, 137, 137, 137, 137…
    $ proto       <chr> "udp", "udp", "udp", "udp", "udp", "udp", "udp", "udp", "u…
    $ trans_id    <int> 33008, 57402, 57402, 57402, 57398, 57398, 57398, 62187, 62…
    $ query       <chr> "*\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\…
    $ unknown1    <chr> "1", "1", "1", "1", "1", "1", "1", "1", "1", "1", "1", "1"…
    $ qclass      <chr> "C_INTERNET", "C_INTERNET", "C_INTERNET", "C_INTERNET", "C…
    $ qclass_name <int> 33, 32, 32, 32, 32, 32, 32, 32, 32, 32, 33, 33, 33, 12, 12…
    $ qtype       <chr> "SRV", "NB", "NB", "NB", "NB", "NB", "NB", "NB", "NB", "NB…
    $ qtype_name  <int> 0, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA,…
    $ rcode       <chr> "NOERROR", NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA,…
    $ rcode_name  <chr> "F", "F", "F", "F", "F", "F", "F", "F", "F", "F", "F", "F"…
    $ AA          <lgl> FALSE, FALSE, FALSE, FALSE, FALSE, FALSE, FALSE, FALSE, FA…
    $ TC          <lgl> FALSE, TRUE, TRUE, TRUE, TRUE, TRUE, TRUE, TRUE, TRUE, TRU…
    $ RD          <lgl> FALSE, FALSE, FALSE, FALSE, FALSE, FALSE, FALSE, FALSE, FA…
    $ RA          <int> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 1, 1, 1, 1, 0…
    $ Z           <chr> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA…
    $ answers     <chr> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA…
    $ TTLs        <chr> "F", "F", "F", "F", "F", "F", "F", "F", "F", "F", "F", "F"…

## Анализ данных

### Шаг 1. Cколько участников информационного обмена в сети Доброй Организации?

``` r
uni_src_ip <- unique(dns_log$id.orig_h)
uni_dst_ip <- unique(dns_log$id.resp_h)

ips <- unique(c(uni_src_ip, uni_dst_ip))

length(ips)
```

    [1] 1359

``` r
glimpse(ips)
```

     chr [1:1359] "192.168.202.100" "192.168.202.76" "192.168.202.89" ...

### Шаг 2. Какое соотношение участников обмена внутри сети и участников обращений к внешним ресурсам?

``` r
int_res <- sum(grepl("^(10\\.|192\\.168\\.|172\\.(1[6-9]|2[0-9]|3[0-1])\\.)", ips))
ext_res <- length(ips) - int_res
int_res / ext_res
```

    [1] 13.77174

### Шаг 3. Найдите топ-10 участников сети, проявляющих наибольшую сетевую активность.

``` r
dns_log %>% group_by(id.orig_h) %>% count(sort = TRUE) %>% head(10)
```

    # A tibble: 10 × 2
    # Groups:   id.orig_h [10]
       id.orig_h           n
       <chr>           <int>
     1 10.10.117.210   75943
     2 192.168.202.93  26522
     3 192.168.202.103 18121
     4 192.168.202.76  16978
     5 192.168.202.97  16176
     6 192.168.202.141 14967
     7 10.10.117.209   14222
     8 192.168.202.110 13372
     9 192.168.203.63  12148
    10 192.168.202.106 10784

### Шаг 4. Найдите топ-10 доменов, к которым обращаются пользователи сети и соответственное количество обращений

``` r
dns_log %>% group_by(query) %>% count(sort = TRUE) %>% head(10)
```

    # A tibble: 10 × 2
    # Groups:   query [10]
       query                                                                       n
       <chr>                                                                   <int>
     1 "teredo.ipv6.microsoft.com"                                             39273
     2 "tools.google.com"                                                      14057
     3 "www.apple.com"                                                         13390
     4 "time.apple.com"                                                        13109
     5 "safebrowsing.clients.google.com"                                       11658
     6 "*\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x… 10401
     7 "WPAD"                                                                   9134
     8 "44.206.168.192.in-addr.arpa"                                            7248
     9 "HPE8AA67"                                                               6929
    10 "ISATAP"                                                                 6569

### Шаг 5. Опеределите базовые статистические характеристики (функция summary() ) интервала времени между последовательными обращениями к топ-10 доменам.

``` r
time_analysis <- dns_log %>%
  count(query, name = "total_requests") %>%
  slice_max(total_requests, n = 10) %>%
  inner_join(dns_log, by = "query") %>%
  group_by(query) %>%
  arrange(ts, .by_group = TRUE) %>%
  summarise(
    requests = n(),
    intervals = list(diff(sort(ts))),
    .groups = 'drop') %>% mutate(
    min_int = sapply(intervals, min),
    q1_int = sapply(intervals, quantile, 0.25),
    median_int = sapply(intervals, median),
    mean_int = sapply(intervals, mean),
    q3_int = sapply(intervals, quantile, 0.75),
    max_int = sapply(intervals, max),
    sd_int = sapply(intervals, sd)
  ) %>%
  select(-intervals) %>%
  arrange(median_int)

time_analysis
```

    # A tibble: 10 × 9
       query       requests min_int q1_int median_int mean_int q3_int max_int sd_int
       <chr>          <int>   <dbl>  <dbl>      <dbl>    <dbl>  <dbl>   <dbl>  <dbl>
     1 "teredo.ip…    39273       0  0          0         2.94  0.510  50388.   256.
     2 "tools.goo…    14057       0  0          0         8.19  1      50365.   428.
     3 "*\\x00\\x…    10401       0  0.150      0.5      11.2   1.5    52724.   521.
     4 "HPE8AA67"      6929       0  0.75       0.75     16.6  25.5    50044.   602.
     5 "WPAD"          9134       0  0.75       0.75     12.6   1.11   50049.   524.
     6 "ISATAP"        6569       0  0.75       0.760    17.5   1.05   51998.   656.
     7 "safebrows…    11658       0  0          1        10.0   2.01   49952.   464.
     8 "www.apple…    13390       0  0          1         8.61  3.01   50964.   443.
     9 "time.appl…    13109       0  0.370      1.76      8.67  4.72   50924.   446.
    10 "44.206.16…     7248       0  2.09       4        16.0  20.1    49680.   584.

### Шан 6. По периодическим запросам на один и тот же домен можно выявить скрытый DNS канал. Есть ли такие IP адреса в исследуемом датасете?

``` r
periodic_ips <- dns_log %>%
  semi_join(
    count(., query) %>% slice_max(n, n = 10),
    by = "query"
  ) %>%
  group_by(id.orig_h, query) %>%
  filter(n() >= 5) %>%
  arrange(ts) %>%
  summarise(
    request_count = n(),
    time_intervals = list(diff(ts)),
    mean_interval = mean(diff(ts)),
    sd_interval = sd(diff(ts)),
    cv = sd_interval / mean_interval,
    min_interval = min(diff(ts)),
    max_interval = max(diff(ts)),
    range_ratio = max_interval / min_interval,
    .groups = 'drop'
  ) %>%
  
  filter(
    cv < 0.5, request_count >= 10, range_ratio < 10) %>%
  arrange(cv, request_count)

periodic_ips
```

    # A tibble: 3 × 10
      id.orig_h   query request_count time_intervals mean_interval sd_interval    cv
      <chr>       <chr>         <int> <list>                 <dbl>       <dbl> <dbl>
    1 192.168.20… ISAT…            90 <dbl [89]>             0.767       0.121 0.158
    2 192.168.20… ISAT…            33 <dbl [32]>             0.862       0.147 0.171
    3 192.168.0.3 ISAT…           108 <dbl [107]>            0.874       0.313 0.358
    # ℹ 3 more variables: min_interval <dbl>, max_interval <dbl>, range_ratio <dbl>

### Шаг 7. Определите местоположение (страну, город) и организацию-провайдера для топ-10 доменов.

``` r
# Определяем функцию get_geo_info_debug
get_geo_info_debug <- function(ip) {
  cat("  Запрашиваю IP:", ip, "... ")
  
  if(grepl(":", ip)) {
    cat("IPv6 - пропускаю\n")
    return(NULL)
  }
  
  if(is.na(ip) || ip == "") {
    cat("Пустой IP\n")
    return(NULL)
  }
  
  tryCatch({
    response <- GET(paste0("http://ip-api.com/json/", ip), 
                    timeout(5))
    
    cat("Статус:", status_code(response), "... ")
    
    if(status_code(response) == 200) {
      info <- content(response, "parsed")
      
      if(info$status == "success") {
        cat("Успех\n")
        return(data.frame(
          ip = ip,
          country = info$country,
          city = ifelse(!is.null(info$city), info$city, NA),
          isp = ifelse(!is.null(info$isp), info$isp, NA),
          org = ifelse(!is.null(info$org), info$org, NA),
          stringsAsFactors = FALSE
        ))
      } else {
        cat("Ошибка API:", info$message, "\n")
      }
    } else {
      cat("HTTP ошибка\n")
    }
    
  }, error = function(e) {
    cat("Ошибка:", e$message, "\n")
  })
  
  return(NULL)
}

top_domains <- dns_log %>%
  count(query, sort = TRUE) %>%
  head(10)

top_domains
```

    # A tibble: 10 × 2
       query                                                                       n
       <chr>                                                                   <int>
     1 "teredo.ipv6.microsoft.com"                                             39273
     2 "tools.google.com"                                                      14057
     3 "www.apple.com"                                                         13390
     4 "time.apple.com"                                                        13109
     5 "safebrowsing.clients.google.com"                                       11658
     6 "*\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x… 10401
     7 "WPAD"                                                                   9134
     8 "44.206.168.192.in-addr.arpa"                                            7248
     9 "HPE8AA67"                                                               6929
    10 "ISATAP"                                                                 6569

## Оценка результата

1.  Зекрепили практические навыки использования языка программирования R
    для обработки данных
2.  Закрепили знания основных функций обработки данных экосистемы
    tidyverse языка R
3.  Закрепили навыки исследования метаданных DNS трафика

## Вывод

В рамках работы, используя программный пакет dplyr, проанализировали DNS
лог с помощью языка программирования R.
