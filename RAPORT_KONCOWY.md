# System bazodanowy dla warsztatu samochodowego

Projekt relacyjnej bazy danych dla niewielkiego warsztatu samochodowego, przygotowany w T-SQL (Microsoft SQL Server).

---

# 1. Wymagania i opis systemu

## 1.1 Opis systemu

Projekt dotyczy zaprojektowania i implementacji relacyjnej bazy danych dla niewielkiego warsztatu samochodowego. W wielu mniejszych warsztatach ewidencja prowadzona jest w zeszycie lub w arkuszu kalkulacyjnym, co prowadzi do utraty danych, błędów w rozliczeniach i trudności z analizą historii napraw.

Grupy użytkowników systemu:

- **Recepcjonista** – przyjmuje pojazdy, zakłada zlecenia, wystawia faktury.
- **Mechanik** – realizuje naprawy, dopisuje usługi i części do zlecenia.
- **Kierownik warsztatu** – nadzoruje pracę, ma wgląd we wszystkie dane.

## 1.2 Wymagania funkcjonalne

| ID | Opis |
|----|------|
| RF-01 | Rejestracja klienta (imię, nazwisko, telefon, e-mail, adres). |
| RF-02 | Rejestracja pojazdu i powiązanie z klientem (1 klient – wiele pojazdów). |
| RF-03 | Tworzenie zlecenia naprawy z opisem usterki. |
| RF-04 | Przypisanie mechanika prowadzącego do zlecenia. |
| RF-05 | Śledzenie statusu zlecenia: przyjęte / w realizacji / zakończone / anulowane. |
| RF-06 | Dopisywanie wykonanych usług z cennika (z ilością). |
| RF-07 | Dopisywanie zużytych części z magazynu (z ilością). |
| RF-08 | Automatyczna aktualizacja stanu magazynowego po pobraniu części. |
| RF-09 | Wystawienie faktury dla zakończonego zlecenia. |
| RF-10 | Przeglądanie historii napraw pojazdu. |
| RF-11 | Przeglądanie zleceń przypisanych do mechanika (obciążenie). |
| RF-12 | Generowanie raportu przychodów. |
| RF-13 | Zestawienie zakończonych napraw danego dnia – dane dla systemu SMS. |
| RF-14 | Raport historii pojazdu. |
| RF-15 | Raport historii klienta. |

## 1.3 Wymagania niefunkcjonalne

- Integralność danych – relacje pilnowane przez klucze obce (FOREIGN KEY).
- Unikalność – numery rejestracyjne, VIN-y oraz numery faktur są unikalne (UNIQUE).
- Spójność dat – data zakończenia zlecenia nie może być wcześniejsza niż data przyjęcia (CHECK).
- Wsparcie polskich znaków – typy NVARCHAR.

## 1.4 Historyjki użytkownika

| Rola | Historyjka |
|------|------------|
| Recepcjonista | Chcę szybko przyjąć pojazd podając tylko numer rejestracyjny i opis usterki. |
| Mechanik | Chcę dopisywać do zlecenia usługi i części w trakcie naprawy, żeby na koniec faktura wyliczyła się automatycznie. |
| Mechanik | Chcę sprawdzić historię pojazdu, bo klient nie pamięta, kiedy ostatnio wymieniał olej. |
| Kierownik | Chcę widzieć obciążenie mechaników, aby wiedzieć, kogo można przypisać do nowego zlecenia. |
| Recepcjonista | Chcę wystawić fakturę jednym kliknięciem po zakończeniu naprawy. |

---

# 2. Diagram bazy danych (ERD)

Baza składa się z **9 tabel**. Tabelą centralną jest **zlecenia** – łączą się z nią wszystkie pozostałe encje. Relacje N:M (zlecenia–usługi oraz zlecenia–części) zostały rozłożone na tabele asocjacyjne **zlecenia_uslugi** i **zlecenia_czesci**.

Diagram wykonano w narzędziu dbdiagram.io. Edytowalna wersja online: https://dbdiagram.io/d/6a19c263f15b4b045230a0bf

