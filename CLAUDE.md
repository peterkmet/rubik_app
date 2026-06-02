# CLAUDE.md

Pokyny pre Claude Code pri práci v tomto repozitári. Detailná špecifikácia
projektu je v [`docs/PRG.md`](docs/PRG.md) – pri rozporoch má prednosť PRG.md.

## Čo je to za projekt

Interaktívna 3D simulácia Rubikovej kocky (3×3×3) bežiaca v prehliadači.
Jedna HTML stránka s JavaScriptom, **bez build kroku** – spúšťa sa otvorením
`index.html` v prehliadači.

Hlavný scenár: kocka je na začiatku vyriešená → tlačidlo **„Zamiešať“** ju
rozloží `X` náhodnými ťahmi → tlačidlo **„Vyriešiť“** ju solverom zloží späť.
Všetky otočenia vrstiev sú **plne animované**.

## Spustenie

Žiadna inštalácia ani build. Otvor `index.html` v prehliadači.
Pri prvom načítaní je potrebné pripojenie na internet (three.js z CDN).
Pre lokálny vývoj prípadne: `python3 -m http.server` a otvor `localhost:8000`.

## Technológie

- HTML5 + CSS3 + JavaScript (ES moduly)
- 3D grafika: **three.js** cez CDN (`<script type="importmap">`)
- Kamera: OrbitControls (three.js addon)
- Žiadne npm závislosti, žiadny bundler, žiadny transpiler.

## Architektúra (zhrnutie z PRG.md)

- Kocka = **27 cubies** (mriežka 3×3×3), súradnice `(x,y,z)` ∈ `{-1,0,1}`.
- Každá cubie je `THREE.Mesh` (`BoxGeometry`) so 6 materiálmi; povrchové
  steny majú farbu podľa schémy, vnútorné sú čierne.
- Farby: U=`+Y` biela, D=`-Y` žltá, F=`+Z` zelená, B=`-Z` modrá,
  R=`+X` červená, L=`-X` oranžová.
- **Stav** kocky = poloha a orientácia cubies v scéne. Po každom ťahu sa
  pozície **zaokrúhlia na celé čísla** (zabránenie numerickému driftu).
- **Ťahy** v štandardnej notácii (`U D L R F B`, prime `'`, dvojité `2`),
  interne `{os, vrstva, uhol}`.
- **Animácia vrstvy**: pomocný `THREE.Group` pivot → `pivot.attach()` 9
  cubies → plynulé otočenie (rAF + easing) → `scene.attach()` späť →
  zaokrúhlenie pozícií.
- **Solver** rieši **spätným prehraním** uloženého zamiešania (obrátené
  poradie + invertované ťahy). Drž ho ako samostatný modul, aby sa dal
  neskôr nahradiť algoritmickým solverom (Kociemba).

## Konvencie

- Kód aj UI texty a komentáre sú v **slovenčine** (zachovaj tento štýl).
- Drž všetko v `index.html`, pokiaľ nie je dôvod štiepiť na súbory.
- Po každom otáčaní vrstvy **zaokrúhľuj pozície** cubies.
- Počas prebiehajúcej animácie **blokuj tlačidlá**, aby sa ťahy neprekrývali.
- Animácie rob cez `requestAnimationFrame` s easing, nie skokovo.

## Štruktúra súborov

```
rubik_app/
├── index.html      # celá aplikácia (HTML + CSS + JS)
├── CLAUDE.md       # tento súbor
└── docs/
    └── PRG.md      # programová dokumentácia / špecifikácia
```

## Plánované (ďalšia verzia)

Ovládanie kocky myšou – klik/ťah na hranu pootočí vrstvu (raycasting cez
`THREE.Raycaster`, smer ťahu → os a smer otočenia, znovupoužitie animácie).
Ďalej: algoritmický solver, história ťahov (späť/vpred), uloženie stavu,
timer a počítadlo ťahov.
