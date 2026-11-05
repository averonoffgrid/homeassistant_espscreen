# Nocte Showroom — SHT30 + VL53L0X (ESPHome)

Konfiguracja: [examples/showroom.local.yaml](examples/showroom.local.yaml) — CALY plik z zachowanymi ustawieniami wyswietlacza ST7701S, dotyku FT6336U 0x48, PCA9554 0x20 oraz podswietlenia GPIO6/BL_EN.

## Podlaczenie
Oba moduly rownolegle do istniejacej magistrali I2C w ESP32-S3:
- SDA: GPIO15
- SCL: GPIO7
- 3V3 i GND: wspolne z plyta Waveshare
- SHT30: adres 0x44 (ADDR ustawiony na 0x44)
- VL53L0X: adres 0x29; XSHUT zwykle niepotrzebny dla jednego modulu

Uwaga: sprawdz napiecie i nazwy pinow konkretnego kupionego modulu. GPIO15/GPIO7 to numery GPIO procesora, a nie numery stykow na plytce ani zlaczu tasmy LCD.

## Encje w HA
- Nocte Temperatura, Nocte Wilgotnosc
- Nocte Odleglosc (cm), Nocte Strefa odleglosci, Nocte Obiekt przed panelem
- Nocte Czas do wygaszenia (sekundy; unavailable gdy automat wylaczony, brak pomiaru lub osoba jest obecna; 0 przy wygaszonym ekranie)
- Nocte Automatyczne wybudzanie (switch; pierwszy start OFF, po uruchomieniu mozna wlaczyc)
- Nocte Odleglosc aktywacji (30–150 cm, domyslnie 80 cm)
- Nocte Opoznienie wygaszenia (5–300 s, domyslnie 30 s)
- Nocte SHT30 odpowiada, Nocte VL53L0X odpowiada — komunikacja I2C
- Nocte SHT30 pomiary aktualne, Nocte VL53L0X pomiary aktualne — swieze dane
- Nocte Stan automatyki — tekstowy status

## Zasada dzialania
- Odczyt VL53L0X co 750 ms; SHT30 co 20 s; sondowanie sprzetu na I2C co 10 s.
- Dwa kolejne cykle z odlegloscia <= progu wlaczaja stan "obiekt przed panelem".
- Stan aktywny trwa az do dwoch cykli poza progiem +20 cm (histereza).
- Po pojawieniu sie obiektu wywolany jest istniejacy skrypt Tessera wake_display.
- Podczas obecnosci odswiezamy standardowy zegar standby Tessera; dotyk ekranu takze odswieza ten zegar.
- Po zaniku obiektu odliczamy ustawione opoznienie i wywolujemy dim_display. Dodatkowo ustawiamy sleep_requested, zeby minutowe odswiezanie Tessera nie wlaczylo przypadkowo swiatla przy wylaczonym standardowym Auto Standby.
- Wylaczenie przelacznika automatyki przywraca logike standardowej Tessera; ekran automatycznie wygaszony wczesniej przez nowy mechanizm zostaje wybudzony.
- Niewazna odleglosc (NaN) przy prawidlowej komunikacji oznacza "poza zasiegiem" i pozwala rozpoczac odliczanie.
- Brak komunikacji lub brak nowych probek w ciagu 4 s powoduje wylaczenie sterowania nowa automatyka (fail-safe). Dotychczasowy mechanizm standby Tessera pozostaje niezalezny.

## Odporność na awarie
- Timeout magistrali I2C wynosi 10 ms (ESP-IDF).
- Odczyty i proby diagnostyczne nie sa wykonywane w ciasnych petlach ani z nieograniczonym ponawianiem.
- Brak czujnika nie powinien blokowac logiki ekranu; encje diagnostyczne pokazuja przyczyne.
- Awaria fizyczna wspolnej magistrali (np. zwarcie SDA/SCL, nieprawidlowe zasilanie lub duze zaklocenia) moze wplynac rowniez na FT6336U i PCA9554. Sam firmware nie gwarantuje dzialania tych urzadzen przy uszkodzonym okablowaniu. W razie potrzeby zastosuj izolacje/oddzielna magistrale.
- SHT30 i VL53L0X nie byly jeszcze fizycznie podlaczone do panelu; po montazu wymagany jest test rzeczywistych odczytow i poprawnosci wake/dim.

## Instalacja
1. Zachowaj lokalne `secrets.yaml` i dotychczasowe klucze API/OTA/Wi-Fi.
2. Skopiuj CALY plik `examples/showroom.local.yaml` jako `/config/esphome/showroom.local.yaml`. Nie dopisuj fragmentow na koncu starszej wersji.
3. Glowny `showroom.yaml` powinien zawierac `local_overrides: !include showroom.local.yaml`, pakiety i komponenty z galezi `nocte-v0.4.90`.
4. W ESPHome Device Builder uruchom Validate, nastepnie Install → Wirelessly. Gdy zmiany zaleznosci nie sa widoczne, uzyj Clean Build Files.
5. Sprawdz w logach skan I2C (0x20, 0x29, 0x44, 0x48), nastepnie encje `odpowiada` i `pomiary aktualne`.
6. Najpierw wykonaj pomiary odleglosci poza szyba oraz za szyba. Gdy wszystko dziala, wlacz `Nocte Automatyczne wybudzanie`.
7. Jesli wyswietlacz zle reaguje, wylacz switch, bez zmiany firmware.

Uwaga: pliki w repozytorium nie zawieraja hasel ani kluczy z Twojego aktualnego panelu. Nie uruchamiaj osobno `showroom.local.yaml`.
