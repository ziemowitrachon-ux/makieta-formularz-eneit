# Formularz ENEIT na kalkulatorzuzyciapradu.pl – zmiany do wdrożenia

Dotyczy: `https://eneit.applicloud.pl/embed/kalkulator` osadzonego w ramce na
`https://kalkulatorzuzyciapradu.pl` (strona główna i podstrony `/branze/<slug>`).
Data: 03.11.2026. Zamawiający: Ziemowit (kalkulatorzuzyciapradu.pl).

W paczce:

- `ZMIANY-FORMULARZA.md` – ten plik (lista zmian z dokładnymi tekstami, kolorami i zachowaniem),
- `makieta/` – klikalna makieta. Otwórz `makieta/index.html` (cała strona z ramką)
  albo `makieta/formularz-makieta.html` (sam formularz). Przełącznik **TEST: Teraz / Makieta**
  w prawym dolnym rogu strony pokazuje w ramce obecny formularz albo wersję po zmianach.

Makietę najlepiej uruchomić przez dowolny lokalny serwer, np. `npx serve makieta`
albo `python -m http.server -d makieta`, i otworzyć podany adres. Otwarta prosto z dysku
(`file://`) też działa, ale Chrome blokuje wtedy czcionki i animację w tle strony.

Makieta jest statyczna (HTML + CSS + trochę JS), zbudowana na Waszym skompilowanym CSS
i Waszych klasach Tailwinda, więc większość klas da się przenieść 1:1. Klasy z prefiksem `m-`
to style makiety, których nie było w Waszym CSS – w kodzie produkcyjnym zamieńcie je na Tailwind.
Nazwy `data-test-id` w opisie poniżej pochodzą z Waszego obecnego formularza.

Skróty do etapów makiety (dopisz do adresu `formularz-makieta.html`):
`?etap=analiza` (ekran analizy w pętli), `?etap=wynik` (wynik i kontakt), `?etap=dziekujemy`.

> Kwota 4 860 zł i tekst podziękowania w makiecie są przykładowe. Wszystkie liczby muszą
> pochodzić z Waszych obliczeń.

---

## 1. Wysokość ramki – formularz musi wysyłać swoją prawdziwą wysokość (najważniejsze)

**Teraz:** formularz wysyła `{ type: 'eneit-embed-height', height: 900 }` raz, po załadowaniu,
niezależnie od szerokości ekranu i treści. Sprawdzone: przy ramce ustawionej na 400 px, 2000 px
i przy szerokości 335 px (telefon) zawsze przychodzi 900. Po rozwinięciu treści
(np. „Pokaż klauzulę”) nie przychodzi nic. Do tego `main#eneit-embed-content` ma `h-dvh overflow-hidden`,
a treść przewija się w środku ramki – na telefonie i na etapie kontaktu część formularza jest schowana.

**Po zmianie:** formularz wysyła wysokość treści przy każdej zmianie (etap, lista plików,
komunikat błędu, szerokość ekranu).

Strona już nasłuchuje tego komunikatu (nic nie zmieniamy w formacie):

```js
// kalkulatorzuzyciapradu.pl – istniejący odbiornik
window.addEventListener('message', (event) => {
  if (event.origin !== 'https://eneit.applicloud.pl' || event.source !== iframe.contentWindow) return;
  if (event.data?.type !== 'eneit-embed-height') return;
  const height = Number(event.data.height);
  if (Number.isFinite(height) && height >= 300 && height <= 12000) iframe.style.height = `${Math.ceil(height)}px`;
});
```

### Umowa między stroną a formularzem

- **Formularz zawsze wysyła naturalną wysokość swojej treści** – tyle, ile treść potrzebuje,
  żeby nic nie przewijało się w środku. Na każdym etapie, przy każdej szerokości.
- **Minimalną wysokość pilnuje strona:** na komputerze (szerokość strony ≥ 900 px) ramka ma
  `min-height: 545px` (równo z tekstem w sekcji hero obok), na telefonie minimum nie ma.
  Gdy treść jest niższa niż 545 px, formularz ma wypełnić ramkę (np. `min-h-dvh` na `main`,
  stopka przyklejona na dół), ale **mierzyć i wysyłać trzeba wysokość treści, nie ramki**.
