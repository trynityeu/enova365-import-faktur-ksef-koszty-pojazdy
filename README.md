# Import faktur kosztowych i samochodowych z KSeF bez wyboru matrycy (enova365)

> Element większej całości: **[Obieg faktur zakupu z KSeF w enova365 — mapa rozwiązania](../../trynityeu/enova365-obieg-faktur-ksef)**

Dodatek do systemu ERP **enova365** (Soneta sp. z o.o.), automatyzujący import
faktur zakupu odebranych przez **Krajowy System e-Faktur (KSeF)** do dwóch
rodzajów dokumentów ewidencji: **ZKE — zakupy kosztowe** i **ZSE — zakupy
samochodowe**. Rodzaj dokumentu i wariant faktury rozpoznaje sam z pliku,
bez okna wyboru i bez pytania operatora.

To repozytorium zawiera wyłącznie **opis funkcjonalny** — bez kodu
źródłowego, bez danych klienta, bez konfiguracji wdrożeniowej.

Pokrewne dodatki tej samej rodziny:

- [Import faktur zakupu materiałowego (ZME) z dopasowaniem do zamówień](../../trynityeu/enova365-import-faktur-ksef-dopasowanie)
  — inny proces: tam koszt wynika z zamówienia zakupu, tutaj z karty
  kontrahenta;
- [Administracja mapowaniem pól faktury KSeF per dostawca](../../trynityeu/enova365-mapowanie-pol-faktury-ksef)
  — schematy mapowania XML; ten dodatek z nich **nie korzysta**.

## Problem, który rozwiązuje

Standardowy import faktury z KSeF w enova365 prowadzi przez okno „Utwórz
dokument", w którym operator musi **wskazać właściwą matrycę** — zestaw
reguł przekształcających XML faktury w dokument ewidencji. Matryc było
siedem (osobna dla faktury zwykłej, korekty, zaliczkowej, korekty
zaliczkowej i rozliczeniowej po stronie kosztowej, oraz zwykłej i korekty
po stronie pojazdowej), a wybór zależał od pola w XML, którego operator
nie widzi przed otwarciem pliku. Pomyłka oznaczała dokument zaksięgowany
według złych reguł.

Dodatkowo matryce żyją **w konfiguracji bazy danych**, nie w kodzie: nie
mają historii zmian, nie da się ich sensownie przetestować ani porównać
między środowiskami, a rozjazd między matrycą a resztą procesu wychodził
dopiero na zaksięgowanym dokumencie.

## Jak działa

- **Jeden przycisk na liście pobranych plików KSeF** — bez okna wyboru
  matrycy i bez okna „Utwórz dokument".
- **Automatyczne rozpoznanie rodzaju** dokumentu (kosztowy vs samochodowy)
  z rodzaju pliku KSeF oraz **wariantu faktury** (zwykła VAT, korekta,
  zaliczkowa, korekta zaliczkowa, rozliczeniowa) wprost z treści XML.
  Dyspozytor wybiera właściwą ścieżkę przetwarzania per plik.
- **Pliki innego rodzaju są pomijane** — faktury zakupu materiałowego
  obsługuje osobny dodatek, więc w mieszanym zaznaczeniu każda faktura
  trafia tam, gdzie powinna, zamiast zostać przetworzona złymi regułami.
- **Cała logika siedmiu matryc przeniesiona do kodu dodatku** — import
  wykonuje natywny mechanizm enova365, ale bez wykonywania jakiejkolwiek
  matrycy z konfiguracji; opis analityczny i cechy dokłada dodatek. Skutek
  praktyczny: logika księgowania ma historię zmian, wersję i daje się
  wdrożyć jednym plikiem, a wdrożenie nie wymaga zakładania matrycy
  w konfiguracji bazy.
- **Kontrola przed importem, nie po** — faktura rozliczeniowa bez kwoty do
  zapłaty wymaga istniejącej, poprawnie zarejestrowanej faktury zaliczkowej.
  Warunek jest sprawdzany **zanim** cokolwiek powstanie: plik z problemem
  zostaje pominięty z czytelnym komunikatem, zamiast zostawić w bazie
  dokument-sierotę po przerwanym księgowaniu.
- **Import wsadowy odporny na błąd pojedynczego pliku** — przy zaznaczeniu
  wielu faktur błąd jednej z nich nie wycofuje pracy wykonanej dla
  pozostałych, a operator dostaje **jeden zbiorczy raport** całej partii
  (ile dokumentów powstało, co pominięto i dlaczego), nie serię okien.
- **Ustawienie porządkujące po imporcie** — na utworzonym dokumencie
  kwoty VAT nie są przenoszone na płatności (wymóg procesu księgowego
  wdrożenia).

