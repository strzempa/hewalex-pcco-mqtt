# Hewalex PCCO — Mapa rejestrów RS485 (Protokół GECO)

Repozytorium zawiera opis i mapę rejestrów dla pompy ciepła **Hewalex PCCO** (monoblok R32) ze sterownikiem **PG-426** oraz modułem wykonawczym **MG-426**. 

Dane zostały wyciągnięte poprzez analizę i obserwację ruchu na porcie RS485 modułu (tzw. port modemowy, wykorzystywany fabrycznie przez moduł internetowy EKO-LAN). Wpisy powstały m.in. podczas prac nad integracją Home Assistant (`hewalex2mqtt`).

> ⚠️ **Uwaga:** Zaprezentowana mapa jest wynikiem inżynierii wstecznej prowadzonej na pojedynczej instalacji i nie stanowi oficjalnej dokumentacji producenta. Zapisywanie parametrów robisz na własną odpowiedzialność.

---

## 📌 Legendy i oznaczenia pewności danych

- ✅ **Potwierdzone:** Sprawdzone fizycznie na panelu pompy ciepła lub potwierdzone testem zapisu/odczytu.
- 🟡 **Prawdopodobne:** Wartości pokrywają się z obserwacjami pracy urządzenia, lecz brak pełnego testu.
- ❓ **Nieznane / Do rozszyfrowania:** Rejestr zmienia wartości lub ma wartość stałą, ale jego dokładna funkcja nie została ustalona.

---

## 1. Połączenie i protokół

Urządzenie **nie używa standardowego protokołu Modbus RTU**, lecz dedykowanego protokołu **GECO** (Hewalex/Geco).

* **Parametry transmisji:** RS485, **38400 bps**, 8 data bits, No parity, 1 stop bit (`8N1`).
* **Konfiguracja sprzętowa:** Do komunikacji można użyć konwertera RS485 $\rightarrow$ TCP (np. USR-DR164 / Waveshare) skonfigurowanego na jednego klienta TCP naraz.
* **Adresacja:**
  * Urządzenie fizyczne: `4`
  * Urządzenie logiczne: `2`
  * Kontroler (nasze oprogramowanie): `1` / `1`

### Budowa ramki GECO

Kompilacja ramki składa się z nagłówka oraz części właściwej:

$$\text{Ramka} = \underbrace{\texttt{69 <to> <from> 84 00 00 <len> <crc8>}}_{\text{Nagłówek}} \; \underbrace{\texttt{<toSoft16 LE> <fromSoft16 LE> <FNC> 80 00 <regLen> <regStart16 LE> [dane] <crc16>}}_{\text{Treść}}$$

* **CRC nagłówka:** CRC-8 (wielomian `0xD5`).
* **CRC treści:** CRC-16 (wielomian `0x1021`, Big-Endian).
* **Kody funkcji (`FNC`):**
  * `0x40` – żądanie odczytu
  * `0x50` – odpowiedź na odczyt
  * `0x60` – żądanie zapisu
  * `0x70` – potwierdzenie zapisu
* **NACK:** Odpowiedź `0x50` z bajtem `0xC0` bez danych.
* **Limit:** Maksymalnie 226 rejestrów (słów 16-bitowych) na jedno zapytanie.
* **Sposób adresowania:** Krok adresacji wynosi 2 bajty (jeden rejestr = word 16-bit, adresy idą co 2).

### Architektura danych

Panel PG-426 działa jako główny kontroler komunikujący się z modułem MG-426. Nasz interfejs RS485 podłączony jest bezpośrednio do **modułu**. 
* Rejestry od adresu `120` w górę stanowią w większości kopie stanu panelu.
* Nastawy w bloku `1070+` **można zapisywać** — panel automatycznie przejmuje nowe wartości.
* **Zegara nie da się nadpisać** przez RS485 — panel natychmiast go przywraca. Zegar należy ustawiać z poziomu panelu fizycznego.

### Typy danych

| Typ | Odczyt / Przelicznik | Uwagi |
|---|---|---|
| `te10` | `int16` / 10 $\rightarrow$ °C | $-50.0\,^\circ\text{C}$ = czujnik rozwarty („Rozw.”)<br>$130.0\,^\circ\text{C}$ = czujnik zwarty („Zwar.”) |
| `fl10` | `uint16` / 10 | Prąd $[A]$, ciśnienie $[bar]$ |
| `word` | `uint16` | Liczby bez znaku, tryby, maski bitowe |
| `s16` | `int16` | Liczby ze znakiem |

### Dostępne bloki adresów

