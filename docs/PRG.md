# Rubikova kocka – simulácia (programová dokumentácia)

## 1. Účel programu

Aplikácia je interaktívna 3D simulácia Rubikovej kocky (3×3×3) bežiaca
priamo v prehliadači. Je napísaná ako jedna HTML stránka s JavaScriptom,
bez nutnosti inštalácie čohokoľvek – stačí otvoriť súbor `index.html`.

Základný scenár použitia:

1. Na začiatku je kocka **postavená** (vyriešená).
2. Po stlačení tlačidla **„Zamiešať“** sa kocka rozloží zadaným počtom
   náhodných ťahov `X` (počet je možné nastaviť).
3. Po stlačení tlačidla **„Vyriešiť“** ju **solver** automaticky zloží
   späť do vyriešeného stavu.

Všetky pohyby vrstiev sú **plne animované** – otáčanie vrstiev prebieha
plynule s časovaním a zjemnením (easing).

## 2. Použité technológie

| Vrstva            | Technológia                                   |
|-------------------|-----------------------------------------------|
| Štruktúra stránky | HTML5                                         |
| Vzhľad / UI       | CSS3                                          |
| Logika a render   | JavaScript (ES moduly)                         |
| 3D grafika        | [three.js](https://threejs.org) (cez CDN)     |
| Ovládanie kamery  | OrbitControls (three.js addon)                |

> three.js sa načítava cez CDN pomocou `<script type="importmap">`.
> Na spustenie je preto pri prvom načítaní potrebné pripojenie na internet.

## 3. Architektúra a dátový model

### 3.1 Kocka ako sada „cubies“

Kocka sa skladá z **27 malých kociek** (tzv. *cubies*) usporiadaných do
mriežky 3×3×3. Každá malá kocka má súradnice `(x, y, z)`, kde každá zložka
nadobúda hodnoty `-1`, `0`, `1`.

Každá malá kocka je objekt `THREE.Mesh` (`BoxGeometry`) so šiestimi
materiálmi – jeden pre každú stenu. Steny, ktoré sú na povrchu veľkej
kocky, dostanú farbu podľa štandardnej schémy, vnútorné steny sú čierne.

Štandardná farebná schéma:

| Stena      | Os    | Farba    |
|------------|-------|----------|
| Hore (U)   | `+Y`  | biela    |
| Dole (D)   | `-Y`  | žltá     |
| Vpredu (F) | `+Z`  | zelená   |
| Vzadu (B)  | `-Z`  | modrá    |
| Vpravo (R) | `+X`  | červená  |
| Vľavo (L)  | `-X`  | oranžová |

### 3.2 Reprezentácia stavu

Stav kocky je daný priamo polohou a orientáciou jednotlivých cubies v
scéne. Po každom otočení vrstvy sa pozície zaokrúhlia na celé čísla, aby
nedochádzalo k hromadeniu numerickej chyby (drift) pri mnohých ťahoch.

### 3.3 Ťahy (moves)

Používa sa štandardná notácia Rubikovej kocky:

```
U  D  L  R  F  B            – otočenie steny o 90° (jeden smer)
U' D' L' R' F' B'           – otočenie o 90° v opačnom smere (prime)
U2 D2 L2 R2 F2 B2           – otočenie o 180°
```

Každý ťah je interne popísaný osou otáčania (`x`/`y`/`z`), hodnotou
vrstvy (`-1`/`+1`) a uhlom (`±90°` alebo `180°`).

### 3.4 Animácia otočenia vrstvy

Otočenie jednej vrstvy prebieha takto:

1. Vytvorí sa pomocný objekt **pivot** (`THREE.Group`) v strede kocky.
2. Všetkých 9 cubies danej vrstvy sa doň „pripne“ pomocou `pivot.attach()`
   (zachová sa ich svetová transformácia).
3. Pivot sa plynule otočí o cieľový uhol (animované cez `requestAnimationFrame`
   s easing funkciou).
4. Po dokončení sa cubies vrátia späť do scény (`scene.attach()`), čím sa
   ich nová poloha aj orientácia natrvalo zapíše. Pozície sa zaokrúhlia.

## 4. Miešanie a riešenie

### 4.1 Miešanie (scramble)

Pri zamiešaní sa vygeneruje `X` náhodných ťahov (parameter z UI). Pri
generovaní sa vyhýbame opakovaniu tej istej steny dvakrát za sebou, aby
miešanie bolo zmysluplné. Zoznam použitých ťahov sa **uloží**.

### 4.2 Riešenie (solver)

V aktuálnej verzii solver pracuje **spätným prehraním** uloženej
postupnosti miešania:

```
riešenie = obrátené poradie(zamiešanie), každý ťah invertovaný
napr. (U  R' F2)  ->  (F2  R  U')
```

Tento prístup je 100 % spoľahlivý a vždy vráti kocku do vyriešeného
stavu. Je oddelený do samostatného modulu, takže ho je možné v budúcnosti
nahradiť plnohodnotným algoritmickým solverom (napr. Kociembov
dvojfázový algoritmus) bez zásahu do zvyšku aplikácie.

## 5. Používateľské rozhranie

| Prvok                | Funkcia                                                   |
|----------------------|-----------------------------------------------------------|
| Pole „Počet ťahov“   | Nastavenie `X` – počtu miešacích ťahov.                   |
| Tlačidlo „Zamiešať“  | Zamieša kocku `X` náhodnými ťahmi (animovane).            |
| Tlačidlo „Vyriešiť“  | Spustí solver, ktorý kocku zloží (animovane).             |
| Tlačidlo „Reset“     | Okamžite vráti kocku do vyriešeného stavu.                |
| Klik na stenu kocky  | Pootočí danú vrstvu „smerom od kurzora“ (stačí kliknúť).  |
| Ťah myšou            | Otáčanie pohľadu kamery okolo kocky (OrbitControls).      |
| Koliesko myši        | Priblíženie / oddialenie.                                 |

Počas prebiehajúcej animácie sú tlačidlá zablokované, aby nedošlo k
prekrývaniu ťahov.

## 6. Štruktúra súborov

```
rubik_app/
├── index.html        # celá aplikácia (HTML + CSS + JS)
└── docs/
    └── PRG.md         # tento dokument
```

## 7. Ovládanie kocky myšou (implementované)

**Kliknutím** na ktorúkoľvek stenu kocky sa príslušná vrstva pootočí o 90°
– stačí kliknúť, netreba ťahať. Funguje takto:

1. Pri kliknutí sa cez `THREE.Raycaster` zistí, na ktorú malú kocku a stenu
   používateľ klikol (vrátane **presného bodu** dopadu); svetová normála
   steny sa prichytí na dominantnú os (kvôli zaobleným hranám).
2. Z polohy kliknutia **v rámci malej kocky** (odchýlka od jej stredu v
   rovine steny) sa určí dominantná dotyková os – teda či používateľ klikol
   bližšie k vodorovnej alebo zvislej hrane. Tým sa rozhodne, či sa otočí
   **horizontálny alebo vertikálny slice**, a v ktorom smere.
3. Os otáčania = `normála × smer kliknutia`; vrstva je daná súradnicou
   klinutej malej kocky pozdĺž tejto osi.
4. Použije sa **existujúci animačný mechanizmus** otáčania vrstiev (rovnaký
   ako pri miešaní/riešení).

Rozlišuje sa klik vs. ťah podľa prejdenej vzdialenosti: krátky klik otočí
vrstvu, ťah (pohyb myši) necháva otáčať pohľad kamery (OrbitControls).
Smer otáčania je v konštante `TURN_SIGN` (ľahké prehodenie).

## 8. Plánované rozšírenia (ďalšia verzia)

- Plnohodnotný algoritmický solver (Kociemba) so zobrazením krokov.
- História ťahov + krok späť / vpred.
- Uloženie a načítanie stavu kocky.
- Meranie času (timer) a počítadlo ťahov pre režim skladania.
