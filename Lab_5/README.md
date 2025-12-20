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

Используя программные пакеты dplyr и tidyverse проанализировать журналы
tcpdump и airodump-ng с помощью языка программирования R.

## Подготовка данных

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
library(lubridate)
library(dplyr)
library(stringr)
```

1.  Загрузка и подготовка данных

``` r
# Чтение данных о точках доступа
access_points <- read_csv(
  'P2_wifi_data.csv',
  n_max = 167,
  col_types = cols(
    .default = col_character(),
    First.time.seen = col_datetime(format = "%Y-%m-%d %H:%M:%S"),
    Last.time.seen = col_datetime(format = "%Y-%m-%d %H:%M:%S"),
    Channel = col_integer(),
    Speed = col_number(),
    Power = col_number(),
    `# beacons` = col_integer(),
    `# IV` = col_number()
  )
)
```

    Warning: The following named parsers don't match the column names:
    First.time.seen, Last.time.seen, Channel

``` r
# Чтение данных о клиентах
client_requests <- read_csv(
  'P2_wifi_data.csv',
  skip = 169,
  col_types = cols(
    .default = col_character(),
    First.time.seen = col_datetime(format = "%Y-%m-%d %H:%M:%S"),
    Last.time.seen = col_datetime(format = "%Y-%m-%d %H:%M:%S"),
    Power = col_number(),
    `# packets` = col_integer()
  )
)
```

    Warning: The following named parsers don't match the column names:
    First.time.seen, Last.time.seen

    Warning: One or more parsing issues, call `problems()` on your data frame for details,
    e.g.:
      dat <- vroom(...)
      problems(dat)

``` r
# Переименование столбцов для удобства
access_points <- access_points %>%
  rename(
    beacons = `# beacons`,
    iv_count = `# IV`
  )

client_requests <- client_requests %>%
  rename(
    packets = `# packets`
  )

