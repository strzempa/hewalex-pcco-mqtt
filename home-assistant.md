# Hewalex PCCO w Home Assistant — krok po kroku

Co trzeba zrobić, żeby pompa Hewalex PCCO (panel PG-426, moduł MG-426) była widoczna i sterowalna w Home Assistant. Tor danych:

```
pompa (MG-426, port RS485 „modemowy”)
   └─ USR-DR164 (RS485 → Wi-Fi, TCP Server :8899)
        └─ hewalex2mqtt (dodatek HA, Python, protokół GECO)
             └─ MQTT (Mosquitto)
                  └─ Home Assistant (MQTT discovery: sensory, nastawy, climate, water_heater)
```

Sprzęt (podłączenie DR164 do pompy) opisuje [`hewalex-hardware.md`](hewalex-hardware.md), a znaczenie rejestrów [`README.md`](README.md).

---

## 1. Konwerter USR-DR164: sieć i tryb pracy

Port szeregowy (38400 / 8 / None / 1) jest opisany w części sprzętowej. Do pracy z HA ustaw jeszcze sieć:

1. **Tryb Wi-Fi: STA** (klient). Konwerter łączy się z domową siecią. Najlepiej 2,4 GHz, z dobrym zasięgiem przy jednostce wewnętrznej.
2. **Stały adres IP.** Ustaw go w DR164 albo zrób rezerwację DHCP na routerze. HA łączy się po adresie IP, a jego zmiana = brak danych.
3. **Socket: TCP Server, port 8899** (domyślny w DR164), protokół *None* (transparent). Jeśli ustawisz inny port, wpisz go potem w opcjach dodatku.
4. **Jeden klient naraz.** DR164 obsługuje jedno połączenie TCP. Nie podłączaj równocześnie innego programu (np. testowego skryptu z laptopa), bo dodatek straci połączenie.
5. Wyłącz w DR164 funkcje, które wtrącają się w strumień danych: heartbeat, rejestrację (*Registry*), Modbus Gateway.

Kontrola: dioda **COM** miga, gdy przez magistralę idą dane.

## 2. Panel PG-426: port RS485

W menu serwisowym panelu (hasło serwisowe jest w instrukcji serwisowej PCCO) sprawdź ustawienia portu RS485 modułu:

| Parametr | Wartość |
|---|---|
| Prędkość | **38400** |
| Adres fizyczny | **4** |
| Adres logiczny | **2** |

Dodatek przedstawia się jako kontroler o adresie 1 / 1 i pyta urządzenie 4 / 2. Jeśli w panelu masz inne adresy, ustaw zmienną `HEWALEX_IDS` w `run.sh` (domyślnie `1,1;4,2`, format `kontroler;urządzenie`).

**Panel musi być włączony.** Gdy panel jest wyłączony, moduł ignoruje zapisy z zewnątrz.

## 3. Home Assistant: broker MQTT

1. **Ustawienia → Dodatki → Sklep** → zainstaluj **Mosquitto broker** i uruchom go.
2. **Ustawienia → Urządzenia i usługi**: HA sam zaproponuje integrację **MQTT**. Zatwierdź ją.

Dodatek `hewalex2mqtt` pobiera dane brokera automatycznie od Supervisora. Zewnętrzny broker ustawisz w opcji `mqtt_override`.

## 4. Dodatek `hewalex2mqtt` (dodatek lokalny)

Dodatek instaluje się jako **dodatek lokalny**, czyli katalog w `/addons` na HA.

> Kod dodatku nie jest jeszcze opublikowany w tym repozytorium (będzie w katalogu `hewalex2mqtt/`). Korzysta z biblioteki `hewalex_geco` z projektu Domoticz-Hewalex i dodaje własną klasę urządzenia PCCO.

Instalacja:

1. Skopiuj katalog `hewalex2mqtt/` (pliki `config.yaml`, `build.yaml`, `Dockerfile`, `run.sh`, `hewalex2mqtt.py`) do `/addons/hewalex2mqtt/`. Możesz to zrobić przez dodatek **Samba share** (udział `addons`), **Studio Code Server** albo **SSH**.
2. **Ustawienia → Dodatki → Sklep → ⋮ → Sprawdź aktualizacje.** Na liście pojawi się sekcja **Dodatki lokalne** z „Hewalex PCCO to MQTT”.
3. Zainstaluj dodatek (budowa obrazu trwa kilka minut), uzupełnij opcje i go uruchom.
4. W zakładce **Dziennik** powinno być widać: `polaczono z <IP>:8899 (modul 4/2)`, `polaczono z brokerem` i `opublikowano discovery dla … encji`.

### Opcje dodatku

| Opcja | Domyślnie | Co robi |
|---|---|---|
| `host` | — | **adres IP konwertera DR164** |
| `port` | `8899` | port TCP Server w DR164 |
| `interval` | `15` | co ile sekund czytać stan (5–300) |
| `config_every` | `4` | co który cykl czytać blok nastaw 1070+ |
| `device_name` | `Pompa ciepła Hewalex` | nazwa urządzenia w HA |
| `base_topic` | `hewalex` | prefiks tematów MQTT |
| `allow_write` | `true` | zapis nastaw; `false` = tylko odczyt (encje `number`/`climate` bez sterowania) |
| `eko_rowne_komfort` | `true` | każdy zapis temperatury komfort idzie też do rejestru eko (1176 → 1178), więc harmonogram panelu przestaje mieć znaczenie |
| `cwu_eko_reg` | `0` | rejestr CWU eko (0 = nie zapisuj; kandydat 1188, niepotwierdzony) |
| `co1_lustro_1182` | `false` | zapis zadanej CO1 także do 1182 (w testach) |
| `skan_rejestrow` | `true` | wolny skan całej pamięci modułu i dziennik zmian w `/share/hewalex2mqtt/skan/` (do rozpracowywania rejestrów; można wyłączyć) |
| `skan_od` / `skan_do` | `0` / `2600` | zakres skanu |
| `log_level` | `info` | `debug` pokazuje ramki |
| `mqtt_override` | wyłączone | ręczny broker MQTT (host/port/user/hasło) |