- Wysokość obcinacie do zakresu 300–12000 px (poza nim strona ignoruje komunikat).
- Po naszej stronie: domyślna wysokość ramki na komputerze 545 px zamiast 900, `min-height: 545px`
  i płynne przejście `transition: height .35s`. To wdrażamy my.

Efekt (zmierzony w makiecie): Faktury i Analiza – 545 px, Wynik + Kontakt – ok. 950 px,
Dziękujemy – 545 px na komputerze. Makieta tylko to symuluje (wysyła 545 px albo wysokość treści),
obowiązuje opis powyżej.

### Propozycja kodu (React)

Mierzymy **trwały element bez rozciągania** (wrapper treści wewnątrz `main`, ten sam przez
wszystkie etapy), a nie `main` z `h-dvh` – inaczej pomiar zależy od wysokości ramki, którą sam zmienia.
Na etapach, które mają rosnąć, nie może być wewnętrznego `overflow-y-auto` z ograniczoną wysokością
(dziś `div.-mx-1.overflow-y-auto` w środku – ResizeObserver nie zauważy wzrostu treści przewijanej
w elemencie o stałym rozmiarze).

```tsx
const contentRef = useRef<HTMLDivElement>(null); // wrapper całej treści formularza (z paddingiem)

useEffect(() => {
  const el = contentRef.current;
  if (!el || window.parent === window) return;
  const send = () => {
    const h = Math.min(12000, Math.max(300, Math.ceil(el.getBoundingClientRect().height)));
    window.parent.postMessage({ type: 'eneit-embed-height', height: h }, 'https://kalkulatorzuzyciapradu.pl');
  };
  const ro = new ResizeObserver(send);
  ro.observe(el, { box: 'border-box' });
  send();
  return () => ro.disconnect();
}, []);
```

`targetOrigin` = `https://kalkulatorzuzyciapradu.pl` wystarcza: Wasz CSP (`frame-ancestors`)
i tak pozwala osadzić formularz tylko na tej domenie, a lokalnie i w podglądach Vercela strona
pokazuje w miejscu ramki zaślepkę. Jeśli chcecie testować u siebie, dopuśćcie listę originów
(np. `localhost`) i wysyłajcie do `document.referrer`-owego originu tylko, gdy jest na liście.

---

## 2. Etap 1 „Faktury”

### 2.1. Bez checkboxa zgody

- Usunąć checkbox `embed-consent-checkbox` z tekstem „Zapoznałem/am się z klauzulą…”.
- Usunąć przycisk „Pokaż klauzulę” (`embed-consent-toggle`) i rozwijaną klauzulę (`embed-consent-clause`).
  Treść klauzuli zostaje w polityce prywatności.
- Pole do wgrywania jest **od razu aktywne** (bez `aria-disabled`, bez `opacity-60`,
  bez komunikatu „Zaznacz zgodę, aby dodać faktury.”).

W stopce (`footer`, nad linkiem „Polityka prywatności”), tylko na tym etapie, dwie linie:

1. z ikoną kłódki (lucide `Lock`, 14 px, kolor `#416e19`, tekst pogrubiony `#416e19`):
   **„Faktury przesyłamy szyfrowanym połączeniem HTTPS”**
2. zwykły tekst (`text-xs text-[#60766a] leading-relaxed`):
   **„Wgrywając fakturę, wyrażam zgodę na przetwarzanie moich danych w celu przygotowania analizy.”**

> Do decyzji po Waszej stronie (prawnik): czy podstawą przetwarzania na tym etapie zostaje zgoda
> wyrażona przez wgranie pliku, czy działanie na żądanie osoby przed zawarciem umowy
> (art. 6 ust. 1 lit. b RODO) – wtedy zdanie może mieć charakter czysto informacyjny.
> Wygląd i miejsce zostają takie same. Do czasu decyzji obowiązuje tekst podany wyżej.
>
> Dziś zgodę z checkboxa zapisujecie osobno („Zgoda została zapisana.”). Po zmianie zapiszcie
> to samo zdarzenie przy pierwszym wgranym pliku: treść zdania, jego wersję, datę i godzinę.

### 2.2. Pole do wgrywania (`embed-upload-zone`)

