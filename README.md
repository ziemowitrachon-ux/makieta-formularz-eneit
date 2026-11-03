# Makieta zmian formularza kalkulatora (kalkulatorzuzyciapradu.pl)

Wersja testowa, nie produkcja. Nic tu nie wysyła danych.

- **Podgląd online:** https://ziemowitrachon-ux.github.io/makieta-formularz-eneit/
  (przełącznik **TEST: Teraz / Makieta** w prawym dolnym rogu)
- **Sam formularz:** https://ziemowitrachon-ux.github.io/makieta-formularz-eneit/formularz-makieta.html
  – etapy: `?etap=analiza`, `?etap=wynik`, `?etap=dziekujemy`

## Co gdzie jest

| Plik | Co zawiera |
|---|---|
| [`ZMIANY-FORMULARZA.md`](ZMIANY-FORMULARZA.md) | **Specyfikacja zmian** – teksty, kolory, rozmiary, zachowanie, kod wysyłania wysokości ramki |
| [`formularz-makieta.html`](formularz-makieta.html) | Formularz **po zmianach** – cały HTML, CSS (blok `<style>`, klasy `m-…`) i JS w jednym pliku |
| [`formularz-teraz.html`](formularz-teraz.html) | Kopia obecnego formularza (ekran startowy) do porównania |
| `index.html` | Kopia strony głównej z ramką kalkulatora (tylko do podglądu) |
| `css/`, `media/` | Skompilowany CSS i czcionki z obecnego formularza |

Różnice najwygodniej zobaczyć, porównując `formularz-teraz.html` z `formularz-makieta.html`
oraz czytając `ZMIANY-FORMULARZA.md` punkt po punkcie.

Lokalnie: `npx serve .` albo `python -m http.server` w tym katalogu.