glimpse(access_points)
```

    Rows: 167
    Columns: 15
    $ BSSID             <chr> "BE:F1:71:D5:17:8B", "6E:C7:EC:16:DA:1A", "9A:75:A8:…
    $ `First time seen` <chr> "2023-07-28 09:13:03", "2023-07-28 09:13:03", "2023-…
    $ `Last time seen`  <chr> "2023-07-28 11:50:50", "2023-07-28 11:55:12", "2023-…
    $ channel           <chr> "1", "1", "1", "7", "6", "6", "11", "11", "11", "1",…
    $ Speed             <dbl> 195, 130, 360, 360, 130, 130, 195, 130, 130, 195, 18…
    $ Privacy           <chr> "WPA2", "WPA2", "WPA2", "WPA2", "WPA2", "OPN", "WPA2…
    $ Cipher            <chr> "CCMP", "CCMP", "CCMP", "CCMP", "CCMP", NA, "CCMP", …
    $ Authentication    <chr> "PSK", "PSK", "PSK", "PSK", "PSK", NA, "PSK", "PSK",…
    $ Power             <dbl> -30, -30, -68, -37, -57, -63, -27, -38, -38, -66, -4…
    $ beacons           <int> 846, 750, 694, 510, 647, 251, 1647, 1251, 704, 617, …
    $ iv_count          <dbl> 504, 116, 26, 21, 6, 3430, 80, 11, 0, 0, 86, 0, 0, 0…
    $ `LAN IP`          <chr> "0.  0.  0.  0", "0.  0.  0.  0", "0.  0.  0.  0", "…
    $ `ID-length`       <chr> "12", "4", "2", "14", "25", "13", "12", "13", "24", …
    $ ESSID             <chr> "C322U13 3965", "Cnet", "KC", "POCO X5 Pro 5G", NA, …
    $ Key               <chr> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, …

``` r
glimpse(client_requests)
```

    Rows: 12,081
    Columns: 7
    $ `Station MAC`     <chr> "CA:66:3B:8F:56:DD", "96:35:2D:3D:85:E6", "5C:3A:45:…
    $ `First time seen` <chr> "2023-07-28 09:13:03", "2023-07-28 09:13:03", "2023-…
    $ `Last time seen`  <chr> "2023-07-28 10:59:44", "2023-07-28 09:13:03", "2023-…
    $ Power             <dbl> -33, -65, -39, -61, -53, -43, -31, -71, -74, -65, -4…
    $ packets           <int> 858, 4, 432, 958, 1, 344, 163, 3, 115, 437, 265, 77,…
    $ BSSID             <chr> "BE:F1:71:D5:17:8B", "(not associated)", "BE:F1:71:D…
    $ `Probed ESSIDs`   <chr> "C322U13 3965", "IT2 Wireless", "C322U21 0566", "C32…

## Анализ данныз

### Шаг 1. Определить небезопасные точки доступа (без шифрования – OPN)

``` r
AP_nOPN <- access_points %>% filter(str_detect(toupper(Privacy), "OPN")) %>% select(BSSID, ESSID)
AP_nOPN
```

    # A tibble: 42 × 2
       BSSID             ESSID        
       <chr>             <chr>        
     1 E8:28:C1:DC:B2:52 MIREA_HOTSPOT
     2 E8:28:C1:DC:B2:50 MIREA_GUESTS 
     3 E8:28:C1:DC:B2:51 <NA>         
     4 E8:28:C1:DC:FF:F2 <NA>         
     5 00:25:00:FF:94:73 <NA>         
     6 E8:28:C1:DD:04:52 MIREA_HOTSPOT
     7 E8:28:C1:DE:74:31 <NA>         
     8 E8:28:C1:DE:74:32 MIREA_HOTSPOT
     9 E8:28:C1:DC:C8:32 MIREA_HOTSPOT
    10 E8:28:C1:DD:04:50 MIREA_GUESTS 
    # ℹ 32 more rows

### Шаг 2. Определить производителя для каждого обнаруженного устройства

``` r
oui_raw <- readLines("oui.txt")
pattern <- "^([0-9A-Fa-f]{2}-[0-9A-Fa-f]{2}-[0-9A-Fa-f]{2})\\s+\\(hex\\)\\s+(.+)$"
matches <- regexec(pattern, oui_raw)
parsed <- regmatches(oui_raw, matches)
parsed <- parsed[sapply(parsed, length) == 3]
oui_df <- data.frame(
  OUI = toupper(gsub("-", ":", sapply(parsed, `[`, 2))),
  Manufacturer = sapply(parsed, `[`, 3),
  stringsAsFactors = FALSE
)

AP_nOPN$OUI <- toupper(substr(AP_nOPN$BSSID, 1, 8))
AP_nOPN <- merge(AP_nOPN, oui_df, by="OUI", all.x=TRUE)
AP_nOPN$Manufacturer[is.na(AP_nOPN$Manufacturer)] <- "Unknown"
head(AP_nOPN)
```

           OUI             BSSID   ESSID                 Manufacturer
    1 00:03:7A 00:03:7A:1A:03:56 MT_FREE        Taiyo Yuden Co., Ltd.
    2 00:03:7F 00:03:7F:12:34:56 MT_FREE Atheros Communications, Inc.
    3 00:25:00 00:25:00:FF:94:73    <NA>                  Apple, Inc.
    4 00:26:99 00:26:99:F2:7A:E0    <NA>           Cisco Systems, Inc
    5 00:26:99 00:26:99:F2:7A:EF    <NA>           Cisco Systems, Inc
    6 00:3E:1A 00:3E:1A:5D:14:45 MT_FREE                      Unknown

### Шаг 3. Выявить устройства, использующие последнюю версию протокола шифрования WPA3, и названия точек доступа, реализованных на этих устройствах

``` r
wpa3_devices <- access_points %>%
  filter(str_detect(toupper(Privacy), "WPA3")) %>%
  select(BSSID, ESSID, Privacy, Cipher, Speed, channel) %>%
  arrange(desc(Speed))

  print(wpa3_devices)