> Diagram ERD: [otwórz edytowalny model w dbdiagram.io](https://dbdiagram.io/d/6a19c263f15b4b045230a0bf). Plik obrazu nie jest dołączony do tego repozytorium.

| Relacja | Kardynalność | Opis |
|---|---|---|
| klienci – pojazdy | 1 : N | Klient może mieć wiele pojazdów. |
| pojazdy – zlecenia | 1 : N | Pojazd może mieć wiele zleceń. |
| pracownicy – zlecenia | 1 : N | Mechanik może prowadzić wiele zleceń. |
| zlecenia – zlecenia_uslugi | 1 : N | Zlecenie zawiera wiele usług. |
| uslugi – zlecenia_uslugi | 1 : N | Usługa może wystąpić w wielu zleceniach. |
| zlecenia – zlecenia_czesci | 1 : N | Zlecenie zawiera wiele części. |
| czesci – zlecenia_czesci | 1 : N | Część może wystąpić w wielu zleceniach. |
| zlecenia – faktury | 1 : 1 | Jedno zlecenie – co najwyżej jedna faktura. |

---

# 3. Kod DDL tabel i opis struktury

## 3.1 Tabela: klienci

Słownik klientów warsztatu. Jeden klient może mieć wiele pojazdów (relacja 1:N z tabelą pojazdy).

| Atrybut | Typ | Ograniczenia | Opis |
|---|---|---|---|
| id_klienta | INT | PK, IDENTITY | Klucz główny. |
| imie | NVARCHAR(50) | NOT NULL | Imię klienta. |
| nazwisko | NVARCHAR(50) | – | Nazwisko klienta. |
| telefon | NVARCHAR(20) | – | Numer kontaktowy. |
| email | NVARCHAR(100) | – | Adres e-mail (opcjonalny). |
| adres | NVARCHAR(200) | – | Adres zamieszkania / do faktury. |

```sql
CREATE TABLE klienci (
    id_klienta INT IDENTITY(1,1) PRIMARY KEY,
    imie NVARCHAR(50) NOT NULL,
    nazwisko NVARCHAR(50),
    telefon NVARCHAR(20),
    email NVARCHAR(100),
    adres NVARCHAR(200)
)
```

## 3.2 Tabela: pojazdy

Pojazdy zarejestrowane w systemie. Każdy pojazd należy do jednego klienta.

| Atrybut | Typ | Ograniczenia | Opis |
|---|---|---|---|
| id_pojazdu | INT | PK, IDENTITY | Klucz główny. |
| id_klienta | INT | FK, NOT NULL | Referencja do klienci(id_klienta). |
| marka | NVARCHAR(50) | – | Marka pojazdu. |
| model | NVARCHAR(50) | – | Model pojazdu. |
| rok_produkcji | INT | – | Rok produkcji pojazdu. |
| nr_rejestracyjny | NVARCHAR(15) | UNIQUE, NOT NULL | Numer rejestracyjny. |
| vin | NVARCHAR(17) | UNIQUE | Numer VIN. |
| przebieg | INT | – | Aktualny przebieg w km. |

```sql
CREATE TABLE pojazdy (
    id_pojazdu INT IDENTITY(1,1) PRIMARY KEY,
    id_klienta INT NOT NULL,
    marka NVARCHAR(50),
    model NVARCHAR(50),
    rok_produkcji INT,
    nr_rejestracyjny NVARCHAR(15) NOT NULL UNIQUE,
    vin NVARCHAR(17) UNIQUE,
    przebieg INT,
    CONSTRAINT FK_pojazdy_klienci FOREIGN KEY (id_klienta)
        REFERENCES klienci(id_klienta)
)
```

## 3.3 Tabela: pracownicy

Wszyscy pracownicy warsztatu: mechanicy, recepcjoniści, kierownik. Pole stanowisko decyduje o roli.

| Atrybut | Typ | Ograniczenia | Opis |
|---|---|---|---|
| id_pracownika | INT | PK, IDENTITY | Klucz główny. |
| imie | NVARCHAR(50) | NOT NULL | Imię pracownika. |
| nazwisko | NVARCHAR(50) | NOT NULL | Nazwisko pracownika. |
| stanowisko | NVARCHAR(50) | – | mechanik / recepcjonista / kierownik. |
| telefon | NVARCHAR(20) | – | Numer służbowy. |
| email | NVARCHAR(100) | – | Służbowy e-mail. |
| stawka_godzinowa | DECIMAL(8,2) | – | Stawka w PLN/h. |
| data_zatrudnienia | DATE | – | Data rozpoczęcia pracy. |

```sql
CREATE TABLE pracownicy (
    id_pracownika INT IDENTITY(1,1) PRIMARY KEY,
    imie NVARCHAR(50) NOT NULL,
    nazwisko NVARCHAR(50) NOT NULL,
    stanowisko NVARCHAR(50),
    telefon NVARCHAR(20),
    email NVARCHAR(100),
    stawka_godzinowa DECIMAL(8,2),
    data_zatrudnienia DATE
)
```

## 3.4 Tabela: uslugi

Cennik usług warsztatowych. Cena robocizny jest kopiowana do zlecenia_uslugi w momencie dodania, dzięki czemu zmiana cennika nie wpływa wstecznie na stare zlecenia.

| Atrybut | Typ | Ograniczenia | Opis |
|---|---|---|---|
| id_uslugi | INT | PK, IDENTITY | Klucz główny. |
| nazwa | NVARCHAR(100) | NOT NULL | Nazwa usługi. |
| opis | NVARCHAR(MAX) | – | Szczegółowy opis usługi. |
| czas_pracy_h | DECIMAL(5,2) | – | Normatywny czas pracy w godzinach. |
| cena_robocizny | DECIMAL(10,2) | – | Cena robocizny w PLN. |

```sql
CREATE TABLE uslugi (
    id_uslugi INT IDENTITY(1,1) PRIMARY KEY,
    nazwa NVARCHAR(100) NOT NULL,
    opis NVARCHAR(MAX),
    czas_pracy_h DECIMAL(5,2),
    cena_robocizny DECIMAL(10,2)
)
```

## 3.5 Tabela: czesci

Magazyn części zamiennych. Stan magazynowy jest aktualizowany przez trigger trg_update_magazyn przy każdym pobraniu części do zlecenia.

| Atrybut | Typ | Ograniczenia | Opis |
|---|---|---|---|
| id_czesci | INT | PK, IDENTITY | Klucz główny. |
| nazwa | NVARCHAR(100) | – | Nazwa handlowa części. |
| numer_katalogowy | NVARCHAR(50) | UNIQUE | Unikalny numer katalogowy. |
| producent | NVARCHAR(50) | – | Nazwa producenta. |
| cena_sprzedazy | DECIMAL(10,2) | – | Cena sprzedaży dla klienta. |
| stan_magazynowy | INT | DEFAULT 0 | Aktualna ilość sztuk na stanie. |

```sql
CREATE TABLE czesci (
    id_czesci INT IDENTITY(1,1) PRIMARY KEY,
    nazwa NVARCHAR(100),
    numer_katalogowy NVARCHAR(50) UNIQUE,
    producent NVARCHAR(50),
    cena_sprzedazy DECIMAL(10,2),
    stan_magazynowy INT DEFAULT 0
)
```

## 3.6 Tabela: zlecenia

Centralna tabela systemu. Każde zlecenie dotyczy jednego pojazdu i ma jednego mechanika prowadzącego. Ograniczenia CHECK pilnują poprawności statusu i dat.

| Atrybut | Typ | Ograniczenia | Opis |
|---|---|---|---|
| id_zlecenia | INT | PK, IDENTITY | Klucz główny. |
| id_pojazdu | INT | FK, NOT NULL | Referencja do pojazdy(id_pojazdu). |
| id_mechanika | INT | FK, NOT NULL | Referencja do pracownicy(id_pracownika). |
| data_przyjecia | DATETIME | NOT NULL | Data i godzina przyjęcia pojazdu. |
| data_zakonczenia | DATETIME | – | Data zakończenia – NULL gdy trwa. |
| status | NVARCHAR(20) | CHECK | przyjete / w_realizacji / zakonczone / anulowane. |
| opis_usterki | NVARCHAR(MAX) | – | Opis usterki zgłoszonej przez klienta. |

```sql
CREATE TABLE zlecenia (
    id_zlecenia INT IDENTITY(1,1) PRIMARY KEY,
    id_pojazdu INT NOT NULL,
    id_mechanika INT NOT NULL,
    data_przyjecia DATETIME NOT NULL DEFAULT GETDATE(),
    data_zakonczenia DATETIME,
    status NVARCHAR(20) NOT NULL DEFAULT 'przyjete',
    opis_usterki NVARCHAR(MAX),
    CONSTRAINT FK_zlecenia_pojazdy FOREIGN KEY (id_pojazdu)
        REFERENCES pojazdy(id_pojazdu),
    CONSTRAINT FK_zlecenia_pracownicy FOREIGN KEY (id_mechanika)
        REFERENCES pracownicy(id_pracownika),
    CONSTRAINT CK_zlecenia_status CHECK (
        status IN ('przyjete','w_realizacji','zakonczone','anulowane')
    ),
    CONSTRAINT CK_zlecenia_daty CHECK (
        data_zakonczenia IS NULL OR data_zakonczenia >= data_przyjecia
    )
)
```

## 3.7 Tabela: zlecenia_uslugi

Tabela asocjacyjna łącząca zlecenia z usługami (relacja N:M). Przechowuje ilość i cenę z momentu wykonania.

| Atrybut | Typ | Ograniczenia | Opis |
|---|---|---|---|
| id | INT | PK, IDENTITY | Klucz główny wiersza. |
| id_zlecenia | INT | FK, NOT NULL | Referencja do zlecenia. |
| id_uslugi | INT | FK, NOT NULL | Referencja do usługi. |
| ilosc | INT | DEFAULT 1 | Ile razy wykonano usługę. |
| cena_jednostkowa | DECIMAL(10,2) | NOT NULL | Kopia ceny z cennika. |

```sql
CREATE TABLE zlecenia_uslugi (
    id INT IDENTITY(1,1) PRIMARY KEY,
    id_zlecenia INT NOT NULL,
    id_uslugi INT NOT NULL,
    ilosc INT NOT NULL DEFAULT 1,
    cena_jednostkowa DECIMAL(10,2) NOT NULL,
    CONSTRAINT FK_zluslugi_zlecenia FOREIGN KEY (id_zlecenia)
        REFERENCES zlecenia(id_zlecenia),
    CONSTRAINT FK_zluslugi_uslugi FOREIGN KEY (id_uslugi)
        REFERENCES uslugi(id_uslugi)
)
```

## 3.8 Tabela: zlecenia_czesci

Tabela asocjacyjna łącząca zlecenia z częściami (relacja N:M). Trigger trg_update_magazyn odejmuje ilość ze stanu magazynowego.

| Atrybut | Typ | Ograniczenia | Opis |
|---|---|---|---|
| id | INT | PK, IDENTITY | Klucz główny wiersza. |
| id_zlecenia | INT | FK, NOT NULL | Referencja do zlecenia. |
| id_czesci | INT | FK, NOT NULL | Referencja do części. |
| ilosc | INT | DEFAULT 1 | Liczba użytych sztuk. |
| cena_jednostkowa | DECIMAL(10,2) | NOT NULL | Kopia ceny z magazynu. |

```sql
CREATE TABLE zlecenia_czesci (
    id INT IDENTITY(1,1) PRIMARY KEY,
    id_zlecenia INT NOT NULL,
    id_czesci INT NOT NULL,
    ilosc INT NOT NULL DEFAULT 1,
    cena_jednostkowa DECIMAL(10,2) NOT NULL,
    CONSTRAINT FK_zlczesci_zlecenia FOREIGN KEY (id_zlecenia)
        REFERENCES zlecenia(id_zlecenia),
    CONSTRAINT FK_zlczesci_czesci FOREIGN KEY (id_czesci)
        REFERENCES czesci(id_czesci)
)
```

## 3.9 Tabela: faktury

Faktura powiązana 1:1 ze zleceniem (UNIQUE na id_zlecenia). Kwoty są wyliczane przez procedurę sp_zamknij_zlecenie.

| Atrybut | Typ | Ograniczenia | Opis |
|---|---|---|---|
| id_faktury | INT | PK, IDENTITY | Klucz główny. |
| id_zlecenia | INT | FK, UNIQUE, NOT NULL | Max jedna faktura na zlecenie. |
| numer_faktury | NVARCHAR(30) | UNIQUE, NOT NULL | Numer faktury, np. FV/2024/001. |
| data_wystawienia | DATE | – | Data wystawienia dokumentu. |
| kwota_netto | DECIMAL(10,2) | – | Kwota netto w PLN. |
| kwota_brutto | DECIMAL(10,2) | – | Kwota brutto w PLN. |
| status_platnosci | NVARCHAR(30) | CHECK | nieoplacona / oplacona / czesciowo_oplacona. |

```sql
CREATE TABLE faktury (
    id_faktury INT IDENTITY(1,1) PRIMARY KEY,
    id_zlecenia INT NOT NULL UNIQUE,
    numer_faktury NVARCHAR(30) NOT NULL UNIQUE,
    data_wystawienia DATE,
    kwota_netto DECIMAL(10,2),
    kwota_brutto DECIMAL(10,2),
    status_platnosci NVARCHAR(30) NOT NULL DEFAULT 'nieoplacona',
    CONSTRAINT FK_faktury_zlecenia FOREIGN KEY (id_zlecenia)
        REFERENCES zlecenia(id_zlecenia),
    CONSTRAINT CK_faktury_status CHECK (
        status_platnosci IN ('nieoplacona','oplacona','czesciowo_oplacona')
    )
)
```

---

# 4. Widoki (Views)

Wszystkie złączenia w widokach używają jawnie wypisanego rodzaju złączenia (INNER JOIN / LEFT OUTER JOIN).

## 4.1 vw_aktywne_zlecenia

Wyświetla zlecenia o statusie przyjęte lub w realizacji wraz z danymi klienta, pojazdu i mechanika.

```sql
CREATE VIEW vw_aktywne_zlecenia AS
SELECT
    z.id_zlecenia,
    z.status,
    z.data_przyjecia,
    k.imie + ' ' + k.nazwisko AS klient,
    k.telefon AS telefon_klienta,
    p.marka + ' ' + p.model AS pojazd,
    p.nr_rejestracyjny,
    pr.imie + ' ' + pr.nazwisko AS mechanik,
    z.opis_usterki
FROM zlecenia z
INNER JOIN pojazdy p ON p.id_pojazdu = z.id_pojazdu
INNER JOIN klienci k ON k.id_klienta = p.id_klienta
INNER JOIN pracownicy pr ON pr.id_pracownika = z.id_mechanika
WHERE z.status IN ('przyjete', 'w_realizacji')
```

Przykład: `SELECT * FROM vw_aktywne_zlecenia`

## 4.2 vw_historia_pojazdu

Historia napraw danego pojazdu, można filtrować po numerze rejestracyjnym (RF-14).

```sql
CREATE VIEW vw_historia_pojazdu AS
SELECT
    p.nr_rejestracyjny,
    p.marka + ' ' + p.model AS pojazd,
    k.imie + ' ' + k.nazwisko AS wlasciciel,
    z.id_zlecenia,
    z.data_przyjecia,
    z.data_zakonczenia,
    z.status,
    z.opis_usterki,
    f.numer_faktury,
    f.kwota_brutto
FROM zlecenia z
INNER JOIN pojazdy p ON p.id_pojazdu = z.id_pojazdu
INNER JOIN klienci k ON k.id_klienta = p.id_klienta
LEFT OUTER JOIN faktury f ON f.id_zlecenia = z.id_zlecenia
```

Przykład: `SELECT * FROM vw_historia_pojazdu WHERE nr_rejestracyjny = 'KR12345'`

## 4.3 vw_raport_przychodow

Lista wszystkich wystawionych faktur z danymi klienta i pojazdu (RF-12).

```sql
CREATE VIEW vw_raport_przychodow AS
SELECT
    f.numer_faktury,
    f.data_wystawienia,
    k.imie + ' ' + k.nazwisko AS klient,
    p.nr_rejestracyjny,
    f.kwota_netto,
    f.kwota_brutto,
    f.status_platnosci
FROM faktury f
INNER JOIN zlecenia z ON z.id_zlecenia = f.id_zlecenia
INNER JOIN pojazdy p ON p.id_pojazdu = z.id_pojazdu
INNER JOIN klienci k ON k.id_klienta = p.id_klienta
```

Przykład: `SELECT * FROM vw_raport_przychodow ORDER BY data_wystawienia DESC`

## 4.4 vw_oblozenie_mechanikow

Liczba aktualnie otwartych zleceń każdego mechanika (RF-11).

```sql
CREATE VIEW vw_oblozenie_mechanikow AS
SELECT
    pr.id_pracownika,
    pr.imie + ' ' + pr.nazwisko AS mechanik,
    COUNT(z.id_zlecenia) AS liczba_otwartych_zlecen
FROM pracownicy pr
LEFT OUTER JOIN zlecenia z
    ON z.id_mechanika = pr.id_pracownika
    AND z.status IN ('przyjete', 'w_realizacji')
WHERE pr.stanowisko = 'mechanik'
GROUP BY pr.id_pracownika, pr.imie, pr.nazwisko
```

Przykład: `SELECT * FROM vw_oblozenie_mechanikow ORDER BY liczba_otwartych_zlecen DESC`

## 4.5 vw_sms_do_wyslania

Zakończone naprawy z bieżącego dnia wraz z telefonami klientów – dane dla zewnętrznego systemu SMS (RF-13).

```sql
CREATE VIEW vw_sms_do_wyslania AS
SELECT
    k.imie + ' ' + k.nazwisko AS klient,
    k.telefon,
    p.marka + ' ' + p.model AS pojazd,
    p.nr_rejestracyjny,
    z.id_zlecenia,
    z.data_zakonczenia
FROM zlecenia z
INNER JOIN pojazdy p ON p.id_pojazdu = z.id_pojazdu
INNER JOIN klienci k ON k.id_klienta = p.id_klienta
WHERE z.status = 'zakonczone'
    AND CAST(z.data_zakonczenia AS DATE) = CAST(GETDATE() AS DATE)
```

Przykład: `SELECT * FROM vw_sms_do_wyslania`

---

# 5. Procedury składowane

Błędy zgłaszane są instrukcją `THROW`. Niestandardowe numery błędów zaczynają się od 50000, jak wymaga tego THROW.

## 5.1 sp_nowe_zlecenie

Tworzy nowe zlecenie naprawy. Przyjmuje ID pojazdu, ID mechanika i opis usterki. Ustawia status przyjete i datę bieżącą, zwraca ID nowego zlecenia.

```sql
CREATE PROCEDURE sp_nowe_zlecenie
    @id_pojazdu INT,
    @id_mechanika INT,
    @opis_usterki NVARCHAR(MAX)
AS
BEGIN
    INSERT INTO zlecenia (id_pojazdu, id_mechanika, data_przyjecia, status, opis_usterki)
    VALUES (@id_pojazdu, @id_mechanika, GETDATE(), 'przyjete', @opis_usterki)

    SELECT SCOPE_IDENTITY() AS nowe_id_zlecenia
END
```

Przykład wywołania:
```sql
EXEC sp_nowe_zlecenie
    @id_pojazdu=3,
    @id_mechanika=2,
    @opis_usterki=N'Stuk w zawieszeniu'
```

## 5.2 sp_dodaj_usluge

Dodaje usługę do zlecenia, kopiując aktualną cenę z cennika.

```sql
CREATE PROCEDURE sp_dodaj_usluge
    @id_zlecenia INT,
    @id_uslugi INT,
    @ilosc INT = 1
AS
BEGIN
    DECLARE @cena DECIMAL(10,2)
    SELECT @cena = cena_robocizny FROM uslugi WHERE id_uslugi = @id_uslugi

    INSERT INTO zlecenia_uslugi (id_zlecenia, id_uslugi, ilosc, cena_jednostkowa)
    VALUES (@id_zlecenia, @id_uslugi, @ilosc, @cena)
END
```

Przykład wywołania: `EXEC sp_dodaj_usluge @id_zlecenia=5, @id_uslugi=1, @ilosc=1`

## 5.3 sp_dodaj_czesc

Dodaje część do zlecenia. Stan magazynowy jest odejmowany automatycznie przez trigger trg_update_magazyn (uruchamia się po INSERT do zlecenia_czesci) – procedura tylko sprawdza wcześniej, czy stanu wystarczy, i zgłasza błąd instrukcją THROW.

```sql
CREATE PROCEDURE sp_dodaj_czesc
    @id_zlecenia INT,
    @id_czesci INT,
    @ilosc INT = 1
AS
BEGIN
    DECLARE @stan INT, @cena DECIMAL(10,2)
    SELECT @stan = stan_magazynowy, @cena = cena_sprzedazy
    FROM czesci WHERE id_czesci = @id_czesci

    IF @stan < @ilosc
        THROW 50001, 'Niewystarczajacy stan magazynowy', 1;

    INSERT INTO zlecenia_czesci (id_zlecenia, id_czesci, ilosc, cena_jednostkowa)
    VALUES (@id_zlecenia, @id_czesci, @ilosc, @cena)
END
```

Przykład wywołania: `EXEC sp_dodaj_czesc @id_zlecenia=5, @id_czesci=1, @ilosc=2`

## 5.4 sp_zamknij_zlecenie

Zamyka zlecenie, oblicza sumę wartości usług i części, wylicza kwotę netto i brutto (VAT 23%) i wystawia fakturę. Dwie modyfikujące dane instrukcje (UPDATE zlecenia oraz INSERT faktury) są spięte jawną transakcją z obsługą BEGIN TRY / BEGIN CATCH – analogicznie do wzorca `p_add_order2` / `p_add_detail` pokazanego na zajęciach – żeby albo wykonały się obie, albo żadna.

```sql
CREATE PROCEDURE sp_zamknij_zlecenie
    @id_zlecenia INT,
    @numer_faktury NVARCHAR(30)
AS
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM zlecenia
        WHERE id_zlecenia = @id_zlecenia
        AND status NOT IN ('zakonczone', 'anulowane')
    )
        THROW 50002, 'Zlecenie nie istnieje lub jest juz zamkniete', 1;

    DECLARE @netto DECIMAL(10,2)
    DECLARE @brutto DECIMAL(10,2)

    SELECT @netto = ISNULL(SUM(ilosc * cena_jednostkowa), 0)
    FROM zlecenia_uslugi WHERE id_zlecenia = @id_zlecenia

    SELECT @netto = @netto + ISNULL(SUM(ilosc * cena_jednostkowa), 0)
    FROM zlecenia_czesci WHERE id_zlecenia = @id_zlecenia

    SET @brutto = ROUND(@netto * 1.23, 2)

    BEGIN TRY
        BEGIN TRANSACTION;

        UPDATE zlecenia
        SET status = 'zakonczone', data_zakonczenia = GETDATE()
        WHERE id_zlecenia = @id_zlecenia;

        INSERT INTO faktury (id_zlecenia, numer_faktury, data_wystawienia,
            kwota_netto, kwota_brutto, status_platnosci)
        VALUES (@id_zlecenia, @numer_faktury, CAST(GETDATE() AS DATE),
            @netto, @brutto, 'nieoplacona');

        COMMIT;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT >= 1
            ROLLBACK;
        THROW;
    END CATCH

    SELECT @netto AS kwota_netto, @brutto AS kwota_brutto
END
```

Przykład wywołania:
```sql
EXEC sp_zamknij_zlecenie
    @id_zlecenia=5,
    @numer_faktury='FV/2025/001'
```

## 5.5 sp_historia_klienta

Zwraca pełną historię napraw wszystkich pojazdów danego klienta – realizuje wymaganie RF-15 (raport historii klienta). Analogiczna do procedury `p_customer_order_total` z zajęć: sprawdza istnienie klienta i zwraca jeden wynikowy zbiór posortowany od najnowszego zlecenia.

```sql
CREATE PROCEDURE sp_historia_klienta
    @id_klienta INT
AS
BEGIN
    IF NOT EXISTS (SELECT 1 FROM klienci WHERE id_klienta = @id_klienta)
        THROW 50005, 'Brak klienta o takim id', 1;

    SELECT
        p.nr_rejestracyjny,
        p.marka + ' ' + p.model AS pojazd,
        z.id_zlecenia,
        z.data_przyjecia,
        z.data_zakonczenia,
        z.status,
        z.opis_usterki
    FROM zlecenia z
    INNER JOIN pojazdy p ON p.id_pojazdu = z.id_pojazdu
    WHERE p.id_klienta = @id_klienta
    ORDER BY z.data_przyjecia DESC
END
```

Przykład wywołania: `EXEC sp_historia_klienta @id_klienta=1`

---

# 6. Triggery

## 6.1 trg_update_magazyn

**Zakres:** tabela zlecenia_czesci, AFTER INSERT.

Po wstawieniu wiersza do zlecenia_czesci automatycznie odejmuje pobraną ilość ze stanu magazynowego w tabeli czesci. Jeśli stanu brakuje, zgłasza błąd instrukcją THROW – SQL Server automatycznie wycofuje wtedy całą instrukcję wstawiającą wiersz, bo trigger działa w tej samej transakcji co INSERT, który go uruchomił (bez potrzeby jawnego ROLLBACK w treści triggera).

```sql
CREATE TRIGGER trg_update_magazyn
ON zlecenia_czesci
AFTER INSERT
AS
BEGIN
    IF EXISTS (
        SELECT 1 FROM inserted i
        INNER JOIN czesci c ON c.id_czesci = i.id_czesci
        WHERE c.stan_magazynowy < i.ilosc
    )
        THROW 50003, 'Brak wystarczajacej ilosci czesci w magazynie', 1;

    UPDATE c
    SET c.stan_magazynowy = c.stan_magazynowy - i.ilosc
    FROM czesci c
    INNER JOIN inserted i ON i.id_czesci = c.id_czesci
END
```

## 6.2 trg_blokada_zamknietego_zlecenia

**Zakres:** tabela zlecenia, AFTER UPDATE.

Blokuje modyfikację zlecenia, które ma już status zakończone lub anulowane, jeśli aktualizacja nie dotyczy zmiany samego statusu.

```sql
CREATE TRIGGER trg_blokada_zamknietego_zlecenia
ON zlecenia
AFTER UPDATE
AS
BEGIN
    IF EXISTS (
        SELECT 1 FROM deleted d
        INNER JOIN inserted i ON i.id_zlecenia = d.id_zlecenia
        WHERE d.status IN ('zakonczone', 'anulowane')
        AND i.status = d.status
    )
        THROW 50004, 'Nie mozna edytowac zamknietego zlecenia', 1;
END
```

## 6.3 trg_zwrot_czesci_do_magazynu

**Zakres:** tabela zlecenia_czesci, AFTER DELETE.

Lustrzane odbicie `trg_update_magazyn`: gdy wiersz zostanie usunięty ze zlecenia_czesci (np. recepcjonista pomylił się przy dodawaniu części), automatycznie zwraca odjętą wcześniej ilość do stanu magazynowego. Korzysta z pseudo-tabeli `deleted` i operacji zbiorowej UPDATE...FROM...JOIN, więc poprawnie obsłuży też usunięcie wielu wierszy naraz.

```sql
CREATE TRIGGER trg_zwrot_czesci_do_magazynu
ON zlecenia_czesci
AFTER DELETE
AS
BEGIN
    UPDATE c
    SET c.stan_magazynowy = c.stan_magazynowy + d.ilosc
    FROM czesci c
    INNER JOIN deleted d ON d.id_czesci = c.id_czesci
END
```

## 6.4 trg_blokada_usuniecia_oplaconej_faktury

**Zakres:** tabela faktury, INSTEAD OF DELETE.

Chroni opłacone faktury przed usunięciem (wymóg audytowy – raz opłaconego dokumentu księgowego nie powinno dało się po prostu skasować). Trigger typu INSTEAD OF całkowicie zastępuje domyślną operację DELETE – dlatego dla faktur nieopłaconych musi sam jawnie wykonać usunięcie, w przeciwnym razie żaden wiersz nigdy by się nie usunął.

```sql
CREATE TRIGGER trg_blokada_usuniecia_oplaconej_faktury
ON faktury
INSTEAD OF DELETE
AS
BEGIN
    IF EXISTS (SELECT 1 FROM deleted WHERE status_platnosci = 'oplacona')
        THROW 50006, 'Nie mozna usunac oplaconej faktury', 1;

    DELETE FROM faktury
    WHERE id_faktury IN (SELECT id_faktury FROM deleted)
END
```

## 6.5 trg_walidacja_przebiegu

**Zakres:** tabela pojazdy, AFTER UPDATE.

Pilnuje sensownej reguły biznesowej: przebieg pojazdu (licznik kilometrów) nie może się zmniejszyć przy aktualizacji danych pojazdu. Porównuje nową wartość (`inserted`) ze starą (`deleted`) dla tego samego id_pojazdu.

```sql
CREATE TRIGGER trg_walidacja_przebiegu
ON pojazdy
AFTER UPDATE
AS
BEGIN
    IF EXISTS (
        SELECT 1 FROM inserted i
        INNER JOIN deleted d ON d.id_pojazdu = i.id_pojazdu
        WHERE i.przebieg < d.przebieg
    )
        THROW 50007, 'Przebieg nie moze sie zmniejszyc', 1;
END
```

---

# 7. Dane testowe

Skrypt INSERT wypełnia bazę przykładowymi danymi: 4 pracowników, 4 klientów, 5 pojazdów, 8 usług w cenniku, 8 rodzajów części, 5 zleceń (3 zakończone, 1 w realizacji, 1 przyjęte) oraz 3 faktury. Dane zawierają polskie znaki, co pozwala zweryfikować poprawne działanie typów NVARCHAR.

Skrypt z przykładowymi danymi nie jest dołączony do tego repozytorium. Powyższy opis określa zakres danych testowych.

---

# 8. Prezentacja

Poniżej zebrano przykładowe zapytania do sprawdzenia działania bazy.

**1. Aktywne zlecenia**
```sql
SELECT * FROM vw_aktywne_zlecenia
```

**2. Historia pojazdu KR12345**
```sql
SELECT * FROM vw_historia_pojazdu
WHERE nr_rejestracyjny = 'KR12345'
```

**3. Raport przychodów**
```sql
SELECT * FROM vw_raport_przychodow
ORDER BY data_wystawienia DESC
```

**4. Obciążenie mechaników**
```sql
SELECT * FROM vw_oblozenie_mechanikow
ORDER BY liczba_otwartych_zlecen DESC
```

**5. Suma przychodów z faktur opłaconych**
```sql
SELECT SUM(kwota_brutto) AS przychod
FROM vw_raport_przychodow
WHERE status_platnosci = 'oplacona'
```

**6. Nowe zlecenie – wywołanie procedury**
```sql
EXEC sp_nowe_zlecenie
    @id_pojazdu=3,
    @id_mechanika=2,
    @opis_usterki=N'Stuk w zawieszeniu'
```

**7. Zamknięcie zlecenia 5 i wystawienie faktury**
```sql
EXEC sp_zamknij_zlecenie
    @id_zlecenia=5,
    @numer_faktury='FV/2025/001'
```

**8. Historia napraw klienta nr 1**
```sql
EXEC sp_historia_klienta @id_klienta=1
```

**9. Test triggera trg_update_magazyn – dodanie części do zlecenia**
```sql
INSERT INTO zlecenia_czesci (id_zlecenia, id_czesci, ilosc, cena_jednostkowa)
VALUES (4, 5, 1, 60.00)

SELECT stan_magazynowy FROM czesci WHERE id_czesci = 5
```

**10. Test triggera trg_blokada_zamknietego_zlecenia – edycja zamkniętego zlecenia**
```sql
UPDATE zlecenia SET opis_usterki = 'zmiana'
WHERE id_zlecenia = 1
```

**11. Test triggera trg_zwrot_czesci_do_magazynu – usunięcie pozycji ze zlecenia**
```sql
SELECT stan_magazynowy FROM czesci WHERE id_czesci = 1

DELETE FROM zlecenia_czesci
WHERE id_zlecenia = 1 AND id_czesci = 1

SELECT stan_magazynowy FROM czesci WHERE id_czesci = 1
```

**12. Test triggera trg_blokada_usuniecia_oplaconej_faktury – próba usunięcia opłaconej faktury**
```sql
DELETE FROM faktury WHERE id_faktury = 1
```

**13. Test triggera trg_walidacja_przebiegu – próba zmniejszenia przebiegu**
```sql
UPDATE pojazdy SET przebieg = 140000
WHERE id_pojazdu = 1
```

