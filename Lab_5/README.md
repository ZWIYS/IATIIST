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
```

1.  Загрузка данных

``` r
Data <- read.csv('./P2_wifi_data.csv')
head(Data)
```

                  BSSID      First.time.seen       Last.time.seen channel Speed
    1 BE:F1:71:D5:17:8B  2023-07-28 09:13:03  2023-07-28 11:50:50       1   195
    2 6E:C7:EC:16:DA:1A  2023-07-28 09:13:03  2023-07-28 11:55:12       1   130
    3 9A:75:A8:B9:04:1E  2023-07-28 09:13:03  2023-07-28 11:53:31       1   360
    4 4A:EC:1E:DB:BF:95  2023-07-28 09:13:03  2023-07-28 11:04:01       7   360
    5 D2:6D:52:61:51:5D  2023-07-28 09:13:03  2023-07-28 10:30:19       6   130
    6 E8:28:C1:DC:B2:52  2023-07-28 09:13:03  2023-07-28 11:55:38       6   130
      Privacy Cipher Authentication Power X..beacons     X..IV           LAN.IP
    1    WPA2   CCMP            PSK   -30        846       504    0.  0.  0.  0
    2    WPA2   CCMP            PSK   -30        750       116    0.  0.  0.  0
    3    WPA2   CCMP            PSK   -68        694        26    0.  0.  0.  0
    4    WPA2   CCMP            PSK   -37        510        21    0.  0.  0.  0
    5    WPA2   CCMP            PSK   -57        647         6    0.  0.  0.  0
    6     OPN                         -63        251      3430  172. 17.203.197
      ID.length           ESSID Key
    1        12    C322U13 3965  NA
    2         4            Cnet  NA
    3         2              KC  NA
    4        14  POCO X5 Pro 5G  NA
    5        25                  NA
    6        13   MIREA_HOTSPOT  NA

1.  Привести датасеты в вид “аккуратных данных”, преобразовать типы
    столбцов в соответствии с типом данных

``` r
access_points <- read.csv('./P2_wifi_data.csv', nrows = 167, stringsAsFactors = FALSE)
access_points$First.time.seen <- as.POSIXct(access_points$First.time.seen, format="%Y-%m-%d %H:%M:%S")
access_points$Last.time.seen  <- as.POSIXct(access_points$Last.time.seen, format="%Y-%m-%d %H:%M:%S")
access_points$channel         <- as.integer(access_points$channel)
access_points$Speed           <- as.numeric(access_points$Speed)
access_points$Power           <- as.numeric(access_points$Power)
access_points$X..beacons      <- as.integer(access_points$X..beacons)
access_points$X..IV           <- as.numeric(access_points$X..IV)
access_points$ESSID <- as.character(access_points$ESSID)
access_points$Key   <- as.character(access_points$Key)

