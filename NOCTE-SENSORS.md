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
- Nocte Tryb ekranu: Wake / Sleep / OFF / Tessera
- Nocte Czas do wygaszenia: odliczanie tylko po wyjsciu poza Odleglosc 2; 0 po OFF, unavailable w innych strefach
- Nocte Automatyczne wybudzanie: switch z pamiecia stanu, przy pierwszym uruchomieniu OFF
- Nocte Odleglosc 1: 20–150 cm, domyslnie 80 cm
- Nocte Odleglosc 2: 30–200 cm, domyslnie 140 cm
- Nocte Opoznienie wygaszenia: 5–300 s, domyslnie 30 s
- Nocte SHT30 odpowiada, Nocte VL53L0X odpowiada: komunikacja I2C
- Nocte SHT30 pomiary aktualne, Nocte VL53L0X pomiary aktualne: swieze dane
- Nocte Stan automatyki: szczegolowy tekst diagnostyczny

## Zasada dzialania (3 strefy)
1. 0 do **Odleglosc 1**: **Wake**, ekran na normalnej jasnosci.
2. Od **Odleglosc 1** do **Odleglosc 2**: **Sleep**, podswietlenie ustawione na poziom czuwania Tessera.
3. Za **Odleglosc 2**: ekran jest w **Sleep** i rozpoczyna sie odliczanie **Nocte Opoznienie wygaszenia**. Po odliczeniu: **OFF**, sterowanie PWM=0 oraz fizyczne BL_EN=0.
4. Powrot z OFF do strefy 2 przywraca podswietlenie Sleep, a powrot do strefy 1 wykonuje Wake.
5. Dwa kolejne cykle pomiarowe (co 1 s) potwierdzaja przejscie do innej strefy. Przy braku obiektu przy dzialajacym sensorze dystansu stan jest traktowany jako odleglosc > prog 2.
6. Przy niespojnym ustawieniu progow firmware przyjmuje efektywna **Odleglosc 2 >= Odleglosc 1 + 10 cm**. Dla czytelnosci ustaw D2 o co najmniej 10 cm wiecej niz D1 w HA.
7. Dotkniecie ekranu w Sleep lub OFF wybudza panel na co najmniej 10 s (przedluzenie przy kolejnych dotknieciach). W strefie dalekiej licznik OFF jest wtedy resetowany.
8. Oryginalna automatyka Tessera pozostaje dostepna po wylaczeniu przelacznika "Nocte Automatyczne wybudzanie".
9. Tryb OFF jest uwzgledniony w `APPLY_BACKLIGHT`, aby minutowa aktualizacja ustawien Tessera nie wlaczyla ponownie podswietlenia.

**Wazne:** aby rozroznic Sleep i OFF, w ustawieniach Tessera ustaw **Standby brightness** na wartosc wyzsza od 0% (np. 20%). Sleep korzysta z tej wartosci (ewentualnie nocnej jasnosci czuwania), OFF wymusza 0% niezaleznie od niej.

## Odporność na awarie
- Timeout magistrali I2C wynosi 10 ms (ESP-IDF).
- Odczyty i proby diagnostyczne nie sa wykonywane w ciasnych petlach ani z nieograniczonym ponawianiem.
- Brak czujnika nie powinien blokowac logiki ekranu; encje diagnostyczne pokazuja przyczyne.
- Awaria fizyczna wspolnej magistrali (np. zwarcie SDA/SCL, nieprawidlowe zasilanie lub duze zaklocenia) moze wplynac rowniez na FT6336U i PCA9554. Sam firmware nie gwarantuje dzialania tych urzadzen przy uszkodzonym okablowaniu. W razie potrzeby zastosuj izolacje/oddzielna magistrale.
- Uzytkownik potwierdzil poprawne odczyty SHT30 i VL53L0X. Trzystopniowa logika Wake/Sleep/OFF jest nowa i wymaga walidacji kompilacji oraz testu na panelu.

## Instalacja
1. Zachowaj lokalne `secrets.yaml` i dotychczasowe klucze API/OTA/Wi-Fi.
2. Skopiuj CALY plik `examples/showroom.local.yaml` jako `/config/esphome/showroom.local.yaml`. Nie dopisuj fragmentow na koncu starszej wersji.
3. Glowny `showroom.yaml` powinien zawierac `local_overrides: !include showroom.local.yaml`, pakiety i komponenty z galezi `nocte-v0.4.90`.
4. W ESPHome Device Builder uruchom Validate, nastepnie Install → Wirelessly. Gdy zmiany zaleznosci nie sa widoczne, uzyj Clean Build Files.
5. Sprawdz w logach skan I2C (0x20, 0x29, 0x44, 0x48), nastepnie encje `odpowiada` i `pomiary aktualne`.
6. Po poprawnym dzialaniu sensorow ustaw progi D1/D2 i opoznienie. W Tessera ustaw widoczna jasnosc czuwania, np. Standby brightness = 20%. Potem wlacz `Nocte Automatyczne wybudzanie`.
7. Test: 50 cm = Wake; 100 cm = Sleep; 170 cm = Sleep, potem OFF po 30 s (przy D1=80 cm, D2=140 cm). Dotknij ekran w trybie OFF i sprawdz reczne wybudzenie. Jesli ekran zle reaguje, wylacz switch.

Uwaga: pliki w repozytorium nie zawieraja hasel ani kluczy z Twojego aktualnego panelu. Nie uruchamiaj osobno `showroom.local.yaml`.
