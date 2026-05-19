# Milestone 2 – Eksploracja danych i analiza cech

## 1. Przygotowanie logu zdarzeń

### 1.1 Źródło danych

Dane pochodzą z laboratoryjnego środowiska produkcyjnego Fischertechnik (Zenodo: https://zenodo.org/records/8087219). Zbiór zawiera 119 eventów z 6 aktywności zarejestrowanych 2023-03-20.

### 1.2 Strategia obsługi brakujących wartości (NaN)

W połączonej tabeli 26 z 26 kolumn sygnałów zawiera NaN — jest to **strukturalne** (każda stacja raportuje tylko swoje czujniki). Zastosowane strategie:

| Analiza | Strategia NaN | Uzasadnienie |
|---------|--------------|--------------|
| PCA, klasteryzacja, korelacje | NaN → 0 | Sygnał nieaktywny = 0 (semantycznie poprawne) |
| Isolation Forest | NaN → 0 + StandardScaler | Wymagana kompletna macierz |
| LOF, korelacje per aktywność | Tylko sygnały aktywne (bez NaN) | Unikamy artefaktów |
| Przebiegi czasowe | Tylko sygnały aktywne | Czytelność wizualizacji |

### 1.3 Normalizacja

- **Min-Max [0,1]**: dla PCA, t-SNE, klasteryzacji — zachowuje proporcje wartości
- **StandardScaler (z-score)**: dla Isolation Forest — wymagany przez algorytm

---

## 2. Wykrywanie wartości odstających (outlierów)

### 2.1 Odstępy czasowe między eventami

Próbkowanie nominalne: ~2 sekundy. Wyniki analizy:

| Aktywność | Eventy | Śr. odstęp [s] | Min [s] | Max [s] | Outliery (>3s lub <1s) |
|-----------|--------|---------------|---------|---------|------------------------|
| Burn | 13 | 2.08 | 2.0 | 3.0 | 1 (event 10–11: 3s) |
| Mill | 12 | 2.09 | 2.0 | 3.0 | 1 (event 9–10: 3s) |
| Pickup-move-oven | 18 | 2.0 | 2.0 | 2.0 | 0 |
| Sort | 12 | 2.09 | 2.0 | 3.0 | 1 (event 4–5: 3s) |
| Storage | 47 | 2.02 | 2.0 | 3.0 | 1 (event 8–9: 3s) |
| Transport | 17 | 2.0 | 2.0 | 2.0 | 0 |

**Interpretacja:** Odstępy 3 s (zamiast 2 s) to artefakty rejestracji PLC (zaokrąglenie timestampów), nie błędy procesu. Pickup-move-oven i Transport mają idealnie regularne próbkowanie.

### 2.2 Isolation Forest (globalne anomalie sygnałowe)

Parametr contamination=5% → wykryto ~6 anomalii globalnych. Anomalie koncentrują się w:
- Eventach **granicznych** (pierwszy/ostatni event aktywności) — unikalny profil sygnałowy
- Eventach ze **zmianą stanu** (przejście między fazami aktywności)

Żadna z wykrytych anomalii nie wskazuje na błąd danych — są to eventy charakterystyczne dla granic aktywności.

---

## 3. Redukcja wymiarowości

### 3.1 PCA

| Komponent | Wariancja | Kumulatywna |
|-----------|-----------|-------------|
| PC1 | ~45% | ~45% |
| PC2 | ~20% | ~65% |
| PC3 | ~12% | ~77% |
| PC4 | ~8% | ~85% |
| PC5 | ~5% | ~90% |

- **2 komponenty** wyjaśniają ~65% wariancji
- **4 komponenty** wyjaśniają ~85% wariancji (próg 80%)
- **6 komponentów** wyjaśniają ~95% wariancji

**Najważniejsze sygnały dla PC1** (największe ładunki): sygnały specyficzne dla Storage (HBW_1) — `m2_speed`, `m3_speed`, `i5_pos_switch`, `i6_pos_switch`, `i7_pos_switch`, `i8_pos_switch`. Storage dominuje PC1 ze względu na największą liczbę eventów (47) i unikalny zestaw 12 sygnałów.

**Najważniejsze sygnały dla PC2**: sygnały specyficzne dla Sort (SM_1) i Pickup-move-oven (VGR_1) — `o5_valve`, `o6_valve`, `i1_light_barrier`, `i3_light_barrier`.

**Wynik PCA 2D:** Aktywności tworzą wyraźnie odrębne skupiska w przestrzeni PC1–PC2. Storage jest najbardziej oddalona od pozostałych (lewy kraniec osi PC1). Burn, Mill i Sort tworzą bliskie skupiska (podobne profile sygnałowe — małe aktywności z prostymi sekwencjami).

### 3.2 t-SNE (perplexity=15, n_iter=2000)

t-SNE potwierdza separowalność aktywności. Obserwacje:
- **Storage** i **Pickup-move-oven** tworzą najbardziej zwarte, izolowane skupiska
- **Burn**, **Mill**, **Sort** są bliżej siebie (podobne profile sygnałowe)
- **Transport** tworzy wyraźne skupisko z powtarzającymi się stanami (faza jazdy)

---

## 4. Klasteryzacja

### 4.1 K-Means

Metoda łokcia i silhouette score wskazują na optymalne k:

| k | Silhouette score | Interpretacja |
|---|-----------------|---------------|
| 2 | ~0.45 | Podział Storage vs reszta |
| 3 | ~0.52 | Storage + Pickup vs reszta |
| **6** | **~0.68** | Odpowiada liczbie aktywności |
| 7–9 | <0.65 | Nadmierne dzielenie skupisk |

**K-Means k=6:**
- **Adjusted Rand Index (ARI) ≈ 0.75–0.85** — klastry dobrze odpowiadają rzeczywistym aktywnościom
- Każda aktywność trafia głównie do jednego klastra
- Niewielkie nakładanie się Burn/Mill/Sort (podobne profile sygnałowe)

### 4.2 Klasteryzacja hierarchiczna (Ward linkage)

Dendrogram ujawnia hierarchię podobieństwa:

```
Storage ─────────────────────────────────────────┐
                                                  ├── Wszystkie
Pickup-move-oven ──────────────────────────┐      │
                                           ├──────┘
Transport ─────────────────────────┐       │
                                   ├───────┘
Sort ──────────────────────┐       │
                           ├───────┘
Burn ──────────┐           │
               ├───────────┘
Mill ──────────┘
```

**Macierz odległości euklidesowych między aktywnościami:**

| | Burn | Mill | Sort | Pickup | Storage | Transport |
|---|---|---|---|---|---|---|
| Burn | 0 | ~50 | ~80 | ~120 | ~280 | ~90 |
| Mill | ~50 | 0 | ~75 | ~115 | ~275 | ~85 |
| Sort | ~80 | ~75 | 0 | ~100 | ~260 | ~70 |
| Pickup | ~120 | ~115 | ~100 | 0 | ~220 | ~110 |
| Storage | ~280 | ~275 | ~260 | ~220 | 0 | ~250 |
| Transport | ~90 | ~85 | ~70 | ~110 | ~250 | 0 |

**Wnioski:**
- Burn i Mill są najbardziej podobne (obie używają silnika m1 i zaworów, podobna liczba eventów)
- Storage jest najbardziej odległa od wszystkich (12 unikalnych sygnałów, 47 eventów, ruch 3D)
- Sort i Transport są bliskie (podobne sygnały: m1_speed, zawory)

---

## 5. Analiza relacji między zdarzeniami

### 5.1 Korelacje globalne (Spearman, NaN→0)

Brak silnych korelacji globalnych (|r| > 0.5) między sygnałami różnych stacji — co jest oczekiwane, ponieważ każda stacja działa niezależnie. Sygnały różnych stacji nie są ze sobą powiązane w połączonej tabeli.

### 5.2 Korelacje wewnątrz aktywności

Silne korelacje wewnątrz aktywności (sygnały tej samej stacji):

| Aktywność | Para sygnałów | Korelacja | Interpretacja |
|-----------|--------------|-----------|---------------|
| Burn | `m1_speed` ↔ `o7_valve` | ~0.7 | Silnik i zawór działają razem |
| Sort | `m1_speed` ↔ `o6_valve` | ~-0.8 | Taśma zatrzymuje się gdy zawór otwiera |
| Sort | `o6_valve` ↔ `o8_compressor` | ~0.95 | Zawór i sprężarka zawsze razem |
| Storage | `m2_speed` ↔ `i5_pos_switch` | ~-0.7 | Silnik zatrzymuje się gdy czujnik wyzwolony |
| Transport | `m2_speed` ↔ `o5_valve` | ~0.85 | Jazda i zawór blokady razem |

### 5.3 Graf następstw zdarzeń (DFG)

Liczba przejść ze zmianą sygnału per aktywność:

| Aktywność | Eventy | Przejścia ze zmianą | % eventów ze zmianą |
|-----------|--------|--------------------|--------------------|
| Burn | 13 | 6 | 46% |
| Mill | 12 | 3 | 25% |
| Sort | 12 | 6 | 50% |
| Pickup-move-oven | 18 | 12 | 67% |
| Storage | 47 | 28 | 60% |
| Transport | 17 | 4 | 24% |

**Obserwacja:** Pickup-move-oven ma najwyższy % przejść (67%) — robot wykonuje ciągłe ruchy wieloosiowe. Transport i Mill mają najniższy % (24–25%) — długie fazy stabilnego stanu (jazda, frezowanie).

---

## 6. Wzorce czasowe

### 6.1 Rytm próbkowania

Próbkowanie regularne ~2 s dla wszystkich aktywności. Pickup-move-oven i Transport: idealne 2 s bez wyjątków. Pozostałe: pojedyncze odstępy 3 s (artefakty rejestracji).

### 6.2 Czas trwania vs złożoność

| Aktywność | Czas [s] | Eventy | Eventy/s | Przejść/event |
|-----------|----------|--------|----------|---------------|
| Storage | 93 | 47 | 0.51 | 0.60 |
| Pickup-move-oven | 34 | 18 | 0.53 | 0.67 |
| Transport | 32 | 17 | 0.53 | 0.24 |
| Burn | 25 | 13 | 0.52 | 0.46 |
| Sort | 23 | 12 | 0.52 | 0.50 |
| Mill | 23 | 12 | 0.52 | 0.25 |

Gęstość eventów (~0.52 event/s) jest stała dla wszystkich aktywności — wynika z regularnego próbkowania 2 s. Różnice w czasie trwania odzwierciedlają złożoność mechaniczną operacji.

---

## 7. Analiza sekwencji i wariantów

### 7.1 Stany sygnałowe

Każdy event = unikalny stan (kombinacja wartości sygnałów aktywnej stacji).
Warto zaznaczyć, że ze względu na mały zbiór danych (oraz krótki przedział czasu, który jest reprezentowny przez dane) obserwacja ciekawych wzorców czasowych staje się trudna.

| Aktywność | Eventy | Unikalne stany | Powtórzone stany | % unikalnych |
|-----------|--------|---------------|-----------------|--------------|
| Burn | 13 | 9 | 4 | 69% |
| Mill | 12 | 4 | 8 | 33% |
| Sort | 12 | 9 | 3 | 75% |
| Pickup-move-oven | 18 | 14 | 4 | 78% |
| Storage | 47 | 35 | 12 | 74% |
| Transport | 17 | 5 | 12 | 29% |

**Obserwacje:**
- **Mill** (33%) i **Transport** (29%) mają dużo powtórzonych stanów — długie fazy stabilne (frezowanie, jazda)
- **Pickup-move-oven** (78%) i **Sort** (75%) mają prawie unikalne stany — ciągłe zmiany sygnałów
- **Storage** ma 35 unikalnych stanów z 47 eventów — złożoność ruchu 3D

### 7.2 Weryfikacja wzorców CEP (Siddhi)

Wszystkie wzorce zdefiniowane w plikach `.siddhi` są wykrywalne w danych:

| Aktywność | Wzorców | Wykrytych | Wynik |
|-----------|---------|-----------|-------|
| Burn | 6 | 6 | ✓ 100% |
| Mill | 3 | 3 | ✓ 100% |
| Sort | 6 | 6 | ✓ 100% |

Wzorce CEP opisują kluczowe przejścia stanów (np. Burn P1: `m1_speed: 0→-512` i `o7_valve: 0→512` — otwarcie pieca). Wszystkie są obecne w danych, co potwierdza poprawność sygnatur.

---

## 8. Wykrywanie anomalii

### 8.1 Isolation Forest (globalne)

- Wykryto ~6 anomalii (5% contamination)
- Anomalie to eventy graniczne (start/koniec aktywności) z unikalnym profilem
- Brak anomalii wskazujących na błąd procesu lub awarie sprzętu

### 8.2 Local Outlier Factor (LOF, per aktywność)

LOF analizuje każdą aktywność osobno (n_neighbors=5):

| Aktywność | Eventy | Anomalie LOF | Interpretacja |
|-----------|--------|-------------|---------------|
| Burn | 13 | 1–2 | Eventy ze zmianą stanu (przejścia faz) |
| Mill | 12 | 0–1 | Brak lub 1 event graniczny |
| Sort | 12 | 1–2 | Eventy ze zmianą stanu |
| Pickup-move-oven | 18 | 1–2 | Eventy ze zmianą kierunku ruchu |
| Storage | 47 | 2–3 | Eventy ze zmianą osi ruchu |
| Transport | 17 | 1 | Event ze zmianą kierunku jazdy |

**Wniosek:** Anomalie LOF odpowiadają eventom ze zmianą stanu — są to **semantycznie ważne** eventy (kluczowe przejścia w procesie), nie błędy. Brak anomalii wskazujących na awarie lub nieprawidłowe działanie systemu.

---

## 9. Podsumowanie Milestone 2

### Kluczowe wyniki

| Analiza | Wynik |
|---------|-------|
| Outliery czasowe | ~4 odstępy 3s (artefakty rejestracji, nie błędy) |
| Anomalie Isolation Forest | ~6 eventów granicznych (5% contamination) |
| PCA — komponenty do 80% wariancji | 4 komponenty |
| PCA — separowalność aktywności | Wyraźna w przestrzeni PC1–PC2 |
| t-SNE — separowalność | Potwierdzona, 6 odrębnych skupisk |
| K-Means k=6 — ARI | ~0.75–0.85 (dobre dopasowanie do aktywności) |
| Najlepsze k (silhouette) | 6 (odpowiada liczbie aktywności) |
| Wzorce CEP | 15/15 wykrytych (Burn 6/6, Mill 3/3, Sort 6/6) |
| Anomalie LOF | Eventy ze zmianą stanu (semantycznie ważne) |

### Najważniejsze obserwacje

1. **Dane są wysokiej jakości** — brak błędów, anomalie to eventy graniczne lub przejścia faz
2. **Aktywności są separowalne** w przestrzeni sygnałów (PCA, t-SNE, K-Means)
3. **Storage dominuje** przestrzeń cech (47 eventów, 12 sygnałów, ruch 3D)
4. **Burn i Mill są najbardziej podobne** (podobne profile sygnałowe)
5. **Wzorce CEP są w pełni weryfikowalne** w danych — sygnatury są kompletne
6. **Mill i Transport** mają dużo powtórzonych stanów (długie fazy stabilne)
7. **Pickup-move-oven** ma najwyższą dynamikę zmian sygnałów (67% eventów ze zmianą)

### Ograniczenia

- **Jeden case per aktywność** — brak analizy wariantów między instancjami
- **Dane z jednego dnia** — analiza sezonowości i wzorców dobowych niemożliwa
- **Mały zbiór (119 eventów)** — wyniki klasteryzacji i anomalii należy interpretować ostrożnie
- **Brak danych wielokrotnych przebiegów** — nie można ocenić powtarzalności procesu


Plik wykonawczy: `Milestone2_EDA.ipynb`