| Start | Liczba słów | Zawartość |
|---|---|---|
| `120` | 182 | Data, czas, temperatury T1–T15, słowa stanu, przekaźniki wyjściowe |
| `300` | 32 | Obieg chłodniczy (parametry pracy sprężarki, ciśnienia, zawór EV) |
| `408` | 64 | Wartości wyliczone, alarmy, tryb pracy |
| `1070` | 174 | Nastawy i progi temperatur |
| `2120` | 4 | Kody statusowe P/F jednostki zewnętrznej (moduł zgłasza NACK) |

> ℹ️ **Skan adresów (0–2600):** Moduł odpowiada poprawnie głównie w zakresie **120–1355**. Pozostałe bloki zwracają NACK (co może być również spowodowane chwilową zajętością magistrali).

---

## 2. Data i czas (Tylko odczyt) ✅

| Rejestr | Bajt niski | Bajt wysoki |
|---|---|---|
| `120` | Rok ($\% 100$) | Miesiąc |
| `122` | Dzień | Dzień tygodnia ($0$ = poniedziałek) 🟡 |
| `124` | Godzina | Minuta |
| `126` | Sekunda | ❓ |

---

## 3. Czujniki temperatur (`te10`) ✅

| Rejestr | Oznaczenie | Opis / Funkcja |
|---|---|---|
| `130` | **T1** | Zasobnik CWU |
| `132` | **T2** | Bufor CO |
| `134` | T3 | Niepodłączony / Rezerwowy |
| `136` | T4 | Niepodłączony / Rezerwowy |
| `138` | T5 | Niepodłączony / Rezerwowy |
| `140` | T6 | Niepodłączony / Rezerwowy |
| `142` | T7 | Niepodłączony / Rezerwowy |
| `144` | T8 | Niepodłączony / Rezerwowy |
| `146` | T9 | Niepodłączony / Rezerwowy |
| `148` | T10 | Niepodłączony / Rezerwowy |
| `150` | T11 | Niepodłączony / Rezerwowy |
| `152` | **T12** | Temperatura pokojowa obiegu CO1 |
| `154` | T13 | Niepodłączony / Rezerwowy |
| `156` | **T14** | Temperatura zewnętrzna |
| `158` | T15 | Niepodłączony / Rezerwowy |
| `160` | EXT | Czujnik zewnętrzny dodatkowy |

---

## 4. Wyjścia i stan urządzenia

| Rejestr | Znaczenie / Funkcja | Pewność |
|---|---|---|
| `162`–`192` | Słowa stanu wewnętrznego | ❓ Nieustalone |
| `194` | Maska stanu (domyślnie `0xA37F`). **Bit 7 miga co ~15 s przy pracy sprężarki** | 🟡 |
| `196` | **Bit 1 = ON** przy aktywnym grzaniu CWU | 🟡 |
| `198` | Słowo stanu | ❓ Nieustalone |
| **`208`** | **Wyjścia przekaźnikowe sterownika** (szczegóły poniżej) | ✅ / 🟡 |

### Rejestr 208 — Wyjścia przekaźnikowe

| Bit | Maska | Funkcja / Wyjście | Pewność | Metoda weryfikacji |
|---|---|---|---|---|
| 0 | `0x01` | Sprężarka / Pompa ładująca | 🟡 | Aktywne przy grzaniu; gaśnie ~3 min po bicie 3 (dobieg) |
| 2 | `0x04` | Sprężarka / Pompa ładująca | 🟡 | Zachowanie tożsame z bitem 0 |
| 3 | `0x08` | **Grzanie CWU** (Zawór 3-drogowy) | ✅ | ON przy grzaniu wody, wyłącza się po osiągnięciu zadanej T1 |
| 5 | `0x20` | **Pompa obiegowa CO1** (podłogówka) | ✅ | Test: ustawienie zadanej CO1 > T12 wymusza natychmiastowe ON |
| 6 | `0x40` | Taktowanie co 15–30 s podczas pracy CO1 | ❓ | Zawór mieszacza / sterowanie pompą? |
| 7 | `0x80` | Nieaktywny | ❓ | — |

---

## 5. Obieg chłodniczy (Blok 300)

| Rejestr | Nazwa | Opis | Typ | Pewność |
|---|---|---|---|---|
| `300` | **Ta1** | Temperatura powietrza (jednostka zewnętrzna) | `te10` | ✅ |
| `302` | **Tp1** | Temperatura parowania | `te10` | ✅ |
| `304` | **Td1** | Temperatura tłoczenia | `te10` | ✅ |
| `306` | **Ts1** | Temperatura ssania | `te10` | ✅ |
| `316` | **EV1** | Elektoniczny zawór rozprężny (krok) | `word` | ✅ |
| `318` | **I1** | Prąd sprężarki $[A]$ | `fl10` | ✅ |
| `320` | **U1** | Napięcie zasilania $[V]$ | `word` | ✅ |
| `322` | — | Wydajność / Częstotliwość sprężarki (zakres 36–67 przy pracy, 0 przy postoju) | `word` | 🟡 |
| `326` | **HP1** | Wysokie ciśnienie $[bar]$ | `fl10` | ✅ |
| `330` | **LP1** | Niskie ciśnienie $[bar]$ | `fl10` | ✅ |
| `334` | — | Bieg wentylatora? (wartości 64 / 80) | `word` | 🟡 |
| `338` | — | Postój / Blokada czasowa ponownego startu (bit 12: `4096`) | `word` | ❓ |

