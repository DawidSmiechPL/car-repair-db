**Temat:** System bazodanowy dla warsztatu samochodowego

**Autorzy:** Dawid Śmiech, Krystian Jasiura

**Grupa:** 1K232

**Zespół:** numer 3

# 1. Zakres i krótki opis systemu

Projekt dotyczy zaprojektowania bazy danych dla niewielkiego warsztatu samochodowego. Pomysł wziął się stąd, że w wielu mniejszych warsztatach nadal wszystko prowadzi się w zeszycie albo w Excelu, co prędzej czy później kończy się gubieniem informacji – nie wiadomo, jaki pojazd był naprawiany, kto co robił, ile zostało części w magazynie.

Tematykę warsztatu samochodowego wybraliśmy z powodu „osobistego powiązania" z tą tematyką, oboje mieliśmy styczność z lokalnymi małymi warsztatami, naprawami przez znajomych oraz obsługę przez ASO standardowych napraw, a w tym realizacji szkód z OC sprawcy. Z uwagi na fakt, że projekt był nam najbliższy, podjęliśmy decyzję o jego realizacji.

Zakładamy, że z systemu będą korzystać trzy grupy osób:

- **recepcjonista** – przyjmuje pojazdy do naprawy, zakłada zlecenia, na koniec wystawia faktury,
- **mechanik** – realizuje naprawy, dopisuje do zlecenia wykonane czynności i zużyte części,
- **kierownik warsztatu** – nadzoruje pracę i ma wgląd we wszystko.

W skrócie cały proces wygląda tak: klient przyjeżdża do warsztatu z usterką (standardowy warsztat, bez napraw z OC/AC), recepcjonista zakłada zlecenie i przypisuje do niego mechanika. Mechanik w trakcie naprawy dodaje do zlecenia kolejne wykonane usługi (z cennika) i części pobrane z magazynu. Po zakończeniu naprawy system pozwala wystawić fakturę.

Baza powinna pamiętać też dane historyczne – chcemy móc sprawdzić, jakie naprawy były robione na danym pojeździe wcześniej (np. czy klocki hamulcowe były niedawno wymieniane).


# 2. Wymagania i funkcje systemu

## 2.1. Wymagania funkcjonalne

System powinien umożliwiać:

1. Rejestrację klienta [imię, nazwisko, telefon, ewentualnie e-mail (raczej mało wykorzystywany w warsztatach) i adres].
2. Dodawanie pojazdów i powiązanie ich z konkretnym klientem (jeden klient może mieć kilka samochodów).
3. Tworzenie zleceń naprawy dla wybranego pojazdu, z opisem zgłoszonej usterki.
4. Przypisanie do zlecenia mechanika prowadzącego.
5. Śledzenie statusu zlecenia – przyjęte, w realizacji, zakończone, anulowane.
6. Dopisywanie do zlecenia wykonanych usług z cennika (z możliwością podania ilości).
7. Dopisywanie do zlecenia zużytych części zamiennych.
8. Aktualizację stanu magazynowego po pobraniu części.
9. Wystawienie faktury dla zakończonego zlecenia.
10. Przeglądanie historii napraw konkretnego pojazdu.
11. Przeglądanie zleceń przypisanych do konkretnego mechanika (np. żeby sprawdzić jego obłożenie).
12. Generowanie raportów przychodów.
13. Wyciągać informację o zrealizowanych naprawach danego dnia i przesyłać SMS do klientów o zakończeniu serwisu i zaproszenie po odbiór samochodu (dane te byłyby przesyłane do zewnętrznego systemu, np. SMSAPI).
14. Raport historii pojazdu.
15. Raport historii klienta.

## 2.2. Wymagania niefunkcjonalne

- Dane muszą być spójne – relacje między tabelami pilnowane przez klucze obce.
- Numery rejestracyjne, VIN-y i numery faktur muszą być unikalne.
- Daty zakończenia zlecenia nie mogą być wcześniejsze niż daty przyjęcia.
- Wsparcie polskich znaków (UTF-8).

## 2.3. Historyjki użytkownika

Spróbowaliśmy opisać kilka typowych sytuacji z perspektywy użytkowników:

- *Jako recepcjonista chciałbym szybko przyjąć pojazd do naprawy, podając tylko numer rejestracyjny i opis usterki – żeby klient nie musiał stać długo przy ladzie.*
- *Jako mechanik chcę móc dopisywać do zlecenia kolejne wykonane usługi i części w trakcie naprawy, żeby na koniec faktura wyliczyła się sama.*
- *Jako mechanik chcę sprawdzić, co było już robione na tym pojeździe wcześniej – czasem klient nie pamięta, kiedy ostatnio wymieniał olej.*
- *Jako kierownik chcę widzieć, ilu mechaników aktualnie ma otwarte zlecenia, żeby wiedzieć, kogo można obciążyć kolejną naprawą.*
- *Jako recepcjonista chcę wystawić fakturę jednym kliknięciem po zakończeniu naprawy.*


