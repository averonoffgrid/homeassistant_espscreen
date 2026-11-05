# Nocte ESPHome screen

Dostosowanie interfejsu [Tessera / homeassistant_espscreen](https://github.com/MaxGramser/homeassistant_espscreen) do panelu Nocte (Waveshare ESP32-S3-LCD-Driver-Board, ekran 480×480, ST7701S, FT6336U).

## Zasady
- **Nie publikuj** prywatnego `showroom.yaml` zawierającego klucze ani `secrets.yaml`. W `examples/showroom.yaml` znajduje się wyłącznie publiczny szablon bez haseł.
- Zachowuj działającą konfigurację ST7701S, dotyku pod adresem I²C 0x48 i BL_EN.
- Prawidłowe źródła zależności: pakiety Tessera i `smart_display` z **tej samej wersji** kodu.
- Obrazy logo są częścią firmware; ich podmiana wymaga nowej kompilacji i aktualizacji OTA.
- Modyfikacje Nocte będą przygotowywane w osobnej gałęzi i sprawdzane przed wdrożeniem.

Projekt pierwotny udostępniony jest na licencji AGPL-3.0. Zachowaj jego licencję i informacje o autorach przy dalszej dystrybucji.

## Pliki do ESPHome
- [examples/showroom.yaml](examples/showroom.yaml) — bezpieczny publiczny szablon. Przed kompilacją na urządzeniu uzupełnij lokalne sekrety albo zachowaj dotychczasowe wpisy API/OTA/Wi-Fi ze swojej konfiguracji.
- [examples/showroom.local.yaml](examples/showroom.local.yaml) — pełna konfiguracja Waveshare/ST7701S/FT6336U/PWM, bez kluczy i haseł; skopiuj ją obok `showroom.yaml` jako `showroom.local.yaml`.

Nie należy wybierać `showroom.local.yaml` jako osobnego urządzenia w ESPHome Device Builder. Kompiluj wyłącznie główny `showroom.yaml`, który zawiera `local_overrides: !include showroom.local.yaml`.
