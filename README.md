# Hewalex PCCO — mapa rejestrów RS485 (protokół GECO) + integracja z Home Assistant

> **EN:** Register map of the **Hewalex PCCO** heat pump (PG-426 controller, MG-426 executive module), reverse-engineered over RS485 (GECO protocol, not Modbus). Used with a USR-DR164 RS485→Wi-Fi converter and a Python → MQTT bridge for Home Assistant. Unofficial; writing settings is at your own risk. Docs are in Polish — tables are mostly self-explanatory.

**Dokumenty:**

- [`README.md`](README.md) — mapa rejestrów (ten plik),
- [`hewalex-hardware.md`](hewalex-hardware.md) — podłączenie konwertera USR-DR164 do pompy,
- [`home-assistant.md`](home-assistant.md) — co zrobić, żeby pompa działała w Home Assistant.

Rejestry pompy ciepła **Hewalex PCCO** (monoblok R32) ze sterownikiem **PG-426** i modułem wykonawczym **MG-426**, czytane przez port RS485 modułu (port „modemowy”, ten sam, którego używa EKO-LAN). Opis powstał przy budowie dodatku Home Assistant `hewalex2mqtt`. To obserwacje z jednej instalacji, a nie dokumentacja producenta. Stan na **16.10.2026**.

Oznaczenia pewności:

- ✅ **potwierdzone**: sprawdzone z panelem pompy albo testem;
- 🟡 **prawdopodobne**: zgadza się z obserwacją, ale nie ma twardego testu;
- ❓ **nieznane**: rejestr się zmienia albo ma stałą wartość, znaczenie nieustalone.

---

## 1. Połączenie i protokół

- **Interfejs:** RS485, 38400 bps, 8N1. Konwerter RS485 → TCP (np. USR-DR164), jeden klient TCP naraz.
- **Adresy:** urządzenie fizyczne 4, logiczne 2; kontroler (my) 1 / 1.
- **To NIE jest Modbus**, tylko protokół **GECO** (Hewalex/Geco).

### Ramka

Ramka składa się z nagłówka i części właściwej:

```
nagłówek:  69 <to> <from> 84 00 00 <len> <crc8>
treść:     <toSoft16 LE> <fromSoft16 LE> <FNC> 80 00 <regLen> <regStart16 LE> [dane] <crc16>
```

- CRC nagłówka: CRC-8, wielomian 0xD5.
- CRC treści: CRC-16, wielomian 0x1021, big-endian.
- Kody funkcji `FNC`:

| FNC | Znaczenie |
|---|---|
| `0x40` | żądanie odczytu |
| `0x50` | odpowiedź na odczyt |
| `0x60` | żądanie zapisu |
| `0x70` | potwierdzenie zapisu |

- **NACK** to odpowiedź `0x50` z bajtem `0xC0` i bez danych.
- Maksymalnie **226 rejestrów** (słów) na jedno zapytanie.
- Adresy rejestrów są parzyste: jeden „rejestr” to słowo 16-bit, a adresy idą co 2.

### Architektura

Panel PG-426 jest kontrolerem i rozmawia z modułem MG-426. Port RS485 rozmawia z **modułem**. Rejestry od 120 w górę to w dużej części **kopia stanu panelu**:

- nastawy z bloku 1070+ **dają się zapisać** i panel je przejmuje;
- **zegara nie da się zapisać**, bo panel go nadpisuje i ustawia się go tylko na panelu.

### Typy danych

| Typ | Odczyt | Uwagi |
|---|---|---|
| `te10` | int16 ze znakiem / 10 → °C | −50,0 = czujnik rozwarty („Rozw.”), 130,0 = zwarty („Zwar.”) |
| `fl10` | uint16 / 10 | prąd [A], ciśnienie [bar] |
| `word` | uint16 | liczby, tryby, maski bitowe |
| `s16` | int16 ze znakiem | |

### Bloki czytane przez `hewalex2mqtt`