## 5. Co pojawi się w Home Assistant

Po starcie dodatku HA sam utworzy urządzenie **„Pompa ciepła Hewalex”** (MQTT discovery), z około 70 encjami:

- **Temperatury:** T1 zasobnik CWU, T2 bufor, T12 pokojowa, T14 zewnętrzna (plus T3–T15, jeśli podłączone).
- **Obieg chłodniczy:** Ta1, Tp1, Td1, Ts1, zawór EV1, prąd I1, napięcie U1, ciśnienia HP1/LP1, moc szacowana (U·I) i energia sprężarki.
- **Stan:** tryb pracy, CO1 włączony, CWU zadana aktualna, CO1 zasilanie zadane z krzywej i progi.
- **Wyjścia (rej. 208):** pompy podłogówki CO1, grzanie CWU, pompa ładująca/sprężarka.
- **Alarmy:** lista kodów G/S/C, liczba alarmów, komunikaty informacyjne, binary_sensor „alarm pompy”.
- **Zegar sterownika** i jego rozjazd z czasem HA (zegar ustawia się tylko na panelu).
- **Nastawy (`number`):** CWU komfort, CO1 pokojowa komfort/eko, histereza CO1, krzywa grzewcza.
- **`climate` CO1** (zadana pokojowa + T12) i **`water_heater` CWU** (zadana komfort + T1). Działają na zwykłych kartach termostatu.

Dane lecą też surowo na MQTT: `hewalex/state` (JSON), `hewalex/availability` (online/offline).

### Zapis nastaw

Encje `number`, `climate` i `water_heater` zapisują przez MQTT `hewalex/set/<klucz>` (wartość w °C albo liczba). Dodatek pilnuje zakresów:

| Klucz | Rejestr | Zakres |
|---|---|---|
| `cwu_komfort` | 1070 | 35–55 °C, krok 0,5 |
| `co1_pokojowa_komfort` | 1176 | 15–26 °C, krok 0,1 (+ eko 1178, gdy `eko_rowne_komfort`) |
| `co1_pokojowa_eko` | 1178 | 15–26 °C |
| `co1_histereza` | 1184 | 0,1–2,0 °C |
| `co1_krzywa` | 1192 | 1–9 |

Przykład w automatyzacji:

```yaml
action: number.set_value
target:
  entity_id: number.pompa_ciepla_hewalex_cwu_komfort
data:
  value: 47
```

Zapis jest potwierdzany ramką `0x70`, a odczyt kolejnego cyklu pokazuje nową wartość.

## 6. Dobre praktyki

- **Nie zmieniaj harmonogramów na panelu**, jeśli sterujesz z HA. Edycja programów czasowych potrafi zawiesić i zrestartować PG-426. Zamiast tego HA pisze tę samą wartość do komfort i eko (`eko_rowne_komfort: true`), a harmonogram przestaje mieć znaczenie.
- **Nie zapisuj za często.** Zmieniaj zadane tylko wtedy, gdy wartość naprawdę się zmienia, np. co 30 min, nie co cykl.
- **Zegar** ustawiaj na panelu (format rr-mm-dd), zwłaszcza po zaniku prądu. Encja „Zegar sterownika — różnica” pokaże rozjazd.
- **Zadana CO1 to temperatura pokojowa** (T12), nie temperatura wody. Wodą steruje krzywa grzewcza (rej. 1192) i bufor.
- Na start warto ustawić `allow_write: false`, zobaczyć dane przez dzień-dwa, a dopiero potem włączyć zapis.

## 7. Problemy

| Objaw | Co sprawdzić |
|---|---|
| Dioda COM nie miga | A/B zamienione, zły port na MG-426, brak zasilania 12 V |
| Dodatek: „brak odpowiedzi” / timeout | IP i port DR164, tryb TCP Server, czy inny program nie trzyma połączenia, adresy 4/2 w panelu |
| Pojedyncze „brak odpowiedzi” co kilka-kilkanaście minut | normalne przy słabym Wi-Fi DR164. Dodatek ponawia sam, a encje nie znikają przy pojedynczym błędzie |
| Zapis bez efektu | panel wyłączony, `allow_write: false`, wartość poza zakresem (patrz log) |
| NACK dla bloku | moduł nie udostępnia tego zakresu, np. 2120 (kody P/F) czy adresy < 120 i > 1356. Nie przeszkadza w pracy |
| Encje „niedostępne” po restarcie HA | poczekaj jeden cykl (~15 s). Konfiguracja encji (discovery) i dostępność są w MQTT z flagą retain, a stan przychodzi z najbliższym odczytem |

---

*Opis na podstawie działającej instalacji: Home Assistant OS 2026.10, Mosquitto, USR-DR164, PG-426 (G-426-P03), MG-426 (MG426-P02-2U3), dodatek `hewalex2mqtt` 1.1.2.*