```

    # A tibble: 8 × 6
      BSSID             ESSID                Privacy   Cipher Speed channel
      <chr>             <chr>                <chr>     <chr>  <dbl> <chr>  
    1 26:20:53:0C:98:E8 <NA>                 WPA3 WPA2 CCMP     866 44     
    2 96:FF:FC:91:EF:64 <NA>                 WPA3 WPA2 CCMP     866 44     
    3 CE:48:E7:86:4E:33 iPhone (Анастасия)   WPA3 WPA2 CCMP     866 44     
    4 8E:1F:94:96:DA:FD iPhone (Анастасия)   WPA3 WPA2 CCMP     866 44     
    5 A2:FE:FF:B8:9B:C9 Christie’s           WPA3 WPA2 CCMP     130 6      
    6 BE:FD:EF:18:92:44 Димасик              WPA3 WPA2 CCMP     130 6      
    7 3A:DA:00:F9:0C:02 iPhone XS Max 🦊🐱🦊 WPA3 WPA2 CCMP     130 6      
    8 76:C5:A0:70:08:96 <NA>                 WPA3 WPA2 CCMP     130 6      

### Шаг 4. Отсортировать точки доступа по интервалу времени, в течение которого они находились на связи, по убыванию.

``` r
print(sapply(access_points, class)[grepl("time|Time|seen|Seen", names(access_points))])
```

    First time seen  Last time seen 
        "character"     "character" 

``` r
access_points_times <- access_points %>%
  mutate(
    first_seen = as.POSIXct(`First time seen`, format = "%Y-%m-%d %H:%M:%S"),
    last_seen = as.POSIXct(`Last time seen`, format = "%Y-%m-%d %H:%M:%S")
  )


ap_activity <- access_points_times %>%
  mutate(
    activity_duration = as.numeric(
      difftime(last_seen, first_seen, units = "hours")
    )
  ) %>%
  filter(activity_duration > 0) %>%
  select(BSSID, ESSID, first_seen, last_seen, activity_duration, channel) %>%
  arrange(desc(activity_duration))

ap_activity %>%
  head(10) %>%
  mutate(
    activity_duration = round(activity_duration, 2),
    first_time = format(first_seen, "%H:%M"),
    last_time = format(last_seen, "%H:%M")
  ) %>%
  select(BSSID, ESSID, channel, first_time, last_time, activity_duration)