| Start | Liczba słów | Zawartość |
|---|---|---|
| 120 | 182 | data, temperatury T1–T15, słowa stanu, wyjścia |
| 300 | 32 | obieg chłodniczy |
| 408 | 64 | wartości wyliczone, alarmy, tryb |
| 1070 | 174 | nastawy |
| 2120 | 4 | kody P/F jednostki zewnętrznej (moduł odpowiada NACK) |

### Zasięg (skan wszystkich adresów 0–2600, 16.10.2026)

- Moduł odpowiada w przedziale **ok. 120–1355**.
- Bloki od 0 i od 1356 do 2600 zwracają NACK.
- Przy kolejnych przebiegach NACK zdarzał się też dla pojedynczych bloków w środku zakresu. To prawdopodobnie zajętość magistrali, a nie brak rejestrów.

---

## 2. Data i czas (tylko odczyt) ✅

| Rejestr | Bajt niski | Bajt wysoki |
|---|---|---|
| 120 | rok % 100 | miesiąc |
| 122 | dzień | dzień tygodnia (0 = poniedziałek) 🟡 |
| 124 | godzina | minuta |
| 126 | sekunda | ❓ |

- Panel pokazuje datę w formacie **rr-mm-dd**.
- Zapis zegara przez RS485 nie działa.
- Po zaniku zasilania zegar trzeba ustawić na panelu.

---

## 3. Czujniki temperatur (`te10`) ✅

| Rejestr | Czujnik | Uwagi |
|---|---|---|
| 130 | **T1** | zasobnik CWU |
| 132 | **T2** | bufor CO |
| 134 | T3 | |
| 136 | T4 | |
| 138 | T5 | |
| 140 | T6 | |
| 142 | T7 | |
| 144 | T8 | |
| 146 | T9 | |
| 148 | T10 | |
| 150 | T11 | |
| 152 | **T12** | temperatura pokojowa obiegu CO1 |
| 154 | T13 | |
| 156 | **T14** | temperatura zewnętrzna |
| 158 | T15 | |
| 160 | EXT | |

W tej instalacji nie wszystkie czujniki są podłączone; niepodłączone pokazują −50,0.

---

## 4. Słowa stanu i wyjścia

| Rejestr | Znaczenie | Pewność |
|---|---|---|
| 162–192 | słowa stanu | ❓ do rozszyfrowania |
| 194 | maska stanu, w spoczynku `0xA37F`; **bit 7 miga co ~15 s przy pracy sprężarki** | 🟡 (bit 7 to szum) |
| 196 | **bit 1 = ON przy grzaniu CWU** | 🟡 |
| 198 | słowo stanu | ❓ |
| **208** | **wyjścia sterownika (przekaźniki)**, tabela niżej | ✅ / 🟡 |

### Rejestr 208: wyjścia

| Bit | Maska | Wyjście | Pewność | Jak ustalone |
|---|---|---|---|---|
| 0 | `0x01` | sprężarka / pompa ładująca | 🟡 | ON przy grzaniu; gaśnie ~3 min po bicie 3 (dobieg) |
| 2 | `0x04` | sprężarka / pompa ładująca | 🟡 | jak bit 0 |
| 3 | `0x08` | **grzanie CWU** (zawór 3-drogowy na CWU) | ✅ | ON przy grzaniu wody, gaśnie przy T1 = zadana |
| 5 | `0x20` | **pompy obiegowe CO1** (podłogówka) | ✅ | test: zadana CO1 powyżej T12 → bit 5 od razu ON, T2 spada |
| 6 | `0x40` | pulsuje co ~15–30 s przy pracy CO1 | ❓ | zawór / pompa? |
| 7 | `0x80` | nigdy nie widziany | ❓ | |

Bit wbudowanej grzałki elektrycznej pompy nie jest jeszcze ustalony; trzeba go obserwować przy mrozie, poniżej temperatury biwalentnej.

---

## 5. Obieg chłodniczy (blok 300)

