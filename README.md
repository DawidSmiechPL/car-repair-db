# System bazodanowy warsztatu samochodowego

Projekt relacyjnej bazy danych warsztatu samochodowego w **Microsoft SQL Server (T-SQL)**.

## Zakres

- 9 tabel z kluczami głównymi, obcymi i ograniczeniami integralności;
- rejestr klientów, pojazdów, pracowników i zleceń napraw;
- magazyn części, cennik usług i faktury;
- 5 widoków raportowych;
- 5 procedur składowanych;
- 5 triggerów, m.in. do rozliczania części w magazynie;
- przykładowe zapytania i scenariusze demonstracyjne.

## Dokumentacja

**[Raport końcowy — opis projektu, struktura tabel i kod SQL](RAPORT_KONCOWY.md)**

Raport zawiera fragmenty T-SQL dla tabel, widoków, procedur i triggerów. Nie jest to gotowy pakiet wdrożeniowy: skrypty demonstracyjne wymagają odpowiednich danych testowych i sprawdzenia w środowisku SQL Server.

## Schemat ERD

[Edytowalny diagram bazy danych w dbdiagram.io](https://dbdiagram.io/d/6a19c263f15b4b045230a0bf).

## Technologie

Microsoft SQL Server · T-SQL · modelowanie relacyjne · integralność danych