```

    # A tibble: 10 × 6
       BSSID             ESSID        channel first_time last_time activity_duration
       <chr>             <chr>        <chr>   <chr>      <chr>                 <dbl>
     1 00:25:00:FF:94:73 <NA>         44      09:13      11:56                  2.72
     2 E8:28:C1:DD:04:52 MIREA_HOTSP… 11      09:13      11:56                  2.72
     3 E8:28:C1:DC:B2:52 MIREA_HOTSP… 6       09:13      11:55                  2.71
     4 08:3A:2F:56:35:FE <NA>         14      09:13      11:55                  2.71
     5 6E:C7:EC:16:DA:1A Cnet         1       09:13      11:55                  2.7 
     6 E8:28:C1:DC:B2:50 MIREA_GUESTS 6       09:13      11:55                  2.7 
     7 E8:28:C1:DC:B2:51 <NA>         6       09:13      11:55                  2.7 
     8 48:5B:39:F9:7A:48 <NA>         1       09:13      11:55                  2.7 
     9 E8:28:C1:DC:FF:F2 <NA>         6       09:13      11:55                  2.7 
    10 8E:55:4A:85:5B:01 Vladimir     6       09:13      11:55                  2.7 

### Шаг 5. Обнаружить топ-10 самых быстрых точек доступа.

``` r
access_points %>% arrange(desc(Speed)) %>% head(10)
```

    # A tibble: 10 × 15
       BSSID         `First time seen` `Last time seen` channel Speed Privacy Cipher
       <chr>         <chr>             <chr>            <chr>   <dbl> <chr>   <chr> 
     1 26:20:53:0C:… 2023-07-28 09:15… 2023-07-28 09:3… 44        866 WPA3 W… CCMP  
     2 96:FF:FC:91:… 2023-07-28 09:52… 2023-07-28 10:2… 44        866 WPA3 W… CCMP  
     3 CE:48:E7:86:… 2023-07-28 09:59… 2023-07-28 10:0… 44        866 WPA3 W… CCMP  
     4 8E:1F:94:96:… 2023-07-28 10:08… 2023-07-28 10:1… 44        866 WPA3 W… CCMP  
     5 9A:75:A8:B9:… 2023-07-28 09:13… 2023-07-28 11:5… 1         360 WPA2    CCMP  
     6 4A:EC:1E:DB:… 2023-07-28 09:13… 2023-07-28 11:0… 7         360 WPA2    CCMP  
     7 56:C5:2B:9F:… 2023-07-28 09:17… 2023-07-28 10:2… 1         360 WPA2    CCMP  
     8 E8:28:C1:DC:… 2023-07-28 09:18… 2023-07-28 11:3… 48        360 OPN     <NA>  
     9 E8:28:C1:DC:… 2023-07-28 09:18… 2023-07-28 11:5… 48        360 OPN     <NA>  
    10 E8:28:C1:DC:… 2023-07-28 09:18… 2023-07-28 11:4… 48        360 OPN     <NA>  
    # ℹ 8 more variables: Authentication <chr>, Power <dbl>, beacons <int>,
    #   iv_count <dbl>, `LAN IP` <chr>, `ID-length` <chr>, ESSID <chr>, Key <chr>

### Шаг 6. Отсортировать точки доступа по частоте отправки запросов (beacons) в единицу времени по их убыванию.

``` r
access_points_times <- access_points %>%
  mutate(
    first_seen = as.POSIXct(`First time seen`, format = "%Y-%m-%d %H:%M:%S"),
    last_seen = as.POSIXct(`Last time seen`, format = "%Y-%m-%d %H:%M:%S")
  )


beacon_stats <- access_points_times %>%
  mutate(
    duration_sec = as.numeric(
      difftime(last_seen, first_seen, units = "secs")
    ),
    beacons_per_sec = if_else(
      duration_sec > 0,
      beacons / duration_sec,
      NA_real_
    )
  ) %>%
  filter(!is.na(beacons_per_sec) & beacons_per_sec > 0) %>%
  select(BSSID, ESSID, beacons, duration_sec, beacons_per_sec, channel) %>%
  arrange(desc(beacons_per_sec))