client_requests <- read.csv('./P2_wifi_data.csv', skip = 169, stringsAsFactors = FALSE)
client_requests$First.time.seen <- as.POSIXct(client_requests$First.time.seen, format="%Y-%m-%d %H:%M:%S")
client_requests$Last.time.seen  <- as.POSIXct(client_requests$Last.time.seen, format="%Y-%m-%d %H:%M:%S")
client_requests$Power           <- as.numeric(client_requests$Power)
```

    Warning: NAs introduced by coercion

``` r
client_requests$X..packets      <- as.integer(client_requests$X..packets)
```

    Warning: NAs introduced by coercion

``` r
client_requests$BSSID           <- as.character(client_requests$BSSID)
client_requests$Probed.ESSIDs   <- as.character(client_requests$Probed.ESSIDs)
client_requests$Station.MAC     <- as.character(client_requests$Station.MAC)
```

1.  Просмотрите общую структуру данных с помощью функции glimpse()

``` r
glimpse(access_points)
```

    Rows: 167
    Columns: 15
    $ BSSID           <chr> "BE:F1:71:D5:17:8B", "6E:C7:EC:16:DA:1A", "9A:75:A8:B9…
    $ First.time.seen <dttm> 2023-07-28 09:13:03, 2023-07-28 09:13:03, 2023-07-28 …
    $ Last.time.seen  <dttm> 2023-07-28 11:50:50, 2023-07-28 11:55:12, 2023-07-28 …
    $ channel         <int> 1, 1, 1, 7, 6, 6, 11, 11, 11, 1, 6, 14, 11, 11, 6, 6, …
    $ Speed           <dbl> 195, 130, 360, 360, 130, 130, 195, 130, 130, 195, 180,…
    $ Privacy         <chr> " WPA2", " WPA2", " WPA2", " WPA2", " WPA2", " OPN", "…
    $ Cipher          <chr> " CCMP", " CCMP", " CCMP", " CCMP", " CCMP", " ", " CC…
    $ Authentication  <chr> " PSK", " PSK", " PSK", " PSK", " PSK", "   ", " PSK",…
    $ Power           <dbl> -30, -30, -68, -37, -57, -63, -27, -38, -38, -66, -42,…
    $ X..beacons      <int> 846, 750, 694, 510, 647, 251, 1647, 1251, 704, 617, 13…
    $ X..IV           <dbl> 504, 116, 26, 21, 6, 3430, 80, 11, 0, 0, 86, 0, 0, 0, …
    $ LAN.IP          <chr> "   0.  0.  0.  0", "   0.  0.  0.  0", "   0.  0.  0.…
    $ ID.length       <int> 12, 4, 2, 14, 25, 13, 12, 13, 24, 12, 10, 0, 24, 24, 1…
    $ ESSID           <chr> " C322U13 3965", " Cnet", " KC", " POCO X5 Pro 5G", " …
    $ Key             <chr> NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA, NA…

``` r
glimpse(client_requests)
```

    Rows: 12,269
    Columns: 7
    $ Station.MAC     <chr> "CA:66:3B:8F:56:DD", "96:35:2D:3D:85:E6", "5C:3A:45:9E…
    $ First.time.seen <dttm> 2023-07-28 09:13:03, 2023-07-28 09:13:03, 2023-07-28 …
    $ Last.time.seen  <dttm> 2023-07-28 10:59:44, 2023-07-28 09:13:03, 2023-07-28 …
    $ Power           <dbl> -33, -65, -39, -61, -53, -43, -31, NA, -71, -74, -65, …
    $ X..packets      <int> 858, 4, 432, 958, 1, 344, 163, NA, 3, 115, 437, 265, 7…
    $ BSSID           <chr> " BE:F1:71:D5:17:8B", " (not associated) ", " BE:F1:71…
    $ Probed.ESSIDs   <chr> "C322U13 3965", "IT2 Wireless", "C322U21 0566", "C322U…

## Анализ данныз

### Шаг 1. Определить небезопасные точки доступа (без шифрования – OPN)

``` r
AP_nOPN <- access_points %>% filter(str_detect(toupper(Privacy), "OPN")) %>% select(BSSID)
AP_nOPN
```

                   BSSID
    1  E8:28:C1:DC:B2:52
    2  E8:28:C1:DC:B2:50
    3  E8:28:C1:DC:B2:51
    4  E8:28:C1:DC:FF:F2
    5  00:25:00:FF:94:73
    6  E8:28:C1:DD:04:52
    7  E8:28:C1:DE:74:31
    8  E8:28:C1:DE:74:32
    9  E8:28:C1:DC:C8:32
    10 E8:28:C1:DD:04:50
    11 E8:28:C1:DD:04:51
    12 E8:28:C1:DC:C8:30
    13 E8:28:C1:DE:74:30
    14 E0:D9:E3:48:FF:D2
    15 E8:28:C1:DC:B2:41
    16 E8:28:C1:DC:B2:40
    17 00:26:99:F2:7A:E0
    18 E8:28:C1:DC:B2:42
    19 E8:28:C1:DD:04:40
    20 E8:28:C1:DD:04:41
    21 E8:28:C1:DE:47:D2
    22 02:BC:15:7E:D5:DC
    23 E8:28:C1:DC:C6:B1
    24 E8:28:C1:DD:04:42
    25 E8:28:C1:DC:C8:31
    26 E8:28:C1:DE:47:D1
    27 00:AB:0A:00:10:10
    28 E8:28:C1:DC:C6:B0
    29 E8:28:C1:DC:C6:B2
    30 E8:28:C1:DC:BD:50
    31 E8:28:C1:DC:0B:B2
    32 E8:28:C1:DC:33:12
    33 00:03:7A:1A:03:56
    34 00:03:7F:12:34:56
    35 00:3E:1A:5D:14:45
    36 E0:D9:E3:49:00:B1
    37 E8:28:C1:DC:BD:52
    38 00:26:99:F2:7A:EF
    39 02:67:F1:B0:6C:98
    40 02:CF:8B:87:B4:F9
    41 00:53:7A:99:98:56
    42 E8:28:C1:DE:47:D0

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

           OUI             BSSID                 Manufacturer
    1 00:03:7A 00:03:7A:1A:03:56        Taiyo Yuden Co., Ltd.
    2 00:03:7F 00:03:7F:12:34:56 Atheros Communications, Inc.
    3 00:25:00 00:25:00:FF:94:73                  Apple, Inc.
    4 00:26:99 00:26:99:F2:7A:E0           Cisco Systems, Inc
    5 00:26:99 00:26:99:F2:7A:EF           Cisco Systems, Inc
    6 00:3E:1A 00:3E:1A:5D:14:45                      Unknown

### Шаг 3. Выявить устройства, использующие последнюю версию протокола шифрования WPA3, и названия точек доступа, реализованных на этих устройствах

``` r
access_points %>%
  filter(str_detect(toupper(Privacy), "WPA3")) %>%
  select(BSSID, ESSID) %>%
  distinct()
