# INFORMAČNÍ SYSTÉM SFRD - Studie proveditelnosti databázového modelu a ORM vrstvy

Ing. Lukáš Kubíček
2. 7. 2019

K tomuto dokumentu náleží zdrojové soubory projektu "DatabaseFeasibility" (Visual Studio 2017, .NET Core 2.2) ilustrující implementační proveditelnost diskutovaného řešení. (Instrukce ke spuštění viz níže v sekci WEB API).

## Základní požadavky

Ověřit implementační proveditelnost architektury návrhu propojení databáze (MS SQL Server) a objektové vrstvy (ORM = Object Relational Mapping) s ohledem na rychlost a celkovou efektivitu pro rozsáhlé datasety (statisíce položek, GB dat).

## .NET Core a Entity Framework
.NET Core je zcela přepsaná verze .NET Frameworku of Microsoftu, s následujícími vlastnostmi:

 - open-source
 - multiplatformní (Windows, Linux, Mac OS)
 - vhodná pro psaní webových a databázových aplikací
 - v nezávislých benchmarcích (např TechEmpower Web Framework Benchmarks) se umísťuje na předních místech

**Entity Framework** (EF) je ORM vrstva pro .NET Core (EF Core), která umožňuje namapovat vztahy z relační databáze do objektových vztahů jazyka C#. Aktuální stabilní verze je **EF Core 2.2**, a poslední beta verze je **3.0-preview6**. Vydání stabilní verze 3.0. se očekává na podzim 2019.

Vzhledem k využití technologie Microsoft SQL Serveru je dobré i použití .NET Core a Entity Framework.

### EF - raw SQL
EF Core překládá zápis z objektového jazyka C# (LINQ - Language Integrated Query) do relačního jazyka SQL a výsledky těchto dotazů zase naplňuje objektové struktury.
V případě, že by EF Core neuměl přeložit některé komplikované dotazy z objektové vrstvy do SQL dotazu nebo pokud by tyto SQL dotazy byly pomalé, lze využít metody  pro tzv. "RAW SQL QUERIES". Těmi lze vytvořit část SQL dotazu ručně. Na výsledek lze navázat dalšími C#/LINQ metodamy.
Toto eliminuje problém možného "neproveditelného dotazu", který by mohl při implementaci vzniknout. Vždy je možnost v případě potřeby využít těchto metod.
Ve verzi EF Core 2.2 jde o metodu "FromSql" a "ExecuteSql", ve verzi EF Core 3.0 budou přejmenovány na "FromSqlRaw" a "ExecuteSqlRaw".

Využitý této metody v je v přiloženém projektu v souboru **DataController.cs** a metodě **GetRawItems** kde na první část SQL dotazu navazují již klasické C#/LINQ metody.

## Jeden model - více databází

Projekt SFRD obsahuje více "podprojektů" které mají stejnou strukturu, mohou tedy být implementovány jako samostatné instance databází. Vykonávání SQL dotazů je v takovémto případě rychlejší, protože databáze prochází pouze část položek.
Alternativou by bylo mít všechny projekty v jedné instanci databáze a  využít cluster indexy podle projektů. Toto by značně zkomplikovalo jak návrh databázové struktury, tak propojení s ORM vrstvou.