cat("### Интенсивность отправки beacon-фреймов (топ-10):\n")
```

    ### Интенсивность отправки beacon-фреймов (топ-10):

``` r
if(nrow(beacon_stats) > 0) {
  beacon_stats %>%
    head(10) %>%
    mutate(
      beacons_per_sec = round(beacons_per_sec, 2),
      duration_min = round(duration_sec / 60, 1)
    ) %>%
    select(BSSID, ESSID, channel, beacons, duration_min, beacons_per_sec)
} else {
  cat("Нет данных для анализа beacon-фреймов.\n")
}
```

    # A tibble: 10 × 6
       BSSID             ESSID          channel beacons duration_min beacons_per_sec
       <chr>             <chr>          <chr>     <int>        <dbl>           <dbl>
     1 F2:30:AB:E9:03:ED iPhone (Ulian… 1             6          0.1            0.86
     2 B2:CF:C0:00:4A:60 Михаил's Gala… 6             4          0.1            0.8 
     3 3A:DA:00:F9:0C:02 iPhone XS Max… 6             5          0.1            0.56
     4 02:BC:15:7E:D5:DC MT_FREE        11            1          0              0.5 
     5 00:3E:1A:5D:14:45 MT_FREE        11            1          0              0.5 
     6 76:C5:A0:70:08:96 <NA>           6             1          0              0.5 
     7 D2:25:91:F6:6C:D8 Саня           12            5          0.2            0.38
     8 BE:F1:71:D6:10:D7 C322U21 0566   11         1647        158.             0.17
     9 00:03:7A:1A:03:56 MT_FREE        11            1          0.1            0.17
    10 38:1A:52:0D:84:D7 EBFCD57F-EE81… 11          704         72              0.16

## Данные клиентов

1.  Определить производителя для каждого обнаруженного устройства

``` r
Unique_Tech <- unique(client_requests$BSSID)
oui <- read.table(
  "oui.txt",
  sep = "\t",
  fill = TRUE,
  quote = "",
  stringsAsFactors = FALSE,
  col.names = c("OUI_raw", "Type", "Manufacturer")
)
oui$OUI <- toupper(gsub("-", ":", trimws(oui$OUI_raw)))
oui$OUI <- sub("\\s+.*", "", oui$OUI)   # убираем " (hex)"
clean_mac <- function(x) {
  x <- toupper(trimws(x))
  if (grepl("^([0-9A-F]{2}:){5}[0-9A-F]{2}$", x)) x else NA
}
extract_oui <- function(mac) {
  if (is.na(mac)) return(NA)
  paste(strsplit(mac, ":")[[1]][1:3], collapse=":")
}
find_man <- function(oui_prefix) {
  if (is.na(oui_prefix)) return("Unknown")
  m <- oui$Manufacturer[oui$OUI == oui_prefix]
  if (length(m) == 0) "Unknown" else m
}
mac_clean <- sapply(Unique_Tech, clean_mac)
mac_oui   <- sapply(mac_clean, extract_oui)
manufs    <- sapply(mac_oui, find_man)

result <- data.frame(
  OUI = mac_oui,
  Manufacturer = manufs
)

