# Milestone 1 - Zrozumienie zbioru danych

## 1. Opis zbioru danych i kontekstu

Analizowany podzbior pochodzi z kolekcji danych o procesach przemyslowych i zawiera sygnatury zdarzen dla 6 aktywnosci:
- Burn (Spalanie, pogrzewanie)
- Mill (Mielenie)
- Pickup-move-oven  (Pobierz i podgrzej)
- Sort  (Sortuj)
- Storage   (Przechowaj)
- Transport (Transportuj)

W projekcie znajduja sie:
- pliki sygnatur (`Signature_*.txt`) z sekwencjami eventow JSON,
- wizualizacje sygnatur (`Visual_Signature_*.png`),
- aplikacje CEP/Siddhi (`Detect*App.siddhi`) mapujace wzorce sygnalowe na aktywnosci.

Po uporzadkowaniu struktury repozytorium:
- dane zrodlowe pozostaja w folderze `Dataset_8087219`,
- artefakty analityczne i CEP sa w folderze `Artefakty_Projektu`.

Kazdy event zawiera timestamp oraz zestaw sygnalow czujnikow i aktuatorow dla stacji (np. `OV_1`, `MM_1`, `SM_1`, `WT_1`, `VGR_1`, `HBW_1`).

## 2. Kluczowe atrybuty logu zdarzen

- `case id`: brak jawnego atrybutu w rekordach; na potrzeby analizy odtworzony z nazwy pliku (`case_<activity>_001`).
- `activity`: brak jawnego pola activity w rekordzie; odtworzona z nazwy pliku sygnatury.
- `timestamp`: obecny (`timestamp`, ISO 8601).
- `resource`: brak klasycznego zasobu osobowego; dostepny zasob techniczny (`station`) jako proxy.

## 3. Analiza jakosci danych

Sprawdzone obszary:
- brakujace wartosci,
- duplikaty,
- poprawne parsowanie i monotonicznosc znacznikow czasu,
- spojnosc typow danych sygnalowych.

Wnioski jakosciowe:
- dane sa regularne i dobrze ustrukturyzowane,
- brak krytycznych problemow z timestampami,
- sygnaly maja glownie dyskretne wartosci (0/1 oraz +/-512),
- ograniczeniem jest brak jawnego `case_id` i `activity` na poziomie pojedynczego eventu.

## 4. Eksploracyjna analiza danych (EDA)

W notebooku wykonano:
- timeline zdarzen wedlug aktywnosci,
- rozklad liczby eventow na aktywnosc,
- histogramy wybranych sygnalow,
- zestawienie czasu trwania przypadkow.

## 5. Podstawowe statystyki

W notebooku wyliczono:
- liczbe eventow,
- liczbe przypadkow (cases),
- liczbe aktywnosci,
- liczbe stacji,
- liczbe eventow na aktywnosc i per case.

## 6. Podstawowe wizualizacje

Dostarczone wizualizacje obejmuja:
- timeline (czas vs aktywnosc),
- frequency bar chart (eventy na aktywnosc),
- distribution (histogramy wartosci sygnalow).

## 7. Podsumowanie Milestone 1

Milestone 1 zostal zrealizowany poprzez:
- przygotowanie kompletnej analizy EDA,
- udokumentowanie jakosci i ograniczen danych,
- wskazanie gotowosci zbioru do kolejnych etapow (detekcja aktywnosci, process mining, conformance checking).

Plik wykonawczy: `Milestone1_EDA.ipynb`.