| Rejestr | Znaczenie | Typ | Pewność |
|---|---|---|---|
| 300 | **Ta1**: temperatura powietrza (jednostka zewnętrzna) | te10 | ✅ |
| 302 | **Tp1**: temperatura parowania | te10 | ✅ |
| 304 | **Td1**: temperatura tłoczenia | te10 | ✅ |
| 306 | **Ts1**: temperatura ssania | te10 | ✅ |
| 308, 310, 312, 314 | stałe 1 | word | ❓ |
| 316 | **EV1**: elektroniczny zawór rozprężny (kroki) | word | ✅ |
| 318 | **I1**: prąd sprężarki [A] | fl10 | ✅ |
| 320 | **U1**: napięcie zasilania [V] | word | ✅ |
| 322 | 36–67 w trakcie pracy, 0 po zatrzymaniu; rośnie z I1 → **wydajność / częstotliwość sprężarki** | word | 🟡 |
| 324, 328 | stałe 1 | word | ❓ |
| 326 | **HP1**: wysokie ciśnienie [bar] | fl10 | ✅ |
| 330 | **LP1**: niskie ciśnienie [bar] | fl10 | ✅ |
| 332 | 0 | word | ❓ |
| 334 | 64 / 80 w trakcie pracy, 64 po zatrzymaniu → bieg wentylatora? | word | 🟡 |
| 336 | 8 przez kilka minut przy szczycie prądu → ograniczenie / zabezpieczenie? | word | ❓ |
| 338 | 0 → 4096 (bit 12) przy zatrzymaniu sprężarki → postój / blokada ponownego startu? | word | ❓ |

Moc sprężarki można szacować jako U1 × I1 (moc pozorna, błąd ok. ±15 %).

---

## 6. Wartości wyliczone, alarmy, tryb (blok 408)

| Rejestr | Znaczenie | Typ | Pewność |
|---|---|---|---|
| 400 | ~4883, zmienia się o ±1 → licznik (godziny pracy?) | word | ❓ |
| 402 | `0x5A00` → `0x6400` przy zatrzymaniu (starszy bajt 90 → 100) | word | ❓ |
| 408 | **CWU zadana** (aktualna, z programu komfort/eko) | te10 | ✅ |
| 410 | **CO1 zasilanie zadane** (z krzywej grzewczej) | te10 | ✅ |
| 412 | CO2 zasilanie zadane | te10 | ✅ |
| 414 | **alarmy G** (komunikacja), bity niżej | word | ✅ |
| 418 | **alarmy S** (czujniki), bity niżej | word | ✅ |
| 422 | **alarmy C / S15**, bity niżej | word | ✅ |
| 436 | **bit 8 (`0x100`) = obieg CO1 włączony** | word | ✅ |
| 440 | **tryb pracy**: 1 = chłodzenie, 2 = CWU, 3 = grzanie | word | ✅ |
| 450 | 25–32, zmienia się w trakcie pracy | word | ❓ |
| 452 | 119 → 135 | word | ❓ |
| 456, 466 | 0 / 65 / 120 (też 10, 89, 98, 102) cyklicznie przy pracy → liczniki czasu [s]? | word | ❓ |
| 460 | 1–3 w trakcie pracy → etap pracy? | word | ❓ |
| 468 | **CO1 zasilanie: próg dolny** (= 410 − histereza wody, 5 °C) | te10 | ✅ |
| 470 | **CO1 zasilanie: próg górny** (= 410) | te10 | ✅ |
| 472, 476 | 2 → 0 przy zatrzymaniu sprężarki | word | ❓ |

### Bity alarmów

**414: komunikacja**

| Bit | Kod | Opis |
|---|---|---|
| 0 | G01 | brak komunikacji panel → moduł |
| 1 | G02 | brak komunikacji moduł → panel |
| 2 | G03 | brak komunikacji z jednostką zewnętrzną |
| 3 | G04 | błąd komunikacji |

**418: czujniki**

| Bit | Kod | Czujnik |
|---|---|---|
| 0 | S01 | T1 (CWU) |
| 1 | S02 | T2 (bufor) |
| 2 | S03 | T3 |
| 3 | S04 | T4 |
| 4 | S05 | T5 |
| 5 | S06 | T6 |
| 6 | S07 | T7 |
| 7 | S08 | T8 |
| 8 | S09 | T9 |
| 9 | S10 | T10 |
| 10 | S11 | T11 |
| 11 | S12 | T12 (pokojowy) |
| 12 | S13 | T13 |
| 13 | S14 | T14 (zewnętrzny) |

