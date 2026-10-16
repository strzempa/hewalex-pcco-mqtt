# Hewalex PCCO + Home Assistant – część hardware'owa

Podłączenie konwertera **USR-DR164** (RS485 → Wi-Fi) do pompy ciepła Hewalex PCCO w miejsce modemu GECO EKO-LAN (G567.01-1B1).

## Co było potrzebne

- Konwerter **PUSR USR-DR164** (RS485 ↔ Wi-Fi 2.4/5.8 GHz) + antena SMA (w zestawie)
- Kawałek skrętki Ethernetowej (żyły jednodrutowe, solid-core) na przedłużki
- Szczypce do ściągania izolacji, ew. termokurczki / kostka zaciskowa
- Mały płaski wkrętak (do dźwigni push-in)
- Spinacz / cienki przedmiot do przycisku Reload

## Gdzie się podłączyć

Złącze śrubowe na płycie **MG-426** (moduł wykonawczy) w jednostce wewnętrznej – to samo, do którego był wpięty modem GECO. Są tam 4 przewody:

| Opis przewodu | Funkcja |
|---|---|
| `12V+` | zasilanie + |
| `12V-` | zasilanie – |
| `RS1A+` | RS485 linia A |
| `RS2B-` | RS485 linia B |

Modem GECO został wypięty (Ekontrol nie jest używany, więc nie jest potrzebny). Zasilanie 12V z płyty w zupełności wystarcza – DR164 przyjmuje 5–36 V DC.

## Mapowanie na USR-DR164

Złącze 5-pinowe push-in (od lewej):

| Pin DR164 | Przewód z pompy |
|---|---|
| 1 – DC+ | `12V+` |
| 2 – DC- | `12V-` |
| 3 – RX/A | `RS1A+` |
| 4 – TX/B | `RS2B-` |
| 5 – GND | niepodłączony |

> Jeśli brak komunikacji (dioda COM nie miga) – zamień A i B miejscami. Dopiero w razie zakłóceń dodaj GND.

## Problem z grubością przewodów i rozwiązanie

Przewody Hewalex mają grube tulejki zaciskowe, które nie mieszczą się w drobnym złączu push-in DR164 (rozstaw pinów ok. 2–2,5 mm, zamienników śrubowych praktycznie brak).

**Rozwiązanie:** przedłużki z żył skrętki Ethernetowej (jednodrutowych – idealnie pasują do push-in):

1. Ściągnąć ok. 8–10 mm izolacji z żyły skrętki.
2. Prostą końcówkę wcisnąć w okrągły otwór pod pomarańczową dźwignią (zatrzaśnie się sama). Gdy nie chce wejść lub trzeba wyjąć – wcisnąć dźwignię wkrętakiem.
3. Drugi koniec połączyć z przewodem Hewalex (kostka zaciskowa lub skręcenie + termokurczka).
4. Opisać każdą przedłużkę tak jak oryginał (12V+, 12V-, RS1A+, RS2B-).

## Kolejność prac

1. Wyłączyć zasilanie pompy ciepła (bezpiecznik).
2. Wypiąć 4 przewody z modemu GECO.
3. Podłączyć przedłużki do DR164 wg tabeli wyżej.
4. Wkręcić antenę SMA w DR164.
5. Włączyć zasilanie – świecić powinna dioda **POW**, a **WORK** migać.

## Ustawienia DR164 (wymagane)

Panel: `http://10.10.100.254` (fabrycznie login/hasło `admin`/`admin` – po pierwszym logowaniu zmień hasło; łączysz się z siecią `USR-DR164-xxxx`).

**Port szeregowy:**

- Baud rate: **38400**
- Data bits: **8**
- Parity: **None**
- Stop bits: **1**
- Flow control: **brak**

**Sieć (żeby HA mógł się połączyć):**

- Tryb Wi-Fi **STA** – dołącz do domowej sieci (najlepiej 2,4 GHz), **stały adres IP** (w DR164 albo rezerwacja DHCP na routerze).
- Socket: **TCP Server**, port **8899**, protokół *None* (transparentny); bez heartbeat/registry/Modbus gateway.

Po resecie fabrycznym (przycisk Reload 4–15 s) te wartości trzeba sprawdzić ponownie.

## Weryfikacja

- Dioda **COM** miga = urządzenie odbiera dane z magistrali RS485 pompy.
- Panel sterujący PG-426 musi pozostać włączony – moduł wykonawczy ignoruje komendy zapisu (np. włączenie pompy), gdy panel jest wyłączony.
- W panelu PG-426 (menu serwisowe → port RS485): **38400**, adres fizyczny **4**, logiczny **2**.

## Co dalej (software)

Protokół Hewalex/GECO nie jest standardowym Modbusem. Mapa rejestrów PCCO: [`README.md`](README.md). Most Python → MQTT → Home Assistant (dodatek `hewalex2mqtt`, łączy się z DR164 jako klient TCP) i konfiguracja HA: [`home-assistant.md`](home-assistant.md).

Pokrewne projekty (pompa PCWU / ten sam sterownik G-426): `mvdklip/Domoticz-Hewalex` (`hewalex_geco/devices/pcwu.py`) oraz `aelias-eu/hewalex-geco-protocol`.