result
```

                           OUI                                        Manufacturer
    BE:F1:71:D5:17:8B BE:F1:71                                             Unknown
    (not associated)      <NA>                                             Unknown
    BE:F1:71:D6:10:D7 BE:F1:71                                             Unknown
    1E:93:E3:1B:3C:F4 1E:93:E3                                             Unknown
    E8:28:C1:DC:FF:F2 E8:28:C1                               Eltex Enterprise Ltd.
    00:25:00:FF:94:73 00:25:00                                         Apple, Inc.
    00:26:99:F2:7A:E2 00:26:99                                  Cisco Systems, Inc
    0C:80:63:A9:6E:EE 0C:80:63                       TP-LINK TECHNOLOGIES CO.,LTD.
    E8:28:C1:DD:04:52 E8:28:C1                               Eltex Enterprise Ltd.
    0A:C5:E1:DB:17:7B 0A:C5:E1                                             Unknown
    9A:75:A8:B9:04:1E 9A:75:A8                                             Unknown
    8A:A3:03:73:52:08 8A:A3:03                                             Unknown
    4A:EC:1E:DB:BF:95 4A:EC:1E                                             Unknown
    BE:F1:71:D5:0E:53 BE:F1:71                                             Unknown
    08:3A:2F:56:35:FE 08:3A:2F Guangzhou Juan Intelligent Tech Joint Stock Co.,Ltd
    6E:C7:EC:16:DA:1A 6E:C7:EC                                             Unknown
    2A:E8:A2:02:01:73 2A:E8:A2                                             Unknown
    E8:28:C1:DC:B2:52 E8:28:C1                               Eltex Enterprise Ltd.
    E8:28:C1:DC:C6:B2 E8:28:C1                               Eltex Enterprise Ltd.
    E8:28:C1:DC:C8:32 E8:28:C1                               Eltex Enterprise Ltd.
    56:C5:2B:9F:84:90 56:C5:2B                                             Unknown
    9A:9F:06:44:24:5B 9A:9F:06                                             Unknown
    12:48:F9:CF:58:8E 12:48:F9                                             Unknown
    E8:28:C1:DD:04:50 E8:28:C1                               Eltex Enterprise Ltd.
    AA:F4:3F:EE:49:0B AA:F4:3F                                             Unknown
    3A:70:96:C6:30:2C 3A:70:96                                             Unknown
    E8:28:C1:DC:3C:92 E8:28:C1                               Eltex Enterprise Ltd.
    8E:55:4A:85:5B:01 8E:55:4A                                             Unknown
    5E:C7:C0:E4:D7:D4 5E:C7:C0                                             Unknown
    E2:37:BF:8F:6A:7B E2:37:BF                                             Unknown
    96:FF:FC:91:EF:64 96:FF:FC                                             Unknown
    CE:B3:FF:84:45:FC CE:B3:FF                                             Unknown
    00:26:99:BA:75:80 00:26:99                                  Cisco Systems, Inc
    76:70:AF:A4:D2:AF 76:70:AF                                             Unknown
    E8:28:C1:DC:B2:50 E8:28:C1                               Eltex Enterprise Ltd.
    00:AB:0A:00:10:10 00:AB:0A                                             Unknown
    E8:28:C1:DC:C8:30 E8:28:C1                               Eltex Enterprise Ltd.
    8E:1F:94:96:DA:FD 8E:1F:94                                             Unknown
    E8:28:C1:DB:F5:F2 E8:28:C1                               Eltex Enterprise Ltd.
    E8:28:C1:DD:04:40 E8:28:C1                               Eltex Enterprise Ltd.
    EA:7B:9B:D8:56:34 EA:7B:9B                                             Unknown
    BE:FD:EF:18:92:44 BE:FD:EF                                             Unknown
    7E:3A:10:A7:59:4E 7E:3A:10                                             Unknown
    00:26:99:F2:7A:E1 00:26:99                                  Cisco Systems, Inc
    00:23:EB:E3:49:31 00:23:EB                                  Cisco Systems, Inc
    E8:28:C1:DC:B2:40 E8:28:C1                               Eltex Enterprise Ltd.
    E0:D9:E3:49:04:40 E0:D9:E3                               Eltex Enterprise Ltd.
    3A:DA:00:F9:0C:02 3A:DA:00                                             Unknown
    E8:28:C1:DC:B2:41 E8:28:C1                               Eltex Enterprise Ltd.
    E8:28:C1:DE:74:32 E8:28:C1                               Eltex Enterprise Ltd.
    E8:28:C1:DC:33:12 E8:28:C1                               Eltex Enterprise Ltd.
    92:F5:7B:43:0B:69 92:F5:7B                                             Unknown
    DC:09:4C:32:34:9B DC:09:4C                         HUAWEI TECHNOLOGIES CO.,LTD
    E8:28:C1:DC:F0:90 E8:28:C1                               Eltex Enterprise Ltd.
    E0:D9:E3:49:04:52 E0:D9:E3                               Eltex Enterprise Ltd.
    22:C9:7F:A9:BA:9C 22:C9:7F                                             Unknown
    E8:28:C1:DD:04:41 E8:28:C1                               Eltex Enterprise Ltd.
    92:12:38:E5:7E:1E 92:12:38                                             Unknown
    B2:1B:0C:67:0A:BD B2:1B:0C                                             Unknown
    E8:28:C1:DC:33:10 E8:28:C1                               Eltex Enterprise Ltd.
    E0:D9:E3:49:04:41 E0:D9:E3                               Eltex Enterprise Ltd.
    1E:C2:8E:D8:30:91 1E:C2:8E                                             Unknown
    A2:64:E8:97:58:EE A2:64:E8                                             Unknown
    A6:02:B9:73:2F:76 A6:02:B9                                             Unknown
    A6:02:B9:73:81:47 A6:02:B9                                             Unknown
    AE:3E:7F:C8:BC:8E AE:3E:7F                                             Unknown
    B6:C4:55:B5:53:24 B6:C4:55                                             Unknown
    86:DF:BF:E4:2F:23 86:DF:BF                                             Unknown
    02:67:F1:B0:6C:98 02:67:F1                                             Unknown
    36:46:53:81:12:A0 36:46:53                                             Unknown
    E8:28:C1:DC:0B:B0 E8:28:C1                               Eltex Enterprise Ltd.
    82:CD:7D:04:17:3B 82:CD:7D                                             Unknown
    E8:28:C1:DC:54:B2 E8:28:C1                               Eltex Enterprise Ltd.
    00:03:7F:10:17:56 00:03:7F                        Atheros Communications, Inc.
    00:0D:97:6B:93:DF 00:0D:97                             Hitachi Energy USA Inc.

1.  Обнаружить устройства, которые НЕ рандомизируют свой MAC адрес

``` r
is_global_mac <- function(mac) {
  if (is.na(mac) || mac == "" || mac == "(not associated)") return(FALSE)
  
  mac_clean <- gsub("[-: ]", "", toupper(mac))

  if (!grepl("^[0-9A-F]{12}$", mac_clean)) return(FALSE)
  
  first_octet_hex <- substr(mac_clean, 1, 2)
  first_octet_bin <- strtoi(first_octet_hex, base = 16)
  
  local_bit <- bitwAnd(first_octet_bin, 0x02) != 0
  
  return(!local_bit) 
}

