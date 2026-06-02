# Implementačný plán – Rubikova kocka

Cieľ: **jediný súbor `index.html`** (HTML + CSS + JS pohromade), ktorý sa dá
otvoriť v prehliadači **priamo z GitHubu** cez GitHub Pages – bez build kroku,
bez lokálnych závislostí (three.js sa ťahá z CDN cez `importmap`).

Postupuje sa po krokoch. **Každý krok je samostatne otestovateľný** – po jeho
dokončení sa dá overiť výsledok skôr, než sa pokračuje ďalej. Po každom kroku
commit + push.

---

## Krok 0 – Kostra a publikovanie

**Práca:** Vytvor `index.html` so základným HTML skeletom (`<head>`, titulok,
prázdny `<div id="scene">`, panel s tlačidlami ako statické UI). Zapni
**GitHub Pages** (Settings → Pages → branch).

**Výsledok:** Jeden súbor sa dá otvoriť aj lokálne, aj cez URL GitHub Pages.

**Test:**
- Lokálne otvorenie `index.html` → zobrazí sa titulok a UI panel.
- Otvorenie GitHub Pages URL v prehliadači → tá istá stránka, bez 404.

> Pozn.: `raw.githubusercontent.com` servíruje HTML ako text/plain a JS
> nespustí. Pre „otvorenie priamo z GitHubu“ je potrebné **GitHub Pages**
> (alebo služba typu `htmlpreview`/`raw.githack`).

---

## Krok 1 – 3D scéna (bootstrap three.js)

**Práca:** Cez `<script type="importmap">` načítaj `three` a `OrbitControls`
z CDN. Vytvor `Scene`, `PerspectiveCamera`, `WebGLRenderer`, svetlo,
`OrbitControls`, render-loop (`requestAnimationFrame`) a obsluhu `resize`.

**Výsledok:** Prázdna 3D scéna, ktorá sa dá otáčať myšou a zoomovať.

**Test:**
- Stránka ukáže 3D priestor (napr. dočasná pomocná os/`GridHelper`).
- Ťahanie myšou otáča pohľad, koliesko zoomuje.
- Zmena veľkosti okna nedeformuje obraz (správny aspect ratio).

---

## Krok 2 – Postavená kocka (27 cubies)

**Práca:** Vygeneruj 27 `THREE.Mesh` (`BoxGeometry`) na pozíciách
`(x,y,z) ∈ {-1,0,1}`. Každá má 6 materiálov; povrchové steny dostanú farbu
podľa schémy (U biela, D žltá, F zelená, B modrá, R červená, L oranžová),
vnútorné čierne. Malé medzery medzi cubies.

**Výsledok:** Vizuálne korektná vyriešená kocka.

**Test:**
- Každá z 6 stien má jednu farbu (9 rovnakých štvorčekov).
- Otáčaním kamery skontroluj všetkých 6 stien + čierne škáry.
- V konzole `cubies.length === 27`.

---

## Krok 3 – Engine ťahov (bez animácie)

**Práca:** Implementuj `applyMove(token)` (napr. `"U"`, `"R'"`, `"F2"`):
pivot `Group` → `pivot.attach()` 9 cubies danej vrstvy → otoč o cieľový uhol
→ `scene.attach()` späť → **zaokrúhli pozície** na celé čísla. Mapovanie
ťah → `{os, vrstva, uhol}`.

**Výsledok:** Okamžité (neanimované) korektné otočenie ľubovoľnej vrstvy.

**Test (z konzoly):**
- `applyMove("R")` otočí pravú vrstvu; farby sa premiestnia správne.
- `applyMove("R"); applyMove("R'")` → kocka je opäť vyriešená.
- `applyMove("U2"); applyMove("U2")` → vyriešená.
- Po sérii ťahov sú všetky pozície celé čísla (žiadny drift).

---

## Krok 4 – Animácia + fronta ťahov

**Práca:** Otáčanie pivotu animuj v render-loope (interpolácia uhla s
easing, napr. ~250 ms). Sprav `moveQueue` a stavový `isBusy` – ťahy sa
prehrávajú postupne. Po dobehnutí ťahu bake (ako v kroku 3).

**Výsledok:** Plynulé, postupné animované otáčania.

**Test:**
- Naplň frontu (`["U","R","F"]`) → prehrá sa plynule, jeden ťah po druhom.
- Počas animácie sa ďalší ťah nezačne skôr, než dobehne predošlý.
- Po animácii sú pozície zaokrúhlené (rovnaký invariant ako krok 3).

---

## Krok 5 – Zamiešanie (scramble)

**Práca:** UI: pole „Počet ťahov X“ + tlačidlo **„Zamiešať“**. Vygeneruj `X`
náhodných ťahov (bez opakovania tej istej steny za sebou), **ulož** zoznam,
zaraď do fronty. Počas behu blokuj tlačidlá.

**Výsledok:** Kocka sa animovane rozloží zadaným počtom ťahov.

**Test:**
- `X = 20` → prehrá sa 20 animovaných ťahov, kocka je rozhádzaná.
- Vygenerovaná postupnosť sa zobrazí/zaloguje a nemá dve rovnaké steny
  bezprostredne za sebou.
- Tlačidlá sú počas miešania zablokované.

---

## Krok 6 – Solver (spätné prehranie)

**Práca:** Tlačidlo **„Vyriešiť“**: vezmi uložené zamiešanie, obráť poradie
a invertuj každý ťah (`U`↔`U'`, `U2`↔`U2`), zaraď do fronty. Solver drž ako
samostatnú funkciu/modul.

**Výsledok:** Po stlačení sa kocka animovane zloží do vyriešeného stavu.

**Test:**
- Zamiešať → Vyriešiť → kocka je vizuálne vyriešená.
- Programová kontrola: funkcia `isSolved()` vráti `true` po doriešení.
- Funguje opakovane (zamiešať/vyriešiť viackrát za sebou).

---

## Krok 7 – Reset, stav UI, dokončenie

**Práca:** Tlačidlo **„Reset“** (okamžité postavenie kocky). Stavový text
(fáza: miešam / riešim / hotovo, počet ťahov). Finálny vzhľad, odstránenie
pomocných prvkov z kroku 1.

**Výsledok:** Uzavretý použiteľný UX.

**Test:**
- „Reset“ kedykoľvek vráti vyriešenú kocku, vyčistí uložené zamiešanie.
- „Vyriešiť“ je dostupné len keď existuje zamiešanie.
- Stavový text zodpovedá realite.

---

## Krok 8 – Overenie „jeden súbor z GitHubu“

**Práca:** Skontroluj, že `index.html` nemá **žiadne lokálne** závislosti
(len CDN). Over na GitHub Pages URL.

**Výsledok:** Celá aplikácia funguje z jednej verejnej URL.

**Test:**
- Otvorenie GitHub Pages URL na čistom profile/inom zariadení → kocka beží,
  zamiešanie aj riešenie fungujú.
- View-Source ukáže jediný súbor; žiadne `src="./..."` na lokálne JS/CSS.

---

## Invarianty (platia v každom kroku)

- Po každom otočení vrstvy sú pozície cubies **celé čísla** (žiadny drift).
- Počas animácie sú **tlačidlá zablokované**.
- Kód, UI texty aj komentáre sú v **slovenčine**.
- Všetko zostáva v **jednom súbore** `index.html`.
