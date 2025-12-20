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
fields <- fields <- c(
  "timestamp", "uid", "src_ip", "src_port", "dst_ip", "dst_port",
  "protocol", "transaction_id", "domain", "unused",
  "qclass", "qclass_id", "qtype", "qtype_id",
  "rcode", "rcode_name", "flag_AA", "flag_TC",
  "flag_RD", "flag_RA", "flag_Z", "answers", "ttls"
)
```

1.  Преобразуйте данные в столбцах в нужный формат

``` r
dns_data <- read_tsv(
  file = "dns.log",
  col_names = fields,
  col_types = cols(
    .default = col_character(),
    timestamp = col_double(),
    src_port = col_integer(),
    dst_port = col_integer(),
    transaction_id = col_integer(),
    qclass_id = col_integer(),
    qtype_id = col_integer(),
    flag_AA = col_logical(),
    flag_TC = col_logical(),
    flag_RD = col_logical(),
    flag_RA = col_integer()
  ),
  na = c("-", "(empty)", ""),
  comment = "#",
  show_col_types = FALSE
)
```

1.  Просмотрите общую структуру данных с помощью функции glimpse()

``` r
glimpse(dns_data)
```

    Rows: 427,935
    Columns: 23
    $ timestamp      <dbl> 1331901006, 1331901015, 1331901016, 1331901017, 1331901…
    $ uid            <chr> "CWGtK431H9XuaTN4fi", "C36a282Jljz7BsbGH", "C36a282Jljz…
    $ src_ip         <chr> "192.168.202.100", "192.168.202.76", "192.168.202.76", …
    $ src_port       <int> 45658, 137, 137, 137, 137, 137, 137, 137, 137, 137, 137…
    $ dst_ip         <chr> "192.168.27.203", "192.168.202.255", "192.168.202.255",…
    $ dst_port       <int> 137, 137, 137, 137, 137, 137, 137, 137, 137, 137, 137, …
    $ protocol       <chr> "udp", "udp", "udp", "udp", "udp", "udp", "udp", "udp",…
    $ transaction_id <int> 33008, 57402, 57402, 57402, 57398, 57398, 57398, 62187,…
    $ domain         <chr> "*\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x…
    $ unused         <chr> "1", "1", "1", "1", "1", "1", "1", "1", "1", "1", "1", …
    $ qclass         <chr> "C_INTERNET", "C_INTERNET", "C_INTERNET", "C_INTERNET",…
    $ qclass_id      <int> 33, 32, 32, 32, 32, 32, 32, 32, 32, 32, 33, 33, 33, 12,…
    $ qtype          <chr> "SRV", "NB", "NB", "NB", "NB", "NB", "NB", "NB", "NB", …
    $ qtype_id       <int> 0, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, …
    $ rcode          <chr> "NOERROR", NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, …
    $ rcode_name     <chr> "F", "F", "F", "F", "F", "F", "F", "F", "F", "F", "F", …
    $ flag_AA        <lgl> FALSE, FALSE, FALSE, FALSE, FALSE, FALSE, FALSE, FALSE,…
    $ flag_TC        <lgl> FALSE, TRUE, TRUE, TRUE, TRUE, TRUE, TRUE, TRUE, TRUE, …
    $ flag_RD        <lgl> FALSE, FALSE, FALSE, FALSE, FALSE, FALSE, FALSE, FALSE,…
    $ flag_RA        <int> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 1, 1, 1, 1…
    $ flag_Z         <chr> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA,…
    $ answers        <chr> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA,…
    $ ttls           <chr> "F", "F", "F", "F", "F", "F", "F", "F", "F", "F", "F", …

## Анализ данных

### Шаг 1. Cколько участников информационного обмена в сети Доброй Организации?

``` r
all_hosts <- dns_data %>%
  select(src_ip, dst_ip) %>%
  unlist(use.names = FALSE) %>%
  unique()

length(all_hosts)
```

    [1] 1359

``` r
glimpse(all_hosts)
```

     chr [1:1359] "192.168.202.100" "192.168.202.76" "192.168.202.89" ...

### Шаг 2. Какое соотношение участников обмена внутри сети и участников обращений к внешним ресурсам?

``` r
internal_cnt <- sum(grepl("^(10\\.|192\\.168\\.|172\\.(1[6-9]|2[0-9]|3[0-1])\\.)", all_hosts))
external_cnt <- length(all_hosts) - internal_cnt