devices_real_mac <- client_requests %>%
  filter(!is.na(`Station MAC`) & `Station MAC` != "(not associated)") %>%
  mutate(
    is_global = map_lgl(`Station MAC`, is_global_mac)
  ) %>%
  filter(is_global) %>%
  select(`Station MAC`, BSSID, `Probed ESSIDs`, Power) %>%
  distinct()


if(nrow(devices_real_mac) > 0) {
  devices_real_mac

} 
```

    # A tibble: 217 × 4
       `Station MAC`     BSSID             `Probed ESSIDs`         Power
       <chr>             <chr>             <chr>                   <dbl>
     1 5C:3A:45:9E:1A:7B BE:F1:71:D6:10:D7 C322U21 0566              -39
     2 C0:E4:34:D8:E7:E5 BE:F1:71:D5:17:8B C322U13 3965              -61
     3 10:51:07:CB:33:E7 (not associated)  <NA>                      -43
     4 68:54:5A:40:35:9E 1E:93:E3:1B:3C:F4 C322U06 5179,Galaxy A71   -31
     5 74:4C:A1:70:CE:F7 E8:28:C1:DC:FF:F2 <NA>                      -71
     6 BC:F1:71:D4:DB:04 (not associated)  <NA>                      -45
     7 4C:44:5B:14:76:E3 E8:28:C1:DD:04:52 <NA>                       -1
     8 A0:E7:0B:AE:D5:44 0A:C5:E1:DB:17:7B AndroidAP177B             -37
     9 00:95:69:E7:7F:35 (not associated)  nvripcsuite               -69
    10 00:95:69:E7:7C:ED (not associated)  nvripcsuite               -55
    # ℹ 207 more rows

``` r
mac_stats <- client_requests %>%
  filter(!is.na(`Station MAC`) & `Station MAC` != "(not associated)") %>%
  summarise(
    total_devices = n_distinct(`Station MAC`),
    global_macs = sum(map_lgl(`Station MAC`, is_global_mac), na.rm = TRUE),
    percentage_global = round(global_macs / total_devices * 100, 1)
  )
```

1.  Кластеризовать запросы от устройств к точкам доступа по их именам.
    Определить время появления устройства в зоне радиовидимости и время
    выхода его из нее.

``` r
client_clusters <- client_requests %>%
  filter(!is.na(`Probed ESSIDs`) & `Probed ESSIDs` != "") %>%
  mutate(
    first_time = as.POSIXct(`First time seen`, format = "%Y-%m-%d %H:%M:%S"),
    last_time = as.POSIXct(`Last time seen`, format = "%Y-%m-%d %H:%M:%S")
  ) %>%
  group_by(`Probed ESSIDs`) %>%
  summarise(
    device_count = n_distinct(`Station MAC`),
    first_detection = min(first_time, na.rm = TRUE),  
    last_detection = max(last_time, na.rm = TRUE),    
    avg_power = mean(Power, na.rm = TRUE),
    total_packets = sum(packets, na.rm = TRUE),
    .groups = "drop"
  ) %>%
  mutate(
    observation_period = as.numeric(
      difftime(last_detection, first_detection, units = "hours")
    )
  ) %>%
  arrange(desc(device_count))

