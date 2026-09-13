Title: Fully Autonomous Astronomical Countdown Clock for ESP32 CYD (3.5" ST7796) using ESPHomeHi! I wanted to share my complete, clean, and tested ESPHome configuration for the Sunton ESP32-035 (Cheap Yellow Display 3.5" without PSRAM).It acts as a standalone astronomical station that displays local time via NTP and a dynamic countdown timer (HH:MM:SS) showing exactly how much time is left until sunset.Features:Zero lag / no memory leaks on non-PSRAM boards using the modern mipi_spi platform.Vector graphics interface (rounded borders and section dividers).Smart celestial icon: displays a vector sun with 8 rays during the day and automatically flips to a crescent moon exactly at the minute of sunset.Hardcoded custom network logic with static IP and dual DNS to ensure immediate NTP synchronization.Feel free to use it and modify it for your coordinates!Created by: SP3PM op.Marcin (73!).


# ☀️🌙 Autonomiczny Zegar Astronomiczny dla ESP32 CYD (3.5")

Kompletny, zoptymalizowany i przetestowany plik konfiguracyjny systemu **ESPHome** dla popularnej płytki **Sunton ESP32-035 (Cheap Yellow Display 3.5" bez pamięci PSRAM)**. 

Urządzenie działa jako w pełni niezależna stacja pobierająca czas z serwerów NTP, która na bieżąco oblicza pozycję słońca i prowadzi płynne, sekundowe odliczanie czasu pozostałego do zachodu słońca dla wskazanych współrzędnych geograficznych (konfiguracja bazowa: *Drezdenko, Polska*).

---

## 👨‍💻 Autor Projektu
* **Twórca:** SP3PM op.Marcin
* **Wersja oprogramowania:** v2.0 (Edycja Graficzna z Wektorami)
* **Status:** Stabilny / Produkcyjny

---

## ⚙️ Specyfikacja techniczna

| Parametr | Wartość / Konfiguracja |
| :--- | :--- |
| **Płytka deweloperska** | Sunton ESP32-2432S035R / Cheap Yellow Display 3.5" |
| **Układ SoC** | ESP32-D0WD-V3 (Dual Core, 240MHz) |
| **Pamięć zewnętrzna** | Brak PSRAM (Wymaga lekkiego oprogramowania) |
| **Sterownik LCD** | ST7796 (Rozdzielczość 320x480 pikseli) |
| **Środowisko** | ESPHome v2026.8.0+ (Framework ESP-IDF v5.5.5) |
| **Adresacja IP** | Static IP (`192.168.1.103`) z obsługą podwójnego DNS |

> [!WARNING]
> Urządzenia Sunton CYD 3.5" bez kości PSRAM ulegają przepełnieniu pamięci i zawieszeniu przy użyciu ciężkich silników graficznych (np. Tasmota-LVGL / Haspmota). Niniejszy kod rozwiązuje ten problem poprzez zastosowanie natywnego, maszynowego silnika wektorowego ESPHome `mipi_spi`.

---

## 🎨 Opis Interfejsu Graficznego
Interfejs został zaprojektowany z dbałością o estetykę i czytelność w trudnych warunkach oświetleniowych:
1. **Ramka Wektorowa:** Zamknięta linia zewnętrzna z zaokrąglonymi narożnikami (promień $r=10\text{px}$) nadająca urządzeniu fabryczny wygląd.
2. **Podział Sekcji:** Dwie dyskretne, poziome linie w kolorze głębokiej szarości, dzielące ekran na sekcję czasu, sekcję astronomiczną oraz stopkę.
3. **Inteligentny Wskaźnik Pory Dnia:** Dynamiczny obiekt wektorowy zlokalizowany w prawym górnym rogu:
   * **W dzień:** Wypełniona żółta tarcza słońca otoczona 8 symetrycznymi promieniami wektorowymi.
   * **W nocy:** Jasny, srebrzysto-błękitny sierp księżyca generowany automatycznie przy użyciu maskowania kołowego.
4. **Automatyka Przełączania:** Zmiana ikony oraz przeliczenie czasu na kolejną dobę następuje dokładnie w sekundzie przekroczenia linii horyzontu przez słońce.

---

## 💾 Kod Źródłowy (`zegar.yaml`)

```yaml
esphome:
  name: zegar
  friendly_name: Zegar Astronomiczny Drezdenko

esp32:
  board: esp32dev
  framework:
    type: arduino

wifi:
  ssid: "Ground_Station"
  password: "wi-fi"

  manual_ip:
    static_ip: 192.168.1.103
    gateway: 192.168.1.1
    subnet: 255.255.255.0
    dns1: 192.168.1.1
    dns2: 8.8.8.8

logger:
api:
ota:
  - platform: esphome

# --- Konfiguracja Sprzętowa SPI ---
spi:
  clk_pin: GPIO14
  mosi_pin: GPIO13
  miso_pin: GPIO12

# --- Wymuszenie Stałego Podświetlenia Matrycy (BL) ---
switch:
  - platform: gpio
    pin: GPIO27
    id: backlight_switch
    restore_mode: ALWAYS_ON

# --- Czas NTP i Koordynaty Astronomiczne ---
time:
  - platform: sntp
    id: sntp_time
    timezone: "CET-1CEST,M3.5.0,M10.5.0/3"
    servers:
      - "0.pl.pool.ntp.org"
      - "1.pl.pool.ntp.org"
      - "pool.ntp.org"

sun:
  latitude: 52.0000
  longitude: 15.0000
  id: sun_astronomical

# --- Generowanie Czcionek Google Fonts ---
font:
  - file: "gfonts://Roboto"
    id: font_small
    size: 20
  - file: "gfonts://Roboto"
    id: font_medium
    size: 36
  - file: "gfonts://Roboto"
    id: font_large
    size: 48

# --- Silnik Graficzny i Definicje Warstw ---
display:
  - platform: mipi_spi
    id: cyd_display
    model: ST7796
    cs_pin: GPIO15
    dc_pin: GPIO02
    dimensions:
      width: 320
      height: 480
    rotation: 90°
    update_interval: 1s
    lambda: |-
      Color kolor_jasnoniebieski = Color(0, 182, 255); // #00b6ff
      Color kolor_pomaranczowy = Color(255, 153, 98); // #ff9962
      Color kolor_zolty = Color(255, 200, 0);         // Cieply zolty
      Color kolor_ksiezyc = Color(200, 220, 255);     // Srebrzysty blekit
      Color kolor_bialy = Color(255, 255, 255);
      Color kolor_szary = Color(150, 150, 150);
      Color kolor_ciemnoszary = Color(50, 50, 50);

      // --- 1. RYSOWANIE GEOMETRII INTERFEJSU ---
      it.line(15, 5, 465, 5, kolor_ciemnoszary);     
      it.line(15, 315, 465, 315, kolor_ciemnoszary); 
      it.line(5, 15, 5, 305, kolor_ciemnoszary);     
      it.line(475, 15, 475, 305, kolor_ciemnoszary); 

      it.circle(15, 15, 10, kolor_ciemnoszary);   
      it.circle(465, 15, 10, kolor_ciemnoszary);  
      it.circle(15, 305, 10, kolor_ciemnoszary);  
      it.circle(465, 305, 10, kolor_ciemnoszary); 

      it.line(20, 115, 460, 115, kolor_ciemnoszary); 
      it.line(20, 225, 460, 225, kolor_ciemnoszary); 

      // --- 2. SEKCJA GÓRNA: NAGŁÓWEK I CZAS ---
      it.print(240, 18, id(font_small), kolor_bialy, TextAlign::TOP_CENTER, "AKTUALNY CZAS");

      // Logika renderowania pory dnia (Słońce z promieniami / Księżyc)
      int x0 = 390; 
      int y0 = 35;  

      if (id(sun_astronomical).is_above_horizon()) {
        it.filled_circle(x0, y0, 8, kolor_zolty);
        it.line(x0, y0 - 12, x0, y0 - 18, kolor_zolty); 
        it.line(x0, y0 + 12, x0, y0 + 18, kolor_zolty); 
        it.line(x0 - 12, y0, x0 - 18, y0, kolor_zolty); 
        it.line(x0 + 12, y0, x0 + 18, y0, kolor_zolty); 
        it.line(x0 - 8, y0 - 8, x0 - 13, y0 - 13, kolor_zolty); 
        it.line(x0 + 8, y0 - 8, x0 + 13, y0 - 13, kolor_zolty); 
        it.line(x0 - 8, y0 + 8, x0 - 13, y0 + 13, kolor_zolty); 
        it.line(x0 + 8, y0 + 8, x0 + 13, y0 + 13, kolor_zolty); 
      } else {
        it.circle(x0, y0, 14, kolor_ksiezyc);
        it.circle(x0 - 7, y0, 14, Color(0, 0, 0)); 
      }

      auto czas = id(sntp_time).now();
      if (czas.is_valid()) {
        it.strftime(240, 48, id(font_medium), kolor_jasnoniebieski, TextAlign::TOP_CENTER, "%H:%M:%S", czas);
      } else {
        it.print(240, 48, id(font_medium), kolor_jasnoniebieski, TextAlign::TOP_CENTER, "CZEKAM NA NTP...");
      }

      // --- 3. SEKCJA ŚRODKOWA: LICZNIK ASTRONOMICZNY ---
      it.print(240, 130, id(font_small), kolor_bialy, TextAlign::TOP_CENTER, "DO ZACHODU SLONCA:");

      auto zachod_czas = id(sun_astronomical).sunset(czas, 0.0);
      if (zachod_czas.has_value() && czas.is_valid()) {
        int32_t pozostalo_sekund = zachod_czas.value().timestamp - czas.timestamp;
        
        // Korekta dobowana następny dzień po wystąpieniu zachodu
        if (pozostalo_sekund < 0) {
          pozostalo_sekund += 86400;
        }

        int godziny = pozostalo_sekund / 3600;
        int minuty = (pozostalo_sekund % 3600) / 60;
        int sekundy = pozostalo_sekund % 60;
        
        it.printf(240, 162, id(font_large), kolor_pomaranczowy, TextAlign::TOP_CENTER, "%02d:%02d:%02d", godziny, minuty, sekundy);
      } else {
        it.print(240, 162, id(font_large), kolor_pomaranczowy, TextAlign::TOP_CENTER, "OBLICZANIE...");
      }

      // --- 4. SEKCJA DOLNA: METRYCZKA ---
      it.print(240, 240, id(font_small), kolor_szary, TextAlign::TOP_CENTER, "AUTOR: SP3PM op.Marcin");
```

---

## 📡 Instrukcja implementacji w sieci lokalnej
1. Otwórz swój **ESPHome Dashboard** lub edytor lokalny.
2. Stwórz nowe urządzenie o nazwie `zegar`.
3. Skopiuj i wklej powyższą zawartość do pliku konfiguracyjnego.
4. Przed kompilacją dostosuj parametry sieciowe w sekcji `wifi:` oraz wprowadź własne współrzędne geograficzne w sekcji `sun:` (`latitude`, `longitude`), jeśli urządzenie ma pracować poza Drezdenkiem.
5. Kliknij **Install** i wybierz metodę programowania (kablem USB lub bezprzewodowo OTA).

---
*Projekt udostępniony na licencji Open-Source dla społeczności Home Assistant i pasjonatów Radiotechniki. Wykonał: SP3PM op.Marcin (73!).*
