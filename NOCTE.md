# Nocte ESPHome screen

Dostosowanie interfejsu [Tessera / homeassistant_espscreen](https://github.com/MaxGramser/homeassistant_espscreen) do panelu Nocte (Waveshare ESP32-S3-LCD-Driver-Board, ekran 480×480, ST7701S, FT6336U).

## Zasady
- **Nie publikuj** plików `showroom.yaml`, `secrets.yaml`, kluczy API, haseł OTA ani haseł Wi-Fi w publicznym repozytorium.
- Zachowuj działającą konfigurację ST7701S, dotyku pod adresem I²C 0x48 i BL_EN.
- Prawidłowe źródła zależności: pakiety Tessera i `smart_display` z **tej samej wersji** kodu.
- Obrazy logo są częścią firmware; ich podmiana wymaga nowej kompilacji i aktualizacji OTA.
- Modyfikacje Nocte będą przygotowywane w osobnej gałęzi i sprawdzane przed wdrożeniem.

Projekt pierwotny udostępniony jest na licencji AGPL-3.0. Zachowaj jego licencję i informacje o autorach przy dalszej dystrybucji.