```

                  BSSID                 ESSID
    1 26:20:53:0C:98:E8                      
    2 A2:FE:FF:B8:9B:C9            Christie’s
    3 96:FF:FC:91:EF:64                      
    4 CE:48:E7:86:4E:33    iPhone (Анастасия)
    5 8E:1F:94:96:DA:FD    iPhone (Анастасия)
    6 BE:FD:EF:18:92:44              Димасик 
    7 3A:DA:00:F9:0C:02  iPhone XS Max 🦊🐱🦊
    8 76:C5:A0:70:08:96                      

### Шаг 4. Отсортировать точки доступа по интервалу времени, в течение которого они находились на связи, по убыванию.

``` r
access_points %>% 
  mutate(duration = as.numeric(
    difftime(Last.time.seen, First.time.seen, units = "secs"))
    ) %>% 
  arrange(desc(duration)) %>% select(BSSID, duration) %>% head(10)
```

                   BSSID duration
    1  00:25:00:FF:94:73     9795
    2  E8:28:C1:DD:04:52     9776
    3  E8:28:C1:DC:B2:52     9755
    4  08:3A:2F:56:35:FE     9746
    5  6E:C7:EC:16:DA:1A     9729
    6  E8:28:C1:DC:B2:50     9726
    7  E8:28:C1:DC:B2:51     9725
    8  48:5B:39:F9:7A:48     9725
    9  E8:28:C1:DC:FF:F2     9724
    10 8E:55:4A:85:5B:01     9723

### Шаг 5. Обнаружить топ-10 самых быстрых точек доступа.

``` r
access_points %>% arrange(desc(Speed)) %>% head(10)
```

                   BSSID     First.time.seen      Last.time.seen channel Speed
    1  26:20:53:0C:98:E8 2023-07-28 09:15:45 2023-07-28 09:33:10      44   866
    2  96:FF:FC:91:EF:64 2023-07-28 09:52:54 2023-07-28 10:25:02      44   866
    3  CE:48:E7:86:4E:33 2023-07-28 09:59:20 2023-07-28 10:04:15      44   866
    4  8E:1F:94:96:DA:FD 2023-07-28 10:08:32 2023-07-28 10:15:27      44   866
    5  9A:75:A8:B9:04:1E 2023-07-28 09:13:03 2023-07-28 11:53:31       1   360
    6  4A:EC:1E:DB:BF:95 2023-07-28 09:13:03 2023-07-28 11:04:01       7   360
    7  56:C5:2B:9F:84:90 2023-07-28 09:17:49 2023-07-28 10:27:22       1   360
    8  E8:28:C1:DC:B2:41 2023-07-28 09:18:16 2023-07-28 11:36:43      48   360
    9  E8:28:C1:DC:B2:40 2023-07-28 09:18:16 2023-07-28 11:51:48      48   360
    10 E8:28:C1:DC:B2:42 2023-07-28 09:18:30 2023-07-28 11:43:23      48   360
          Privacy Cipher Authentication Power X..beacons X..IV           LAN.IP
    1   WPA3 WPA2   CCMP        SAE PSK   -85          3     0    0.  0.  0.  0
    2   WPA3 WPA2   CCMP        SAE PSK   -85          9     0    0.  0.  0.  0
    3   WPA3 WPA2   CCMP        SAE PSK   -65          9     0    0.  0.  0.  0
    4   WPA3 WPA2   CCMP        SAE PSK   -67         12     0    0.  0.  0.  0
    5        WPA2   CCMP            PSK   -68        694    26    0.  0.  0.  0
    6        WPA2   CCMP            PSK   -37        510    21    0.  0.  0.  0
    7        WPA2   CCMP            PSK   -64        317    31    0.  0.  0.  0
    8         OPN                         -89          5    23  169.254.175.203
    9         OPN                         -88          5    89  172. 17.203.197
    10        OPN                         -87          5     0    0.  0.  0.  0
       ID.length               ESSID  Key
    1         10                     <NA>
    2         10                     <NA>
    3         27  iPhone (Анастасия) <NA>
    4         27  iPhone (Анастасия) <NA>
    5          2                  KC <NA>
    6         14      POCO X5 Pro 5G <NA>
    7         10          OnePlus 6T <NA>
    8         12        MIREA_GUESTS <NA>
    9         13       MIREA_HOTSPOT <NA>
    10         0                     <NA>

### Шаг 6. Отсортировать точки доступа по частоте отправки запросов (beacons) в единицу времени по их убыванию.

``` r
library(dplyr)