- Tło: **białe `#ffffff`** (dziś `#e9f1e8`, a w stanie zablokowanym przed zgodą `#eef3ed`). Obramowanie bez zmian: `1.5px dashed #a7c2ad`,
  na hover `#6b9a38`.
- Ikona i napis „Przeciągnij pliki lub kliknij” bez przygaszenia (`opacity` 100%).
- Pole działa myszą, przeciąganiem i klawiaturą (Enter i Spacja otwierają wybór pliku).

### 2.3. Przycisk w polu

- Napis: **„Wgraj fakturę za prąd”** (zamiast „Wybierz pliki”).
- Kolor: tło **`#d92d20`**, obramowanie `#d92d20`, tekst biały; hover `#b42318`.
- Rozmiar bez zmian (`min-h-11`, `rounded-[8px]`, `text-[13px] font-bold`).
- Po dodaniu co najmniej jednego pliku napis zmienia się na **„Dodaj kolejną fakturę”**.

### 2.4. Po wgraniu plików – pole się zwęża, widać listę z podglądem

Gdy lista plików nie jest pusta (`compact`):

- pole do wgrywania robi się niskie: jeden wiersz (ikona 20 px + przycisk 36 px wysokości),
  bez napisów „Przeciągnij pliki…” i „PDF, do 20 MB…”,
- pod nim lista plików (`embed-file-list`), każdy wiersz:
  - **miniatura pierwszej strony PDF** 34 × 44 px (obramowanie `#d5e7d5`, `rounded 4px`);
    w makiecie to podgląd pliku z `URL.createObjectURL` w zmniejszonej ramce – u Was może to być
    np. render pierwszej strony (pdf.js) albo miniatura z serwera; gdy podglądu nie ma albo się nie
    wygenerował: napis „PDF” w tym samym kwadracie (wtedy przycisk „Podgląd” może zostać, jeśli plik
    da się otworzyć),
  - nazwa pliku (pogrubiona `#152b24`, obcięta wielokropkiem) i pod nią
    „rozmiar · **przesłano**” (`text-xs #52675c`, „przesłano” pogrubione `#416e19`),
  - przycisk-link **„Podgląd”** (otwiera plik w nowej karcie) i **„Usuń”**.
- Stany wiersza podczas wysyłania, błędu, ponowienia i usuwania (pasek postępu, „Ponów”,
  „Usuń z listy”) oraz zasady, kiedy „Analizuj faktury (n)” jest aktywny – **bez zmian**, jak dziś.
  Zmienia się tylko wygląd wiersza (miniatura, układ, przycisk „Podgląd”).
- Przycisk „Analizuj faktury (n)” (`embed-analyze-button`) bez zmian.

---

## 3. Etap 2 „Analiza” – animacja procesu (`embed-progress`)

Cel: użytkownik ma widzieć, że coś się dzieje. Zamiast statycznej karty z paskiem:

- **Faktury w okręgu:** trzy małe ikony faktur (22 × 28 px, białe, obramowanie `1.5px #416e19`,
  trzy „linijki tekstu” 2 px: pierwsza krótsza, `#416e19`, dwie pozostałe `#a7c2ad`) krążą
  po przerywanym okręgu (`1.5px dashed #b9d4b0`) wokół limonkowego kółka z błyskawicą
  (`#c8ff3d`, ikona lucide `Zap` w kolorze `#0c2338`). Scena 96 × 96 px, przerywany okrąg
  74 × 74 px, kółko z błyskawicą 34 × 34 px, środki faktur 37 px od środka.
  Jeden obieg 4,5 s, faktury nie obracają się wokół własnej osi.
  Kółko lekko pulsuje (poświata `#c8ff3d`, 1,6 s).
- **Zmieniający się nagłówek** (h2, `font-display text-[22px] font-bold #152b24`, `aria-live="polite"`),
  płynne przejście: stary tekst znika (0,35 s, zanik + przesunięcie 6 px w dół), nowy się pojawia
  (kolejne 0,35 s), kolejno:
  1. „Analizujemy Twoje faktury…”
  2. „Sprawdzamy poziom zużycia…”
  3. „Porównujemy dane…”
  4. „Przygotowujemy wyliczenia…”
- **Lista tych samych czterech kroków** pod nagłówkiem (13 px): gotowe – zielone kółko `#416e19`
  z białym ptaszkiem, bieżący – kręcące się kółko i pogrubiony tekst `#152b24`, kolejne – szare `#8aa093`.