internal_cnt / external_cnt
```

    [1] 13.77174

### Шаг 3. Найдите топ-10 участников сети, проявляющих наибольшую сетевую активность.

``` r
dns_data %>%
  count(src_ip, sort = TRUE) %>%
  slice_head(n = 10)
```

    # A tibble: 10 × 2
       src_ip              n
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
dns_data %>%
  count(domain, sort = TRUE) %>%
  slice_head(n = 10)
```

    # A tibble: 10 × 2
       domain                                                                      n
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
interval_stats <- dns_data %>%
  count(domain, name = "req_total") %>%
  slice_max(req_total, n = 10) %>%
  left_join(dns_data, by = "domain") %>%
  arrange(domain, timestamp) %>%
  group_by(domain) %>%
  summarise(
    min_gap = min(diff(timestamp)),
    q1_gap = quantile(diff(timestamp), 0.25),
    median_gap = median(diff(timestamp)),
    mean_gap = mean(diff(timestamp)),
    q3_gap = quantile(diff(timestamp), 0.75),
    max_gap = max(diff(timestamp)),
    sd_gap = sd(diff(timestamp)),
    .groups = "drop"
  ) %>%
  arrange(median_gap)

interval_stats
```

    # A tibble: 10 × 8
       domain               min_gap q1_gap median_gap mean_gap q3_gap max_gap sd_gap
       <chr>                  <dbl>  <dbl>      <dbl>    <dbl>  <dbl>   <dbl>  <dbl>
     1 "teredo.ipv6.micros…       0  0          0         2.94  0.510  50388.   256.
     2 "tools.google.com"         0  0          0         8.19  1      50365.   428.
     3 "*\\x00\\x00\\x00\\…       0  0.150      0.5      11.2   1.5    52724.   521.
     4 "HPE8AA67"                 0  0.75       0.75     16.6  25.5    50044.   602.
     5 "WPAD"                     0  0.75       0.75     12.6   1.11   50049.   524.
     6 "ISATAP"                   0  0.75       0.760    17.5   1.05   51998.   656.
     7 "safebrowsing.clien…       0  0          1        10.0   2.01   49952.   464.
     8 "www.apple.com"            0  0          1         8.61  3.01   50964.   443.
     9 "time.apple.com"           0  0.370      1.76      8.67  4.72   50924.   446.
    10 "44.206.168.192.in-…       0  2.09       4        16.0  20.1    49680.   584.

### Шан 6. По периодическим запросам на один и тот же домен можно выявить скрытый DNS канал. Есть ли такие IP адреса в исследуемом датасете?

``` r
tmp_hosts <- dns_data %>%
  semi_join(
    dns_data %>% count(domain) %>% slice_max(n, n = 10),
    by = "domain"
  ) %>%
  group_by(src_ip, domain) %>%
  filter(n() >= 5) %>%
  arrange(timestamp) %>%
  summarise(
    total = n(),
    avg_int = mean(diff(timestamp)),
    sd_int = sd(diff(timestamp)),
    cv = sd_int / avg_int,
    ratio = max(diff(timestamp)) / min(diff(timestamp)),
    .groups = "drop"
  ) %>%
  filter(cv < 0.5, total >= 10, ratio < 10) %>%
  arrange(cv)

tmp_hosts
```

    # A tibble: 3 × 7
      src_ip          domain total avg_int sd_int    cv ratio
      <chr>           <chr>  <int>   <dbl>  <dbl> <dbl> <dbl>
    1 192.168.202.49  ISATAP    90   0.767  0.121 0.158  2.47
    2 192.168.202.147 ISATAP    33   0.862  0.147 0.171  1.42
    3 192.168.0.3     ISATAP   108   0.874  0.313 0.358  5.04

### Шаг 7. Определите местоположение (страну, город) и организацию-провайдера для топ-10 доменов.

``` r
# Определяем функцию get_geo_info_debug
get_geo_info <- function(ip) {
  if (is.na(ip) || ip == "" || grepl(":", ip)) return(NULL)
  tryCatch({
    r <- httr::GET(paste0("http://ip-api.com/json/", ip), httr::timeout(5))
    if (httr::status_code(r) != 200) return(NULL)
    d <- httr::content(r, "parsed")
    if (d$status != "success") return(NULL)
    data.frame(
      ip = ip,
      country = d$country,
      city = d$city,
      isp = d$isp,
      org = d$org
    )
  }, error = function(e) NULL)
}

popular_domains <- dns_data %>% count(domain, sort = TRUE) %>% slice_head(n = 10)
popular_domains
```

    # A tibble: 10 × 2
       domain                                                                      n
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