**422**

| Bit | Kod | Opis |
|---|---|---|
| 0 | C01 | |
| 1 | C02 | |
| 3 | S15 | brak przepływu (filtr / zapowietrzenie / ciśnienie) |

**2120 / 2121: kody jednostki zewnętrznej** (wg instrukcji serwisowej; ten moduł odpowiada na nie NACK)

| Rejestr | Bit | Kod | Opis |
|---|---|---|---|
| 2120 | 0 | P01 | zasilanie |
| 2120 | 1 | P02 | fazy sprężarki |
| 2120 | 2 | P03 | IPM |
| 2120 | 3 | P04 | korki oleju ⓘ |
| 2120 | 4 | P05 | presostat HP |
| 2120 | 5 | P06 | wysokie ciśnienie |
| 2120 | 6 | P07 | podgrzewanie oleju ⓘ |
| 2120 | 7 | P08 | temperatura tłoczenia |
| 2120 | 8 | P09 | parownik |
| 2120 | 9 | P10 | napięcie AC |
| 2120 | 10 | P11 | T zewnętrzna poza zakresem |
| 2120 | 11 | P12 | prędkość ⓘ |
| 2120 | 12 | P13 | presostat LP |
| 2121 | 0 | F01 | czujnik Ta |
| 2121 | 1 | F02 | czujnik Tp |
| 2121 | 2 | F03 | czujnik Td |
| 2121 | 3 | F04 | czujnik Ts |
| 2121 | 4 | F05 | czujnik HP |
| 2121 | 5 | F06 | czujnik LP |
| 2121 | 6 | F07 | blokada (3× P05/P13) |

ⓘ to komunikaty informacyjne, które pojawiają się przy normalnej pracy, a nie alarmy.

---

## 7. Nastawy (blok 1070+)

**Zapis działa** (FNC `0x60` → `0x70`), panel przejmuje wartość. Zakresy w tabeli to limity przyjęte w `hewalex2mqtt`, a nie limity producenta.

| Rejestr | Nastawa | Typ | Zakres (raw) | Zapis | Pewność |
|---|---|---|---|---|---|
| **1070** | **CWU: temperatura komfort** | te10 | 350–550 (35–55 °C), krok 0,5 | ✅ | ✅ |
| 1072 | CWU eko? | te10 | — | — | ❓ |
| 1074 | ? | te10 | — | — | ❓ |
| **1176** | **CO1: temperatura pokojowa komfort** | te10 | 150–260 (15–26 °C), krok 0,1 | ✅ | ✅ |
| **1178** | **CO1: temperatura pokojowa eko** | te10 | 150–260 | ✅ | ✅ (zgodna z panelem) |
| 1182 | CO1: temperatura pokojowa (trzecia nastawa, np. nocna?); zapis 22,5 → ack, odczyt zgodny | te10 | 150–260 | ✅ | ❓ znaczenie w testach |
| **1184** | **CO1: histereza pokojowa** | fl10 | 1–20 (0,1–2,0 °C) | ✅ | ✅ |
| 1186 | 45,0 °C | te10 | — | — | ❓ |
| 1188 | 40,0 °C → kandydat na **CWU eko** | te10 | — | — | ❓ |
| **1192** | **CO1: krzywa grzewcza** | word | 1–9 | ✅ | ✅ |
| 1020 | −20,0 (to NIE jest „T zewnętrzna wyłączenia”, która na panelu ma −25) | te10 | — | — | ❓ |
| 1240 | 21,0 °C | te10 | — | — | ❓ |
| 1242 | 60,0 °C → max zasilania? | te10 | — | — | ❓ |

Wskazówki praktyczne:

- **Programy czasowe** (komfort/eko) zmienione na panelu potrafią go zawiesić i zrestartować. Bezpieczniej zostawić program w spokoju i zapisywać **tę samą wartość do komfort i eko** (np. 1176 i 1178). Wtedy program przestaje mieć znaczenie, a temperaturą steruje system nadrzędny.
- Biwalentna temperatura dla grzałki CO (na panelu −12,0 °C) nie jest jeszcze znaleziona w bloku nastaw; szukać raw `-120`.

