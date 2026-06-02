# Záznam session – vývoj simulácie Rubikovej kocky

Tento dokument zhŕňa celý priebeh vývojovej session: čo sa riešilo, aké
rozhodnutia padli a v akom poradí. Slúži ako história projektu.

Branch: `claude/sweet-allen-FWJhk`

---

## 1. Zadanie

Vytvoriť simuláciu Rubikovej kocky:

- na začiatku **postavená** kocka,
- tlačidlom sa **rozloží `X` náhodnými ťahmi**,
- ďalším tlačidlom ju **solver vyrieši**,
- má to byť **HTML stránka s JavaScriptom**, kocka **plne animovaná**,
- najprv napísať `docs/PRG.md`,
- do ďalšej verzie ovládanie myšou (klik na hranu → pootočenie vrstvy).

---

## 2. Dokumentácia (pred kódom)

1. **`docs/PRG.md`** – programová dokumentácia / špecifikácia: účel, použité
   technológie (three.js cez CDN), dátový model (27 cubies), notácia ťahov,
   mechanizmus animácie vrstiev (pivot `attach`/`detach`), miešanie a solver,
   UI, štruktúra súborov, plánované rozšírenia. Commitnuté a pushnuté.
2. **`CLAUDE.md`** – pokyny pre prácu v repozitári, odvodené z PRG.md
   (zhrnutie projektu, spustenie, konvencie – slovenčina, jeden súbor,
   zaokrúhľovanie pozícií, blokovanie tlačidiel počas animácie).
3. **`docs/PLAN.md`** – implementačný plán rozdelený na 9 samostatne
   otestovateľných krokov (kostra + Pages → 3D scéna → kocka → engine ťahov →
   animácia + fronta → scramble → solver → reset/UI → overenie „jeden súbor
   z GitHubu“). Pri každom kroku popis, výsledok a testy + invarianty.

Dôležité zistenie zachytené v pláne: `raw.githubusercontent.com` servíruje
HTML ako `text/plain`, takže JS nespustí → na „otvorenie priamo z GitHubu“
treba **GitHub Pages** (prípadne `raw.githack`).

---

## 3. Implementácia aplikácie (`index.html`)

Celá appka v jednom súbore, bez build kroku, závislosti len z CDN
(`importmap` → three.js + OrbitControls + RoundedBoxGeometry).

Postavené naraz (nie krok po kroku – pôvodný zámer používateľa bol len prvý
krok, ale výsledok bol akceptovaný):

- 3D scéna: `Scene`, `PerspectiveCamera`, `WebGLRenderer`, svetlá,
  `OrbitControls`, render-loop, `resize`.
- Kocka = 27 cubies na pozíciách `(x,y,z) ∈ {-1,0,1}`.
- Engine ťahov: `parseToken` (`U`, `R'`, `F2`…) → `{axis, layer, angle,
  turns}`; otočenie vrstvy cez pomocný pivot (`pivot.attach` → animácia →
  `scene.attach` → zaokrúhlenie pozícií).
- Animácia + fronta ťahov (`requestAnimationFrame`, easing), blokovanie
  tlačidiel počas behu (`isBusy`).
- **Zamiešať**: `X` náhodných ťahov (bez opakovania steny za sebou), uloženie
  postupnosti.
- **Vyriešiť**: solver = spätné prehranie zamiešania (obrátené poradie +
  invertované ťahy) – samostatná funkcia, neskôr nahraditeľná Kociembom.
- **Reset**, stavový text, kontrola `isSolved()`.
- Ladiace API v konzole: `window.rubik.applyMove(...)`, `isSolved()`,
  `cubies`.

---

## 4. Ovládanie myšou – iterácie

Funkcia sa vyvíjala v niekoľkých krokoch podľa spätnej väzby:

1. **Ťah na kocku = pootočenie vrstvy** (raycasting, smer ťahu → os a smer).
   Počas ťahu na kocku vypnuté OrbitControls; ťah mimo kocky otáčal pohľad.
2. **Klik namiesto ťahu** – požiadavka: „stačí kliknúť, netreba hýbať myšou“.
   Klik otočil vrstvu „smerom od kurzora“ (vektor klik → stred kocky).
   Ťahanie ostalo na otáčanie pohľadu.
3. **Klik podľa miesta na malej kocke** – požiadavka: prehodiť znamienko a
   rozlišovať, do ktorej časti malej kocky sa klikne:
   - použije sa **presný bod dopadu** raycastu,
   - odchýlka od stredu malej kocky v rovine steny → dominantná dotyková os,
   - klik bližšie k vodorovnej hrane → **vertikálny slice**, bližšie k zvislej
     hrane → **horizontálny slice**,
   - os otáčania = `normála × smer kliknutia`,
   - smer otáčania v konštante `TURN_SIGN` (ľahké prehodenie).

---

## 5. Vzhľad kocky

Na základe referenčného obrázka/videa upravený vzhľad na „klasickú“ kocku:

- telo cubie = `RoundedBoxGeometry` (čierny plast so zaoblenými hranami),
- nálepky = samostatné dlaždice so zaoblenými rohmi (`Shape` →
  `ShapeGeometry`) tesne nad stenou → vystupujúca čierna mriežka,
- jasnejšia, klasická farebná paleta,
- key + fill + ambient svetlo pre rovnomerné, sýte farby,
- `isSolved()` upravené, aby čítalo farby nálepiek; normála kliknutia
  „prichytená“ na dominantnú os (kvôli zaobleným hranám).

---

## 6. Publikovanie (GitHub Pages)

- Cieľ: jediný `index.html` otvoriteľný priamo z GitHubu.
- Na testovanie počas vývoja sa používal `raw.githack.com` (s upozornením na
  CDN cache – treba hard refresh alebo `?v=` parameter).
- Po zapnutí Pages fungoval `…github.io/rubik_app/index.html`, ale koreň
  `…github.io/rubik_app/` hádzal **404** – zakešovaná 404 odpoveď z času,
  keď v koreni ešte nebol `index.html`.
- Riešenie: pridaný **`.nojekyll`** do koreňa → vypnutie Jekyll spracovania a
  vynútenie čistého re-deployu (po builde koreň `/` opäť servíruje
  `index.html`).

---

## 7. Výsledná štruktúra súborov

```
rubik_app/
├── index.html        # celá aplikácia (HTML + CSS + JS)
├── CLAUDE.md         # pokyny pre prácu v repozitári
├── .nojekyll         # vypnutie Jekyll na GitHub Pages
└── docs/
    ├── PRG.md        # programová dokumentácia / špecifikácia
    ├── PLAN.md       # implementačný plán (kroky + testy)
    └── session.md    # tento záznam
```

---

## 8. Poznámky / možné ďalšie kroky

- Smer otáčania pri kliknutí sa dá prehodiť konštantou `TURN_SIGN`.
- Solver je zatiaľ spätné prehranie – do budúcna algoritmický solver
  (Kociemba), história ťahov (späť/vpred), uloženie stavu, timer a počítadlo
  ťahov (viď `docs/PRG.md`, sekcia plánované rozšírenia).
- Pri testovaní cez raw.githack pozor na CDN cache (hard refresh / `?v=`).