- **Pasek postępu** – rośnie z każdym krokiem, gradient `#78aa39 → #c8ff3d → #78aa39`
  z przesuwającym się połyskiem.
- **Licznik** bez zmian: „Przetworzono X z Y faktur”.
- Na tym etapie chowamy zdanie pod tytułem („Prześlij faktury za prąd…”), żeby wszystko zmieściło
  się w 545 px bez przewijania. Usuwamy „Zwykle zajmuje to do 5 minut. Możesz poczekać tutaj.”
- W makiecie kroki zmieniają się co 1,8 s. **W produkcji najlepiej podpiąć je pod prawdziwe etapy
  przetwarzania** (jeśli backend je zna); jeśli nie – kroki mogą iść czasowo, ale ostatni
  („Przygotowujemy wyliczenia…”) powinien trwać, aż przyjdzie wynik.
- `prefers-reduced-motion: reduce`: krążenie, puls, połysk i kręcące się kółko wyłączone,
  teksty dalej się zmieniają.

---

## 4. Etap 3 „Kontakt” – wynik i formularz

### 4.1. Karta z kwotą (`embed-teaser`) – ma robić wrażenie

- Tło: pełna limonka z gradientem
  `radial-gradient(120% 140% at 0% 0%, #e9ffb3 0%, #c8ff3d 45%, #9fe02a 100%)`,
  `border-radius 14px`, poświata `0 0 0 1px #a6e63155, 0 12px 34px -10px #8fd11f99, 0 0 40px -8px #c8ff3d`.
- Teksty:
  - etykieta (11 px, 800, wersaliki, `letter-spacing .12em`, `#1d3b12`): **„Możesz zaoszczędzić rocznie”**
    (zamiast „Szacowane roczne oszczędności”),
  - kwota: **54 px**, `font-display`, 800, `letter-spacing -.06em`, kolor `#0c2338`, `tabular-nums`,
    format z twardą spacją jako separatorem tysięcy: „4 860 zł”,
  - pod kwotą (14 px, `#1d3b12`): **„to ok. 405 zł mniej na rachunkach co miesiąc”**
    – kwota miesięczna = roczna / 12, zaokrąglona do pełnych złotych, pogrubiona `#0c2338`,
  - przypis (11 px, `#2f4d22`, cienka linia nad nim `#0c233826`):
    „Szacunek na podstawie odczytu faktur. Dokładną analizę przygotuje doradca.”
- Animacje: karta wjeżdża (0,7 s, z dołu 10 px + lekkie powiększenie), kwota **nabija się od 0**
  do wyniku w 1,4 s (ease-out), co 3,2 s przez kartę przelatuje ukośny błysk światła.
  Przy `prefers-reduced-motion` bez animacji, od razu pełna kwota.

### 4.2. Formularz kontaktowy (`embed-contact-form`, `.embed-dark-panel`)

- **Usunąć** etykietę „Bezpłatna konsultacja” (to nie jest konsultacja).
- Nagłówek (h2, 19 px): **„Zamów ofertę dla swojej firmy, aby zacząć oszczędzać.”**
  (zamiast „Chcesz poznać szczegóły? Zostaw kontakt”).
- Pola „Imię i nazwisko”, „E-mail firmowy”, „Telefon”: **tło białe `#ffffff`**, tekst `#152b24`,
  obramowanie białe; fokus: `outline 2px #c8ff3d`, `outline-offset 2px`.
  Walidacja pól i komunikaty błędów bez zmian (makieta ich nie pokazuje).
- Przycisk „Zacznij oszczędzać” bez zmian.
  (Do rozważenia: nagłówek kończy się słowami „aby zacząć oszczędzać”, więc przycisk mógłby
  brzmieć „Zamów ofertę” – decyzja zamawiającego, w makiecie zostaje „Zacznij oszczędzać”.)

### 4.3. Obowiązkowa zgoda pod przyciskiem

Pod przyciskiem „Zacznij oszczędzać” checkbox **wymagany**:

> Wyrażam zgodę na przetwarzanie moich danych w celu przygotowania i przesłania szczegółowej
> oferty. Szczegóły w [Polityce prywatności](https://kalkulatorzuzyciapradu.pl/polityka-prywatnosci)
> i [Regulaminie](https://kalkulatorzuzyciapradu.pl/regulamin). *

- Tekst 12 px, `#d7e0dc`, linki i gwiazdka `#c8ff3d`, linki podkreślone, otwierają się w nowej karcie.
- Checkbox 18 px, `accent-color: #c8ff3d`.
- Bez zaznaczenia formularz się nie wysyła: pod zgodą komunikat (12 px, 700, `#ff8a80`,
  `role="alert"`) **„Zaznacz zgodę, aby otrzymać ofertę.”**, checkbox dostaje obwódkę `2px #ff8a80`
  i fokus. Po zaznaczeniu komunikat znika.
- Zgodę zapiszcie razem z kontaktem (treść, wersja, data, godzina), tak jak dziś zapisujecie zgodę z etapu 1.

### 4.4. Informacja o bezpieczeństwie

Pod zgodą ramka (12 px, `padding 10px 12px`, `rounded 8px`, tło `#c8ff3d0f`,
obramowanie `#c8ff3d33`, tekst `#bdc9cc`) z ikoną tarczy z kłódką (20 px, `#c8ff3d`):

> **Twoje dane są bezpieczne.** Przesyłamy je szyfrowanym połączeniem HTTPS
> i używamy tylko do przygotowania oferty.

(„Twoje dane są bezpieczne.” pogrubione, białe.) Proszę potwierdzić, że zdanie
„używamy tylko do przygotowania oferty” zgadza się z tym, jak przetwarzacie dane – wzięliśmy je
z obecnej klauzuli („…a nie w celu nieograniczonych kampanii marketingowych”).

---

## 5. Bez zmian

- Nagłówek etapu 1 „Rozpocznij darmową analizę” i zdanie pod spodem – z jedną poprawką:
  w „Prześlij faktury za prąd — w kilka minut…” pauzę „—” zamieńcie na półpauzę „–”
  (na stronie używamy wyłącznie półpauzy).
- Nagłówek etapów Wynik i Dziękujemy, jak dziś: „Twoja analiza” + „Wstępny wynik na podstawie przesłanych faktur.”
- Stepper „Faktury / Analiza / Kontakt” (ukończone etapy z ptaszkiem `#416e19`).
- Ekran „Dziękujemy” (tekst z Waszego panelu).
- Ekrany błędów i limitów („Kalkulator chwilowo niedostępny”, „Sesja wygasła”, limity plików).
- Link „Polityka prywatności” w stopce (adres z Waszej konfiguracji, nowa karta). W kopii „przed”
  w makiecie link prowadzi do `#` – to tylko uproszczenie kopii.

## 5a. Zachowania tylko w makiecie – nie wdrażać

- Zamknięcie okna wyboru pliku bez wyboru dodaje przykładową fakturę (żeby dało się przeklikać).
- Pliki nie są sprawdzane (typ, rozmiar, strony, liczba) i nigdzie nie są wysyłane.
- Analiza trwa stałe ok. 7 s, kwota jest na sztywno.
- Formularz kontaktowy sprawdza tylko zgodę; po jej zaznaczeniu od razu pokazuje „Dziękujemy”.
- Wysokość ramki jest symulowana (545 px albo wysokość treści) – obowiązuje pkt 1.
- Przełącznik TEST, parametry `?etap=…` i link „Zacznij od nowa (tylko w makiecie)”.

## 5b. Teksty do potwierdzenia

Do czasu decyzji obowiązują teksty z tej specyfikacji. Do potwierdzenia przez Was lub zamawiającego:
zdanie o zgodzie na etapie 1 (pkt 2.1, prawnik), zdanie „używamy tylko do przygotowania oferty”
(pkt 4.4) i ewentualna zmiana przycisku na „Zamów ofertę” (pkt 4.2).

## 6. Kolejność wdrożenia (propozycja)

1. Wysokość ramki (pkt 1) – bez tego etap Kontakt jest ucięty.
2. Etap 1: zgoda, kolory, przycisk, lista plików (pkt 2).
3. Etap 3: karta z kwotą, formularz, zgoda, bezpieczeństwo (pkt 4).
4. Etap 2: animacja (pkt 3).

Pytania: Ziemowit.