# 3. Projekt bazy danych

## 3.1. Schemat bazy danych

Baza składa się z 9 tabel. Centralna jest tabela `zlecenia` – to do niej „podpinają się" wszystkie pozostałe encje (pojazd, mechanik, usługi, części, faktura). Ponieważ jedno zlecenie może obejmować wiele usług i wiele części, te dwie relacje musieliśmy rozłożyć na tabele asocjacyjne (`zlecenia_uslugi` oraz `zlecenia_czesci`). W tych tabelach przechowujemy też ilość i cenę jednostkową – cenę kopiujemy w momencie wpisu, żeby późniejsza zmiana w cenniku nie wpłynęła wstecznie na stare zlecenia.

Diagram został wykonany w narzędziu **dbdiagram.io**. Link do edytowalnej wersji:
[https://dbdiagram.io/d/6a19c263f15b4b045230a0bf](https://dbdiagram.io/d/6a19c263f15b4b045230a0bf)

![Diagram ER bazy danych warsztatu](diagram_dbdiagram.png)

### Legenda oznaczeń na diagramie

W dbdiagram.io przy nazwach pól pojawiają się ikonki – oto co oznaczają (w pliku markdown w zależności od edytora może nie być wykryta czcionka zawierająca emoji ułatwiających poniższą legendę):

| Oznaczenie       | Znaczenie                                                                    |
|------------------|------------------------------------------------------------------------------|
| 🔑 (kluczyk)     | **PK – klucz główny** (Primary Key). Pole jednoznacznie identyfikujące rekord w tabeli, np. `id_klienta`. |
| 🔗 (łańcuszek)   | **FK – klucz obcy** (Foreign Key). Pole, które odwołuje się do klucza głównego w innej tabeli, np. `id_klienta` w tabeli `pojazdy` wskazuje na `klienci.id_klienta`. To te ikonki tworzą widoczne na diagramie linie łączące tabele. |
| 👆 (odcisk palca) | **UQ – ograniczenie unikalności** (Unique). Pole, którego wartość nie może się powtórzyć w obrębie tabeli, np. `nr_rejestracyjny`, `vin`, `numer_faktury`. |
| 📝 (notesik)     | **Note – notatka opisowa**. W kodzie dbdiagram można dodać komentarz do pola (np. dozwolone wartości statusu). Notesik pojawia się przy polach takich jak `status` czy `status_platnosci`, gdzie opisuję możliwe wartości (`przyjete / w_realizacji / zakonczone / anulowane`). |
| `NN`             | **NOT NULL** – pole jest wymagane, nie może być puste. |

## 3.2. Opis poszczególnych tabel

### Tabela: `klienci`

Słownik klientów warsztatu. Jeden klient może mieć wiele pojazdów (relacja 1:N z tabelą `pojazdy`).

| Atrybut       | Typ                  | Uwagi                                       |
|---------------|----------------------|---------------------------------------------|
| id_klienta    | INT                  | PK, auto_increment                          |
| imie          | VARCHAR(50)          | NOT NULL                                    |
| nazwisko      | VARCHAR(50)          |                                             |
| telefon       | VARCHAR(20)          |                                             |
| email         | VARCHAR(100)         | opcjonalne                                  |
| adres         | VARCHAR(200)         |                                             |

### Tabela: `pojazdy`

Pojazdy zarejestrowane w systemie. Każdy pojazd należy do dokładnie jednego klienta.

| Atrybut          | Typ                   | Uwagi                                                  |
|------------------|-----------------------|--------------------------------------------------------|
| id_pojazdu       | INT                   | PK, auto_increment                                     |
| id_klienta       | INT                   | FK → klienci(id_klienta), NOT NULL                     |
| marka            | VARCHAR(50)           |                                                        |
| model            | VARCHAR(50)           |                                                        |
| rok_produkcji    | INT                   |                                                        |
| nr_rejestracyjny | VARCHAR(15)           | UNIQUE, NOT NULL                                       |
| vin              | VARCHAR(17)           | UNIQUE                                                 |
| przebieg         | INT                   | w km, aktualizowany przy każdej wizycie                |

### Tabela: `pracownicy`

Pracownicy warsztatu – nie tylko mechanicy, ale też recepcjoniści i kierownik. Pole `stanowisko` decyduje, kim dana osoba jest. Mechanicy są przypisywani do zleceń.

| Atrybut            | Typ                  | Uwagi                                            |
|--------------------|----------------------|--------------------------------------------------|
| id_pracownika      | INT                  | PK, auto_increment                               |
| imie               | VARCHAR(50)          | NOT NULL                                         |
| nazwisko           | VARCHAR(50)          | NOT NULL                                         |
| stanowisko         | VARCHAR(50)          | np. mechanik, recepcjonista, kierownik           |
| telefon            | VARCHAR(20)          |                                                  |
| email              | VARCHAR(100)         |                                                  |
| stawka_godzinowa   | DECIMAL(8,2)         | w PLN                                            |
| data_zatrudnienia  | DATE                 |                                                  |

### Tabela: `zlecenia`

Centralna tabela całej bazy. Każde zlecenie dotyczy jednego pojazdu i ma jednego mechanika prowadzącego.

| Atrybut          | Typ                  | Uwagi                                                                |
|------------------|----------------------|----------------------------------------------------------------------|
| id_zlecenia      | INT                  | PK, auto_increment                                                   |
| id_pojazdu       | INT                  | FK → pojazdy(id_pojazdu), NOT NULL                                   |
| id_mechanika     | INT                  | FK → pracownicy(id_pracownika), NOT NULL                             |
| data_przyjecia   | DATETIME             | NOT NULL                                                             |
| data_zakonczenia | DATETIME             | NULL dopóki zlecenie nie jest zakończone                             |
| status           | VARCHAR(20)          | przyjete / w_realizacji / zakonczone / anulowane                     |
| opis_usterki     | TEXT                 | to, co klient zgłosił przy przyjęciu                                 |

### Tabela: `uslugi`

Cennik usług – słownik, modyfikowany rzadko. Trzymamy tu normatywny czas pracy i cenę robocizny.

| Atrybut         | Typ                  | Uwagi                                |
|-----------------|----------------------|--------------------------------------|
| id_uslugi       | INT                  | PK, auto_increment                   |
| nazwa           | VARCHAR(100)         | NOT NULL                             |
| opis            | TEXT                 | opcjonalny, szczegółowy              |
| czas_pracy_h    | DECIMAL(5,2)         | normatywny czas w godzinach          |
| cena_robocizny  | DECIMAL(10,2)        | w PLN                                |

### Tabela: `zlecenia_uslugi`

Tabela łącząca (relacja N:M między zleceniami a usługami). Zapisujemy też ilość i cenę z momentu wykonania.

| Atrybut          | Typ                  | Uwagi                                       |
|------------------|----------------------|---------------------------------------------|
| id               | INT                  | PK, auto_increment                          |
| id_zlecenia      | INT                  | FK → zlecenia(id_zlecenia), NOT NULL        |
| id_uslugi        | INT                  | FK → uslugi(id_uslugi), NOT NULL            |
| ilosc            | INT                  | ile razy wykonano                           |
| cena_jednostkowa | DECIMAL(10,2)        | kopia z cennika z momentu wpisu             |

### Tabela: `czesci`

Magazyn części zamiennych.

| Atrybut          | Typ                  | Uwagi                                  |
|------------------|----------------------|----------------------------------------|
| id_czesci        | INT                  | PK, auto_increment                     |
| nazwa            | VARCHAR(100)         |                                        |
| numer_katalogowy | VARCHAR(50)          | UNIQUE                                 |
| producent        | VARCHAR(50)          |                                        |
| cena_sprzedazy   | DECIMAL(10,2)        | cena dla klienta                       |
| stan_magazynowy  | INT                  | aktualna liczba sztuk na stanie        |

### Tabela: `zlecenia_czesci`

Analogicznie do `zlecenia_uslugi`, tylko dla części – też N:M.

| Atrybut          | Typ                  | Uwagi                                       |
|------------------|----------------------|---------------------------------------------|
| id               | INT                  | PK, auto_increment                          |
| id_zlecenia      | INT                  | FK → zlecenia(id_zlecenia), NOT NULL        |
| id_czesci        | INT                  | FK → czesci(id_czesci), NOT NULL            |
| ilosc            | INT                  | liczba użytych sztuk                        |
| cena_jednostkowa | DECIMAL(10,2)        | kopia z magazynu z momentu wpisu            |

### Tabela: `faktury`

Faktura jest powiązana ze zleceniem 1:1 (jedno zlecenie → maksymalnie jedna faktura), stąd UNIQUE na `id_zlecenia`.

| Atrybut          | Typ                  | Uwagi                                                |
|------------------|----------------------|------------------------------------------------------|
| id_faktury       | INT                  | PK, auto_increment                                   |
| id_zlecenia      | INT                  | FK → zlecenia(id_zlecenia), UNIQUE, NOT NULL         |
| numer_faktury    | VARCHAR(30)          | UNIQUE, NOT NULL, np. „FV/2024/001"                  |
| data_wystawienia | DATE                 |                                                      |
| kwota_netto      | DECIMAL(10,2)        | w PLN                                                |
| kwota_brutto     | DECIMAL(10,2)        | w PLN                                                |
| status_platnosci | VARCHAR(20)          | nieoplacona / oplacona / czesciowo_oplacona          |

---

W kolejnym etapie planujemy zaimplementować widoki dla najczęściej wykonywanych zapytań oraz triggery do automatycznej aktualizacji stanu magazynowego po dodaniu części do zlecenia. Jeżeli czas pozwoli i będzie na to przestrzeń, będziemy chcieli jeszcze rozwinąć strukturę bazy danych o np. uzupełnianie godzin pracowników i wyliczanie miesięcznej wypłaty.