> 💡 **Wskazówka:** Szacowaną moc pobieraną przez sprężarkę można obliczyć ze wzoru na moc pozorną:
> $$P \approx U1 \cdot I1 \quad [\text{VA}]$$
> (Błąd pomiarowy wynosi ok. $\pm 15\%$).

---

## 6. Wartości wyliczone, tryby i alarmy (Blok 408)

| Rejestr | Parametr | Typ | Pewność |
|---|---|---|---|
| `408` | **CWU: Wartość zadana** (aktualnie wyliczona z programu komfort/eko) | `te10` | ✅ |
| `410` | **CO1 zasilanie zadane** (wyliczone z krzywej grzewczej) | `te10` | ✅ |
| `412` | CO2 zasilanie zadane | `te10` | ✅ |
| `414` | **Błędy i alarmy G** (komunikacja) | `word` | ✅ |
| `418` | **Błędy i alarmy S** (czujniki) | `word` | ✅ |
| `422` | **Błędy i alarmy C / S15** | `word` | ✅ |
| `436` | **Bit 8 (`0x100`) = Obieg CO1 aktywny** | `word` | ✅ |
| `440` | **Tryb pracy:** `1` = Chłodzenie, `2` = CWU, `3` = Grzanie | `word` | ✅ |
| `468` | **CO1 dolny próg zasilania** ($410 - \text{histereza}$) | `te10` | ✅ |
| `470` | **CO1 górny próg zasilania** ($= 410$) | `te10` | ✅ |

### Bity błędów i alarmów

#### Rejestr `414` (Błędy komunikacji - G)
* **Bit 0 (`G01`):** Brak komunikacji panel $\rightarrow$ moduł
* **Bit 1 (`G02`):** Brak komunikacji moduł $\rightarrow$ panel
* **Bit 2 (`G03`):** Brak komunikacji z jednostką zewnętrzną
* **Bit 3 (`G04`):** Błąd transmisji danych

#### Rejestr `418` (Awarie czujników - S)
* **Bity 0–13 (`S01`–`S14`):** Uszkodzenie / brak odczytu z odpowiadającego czujnika $T1$–$T14$.

#### Rejestr `422`
* **Bit 3 (`S15`):** Brak przepływu wody (filtr, zapowietrzenie układu lub słabe ciśnienie).

---

## 7. Nastawy i progi (Blok 1070+)

Możliwy zapis za pomocą komendy `FNC 0x60`. Panel akceptuje wprowadzane zmiany.

| Rejestr | Parameter | Typ | Zakres testowy | Zapis | Pewność |
|---|---|---|---|---|---|
| **`1070`** | **CWU: Temp. komfortowa** | `te10` | 35.0 – 55.0 °C (krok 0.5) | ✅ | ✅ |
| **`1176`** | **CO1: Temp. pokojowa komfort** | `te10` | 15.0 – 26.0 °C (krok 0.1) | ✅ | ✅ |
| **`1178`** | **CO1: Temp. pokojowa eko** | `te10` | 15.0 – 26.0 °C | ✅ | ✅ |
| **`1184`** | **CO1: Histereza pokojowa** | `fl10` | 0.1 – 2.0 °C | ✅ | ✅ |
| **`1192`** | **CO1: Krzywa grzewcza** | `word` | 1 – 9 | ✅ | ✅ |

> 📌 **Praktyczny tip przy sterowaniu z Home Assistant:**
> Zmiana harmonogramów z poziomu sterownika bywa niestabilna. Zdecydowanie bezpieczniejszym podejściem jest **wpisywanie tej samej wartości temperatury do rejestrów komfortu i eko** (np. jednoczesny zapis do `1176` oraz `1178`). Pozwala to pominąć fabryczny harmonogram i oddać pełne sterowanie zewnętrznej automatyce.

---

## 📝 Referencje i podziękowania

* [mvdklip/Domoticz-Hewalex](https://github.com/mvdklip/Domoticz-Hewalex) – wzorce protokołu GECO dla pomp PCWU.
* [aelias-eu/hewalex-geco-protocol](https://github.com/aelias-eu/hewalex-geco-protocol) – opis struktury ramek GECO.
* Instrukcja serwisowa sterowników Hewalex PCCO (kody błędów i alarmów).

---
*Plik utworzony i zweryfikowany na bazie konfiguracji: PG-426 (G-426-P03), MG-426 (MG426-P02-2U3).*