## Jak powstaje opis analityczny

Opis analityczny to rozbicie dokumentu na linie kosztowe z przypisaniem
wymiarów kontrolingowych. Dodatek buduje go w całości sam — pozycja po
pozycji faktury — zamiast polegać na matrycy z konfiguracji.

### Skąd bierze się dekretacja

Faktura kosztowa — w odróżnieniu od materiałowej — nie ma zamówienia, do
którego można ją dopasować. Informacja „czyj to koszt" pochodzi więc
z **karty kontrahenta** (odnalezionego po numerze NIP z faktury), gdzie
jest ustawiana raz dla danego dostawcy i przenoszona automatycznie na
**każdą** linię opisu analitycznego: centrum kosztów, lokalizacja, projekt
i pracownik. Brak tych ustawień nie blokuje importu — dokument powstaje,
linie zostają bez wymiarów do ręcznego uzupełnienia.

### Jak powstaje pojedyncza linia

- **Jedna linia na pozycję faktury**; pozycje o kwocie zerowej są
  pomijane, żeby nie zaśmiecać dekretacji.
- **Kwota** to netto pozycji — z jednym wyjątkiem: dla branż, w których
  VAT nie podlega odliczeniu (usługi hotelowe i gastronomiczne, rozpoznane
  po typie kontrahenta), linia powstaje **w kwocie brutto**, wyliczonej
  z brutto i stawki, a nie wziętej z pola netto.
- **Stawka VAT** pozycji jest mapowana na słownik stawek w bazie, wraz
  z przypadkami szczególnymi („nie podlega", „zwolniona").
- **Waluta obca** przeliczana jest na złote po kursie z danej pozycji, a
  gdy pozycja go nie niesie — po kursie całego dokumentu. Oryginalna kwota
  walutowa zostaje zapisana w uwagach linii, żeby dekretacja była czytelna
  bez sięgania do pliku źródłowego.
- **Dwa pola opisowe o różnym przeznaczeniu**: pełne uwagi (nazwa pozycji,
  indeks, oznaczenie trybu brutto, kwota walutowa) oraz krótki opis linii,
  przycinany **na granicy słowa** z wielokropkiem, żeby zmieścić się
  w limicie pola i pozostać czytelnym.
- **Opis całego dokumentu** składany jest z nazw kolejnych pozycji, aż do
  wyczerpania limitu — pierwszy rzut oka na listę dokumentów mówi, czego
  faktura dotyczyła, bez jej otwierania.

### Czym różnią się warianty faktury

| Wariant | Co trafia do opisu analitycznego |
|---|---|
| **Zwykła (VAT)** | jedna linia na pozycję, kwoty wprost z faktury |
| **Korekta** | pozycje zestawiane parami „stan przed / stan po"; linia powstaje **tylko dla różnicy** i tylko gdy różnica jest niezerowa. Korekta bez różnic kończy się dokumentem z wyraźną adnotacją, zamiast pustym opisem bez wyjaśnienia |
| **Zaliczkowa** | linie budowane z wierszy zamówienia, nie z pozycji towarowych |
| **Korekta zaliczkowa** | jak zaliczkowa, ale dopuszcza kwoty ujemne i pomija wiersze stanu sprzed korekty |
| **Rozliczeniowa** | kwota końcowa **rozdzielana proporcjonalnie** między pozycje rozliczanej zaliczki; ostatnia pozycja każdej grupy absorbuje resztę z zaokrągleń, więc suma linii zgadza się co do grosza. Dodatkowo obciążenia typu **kaucja** trafiają jako osobne linie ze stawką „nie podlega" |

## Skąd faktura wie, czy jest kosztowa, samochodowa czy materiałowa

Rodzaju dokumentu **nie ustala żaden z dodatków** — przypisuje go sama
enova365 w momencie pobrania pliku z KSeF, na podstawie **cechy
algorytmicznej** skonfigurowanej w bazie: NIP z faktury → kontrahent →
jego typ → rodzaj dokumentu. Dostawca materiałów daje zakup materiałowy
(obsługiwany przez [osobny dodatek](../../trynityeu/enova365-import-faktur-ksef-dopasowanie)),
oznaczenie „pojazdy" — zakup samochodowy, a pozostałe typy **oraz brak
kontrahenta w bazie** — zakup kosztowy; dwa ostatnie obsługuje ten
dodatek.

Ten dodatek tę wartość wyłącznie **czyta** i po niej decyduje, czy plik
należy do niego — pliki cudzego rodzaju pomija zamiast je przetwarzać,
więc pomyłka w klasyfikacji kończy się komunikatem, nie błędnie
zaksięgowanym dokumentem.

