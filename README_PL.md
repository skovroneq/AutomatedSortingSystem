# Automated Sorting System — automatyczny system sortowania

**Projekt sterowania PLC i symulacji | OMRON Sysmac Studio | NJ301-1100 | Ladder Diagram + Structured Text**

[English](README.md) | [Polski](README_PL.md)

## Opis projektu

Automated Sorting System to projekt portfolio przedstawiający zaprojektowanie, implementację i symulację przemysłowej sekwencji sortowania produktów z wykorzystaniem sterownika PLC serii OMRON NJ. Układ koordynuje pracę **trzech przenośników i dwóch siłowników pneumatycznych**, kierując produkty do jednego z trzech miejsc docelowych zależnie od ich typu.

Projekt koncentruje się na modułowym programowaniu PLC, sterowaniu sekwencyjnym, blokadach urządzeń, obsłudze awarii oraz powtarzalnych testach w środowisku symulacyjnym. **Projekt przetestowano w symulatorze sterownika Sysmac Studio, bez użycia fizycznej maszyny.**

## Zasada działania

W demonstracyjnej wersji system przetwarza **jeden produkt na cykl**. Po wykryciu produktu zapamiętywany jest jego typ, a sekwencja oczekuje na sygnał czujnika pozycji sortowania.

| Typ produktu | Działanie | Miejsce docelowe |
| --- | --- | --- |
| **1 — A** | Siłownik A wysuwa się, kieruje produkt na boczny tor i cofa się | Przenośnik 2 |
| **2 — B** | Siłownik B wysuwa się, kieruje produkt na boczny tor i cofa się | Przenośnik 3 |
| **3 — C** | Produkt przejeżdża bez użycia siłowników; zakończenie wymaga wykrycia i zwolnienia czujnika wyjściowego | Wyjście przenośnika 1 |

W stanach oczekiwania i transportu program żąda pracy wszystkich trzech przenośników. Podczas przekierowywania produktu pracuje odpowiedni przenośnik docelowy, natomiast produkt C kontynuuje przejazd przenośnikiem 1. Przyjęcie kolejnego produktu wymaga spełnienia warunków ponownego uzbrojenia czujników.

## Technologie i środowisko

- **Oprogramowanie inżynierskie:** OMRON Sysmac Studio 1.64
- **Sterownik w symulacji:** OMRON NJ301-1100
- **Języki PLC:** Ladder Diagram (LD) i Structured Text (ST)
- **Wykonywanie programu:** `PRG_SortingSystem` przypisany do `PrimaryTask`
- **Sprzęt:** testy demonstracyjne nie wymagają fizycznego PLC, napędów, zaworów ani czujników
- **Format projektu:** natywny plik Sysmac Studio `.smc2`

## Architektura programu

Główny program LD `PRG_SortingSystem` łączy sygnały operatorskie i czujnikowe, wyznacza warunki gotowości oraz wspólny sygnał awarii, wywołuje bloki sekwencji i urządzeń oraz obsługuje połączenia symulacyjne.

| Element | Język | Odpowiedzialność |
| --- | --- | --- |
| `PRG_SortingSystem` | LD | Integracja, blokady, wywołania bloków i sygnały symulacyjne |
| `FB_SortingSequence` | ST | Kierowanie produktami, sekwencja sterowania, START/STOP/RESET, status cyklu i systemu |
| `FB_ConveyorMotor` | ST | Sterowanie silnikiem, kontrola sprzężenia zwrotnego i diagnostyka uruchomienia oraz zaniku potwierdzenia pracy |
| `FB_PneumaticCylinder` | ST | Sterowanie zaworami, kontrola czujników położenia, timeouty wysuwania i cofania |
| `FB_CylinderSimulator` | ST | Symulacja pozycji, opóźnienia ruchu 500 ms, blokada mechaniczna i ręczny reset pozycji |
| `ST_MotorSimulation` | Typ danych | Sygnały symulacyjne i diagnostyczne przenośnika |
| `ST_CylinderSimulation` | Typ danych | Sygnały symulacyjne i diagnostyczne siłownika |

Trzy instancje bloku silnika sterują przenośnikami 1–3. Dwie instancje bloku pneumatycznego i dwie instancje symulatora odpowiadają za siłowniki A i B.

## Sekwencja sterowania procesem sortowania

| Stan | Opis |
| --- | --- |
| `0` | IDLE — oczekiwanie na poprawny START i gotowość urządzeń |
| `10` | WAIT_FOR_PRODUCT — praca przenośników i uzbrajanie detekcji po zwolnieniu czujnika |
| `20` | TRANSPORT — oczekiwanie na dotarcie do pozycji sortowania |
| `30` / `31` | EXTEND_A / RETRACT_A — przekierowanie produktu A i powrót siłownika A |
| `40` / `41` | EXTEND_B / RETRACT_B — przekierowanie produktu B i powrót siłownika B |
| `50` / `51` / `52` | SORT_C / WAIT_EXIT_DETECTION / WAIT_EXIT_CLEAR — kontrola przejazdu produktu C przez wyjście |
| `60` | COMPLETE — sygnał zakończenia cyklu i powrót do oczekiwania |
| `900` | FAULT — zatrzaśnięty stan awarii i blokada poleceń ruchu |

Program wykrywa zbocze narastające START, ignoruje START przy aktywnym STOP, RESET, trybie ręcznym lub awarii i na początku każdego cyklu skanowania zeruje polecenia wyjściowe przed ich ustawieniem przez aktywny stan. Ponowne uzbrajanie czujników ogranicza ryzyko wielokrotnego przyjęcia produktu przy stale aktywnym sygnale.

## Diagnostyka i obsługa awarii

