# Nocte Offline Runtime — plan wdrożenia v0.1

Baza: `nocte-v0.4.90` (showroom: Waveshare ESP32-S3 N8R8, 480x480 ST7701S, FT6336U 0x48).
Nie modyfikować gałęzi produkcyjnej ani prywatnych `showroom.yaml` / `secrets.yaml`.

## Cel pierwszego etapu
Po jednorazowej poprawnej synchronizacji panel ma odtworzyć po restarcie kompletny układ ekranów i kafelków, także bez Wi-Fi i HA. Nawigacja lokalna, dotyk i autowygaszanie Nocte działają offline. Encje HA bez świeżych danych pokazują status niedostępny, nie stan aktywny.

## Potwierdzone punkty integracji z kodem
- `components/smart_display/page_receiver.cpp`: `receive()`, op `begin`, `page`, `tile`, `appearance`, `bar`, `commit` i `ping`; dopiero pomyślny `commit` zatwierdza kompletny layout.
- `components/smart_display/runtime_tiles.h`: `model`, `transfer`, `feed_alive()`, `fresh()`, `render()`, `boot_view()`, `tick()`; osobne lokalne uprawnienie do akcji, niezależne od `ha_connected()`.
- `packages/core.yaml`: ESPHome `api: screen_message` i inicjalizacja `on_boot`.
- `examples/showroom.local.yaml`: działające sterowniki LCD/FT6336U, GPIO6 i ekspander 0x20 — bez zmian funkcjonalnych w etapie offline.

## Moduły
1. `nocte_config_store`: zapis i walidacja ostatniego kompletnego układu (magic, schema_ver, rev, length, CRC, committed), 2 generacje A/B; zapisy wyłącznie po pełnej poprawnej synchronizacji, bez zapisu każdej zmiany stanu.
2. `nocte_layout_restore`: start z ostatniej poprawnej konfiguracji; nie odtwarzać starych tokenów sesji `session`/liczników `seq` 1:1 — utworzyć osobny, kontrolowany mechanizm rekonstruowania modelu lub sesję odtworzeniową z nowymi metadanymi.
3. `nocte_status`: `LOCAL_READY`, `SYNCING`, `ONLINE`, `OFFLINE`, `CONFIG_ERROR`; status sieci ma być dyskretny, nie blokować ekranów.
4. `nocte_event_bus`: niezależne zdarzenia UI, wejść, komunikacji i NVT; docelowo `UI_TOUCH`, `IO_INPUT`, `CAN_RX`, `NVT_CHANGE`.
5. Interfejsy przyszłe bez inicjalizowania nieobecnego sprzętu: `nocte_io`, `nocte_can`, `nocte_nvt`, `nocte_bindings`, `nocte_ha_sync`.

## Kluczowe decyzje
- Konfiguracja widoku to nie bieżące stany encji. Po restarcie offline nie pokazywać zapamiętanego `ON` jako potwierdzonego stanu urządzenia.
- Dla przyszłych akcji: `HA_SERVICE` dostępne tylko online; `LOCAL_IO` i `NOCTE_CAN` nie zależą od HA; wymagana obsługa potwierdzeń i nieznanych stanów.
- Zachować oryginalne schematy transmisji menedżera, w tym późniejsze `appearance` / `bar`, bez konieczności ponownego generowania firmware'u.
- Zasoby graficzne osadzone w firmware zostają dostępne. Grafiki pobierane po HTTP wymagają osobnej polityki cache.
- Sprawdzić rzeczywistą tabelę partycji na 8 MB flash oraz rozmiary dwóch OTA przed przydzieleniem pamięci na dane. Nie zakładać z góry 1–2 MB dostępnego FS.
- Nie zapisywać konfiguracji do Flash przy każdej zmianie stanu, aby ograniczyć zużycie.
- Aktualizacja konfiguracji transakcyjna: najpierw zapis nowego pokolenia, CRC + schema check, następnie atomowe wskazanie aktywnego; na awarię fallback do poprzedniego.
- Nie zapisywać kluczy API/OTA/Wi-Fi w magazynie układu wizualizacji.

## Kryteria akceptacji (pierwszy etap)
1. Po udanym załadowaniu ekranów z HA, odłączyć Wi-Fi i zresetować panel: pełne kafelki i nawigacja wracają bez sieci.
2. Przez 30 min bez HA ekran nie przechodzi na pełnoekranowy błąd połączenia.
3. Stan encji HA bez danych: offline/niedostępny. Lokalny stan może pozostać dostępny po odczycie urządzenia.
4. Test aktualizacji układu + restartu + zmian `appearance`; aktywny zostaje ostatni kompletny układ.
5. Przerwać zapis w połowie / uszkodzić CRC: panel uruchamia poprzednią poprawną konfigurację.
6. Odzyskać Wi-Fi/HA: bez migotania, zbędnego przeładowania, czy utraty działającej lokalnej nawigacji.
7. Działające dotyk FT6336U, ST7701S, podświetlenie i Wake/Sleep/OFF bez regresji.
8. Potwierdzić rozmiar binarny i wolne zasoby RAM/PSRAM/Flash po kompilacji.

## Etap II
Wprowadzić lokalne `IO_OUTPUT`, `IO_INPUT` i testowy kafelek lokalny (bez fizycznego DCAN500); adapter sprzętowy do TCA9535/TCA9534 zostawić wyłączony, dopóki PCB nie istnieje.

## Etap III
TWAI -> DCAN500, NVT, bindingi z Network Designera, mapowanie lokalne, kolejki/priorytety i potwierdzenia. Synchronizacja z HA po Wi-Fi staje się transportem konfiguracji i integracji.
