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
- Nocte Czas do wygaszenia: odliczanie tylko poza Odlegloscia 2; 0 po OFF, unavailable w Wake i Sleep bez licznika
- Nocte Automatyczne wybudzanie: switch, przy pierwszym uruchomieniu OFF
- Nocte Odleglosc 1: 20–150 cm, domyslnie 80 cm
- Nocte Odleglosc 2: 30–200 cm, domyslnie 140 cm
- Nocte Opoznienie wygaszenia: 5–300 s, domyslnie 30 s
- Nocte SHT30 odpowiada, Nocte VL53L0X odpowiada: komunikacja I2C
- Nocte SHT30 pomiary aktualne, Nocte VL53L0X pomiary aktualne: swieze dane
- Nocte Stan automatyki: szczegolowy tekst diagnostyczny

## Zasada dzialania (3 strefy)
1. Dystans **do Odleglosci 1 (D1)**: tryb Wake, normalna jasnosc; wyzerowany licznik OFF.
2. Dystans **powyzej D1 do Odleglosci 2 (D2)**: tryb Sleep, podswietlenie na stale na poziomie czuwania Tessera; w tej strefie **OFF NIE jest odliczany i NIE wlacza sie samoczynnie**.
3. Dystans **powyzej D2**: tryb Sleep oraz start licznika "Nocte Opoznienie wygaszenia". Po uplywie czasu fizyczny OFF (PWM=0, BL_EN=0).
4. Bezposredni skok pomiaru z Wake za D2 (bez zarejestrowania strefy Sleep) rowniez rozpoczyna odliczanie do OFF.
5. Powrot z OFF do strefy Sleep wlacza przyciemnione podswietlenie. Powrot z OFF/Sleep do Wake wlacza normalna jasnosc. Samo pozostawanie poza D2 utrzymuje OFF, bez cyklicznego swiecenia.
6. Przejscie ze strefy dalekiej do Sleep lub Wake zawsze anuluje licznik OFF; nastepne wyjscie poza D2 zaczyna pelne opoznienie od nowa.
7. Dwa kolejne cykle sprawdzajace (co 1 s) zatwierdzaja zmiane strefy. Brak obiektu w zasiegu przy dzialajacym czujniku traktowany jest jako >D2.
8. Przy niespojnych nastawach efektywny prog D2 wynosi co najmniej D1+10 cm. Ustawiaj D2 o >=10 cm wiecej niz D1.
9. Dotyk/rekord Wake z Home Assistant chwilowo wybudza ekran z Sleep/OFF (10 s ochrony). Po zakonczeniu ochrony w strefie dalekiej zaczyna sie pelne opoznienie OFF.
10. Wylaczenie switcha "Nocte Automatyczne wybudzanie" oddaje sterowanie oryginalnej logice Tessera.
11. Tryb OFF jest uwzgledniony w `APPLY_BACKLIGHT` — odswiezanie ustawien Tessera nie wlacza podswietlenia przypadkowo.

**Wazne:** aby Sleep byl rzeczywiscie widoczny, ustaw w Tessera **Standby brightness >0%**, np. 20% (sprawdz tez ustawienie jasnosci nocnej). OFF wymusza 0% i fizyczne wykluczenie BL_EN.

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
7. Przy D1=80 cm, D2=140 cm, opoznienie=5 s wykonaj:
   - Test A: 50 -> 91 cm: Wake -> Sleep; poczekaj 15 s, ekran MA pozostac przyciemniony (bez OFF).
   - Test B: 91 -> 170 cm: Sleep -> licznik 5 s -> OFF.
   - Test C: 50 -> 170 cm w jednym kroku pomiaru: Wake -> licznik 5 s -> OFF.
   - Test D: po OFF wroc na 91 cm: ekran MA natychmiast przywrocic Sleep; nastepnie zejdz na 50 cm: Wake.
   - Test E: sprawdz dotyk i reczne Wake; przelacznik OFF automatyki powinien przywrocic zwykle dzialanie Tessera.


Uwaga: pliki w repozytorium nie zawieraja hasel ani kluczy z Twojego aktualnego panelu. Nie uruchamiaj osobno `showroom.local.yaml`.