→ [Automatyczna klasyfikacja faktur KSeF przed importem](../../trynityeu/enova365-klasyfikacja-faktur-ksef)
— pełny opis reguły, cech pomocniczych do przeglądu przed importem oraz
tego, kiedy poprawia się pojedynczy plik, a kiedy kartę kontrahenta.

## Co dzieje się wcześniej

- Plik faktury musi zostać **pobrany z KSeF** do listy plików w enova365
  (moduł integracji KSeF wbudowany w enova365 — poza zakresem tego
  dodatku) i mieć przypisany rodzaj kosztowy albo samochodowy (patrz
  „Skąd faktura wie…" wyżej).
- Karty kontrahentów kosztowych powinny mieć uzupełnioną dekretację
  (centrum kosztów, lokalizacja, projekt, pracownik) oraz typ kontrahenta
  — inaczej dokument powstanie, ale linie opisu analitycznego będą bez
  przypisania i trzeba je uzupełnić ręcznie.
- Dla faktury rozliczeniowej bez kwoty do zapłaty — odpowiadająca jej
  **faktura zaliczkowa musi już być w bazie** i w stanie pozwalającym na
  rozliczenie.

## Co dzieje się później

- Dokument trafia do zwykłego obiegu ewidencji VAT i księgowania —
  dodatek kończy pracę na utworzeniu dokumentu z gotowym opisem
  analitycznym.
- Faktury zakupu **materiałowego** idą zupełnie inną ścieżką: dopasowanie
  do zamówień zakupu i rozliczenie ilościowe.
  → [Import faktur zakupu materiałowego (ZME) z dopasowaniem do zamówień](../../trynityeu/enova365-import-faktur-ksef-dopasowanie)

## Jakie cechy (pola konfiguracyjne enova365) są wykorzystywane

- **Na karcie kontrahenta** — typ kontrahenta (rozstrzyga tryb brutto dla
  branż bez odliczenia VAT) oraz stała dekretacja dostawcy: centrum
  kosztów, lokalizacja, projekt, pracownik.
- **Na dokumencie ewidencji** — status płatności rozpoznany z faktury
  (zapłacona / do zapłaty), ustalany na podstawie znacznika zapłaty, formy
  płatności albo zbieżności daty wystawienia z terminem płatności.
- **Na liniach opisu analitycznego** — uwagi (pełna nazwa pozycji, indeks,
  oznaczenie trybu brutto i kwota walutowa), stawka VAT, lokalizacja,
  projekt i pracownik.

## Co jest do tego potrzebne

- enova365 z modułami Handel, Księga i Ewidencja VAT oraz aktywną
  integracją KSeF (pobieranie plików faktur do listy „Pobrane").
- Zdefiniowane w bazie rodzaje dokumentów KSeF dla zakupów kosztowych
  i samochodowych oraz odpowiadające im definicje dokumentów ewidencji.
- Uzupełnione cechy dekretacyjne na kartach kontrahentów kosztowych.
- Słownik stawek VAT zgodny z oznaczeniami stawek w fakturach.
- .NET 8 oraz referencje do bibliotek Soneta.Sdk w wersji zgodnej
  z zainstalowaną enovą.

## Uwaga wdrożeniowa: bezpiecznik między bazami

Ten sam zestaw dodatków bywa wgrany do kilku baz różnych firm, które mają
rodzaje dokumentów KSeF o **identycznych symbolach**, ale własne, różniące
się reguły księgowania. Dodatek rozpoznaje bazę, dla której powstał, i w
obcej bazie **nie uruchamia się w ogóle** — zamiast zaksięgować dokument
według cudzych reguł, nie robi nic.

## Zgodność

| | |
|---|---|
| System | enova365 (Soneta sp. z o.o.), wersja referencyjna **2512.9.10** |
| Platforma | .NET 8 |
| Schemat faktury | KSeF **FA(3)** |
| Rodzaje dokumentów | ZKE — zakupy kosztowe, ZSE — zakupy samochodowe |
| Warianty faktury | zwykła (VAT), korekta, zaliczkowa, korekta zaliczkowa, rozliczeniowa |
| Wdrożenie | dodatek instalowany przez interfejs enova365 (Narzędzia → Opcje → Dodatki), bez wymogu podpisu cyfrowego; nie wymaga zakładania matrycy w konfiguracji bazy |

## Czego tu nie ma

Kod źródłowy, treść matryc źródłowych, dane handlowe, dane osobowe oraz
wszelka konfiguracja specyficzna dla wdrożenia pozostają w prywatnym
repozytorium. To repo służy wyłącznie jako publiczny opis funkcji dodatku.