if(nrow(client_clusters) > 0) {
  result <- client_clusters %>%
    mutate(
      first_time_str = format(first_detection, "%H:%M"),
      last_time_str = format(last_detection, "%H:%M"),
      observation_period = round(observation_period, 2),
      avg_power = round(avg_power, 1)
    ) %>%
    select(
      `Probed ESSIDs`, 
      device_count, 
      first_time_str, 
      last_time_str, 
      observation_period, 
      avg_power, 
      total_packets
    )
  result}
```

    # A tibble: 147 × 7
       `Probed ESSIDs`  device_count first_time_str last_time_str observation_period
       <chr>                   <int> <chr>          <chr>                      <dbl>
     1 GIVC                      284 09:13          11:56                       2.72
     2 IT2 Wireless              229 09:13          11:52                       2.66
     3 Gardenpasanauri           115 09:13          09:53                       0.67
     4 region                    111 09:45          10:18                       0.54
     5 Galaxy A31CAED             87 09:43          10:00                       0.3 
     6 Avenue611                  72 09:41          10:00                       0.33
     7 iPhone (Максим)            72 09:50          10:00                       0.17
     8 HONOR 30                   71 09:41          10:00                       0.33
     9 iPhone (Дима )             68 09:51          10:00                       0.17
    10 AndroidShare_15…           51 10:33          11:55                       1.37
    # ℹ 137 more rows
    # ℹ 2 more variables: avg_power <dbl>, total_packets <int>

1.  Оценить стабильность уровня сигнала внури кластера во времени.
    Выявить наиболее стабильный кластер.

``` r
shortest_sd_essid <- client_requests %>%
  mutate(
    first_time = as.POSIXct(`First time seen`, format = "%Y-%m-%d %H:%M:%S"),
    last_time = as.POSIXct(`Last time seen`, format = "%Y-%m-%d %H:%M:%S"),
    duration = as.numeric(difftime(last_time, first_time, units = "secs"))
  ) %>%
  filter(duration > 0 & !is.na(`Probed ESSIDs`) & `Probed ESSIDs` != "") %>%
  group_by(`Probed ESSIDs`) %>%
  summarise(
    Mean = mean(duration, na.rm = TRUE),
    Sd = sd(duration, na.rm = TRUE),
    Count = n(),
    .groups = "drop"
  ) %>%
  filter(!is.na(Sd) & Sd > 0) %>%
  arrange(Sd) %>%
  select(`Probed ESSIDs`, Count, Mean, Sd) %>%
  head(1)  # Берем только самую стабильную
if(nrow(shortest_sd_essid) > 0) {
  result <- shortest_sd_essid %>%
    mutate(
      Mean_min = round(Mean / 60, 2),
      Sd_min = round(Sd / 60, 2),
      `Probed ESSIDs` = if_else(
        nchar(`Probed ESSIDs`) > 40,
        paste0(substr(`Probed ESSIDs`, 1, 37), "..."),
        `Probed ESSIDs`))
      
      result}
```

    # A tibble: 1 × 6
      `Probed ESSIDs` Count  Mean    Sd Mean_min Sd_min
      <chr>           <int> <dbl> <dbl>    <dbl>  <dbl>
    1 nvripcsuite         3  9780  3.46      163   0.06

## Оценка результатов

Используя программные пакеты dplyr и tidyverse проанализировали журналы
tcpdump и airodump-ng с помощью языка программирования R.

## Вывод

1.  Получили знания о методах исследования радиоэлектронной обстановки.
2.  Составили представление о механизмах работы Wi-Fi сетей на канальном
    и сетевом уровне модели OSI.
3.  Зекрепили практические навыки использования языка программирования R
    для обработки данных
4.  Закрепили знания основных функций обработки данных экосистемы
    tidyverse языка R