- **Wspólna awaria:** `AnyDeviceFault` łączy błędy trzech silników i dwóch siłowników.
- **Gotowość:** `DevicesReady` wymaga braku awarii urządzeń, wsunięcia obu siłowników i nieaktywnego STOP.
- **Diagnostyka napędów:** `FB_ConveyorMotor` zawiera timeout uruchomienia i diagnostykę utraty potwierdzenia pracy; dostępne jest również wejście awarii zewnętrznej.
- **Diagnostyka pneumatyki:** `FB_PneumaticCylinder` kontroluje wysuwanie i cofanie z **limitem czasu ruchu 2 s** oraz wykrywa sprzeczne sygnały czujników położenia.
- **Reakcja na awarię:** aktywna awaria przełącza sekwencję do stanu `900`, blokuje polecenia ruchu i uniemożliwia ponowne uruchomienie przez START.
- **RESET:** po ustąpieniu odpowiednich warunków awarii możliwy jest powrót sekwencji do IDLE; **RESET sam nie uruchamia maszyny**. Poszczególne bloki urządzeń mogą mieć dodatkowe warunki kasowania błędów.
- **Sterowanie symulacją:** `SimulateCylinderAJam` / `SimulateCylinderBJam` blokują zakończenie symulowanego ruchu, a `ResetCylinderAPosition` / `ResetCylinderBPosition` niezależnie przywracają symulowane pozycje.

**Ważne:** jest to logika sterowania procesem i diagnostyki, **a nie zweryfikowana funkcja zatrzymania awaryjnego o wymaganym poziomie bezpieczeństwa**. Wdrożenie na rzeczywistej maszynie wymagałoby odpowiedniej architektury bezpieczeństwa, oceny ryzyka, integracji sprzętowej i uruchomienia.

## Testy symulacyjne

W Sysmac Studio przeprowadzono następujące testy:

| Scenariusz | Wynik |
| --- | --- |
| Poprawne sortowanie produktów A, B i C | Zaliczony |
| STOP podczas oczekiwania i transportu | Zaliczony |
| Blokada START przy aktywnym STOP | Zaliczony |
| Propagacja awarii przenośnika i zatrzaskiwanie stanu awarii | Zaliczony |
| Blokada START przy aktywnej awarii urządzenia | Zaliczony |
| RESET przywraca IDLE bez automatycznego restartu | Zaliczony |
| Timeout wysuwania i cofania siłownika A | Zaliczony |
| Timeout wysuwania i cofania siłownika B | Zaliczony |
| Ręczne przywracanie symulowanych pozycji siłowników | Zaliczony |
| Test regresyjny START/STOP po uporządkowaniu kodu | Zaliczony |

Wyniki dotyczą **ręcznych testów w symulatorze**, a nie automatycznego zestawu testowego ani odbioru sprzętowego.

## Jak otworzyć i uruchomić projekt

1. Zainstaluj zgodną wersję **OMRON Sysmac Studio** (projekt opracowano w wersji **1.64**).
2. Otwórz natywny plik projektu `.smc2` z katalogu `project/` w Sysmac Studio.
3. Sprawdź konfigurację sterownika **NJ301-1100** oraz przypisanie `PRG_SortingSystem` do `PrimaryTask`.
4. Skompiluj projekt sterownika i uruchom symulację w Sysmac Studio.
5. W Watch Tab ustawiaj wejścia `SortingSim` oraz obserwuj `SortingState`, `AnyDeviceFault`, status przenośników i siłowników.
6. Dla normalnego cyklu włącz AUTO, wyłącz STOP i RESET oraz sprawdź gotowość urządzeń. Podaj impuls START, ustaw `ProductType` na `1`, `2` lub `3`, podaj impuls `ProductDetected`, a następnie aktywuj `AtSortingPosition`. Dla produktu C aktywuj i zwolnij `ProductAtExit`, aby zakończyć cykl.
7. Aby przetestować awarię, zasymuluj błąd przenośnika lub blokadę siłownika i obserwuj sygnały diagnostyczne oraz stan `900`. Usuń symulowaną przyczynę awarii przed resetem; w razie potrzeby skorzystaj z osobnych wejść przywracania pozycji symulatora.

## Struktura repozytorium

```text
AutomatedSortingSystem/
├── README.md
├── README_PL.md
├── project/
│   └── AutomatedSortingSystem.smc2
└── screenshots/
    └── (zrzuty ekranu z Sysmac Studio)
```

## Zakres i ograniczenia

- Jeden produkt na cykl; równoczesne śledzenie wielu produktów nie jest obsługiwane.
- Sygnał potwierdzenia pracy silnika jest symulowany bezpośrednio z wyjścia `MotorRun`, dlatego **nie zweryfikowano niezależnie** zachowania przy rzeczywistym zaniku sprzężenia zwrotnego.
- Ruch siłowników odwzorowuje uproszczony model czasowy, a nie fizyczną dynamikę pneumatyki.
- Dla transportu produktu i oczekiwania na czujnik wyjściowy nie zaimplementowano osobnych timeoutów.
- Projekt nie zawiera fizycznych wejść/wyjść, urządzeń fieldbus, HMI ani sprzętowego systemu bezpieczeństwa.

## Możliwe rozszerzenia

Niezależna symulacja awarii sprzężenia zwrotnego silników, diagnostyka timeoutów transportu, wizualizacja HMI, śledzenie wielu produktów i integracja z rzeczywistym sterownikiem oraz modułami I/O.

## Cel projektu

Projekt powstał jako demonstracja kompetencji inżynierskich w zakresie programowania PLC, systematycznej diagnostyki, modułowej architektury sterowania i weryfikacji w środowisku symulacyjnym OMRON Sysmac Studio.
