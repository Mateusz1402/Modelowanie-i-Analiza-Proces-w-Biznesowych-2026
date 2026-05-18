# Milestone 1 - Zrozumienie zbioru danych

## 1. Kontekst i opis zbioru danych

### Środowisko produkcyjne

Analizowany zbiór pochodzi z laboratoryjnego środowiska produkcyjnego **Fischertechnik** (model fabryki modułowej), opisanego w publikacji na platformie Zenodo (https://zenodo.org/records/8087219). Jest to cyber-fizyczny system produkcyjny złożony z kilku stacji roboczych połączonych transporterem. Każda stacja realizuje określoną operację technologiczną na detalu (workpiece).

Stacje produkcyjne w zbiorze:
- **OV_1** – piec (oven) – realizuje aktywność **Burn** (wypalanie/podgrzewanie detalu)
- **MM_1** – frezarka (milling machine) – realizuje aktywność **Mill** (frezowanie)
- **SM_1** – sortownica (sorting machine) – realizuje aktywność **Sort** (sortowanie)
- **VGR_1** – robot chwytakowy (vacuum gripper robot) – realizuje aktywność **Pickup-move-oven** (pobranie i przeniesienie do pieca)
- **HBW_1** – magazyn wysokiego składowania (high-bay warehouse) – realizuje aktywność **Storage** (składowanie)
- **WT_1** – wózek transportowy (workpiece transport) – realizuje aktywność **Transport** (transport)

### Rodzaje czujników i aktuatorów

Każde zdarzenie zawiera wektor sygnałów ze stacji. Sygnały dzielą się na:

- **Czujniki pozycji (i*_pos_switch)** – przełączniki krańcowe, wartości binarne: `0` (nie wyzwolony) lub `1` (wyzwolony)
- **Bariery świetlne (i*_light_barrier)** – czujniki obecności detalu, wartości binarne: `0` (brak detalu) lub `1` (detal wykryty)
- **Silniki (m*_speed)** – prędkość silnika, wartości: `0` (stop), `512` (obrót w przód), `-512` (obrót wstecz)
- **Zawory (o*_valve)** – sterowanie pneumatyczne, wartości: `0` (zamknięty) lub `512` (otwarty)
- **Sprężarki/kompresory (o*_compressor)** – wartości: `0` (wyłączona) lub `512` (włączona)

Wartości `512` i `-512` (zamiast `1` i `-1`) wynikają z reprezentacji sygnałów PLC w skali 10-bitowej.

### Struktura pojedynczego rekordu

Każdy rekord JSON w pliku `Signature_*.txt` ma postać:
```json
{"station": "OV_1", "timestamp": "2023-03-20T10:11:01Z", "events": {"i1_pos_switch": 0, "i2_pos_switch": 0, ...}}
```

Pola:
- `station` – identyfikator stacji (zasób techniczny)
- `timestamp` – znacznik czasu ISO 8601 (UTC), próbkowanie co ~2 sekundy
- `events` – słownik sygnałów aktywnych w danej stacji

---

## 2. Kluczowe atrybuty logu zdarzeń

| Atrybut | Dostępny | Źródło | Uwagi |
|---------|----------|--------|-------|
| `case_id` | Tak (odtworzony) | Nazwa pliku `Signature_*` | Brak jawnego `case_id` w rekordzie JSON |
| `activity` | Tak (odtworzona) | Nazwa pliku `Signature_*` | Brak jawnego pola `activity` w rekordzie |
| `timestamp` | Tak | Pole `timestamp` | Format ISO 8601, UTC, próbkowanie ~2s |
| `resource` | Tak (proxy) | Pole `station` | Brak zasobu ludzkiego; dostępna stacja/maszyna |

**Ograniczenia:** Każdy plik sygnatury reprezentuje **jeden przypadek (case)** jednej aktywności. W rzeczywistym procesie produkcyjnym detal przechodzi przez wiele stacji – pełny log procesu wymagałby połączenia sygnatur z wielu stacji w jeden ślad przypadku.

---

## 3. Podstawowe statystyki zbioru

### Statystyki globalne

| Metryka | Wartość |
|---------|---------|
| Łączna liczba eventów | **119** |
| Liczba przypadków (cases) | **6** |
| Liczba aktywności | **6** |
| Liczba stacji | **6** |
| Łączna liczba kolumn sygnałów | **26** |
| Zakres czasowy | 2023-03-20, 10:11 – 11:08 UTC |

### Liczba eventów i czas trwania per aktywność

| Aktywność | Stacja | Liczba eventów | Czas trwania | Liczba sygnałów |
|-----------|--------|---------------|--------------|-----------------|
| Burn | OV_1 | **13** | ~25 s (10:11:01–10:11:26) | 6 |
| Mill | MM_1 | **12** | ~23 s (10:34:25–10:34:48) | 8 |
| Sort | SM_1 | **12** | ~23 s (10:12:06–10:12:29) | 10 |
| Pickup-move-oven | VGR_1 | **18** | ~34 s (11:08:02–11:08:36) | 10 |
| Storage | HBW_1 | **47** | ~93 s (10:30:28–10:32:01) | 12 |
| Transport | WT_1 | **17** | ~32 s (10:57:07–10:57:39) | 6 |
| **SUMA** | | **119** | | |

**Obserwacje:**
- Aktywność **Storage** dominuje liczbą eventów (47 z 119, tj. **39,5%** wszystkich zdarzeń) i czasem trwania (93 s). Wynika to ze złożoności operacji magazynowania – robot HBW_1 wykonuje wiele ruchów w trzech osiach (m1, m2, m3, m4_speed).
- Aktywności **Burn**, **Mill** i **Sort** mają zbliżoną liczbę eventów (12–13) i czas trwania (~23–25 s).
- **Pickup-move-oven** i **Transport** są pośrednie (17–18 eventów, ~32–34 s).
- Próbkowanie jest regularne – co ~2 sekundy, z nielicznymi wyjątkami (np. Burn: event 10–11 co 3 s).

### Kolejność aktywności w czasie (timeline)

Wszystkie aktywności zarejestrowano tego samego dnia (2023-03-20). Kolejność chronologiczna:

```
10:11  Burn          (OV_1)
10:12  Sort          (SM_1)
10:30  Storage       (HBW_1)
10:34  Mill          (MM_1)
10:57  Transport     (WT_1)
11:08  Pickup-move-oven (VGR_1)
```

Aktywności **nie nakładają się** w czasie – każda sygnatura reprezentuje odrębny, sekwencyjny przypadek.

---

## 4. Analiza jakości danych

### Wyniki kontroli jakości

| Sprawdzenie | Wynik | Interpretacja |
|-------------|-------|---------------|
| Duplikaty wierszy | **0** | Brak zduplikowanych rekordów |
| Duplikaty (case_id, timestamp, station) | **0** | Brak zduplikowanych zdarzeń |
| Błędy parsowania timestamp | **0** | Wszystkie 119 timestampów poprawnie sparsowane |
| Niemonotoniczność timestampów | **0** przypadków | Czasy rosną monotonicznie w każdym case |
| Błędy typów sygnałów | **0** | Wszystkie wartości sygnałów są liczbowe |
| Kolumny z brakującymi wartościami | **26 z 32** | Patrz wyjaśnienie poniżej |

### Brakujące wartości – wyjaśnienie

W połączonej tabeli `df_full` (119 wierszy × 32 kolumny) większość kolumn sygnałów zawiera wartości `NaN`. Jest to **zachowanie oczekiwane i poprawne**, nie błąd danych.

Przyczyna: każda stacja używa tylko swoich czujników i aktuatorów. Np.:
- Stacja **OV_1** (Burn) raportuje tylko: `i1_pos_switch`, `i2_pos_switch`, `i5_light_barrier`, `m1_speed`, `o7_valve`, `o8_compressor` – pozostałe 20 kolumn to `NaN`
- Stacja **HBW_1** (Storage) raportuje: `i1–i4_light_barrier`, `i5–i8_pos_switch`, `m1–m4_speed` – 12 sygnałów

Tabela pokrycia sygnałów per aktywność:

| Aktywność | Sygnały aktywne | Sygnały nieaktywne (NaN) |
|-----------|----------------|--------------------------|
| Burn | 6 | 20 |
| Mill | 8 | 18 |
| Sort | 10 | 16 |
| Pickup-move-oven | 10 | 16 |
| Storage | 12 | 14 |
| Transport | 6 | 20 |

**Wniosek:** Brakujące wartości wynikają z architektury systemu (każda stacja ma własny zestaw czujników), a nie z błędów pomiarowych. Dane są kompletne w zakresie sygnałów właściwych dla każdej stacji.

### Wartości sygnałów – analiza

Sygnały przyjmują wyłącznie wartości ze zbioru `{-512, 0, 512}` (dla silników i zaworów) lub `{0, 1}` (dla czujników binarnych). Brak wartości odstających ani anomalii numerycznych.

Przykładowe wartości dla aktywności **Burn** (OV_1):
- `m1_speed`: przyjmuje wartości `-512` (cofanie), `0` (stop), `512` (do przodu) – silnik napędu pieca
- `o7_valve`: `0` lub `512` – zawór pneumatyczny
- `i1_pos_switch`, `i2_pos_switch`: `0` lub `1` – przełączniki krańcowe pozycji

---

## 5. Eksploracyjna analiza danych (EDA)

### Rozkład eventów per aktywność

```
Storage          ████████████████████████████████████████████████  47 (39.5%)
Pickup-move-oven ██████████████████  18 (15.1%)
Transport        █████████████████  17 (14.3%)
Burn             █████████████  13 (10.9%)
Mill             ████████████  12 (10.1%)
Sort             ████████████  12 (10.1%)
```

### Analiza sygnałów per aktywność

#### Burn (OV_1) – 13 eventów, 6 sygnałów
Sekwencja opisuje cykl pieca: otwarcie drzwi (`m1_speed = -512`), załadunek detalu, zamknięcie (`m1_speed = 512`), oczekiwanie, otwarcie i wyładunek. Zawór `o7_valve = 512` aktywny przez większość cyklu (podtrzymanie pozycji pneumatycznej).

#### Mill (MM_1) – 12 eventów, 9 sygnałów
Frezarka używa sprężarki (`o8_compressor = 512`) do chłodzenia/mocowania. Silniki `m1`, `m2`, `m3` sterują osiami XYZ. Czujniki `i1–i3_pos_switch` i `i4_light_barrier` monitorują pozycję i obecność detalu.

#### Sort (SM_1) – 12 eventów, 10 sygnałów
Sortownica używa taśmociągu (`m1_speed = -512` przez większość cyklu) i zaworów pneumatycznych (`o5_valve`, `o6_valve`) do kierowania detalu. Bariery świetlne `i1`, `i3`, `i6`, `i7`, `i8_light_barrier` śledzą pozycję detalu na taśmie.

#### Pickup-move-oven (VGR_1) – 18 eventów, 10 sygnałów
Robot chwytakowy wykonuje najbardziej złożony ruch: 3 silniki (`m1`, `m2`, `m3_speed`) sterują osiami, `o7_compressor` i `o8_valve` sterują chwytakiem próżniowym. Sekwencja obejmuje: opuszczenie, chwycenie detalu, podniesienie, obrót, przeniesienie do pieca.

#### Storage (HBW_1) – 47 eventów, 12 sygnałów
Magazyn wysokiego składowania jest najdłuższą aktywnością. Używa 4 silników (`m1–m4_speed`) do ruchu w 3 osiach + mechanizmu wyciągania. Czujniki `i1–i4_light_barrier` i `i5–i8_pos_switch` monitorują pozycję w regale. Duża liczba eventów wynika z konieczności precyzyjnego pozycjonowania w 3D.

#### Transport (WT_1) – 17 eventów, 6 sygnałów
Wózek transportowy używa jednego silnika (`m2_speed`) do jazdy i zaworów (`o5_valve`, `o6_valve`) do zatrzymania/blokady. Czujniki `i3_pos_switch`, `i4_pos_switch` wykrywają pozycje krańcowe.

### Porównanie liczby sygnałów vs liczby eventów

Aktywności z większą liczbą sygnałów (Storage: 12, Sort/Pickup: 10) niekoniecznie mają więcej eventów – wyjątkiem jest Storage, gdzie złożoność mechaniczna (ruch 3D) generuje zarówno więcej sygnałów, jak i więcej kroków.

---

## 6. Analiza jakości pod kątem process mining

### Dostępność atrybutów XES

| Atrybut XES | Status | Uwagi |
|-------------|--------|-------|
| `case:concept:name` |  Odtworzony | Z nazwy pliku: `case_Burn_001`, `case_Mill_001`, itd. |
| `concept:name` (activity) |  Odtworzona | Z nazwy pliku: Burn, Mill, Sort, itd. |
| `time:timestamp` |  Dostępny | ISO 8601 UTC, monotoniczny, bez błędów |
| `org:resource` |  Proxy | Pole `station` jako zasób techniczny |

### Ograniczenia dla process mining

1. **Jeden case per aktywność** – każdy plik sygnatury to jeden przypadek jednej aktywności. Brak wielokrotnych instancji tej samej aktywności.
2. **Brak case_id na poziomie rekordu** – `case_id` odtworzony z nazwy pliku, nie z danych.
3. **Brak zasobu ludzkiego** – tylko zasób techniczny (stacja).
4. **Sygnały niskopoziomowe** – dane to surowe sygnały PLC, nie zdarzenia procesowe wysokiego poziomu. Aplikacje CEP (Siddhi) służą do mapowania sygnałów na aktywności.
5. **Brak wariantów** – każda aktywność ma dokładnie jeden przypadek, więc analiza wariantów procesu nie jest możliwa na tym zbiorze.

---

## 7. Podsumowanie Milestone 1

Zbiór zawiera **119 eventów** z **6 aktywności** zarejestrowanych w laboratoryjnym środowisku produkcyjnym Fischertechnik w dniu 2023-03-20. Dane są wysokiej jakości: brak duplikatów, błędów timestampów i anomalii numerycznych. Brakujące wartości są strukturalne (każda stacja raportuje tylko swoje sygnały) i nie stanowią problemu jakościowego.

Kluczowe obserwacje:
- Aktywność **Storage** dominuje (47 eventów, 39,5%), co odzwierciedla złożoność operacji magazynowania 3D
- Próbkowanie regularne (~2 s), sygnały przyjmują wartości `{-512, 0, 512}` lub `{0, 1}`
- Dane gotowe do dalszej analizy: detekcji aktywności (CEP/Siddhi), konwersji do XES i process mining

Plik wykonawczy z kodem i wizualizacjami: `Milestone1_EDA.ipynb`.