---

## 8. Do zrobienia

- Rozszyfrować 162–192, 194/196/198 i bit wbudowanej grzałki (prawdopodobnie w 208 albo 196).
- Potwierdzić znaczenie 1182 (nocna zadana?) i 1188 (CWU eko?).
- Znaleźć rejestr, który pokazuje aktywny program komfort/eko.
- Potwierdzić kandydatów z bloków 300 i 400 (322, 334, 336, 338, 400, 402, 456/466, 460) na kilku dobach danych: grzanie CWU, CO1, odszranianie, antylegionella.
- Sprawdzić, czy 436 i 440 przyjmują zapis.
- Przy NACK dzielić blok na mniejsze; zakres 0–119 i powyżej 1356 może być częściowo dostępny.

---

## 9. Podpowiedzi z pompy PCWU (inna mapa, ten sam sterownik G-426)

W `mvdklip/Domoticz-Hewalex` (`devices/pcwu.py`) jest mapa dla pompy CWU **PCWU** (PG-426-P01 + MG-426-P01). Adresy **nie pokrywają się** z PCCO (np. T1 = 128, nastawy od 302), ale układ pól daje tropy:

- **Maska wyjść PCWU (rej. 196):** bit 0 = wentylator, bit 2 = pompa wody, bit 5 = „pompa ciepła ON”, bit 11 = sprężarka, bit 12 = grzałka E. W PCCO rej. 208 bity 0 i 2 też zapalają się przy grzaniu i gasną z dobiegiem → 🟡 **208 bit 0 = wentylator, bit 2 = pompa ładująca**. Bity 11/12 w 208 warto obserwować jako kandydatów na sprężarkę / grzałkę wbudowaną.
- **WaitingStatus PCWU (rej. 202):** 0 = gotowa, 2 = wyłączona, 32 = „właśnie zatrzymana, czeka na restart”, 64 = niski COP; **WaitingTimer (206)** odlicza do 0. W PCCO kandydaci: 338 (bit 12 po zatrzymaniu) i 456/466 (odliczanie).
- **Rejestr 252 (status panelu)** i **256 (ostatnio zmieniony parametr)** w komunikacji panel ↔ moduł PCWU: po zmianie parametru na panelu moduł czyta 8 rejestrów od 256 (kod parametru, urządzenie, wartość). W PCCO 252/256 leżą w czytelnym zakresie — **zmiana jednego parametru na panelu (nie harmonogramu!) powinna pokazać w skanie, który rejestr nastaw go przechowuje** (np. biwalentna −12).

---

## Historia zmian

- **16.10.2026 (2):** sekcja 9 — tropy z mapy PCWU (208 bit 0/2, status oczekiwania, 252/256); `hewalex2mqtt` 1.1.2 dzieli bloki z NACK, żeby znaleźć częściowo dostępne zakresy 0–119 i > 1356 (wynik: `obraz.json` → `zakresy`).
- **16.10.2026:** pierwsza wersja: mapa z `hewalex2mqtt` 1.1.1, wyjścia z rejestru 208 (test 15.10), pierwszy przegląd skanu 0–2600 (kandydaci w blokach 300 i 400).

---

## Źródła i podziękowania

- [mvdklip/Domoticz-Hewalex](https://github.com/mvdklip/Domoticz-Hewalex) (GPL-3.0): protokół GECO, typy rejestrów (PCWU).
- [aelias-eu/hewalex-geco-protocol](https://github.com/aelias-eu/hewalex-geco-protocol) (MIT): opis ramek GECO.
- Instrukcja serwisowa Hewalex PCCO: kody alarmów G/S/C/P/F.
- Własne obserwacje: odczyty przez RS485, testy zapisu, porównanie z panelem PG-426, pełny skan adresów 0–2600 (10–16.10.2026).

Rejestry pochodzą z jednej instalacji (PG-426 G-426-P03, MG-426 MG426-P02-2U3). Inne wersje firmware mogą się różnić. Zapis nastaw robisz na własne ryzyko.