access_points %>%
  mutate(
    duration = as.numeric(difftime(Last.time.seen, First.time.seen, units = "secs")),
    beacons_per_second = X..beacons / duration
  ) %>%
  filter(duration != 0) %>%
  arrange(desc(beacons_per_second)) %>%
  select(BSSID, beacons_per_second) %>%
  head()
```

                  BSSID beacons_per_second
    1 F2:30:AB:E9:03:ED          0.8571429
    2 B2:CF:C0:00:4A:60          0.8000000
    3 3A:DA:00:F9:0C:02          0.5555556
    4 02:BC:15:7E:D5:DC          0.5000000
    5 00:3E:1A:5D:14:45          0.5000000
    6 76:C5:A0:70:08:96          0.5000000

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
                           <NA>                                             Unknown
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
    MIREA_HOTSPOT          <NA>                                             Unknown
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
    AndroidShare_6329      <NA>                                             Unknown
     DC:09:4C:32:34:9B DC:09:4C                         HUAWEI TECHNOLOGIES CO.,LTD
     E8:28:C1:DC:F0:90 E8:28:C1                               Eltex Enterprise Ltd.
     E0:D9:E3:49:04:52 E0:D9:E3                               Eltex Enterprise Ltd.
     22:C9:7F:A9:BA:9C 22:C9:7F                                             Unknown
     E8:28:C1:DD:04:41 E8:28:C1                               Eltex Enterprise Ltd.
    TP-Link_9872_5G        <NA>                                             Unknown
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
library(dplyr)
library(stringr)

NotUniqueDevice <- client_requests %>%
  filter(!is.na(Station.MAC)) %>%                         # убираем пустые
  filter(!toupper(substr(Station.MAC, 2, 2)) %in% c("2","6","A","E")) %>%  # локальный бит
  select(Station.MAC) %>%
  distinct()  

head(NotUniqueDevice)
```

            Station.MAC
    1 5C:3A:45:9E:1A:7B
    2 C0:E4:34:D8:E7:E5
    3 10:51:07:CB:33:E7
    4 68:54:5A:40:35:9E
    5 74:4C:A1:70:CE:F7
    6 BC:F1:71:D4:DB:04

1.  Кластеризовать запросы от устройств к точкам доступа по их именам.
    Определить время появления устройства в зоне радиовидимости и время
    выхода его из нее.

``` r
library(dplyr)

grouped_data <- client_requests %>%
  group_by(Probed.ESSIDs) %>%
  summarise(
    unique_devices = n_distinct(Station.MAC),
    first_time_seen = min(First.time.seen, na.rm = TRUE),
    last_time_seen  = max(Last.time.seen, na.rm = TRUE),
    .groups = "drop"
  )

head(grouped_data)
```

    # A tibble: 6 × 4
      Probed.ESSIDs           unique_devices first_time_seen     last_time_seen     
      <chr>                            <int> <dttm>              <dttm>             
    1 ""                               10652 2023-07-28 09:13:04 2023-07-28 11:56:24
    2 "-D-13-"                            16 2023-07-28 09:14:42 2023-07-28 10:26:42
    3 "1"                                 31 2023-07-28 10:36:12 2023-07-28 11:56:13
    4 "107"                                1 2023-07-28 10:29:43 2023-07-28 10:29:43
    5 "531"                                1 2023-07-28 10:57:04 2023-07-28 10:57:04
    6 "AAAAAOB/CC0ADwGkRedmi…              3 2023-07-28 09:34:20 2023-07-28 11:44:40

1.  Оценить стабильность уровня сигнала внури кластера во времени.
    Выявить наиболее стабильный кластер.

``` r
library(dplyr)

shortest_sd_essid <- client_requests %>%
  mutate(duration = as.integer(difftime(Last.time.seen, First.time.seen, units = "secs"))) %>%
  filter(duration != 0, !is.na(Probed.ESSIDs)) %>%
  group_by(Probed.ESSIDs) %>%
  summarise(
    Mean = mean(duration, na.rm = TRUE),
    Sd   = sd(duration, na.rm = TRUE),
    .groups = "drop"
  ) %>%
  filter(!is.na(Sd) & Sd != 0) %>%
  arrange(Sd) %>%
  select(Probed.ESSIDs, Mean, Sd) %>%
  slice_head(n = 1)

shortest_sd_essid
```

    # A tibble: 1 × 3
      Probed.ESSIDs  Mean    Sd
      <chr>         <dbl> <dbl>
    1 nvripcsuite    9780  3.46

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
