# Исследование метаданных DNS трафика
p.pluvkov@yandex.ru

## Цель работы

1.  Получить знания о методах исследования радиоэлектронной обстановки.
2.  Составить представление о механизмах работы Wi-Fi сетей на канальном
    и сетевом уровне модели OSI.
3.  Зекрепить практические навыки использования языка программирования R
    для обработки данных
4.  Закрепить знания основных функций обработки данных экосистемы
    tidyverse языка R

## Исходные данные

1.  Програмное обеспеченье MacOS
2.  RStudio
3.  Интерпретатор языка R 4.5.1
4.  Файл P2_wifi_data.csv

## План

Используя программный пакет dplyr, проанализировать DNS лог с помощью
языка программирования R.

## Ход работы

### Подготовка данных

1.  Импорт библиотек

``` r
library(tidyverse)
```

    Warning: package 'ggplot2' was built under R version 4.5.2

    Warning: package 'readr' was built under R version 4.5.2

    ── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ──
    ✔ dplyr     1.1.4     ✔ readr     2.1.6
    ✔ forcats   1.0.1     ✔ stringr   1.5.2
    ✔ ggplot2   4.0.1     ✔ tibble    3.3.0
    ✔ lubridate 1.9.4     ✔ tidyr     1.3.1
    ✔ purrr     1.2.0     
    ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ✖ dplyr::filter() masks stats::filter()
    ✖ dplyr::lag()    masks stats::lag()
    ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors

``` r
library(readr)
library(dplyr)
library(tidyr)
library(lubridate)
library(ggplot2)
```

1.  Загрузка данных

``` r
data_raw <- read_csv("P2_wifi_data.csv", skip = 1, col_names = FALSE)
```

    Warning: One or more parsing issues, call `problems()` on your data frame for details,
    e.g.:
      dat <- vroom(...)
      problems(dat)

    Rows: 12250 Columns: 15
    ── Column specification ────────────────────────────────────────────────────────
    Delimiter: ","
    chr (15): X1, X2, X3, X4, X5, X6, X7, X8, X9, X10, X11, X12, X13, X14, X15

    ℹ Use `spec()` to retrieve the full column specification for this data.
    ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
glimpse(data_raw)
```

    Rows: 12,250
    Columns: 15
    $ X1  <chr> "BSSID", "BE:F1:71:D5:17:8B", "6E:C7:EC:16:DA:1A", "9A:75:A8:B9:04…
    $ X2  <chr> "First time seen", "2023-07-28 09:13:03", "2023-07-28 09:13:03", "…
    $ X3  <chr> "Last time seen", "2023-07-28 11:50:50", "2023-07-28 11:55:12", "2…
    $ X4  <chr> "channel", "1", "1", "1", "7", "6", "6", "11", "11", "11", "1", "6…
    $ X5  <chr> "Speed", "195", "130", "360", "360", "130", "130", "195", "130", "…
    $ X6  <chr> "Privacy", "WPA2", "WPA2", "WPA2", "WPA2", "WPA2", "OPN", "WPA2", …
    $ X7  <chr> "Cipher", "CCMP", "CCMP", "CCMP", "CCMP", "CCMP", NA, "CCMP", "CCM…
    $ X8  <chr> "Authentication", "PSK", "PSK", "PSK", "PSK", "PSK", NA, "PSK", "P…
    $ X9  <chr> "Power", "-30", "-30", "-68", "-37", "-57", "-63", "-27", "-38", "…
    $ X10 <chr> "# beacons", "846", "750", "694", "510", "647", "251", "1647", "12…
    $ X11 <chr> "# IV", "504", "116", "26", "21", "6", "3430", "80", "11", "0", "0…
    $ X12 <chr> "LAN IP", "0.  0.  0.  0", "0.  0.  0.  0", "0.  0.  0.  0", "0.  …
    $ X13 <chr> "ID-length", "12", "4", "2", "14", "25", "13", "12", "13", "24", "…
    $ X14 <chr> "ESSID", "C322U13 3965", "Cnet", "KC", "POCO X5 Pro 5G", NA, "MIRE…
    $ X15 <chr> "Key", NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA,…

1.  Привести датасеты в вид “аккуратных данных”, преобразовать типы
    столбцов в соответствии с типом данных

2.  Просмотрите общую структуру данных с помощью функции glimpse()