![diagram](mermaid-diagram-20190703134452.svg "Jeden model - více databází" )
```

### Možné problémy
Přístup "jeden model - více databází" je méně obvyklý a má následující nevýhody:

 1. komplikovanější inicializace modelu (obvykle se počítá s připojením
    jednoho modelu k jedné databázi)
2. komplikovanější migrace (změny v modelu) a úprava databázových schémat
3. méně dokumentace

### Řešení
1. Připojení jednoho modelu k více databázím lze provést v metodě **OnConfiguring** při inicializování modelu. Zde v Connectoin stringu nahradíme jméno instance databáze, kterou nastavíme při vytváření objektu konstruktorem. Ukázka je v souboru **ProductsContext.cs**
2. Při změně modelu (přidání položek atd) využívá EF Core mechanismus Migrací (vytvoření rozdílového souboru např **20190628170358_newproperty1.cs** a následně aktualizuje schéma databáze. Pro aktualizaci více databázových instancí jedním modelem lze použít vytvořený PowerShell script (**UpdateDatabase.ps1**). Ten načte jména databází z textového souboru (**databases.txt**) a provede jejich migraci postupně tak, že nastaví environment variable na název databáze kterou migruje a v metodě "OnConfiguring" Modelu je toto využito na úpravu connection stringu po dobu migrace.
3. Jde sice o méně běžný případ, ale občas se využívá, vyskytuje se na toto téma několik dotazů v Issues na Githubu EF projektu.

Všechny problémy s přístupem "jeden model - více databází" jsou tedy řešitelné a jejich ukázková implementace je přiložena.

## Ukázka WEB API

Přiložený projekt po spuštění zpřístupní WEB API (asp.net core) kde je několik ukázkových přístupů do databáze, čtení a zápis dat.

Pro spuštění je potřeba upravit connection stringy k existujícímu SQL severu v souborech ProductsContext.cs a UsersContext.cs.
Jako první je potřeba zavolat `/api/data/populate` a `/api/users/populate`
Tato dvě volání vytvoří 2 databáze zalořené na modelu Procuct (productA, pruductB) a databázi uživatelů SFRDUsers.

**Čtení dat a určení databáze z parametru**
Metody nad databází uživatelů v souboru **UsersController.cs** ilustrují klasické akce EF Core pro práci s daty.
Metody v **DataController.cs** ilustrují přístup k více databázím jedním modelem. Např metoda **GetProducts** voláním `/api/data/productA/products` vrátí JSON se seznamem produktů v databázi "productA" - tento parametr z URL je možné transformovat, zde je použito přímo jméno databáze pro jednoduchost.

## Spojení projektových databází s databází klientů
Seznam klientů je v samostatné databázi. Některé dotazy mají kombinovat data z produktových databází a databáze klientů.

Pro takový případ je vhodné použití **cross database query**. MS SQL Server umí zpracovat SQL dotaz nad více databázemi, pokud jsou všechny lokálně přístupné. SQL server provede optimalizaci a dotaz vyhodnocuje podobně jako dotazy z jedné databáze. Cross-DB bohužel není kompatibilní s optimalizací **"in-memory-oltp"** viz [odkaz](https://docs.microsoft.com/en-us/sql/relational-databases/in-memory-oltp/cross-database-queries?view=sql-server-2017)

> Starting with SQL Server 2014 (12.x), memory-optimized tables do not
> support cross-database transactions. You cannot access another
> database from the same transaction or the same query that also
> accesses a memory-optimized table. You cannot easily copy data from a
> table in one database, to a memory-optimized table in another
> database.

Vzhledem k velikosti datasetu, se jeví jako vhodné použít **in-memoy tables**. V tom případě by ale nešlo využít cross-database queries. Jako řešení by mohla data s klienty být v cache ve statické proměnné v programu. Konkrétní řešení závisí na konkrétních vztazích datových struktur (jejich komplexnost, propojenost, možnost nahrání do paměti atd). Je také možní že in-memory tabulky s dobrými indexy budou zvládat bez problému i dotaz rozdělený (1 část dotazu do produktové db, druhá to uživatelské nebo naopak).

Spojení dat z více databází není neřešitelný problém, zvolení způsobu řešení závisí na konkrétních vlastnostech reálného datasetu. Navrhuji otestovat zde popsaná řešení nad datasetem s odpovídající komplexností a velikostí.

## Využiží Cache v .NET Core

.NET Core nabízí několik mechanizmů pro využití Cache. Jak pro Entity Framework (Query cache),  tak pro API requesty (response caching). Také je možné využít vlastní statické třídy kolekcí (zejména Dictionary a Lookup jsou velmi rychlé) a ručně ovládat jejich životnost.
