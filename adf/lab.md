Do wykonania zadania będą Ci potrzebne:
- usługa Azure Data Factory
- usługa Azure Storage z utworzonym kontenerem "mydata"
- usługa Azure SQL z najmniejszą bazą danych (może być serverless)
- plik z danymi ("hw_25000.csv") dołączony do tego zadania

> UWAGA: Przy bazie Azure SQL ustaw opcję "Allow Azure Services" na ON. Jest to niezbędne do połączenia Azure SQL z Azure Data Factory. Konfiguruję się tę opcją w zakładce "Firewalls and virtual networks".
## ZADANIE 1 - Kopiowanie danych z Blob storage do Azure SQL
Dane w pliku ("hw_25000.csv") zawierją informacje o wzroście i wadze pewnej grupy ludzi. Plik ma zatem 3 kolumny:
- id
- wzrost (w calach)
- waga (w funtach)

1. W pierwszym kroku chcemy załadować nasze dane z pliku .csv do bazy danych. Zanim jednak to zrobimy musimy utworzyć tabelkę w naszej bazie. W tym celu zaloguj się do bazy i wykonaj zapytanie:

```
CREATE TABLE people_raw
(
    id bigint,
    height float,
    weight float
);
```

Ta tabela będzie nam potrzebna na przechowywanie danych zaimportowanych z Azure Storage.

1. Następnie umieść plik ("hw_25000.csv") w swojeje usłudze Azure Storage w kontenerze "mydata".
2. Zaloguj się do portalu azure, wybierz usługę Azure Data Factory i otwórz UI. ![VSCode](img/webui.png)
3. W UI Azure Data Factory wybierz "Copy Data" ![VSCode](img/adf-ui.png)
4. Skonfiguruj cały proces tak, żeby dane były kopiowane z kontenera "mydata" do tabelki "people_raw" w bazie danych. Póki co mapowanie niech będzie proste: 1:1, choć **Pamiętaj o zdefiniowaniu odpowiednich typów danych.** Gdybyś miał problem z tym zadaniem włącz jeszcze raz lekcję "copy data" z tego tygodnia :)
- jako źródło wybierz Azure Storage
- jako cel wybierz Azure SQL i tabelkę "people_raw"
5. Po ukończeniu kopiowania danych wykonaj na swojej bazie polecenie SQL: `select count(*) from people_raw` (powinieneś otrzymać 25000 wierszy w tabeli).

## KROK 2 - zbudowanie prostej transformacji danych
W tym kroku będziemy chcieli nasze dane dalej przetransformować tak, aby wypełnić dwie nowe tabele w bazie danych:
- Tabela 1 będzie zawierać wskaźnik BMI dla osób z niedowagą
- Tabela 2 będzie zawierać jeden wiersz ze średnią wartością wagi dla wszystkich osób

1. Utwórz tabelę 1:
```
CREATE TABLE people_bmi
(
    id bigint,
    height float,
    weight float,
    bmi float
);
```
2. Utwórz tabelę 2:
```
CREATE TABLE avg_weight
(
    avg_weight float
);
```
3. W Azure Data Factory utwórz dwa nowe Dataset'y dla obu tabel.
4. Następnie utwórz nowy Data Flow, w którym zdefiniujesz odpowiednie transformacje. Twój data flow powinien rozdzielać dane źródłowe na dwie ścieżki. Schematycznie data flow powinien wyglądać tak:
```   
SOURCE -> kolumna BMI -> filtrowanie -> sink (people_bmi)
SOURCE -> liczenie średniej -> sink (avg_weight)
```
SOURCE będzie w naszym przypadku wspólne dla obu ścieżek.

1. W pierwszej ścieżce musisz dodać nową kolumnę o nazwie BMI. W tym celu dodaj krok wychodzący ze źródła danych typu **Derived Column**. Wartością nowej kolumny będzie bmi dla każdej osoby wyliczone wg wzoru: https://pl.wikipedia.org/wiki/Wska%C5%BAnik_masy_cia%C5%82a. **Pamiętaj, że w danych wzrost jest podany w calach, a waga w funtach**
2. Następnie w tej samej ścieżce dodaj filtr, który przepuści tylko wartości BMI mniejsze niż 18.5 (niedowaga).
3. Na końcu tej ścieżki dodaj sink wskazujący na tabelkę "people_bmi" w bazie danych.
4. Teraz utwórz drugą ścieżkę, w tym celu dodaj nową gałąź wychodzącą ze źródła (source).
5.  Dla tej nowej gałęzi dodaj nowy krok data flow typu "Aggregate". W "Aggregate Setting" kolumnę w opcji "Group By" zostaw pustą (chcemy policzyć średnią dla wszystkich wierszy łącznie), natomiast w opcji "Aggregates" wybierz kolumnę "weight", a jako expression wpisz "avg(weight)".
6.  Na końcu drugiej ścieżki dodaj nowy sink wskazujący na tabelę "avg_weight" w bazie danych.
7.  Kliknij przycisk "Validate All", a następnie "Publish All".

## ZADANIE 3 - utworzenie pipeline, który spina wszystko w całość
W ostanim zadaniu chcemy spiąć wszystko w jedną całość. Czyli zbudować proces ETLowy, który:
- pobiera dane z pliku na Blob Storage
- umieszcza te dane w tabeli "people_raw"
- Transformuje dane z tabeli "people_raw" do dwóch kolejnych tabel "people_bmi" oraz "avg_weight".
1. Na początku usuń dane, które zostały zaimportowane w Zadaniu 1, żeby mieć czyste środowisko. Na bazie danych wykonaj polecenie: `delete from people_raw`.
2. Przejdź do usługi Azure Data Factory. W zakładce "Pipelines" powinieneś mieć pipeline utworzony automatycznie podczas wykonywania Zadania 1.
3. Do tego pipeline przeciągnij data flow utworzony w Zadaniu 2.
4. Twój pipeline powinien wyglądać mniej więcej tak:
![VSCode](img/pipeline.png)
5. Kliknij przycisk "Validate All", a następnie "Publish All".
6. Uruchom swój pipeline klikając przycisk "Add trigger" -> "Trigger now" i "Finish".
7. Przejdź teraz do zakładki "Monitoring". Powinieneś tam zobaczyć, że Twój pipeline się wykonuje. Jeżeli pojawią się jakieś problemy, odczytaj komunikat błędu i spróbuj go rozwiązać. W przypadku innych trudności napisz na naszej grupie na Facebooku.
8. Po zakończeniu wykonywania pipelinu, wykonaj na bazie danych dwa zapytania SQL:
- `select count(*) from people_bmi;` -> powinienieś dostać 7453 wiersze
- `select * from avg_weight;` -> średnia waga powinna wynieść 127.079

## KONIEC

> **WAŻNE**:
> PAMIĘTAJ O USUNIĘCIU UTWORZONYCH ZASOBÓW W AZURE PO WYKONANIU ĆWICZEŃ