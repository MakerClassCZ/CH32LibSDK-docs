# Příklady herních technik z reálných her

## Úvod

Tento dokument obsahuje konkrétní techniky a vzory extrahované z her **ch32v002-Fifteen** a **ch32v002-Train**, které byly vytvořeny autorem frameworku. Tyto příklady ukazují zamýšlený styl práce s CH32LibSDK a osvědčené postupy.

## CH32V002-FIFTEEN - Puzzle hra

Fifteen je implementace klasické puzzle hry 15 (posuvné kostky v mřížce 4×4).

**Lokace:** `/home/user/ch32-kit/ch32libsdk/Rvpc/Games/ch32v002-Fifteen/src/main.c`

### 1. Tile-based rendering s box-drawing znaky

Hra používá speciální znaky pro vytvoření vizuálního efektu 3D dílků:

```c
#define TILEW  4    // Šířka dílce v znacích
#define TILEH  3    // Výška dílce v řádcích

void DrawBoard() {
    for (i = 0; i < TILESYNUM; i++) {
        for (j = 0; j < TILESXNUM; j++) {
            x = j*TILEW + TILEX;
            y = i*TILEH + TILEY;
            b = Board[i*TILESXNUM + j];

            if (b == TILE_EMPTY) {
                // Prázdné pole = černé znaky
                PrintCharAt(1, x, y, COL_BLACK);
                // ... 4×3 černé znaky
            } else {
                // Rám dílku s číslem
                PrintCharAt(156, x, y, COL_BLACK);     // ╔ horní levý roh
                PrintCharAt(149, x+1, y, COL_BLACK);   // ═ horní hrana
                PrintCharAt(153, x+3, y, COL_BLACK);   // ╗ horní pravý roh

                PrintCharAt(154, x, y+1, COL_BLACK);   // ║ levá hrana
                // Číslo uprostřed
                PrintCharAt((b<9) ? (' '+128) : ('1'+128), x+1, y+1, COL_BLACK);
                PrintCharAt(((b+1) % 10)+'0'+128, x+2, y+1, COL_BLACK);
                PrintCharAt(154, x+3, y+1, COL_BLACK); // ║ pravá hrana

                PrintCharAt(150, x, y+2, COL_BLACK);   // ╚ dolní levý roh
                PrintCharAt(149, x+1, y+2, COL_BLACK); // ═ dolní hrana
                PrintCharAt(147, x+3, y+2, COL_BLACK); // ╝ dolní pravý roh
            }
        }
    }
}
```

**Výhody:**
- Využití box-drawing znaků pro estetický vzhled
- Minimální paměť (žádné sprite bitmapy)
- Rychlé renderování

### 2. 1D pole místo 2D mřížky

```c
u8 Board[TILESNUM];  // 16 elementů = 4×4 mřížka
u8 Pos[TILESNUM];    // Lookup pozice každého dílku

// Převod index ↔ souřadnice
int index = y*TILESXNUM + x;  // (x,y) → index
int x = index & 3;             // index → X (bitová operace místo %)
int y = index >> 2;            // index → Y (shift místo /)

// Inline funkce pro souřadnice díry
INLINE u8 HoleX() { return Hole & 3; }
INLINE u8 HoleY() { return Hole >> 2; }
```

**Výhody:**
- Úspora paměti
- Rychlejší lineární procházení
- Bitové operace místo násobení/dělení

### 3. Inkrementální aktualizace metrik

Místo přepočítávání celé distance metriky po každém tahu:

```c
void Shift(s8 shift)  // shift: -1, +1, -4, +4
{
    u8 oldinx = Hole;
    u8 newinx = oldinx + shift;
    u8 b = Board[oldinx + shift];

    // PŘED posunem - vypočti starou vzdálenost dílku
    int old_x = (oldinx & 3) - (b & 3);
    int old_y = (oldinx >> 2) - (b >> 2);
    if (Solved[b]) {
        old_x *= 4;
        old_y *= 4;
    }
    int old_dist = old_x*old_x + old_y*old_y;

    // POSUN
    Board[oldinx] = b;
    Board[newinx] = TILE_EMPTY;
    Hole = newinx;

    // PO posunu - vypočti novou vzdálenost
    int new_x = (newinx & 3) - (b & 3);
    int new_y = (newinx >> 2) - (b >> 2);
    if (Solved[b]) {
        new_x *= 4;
        new_y *= 4;
    }
    int new_dist = new_x*new_x + new_y*new_y;

    // Inkrementální update
    Dist -= old_dist;
    Dist += new_dist;

    // Update PosErr
    if ((b == newinx) && Solved[b]) PosErr++;
    if ((b == oldinx) && Solved[b]) PosErr--;
}
```

**Výhoda:** O(1) místo O(n) přepočítání

### 4. Depth-First Search s pruningem

AI solver používá DFS s adaptivní hloubkou:

```c
Bool Solve1()  // Vrací True = nalezeno řešení
{
    if (PosErr == 0) return True;  // Goal test

    int movesnum = MovesNum;
    if (movesnum >= DepthMax) return False;  // Pruning: max. hloubka
    MovesNum = movesnum+1;

    // Vyzkoušej pohyb vlevo
    if ((HoleX() > 0) && !Locked[Board[Hole - 1]]) {
        Shift(-1);
        Moves[movesnum] = -1;
        Bool res = Solve1();  // Rekurze
        Shift(1);             // Backtrack
        if (res) return True;
    }

    // Vyzkoušej pohyb vpravo
    if ((HoleX() < TILESXNUM-1) && !Locked[Board[Hole + 1]]) {
        Shift(1);
        Moves[movesnum] = 1;
        Bool res = Solve1();
        Shift(-1);
        if (res) return True;
    }

    // Vyzkoušej pohyb nahoru
    if ((HoleY() > 0) && !Locked[Board[Hole - TILESXNUM]]) {
        Shift(-TILESXNUM);
        Moves[movesnum] = -TILESXNUM;
        Bool res = Solve1();
        Shift(TILESXNUM);
        if (res) return True;
    }

    // Vyzkoušej pohyb dolů
    if ((HoleY() < TILESYNUM-1) && !Locked[Board[Hole + TILESXNUM]]) {
        Shift(TILESXNUM);
        Moves[movesnum] = TILESXNUM;
        Bool res = Solve1();
        Shift(-TILESXNUM);
        if (res) return True;
    }

    MovesNum = movesnum;
    return False;
}
```

### 5. Hybridní solver s adaptivní hloubkou

```c
// Dvoustupňový přístup:
// 1. PreSolve1(): Heurističtější, rychlejší, menší hloubka (7)
// 2. Solve1(): Exaktní, pomalejší, větší hloubka (17)

int faster1 = 7;   // Ovládá PREDEPTH_MAX
int faster2 = 10;  // Ovládá DEPTH_MAX

void Solver() {
    // ... výpočet ...

    if (!ok && !Check()) {
        // Řešitel selhal, zkus větší hloubkou
        if (faster1 > 0) faster1--;
    }

    if (BestMovesNum > 0) {
        // Našli jsme řešení, zrychli
        if (faster2 < 10) faster2++;
    }
}
```

**Výhoda:** Balancuje rychlost vs optimalitu

### 6. Animace čekání během výpočtu

```c
void WaitOn()  { PrintCharAt(8, x+1, y+1, COL_BLACK); }  // Animovaný znak 8
void WaitOn2() { PrintCharAt(9, x+1, y+1, COL_BLACK); }  // Animovaný znak 9
void WaitOff() { PrintCharAt(1, x+1, y+1, COL_BLACK); }  // Smazání

// V solver loop:
for (int step = 0; step < steps; step++) {
    if ((step & 1) == 0) WaitOn(); else WaitOn2();  // Blikání
    // ... výpočet ...
}
WaitOff();
```

### 7. Shuffle algoritmus - garantovaná řešitelnost

```c
void Shuffle()
{
    // 5000 náhodných pohybů z vyřešeného stavu
    for (i = 5000; i > 0; i--) {
        r = RandU8();

        if (r >= 0x80) {  // Horní bit určuje horizontální/vertikální
            if ((r & 1) == 0) {
                if (HoleX() > 0) Shift(-1);
            } else {
                if (HoleX() < TILESXNUM-1) Shift(1);
            }
        } else {
            if ((r & 1) == 0) {
                if (HoleY() > 0) Shift(-TILESXNUM);
            } else {
                if (HoleY() < TILESYNUM-1) Shift(TILESXNUM);
            }
        }

        // Vizuální feedback
        if ((i & 0x7f) == 0) {  // Každých 128 iterací
            DrawBoard();
            WaitMs(15);
            PlayTone(5000);
        }
    }
}
```

**Proč:** Zaručuje, že puzzle je řešitelné

### 8. Zvuková zpětná vazba

```c
const sMelodyNote MoveSound[] = { { 1, NOTE_C6 }, { 0, 0 } };
const sMelodyNote BumpSound[] = { { 4, NOTE_C3 }, { 0, 0 } };

// V game loop:
switch(JoyGet()) {
    case KEY_RIGHT:
        if (HoleX() > 0) {
            Shift(-1);
            PlayMelody(MoveSound);  // Validní pohyb
            DrawBoard();
        } else {
            PlayMelody(BumpSound);  // Kolize se stěnou
        }
        break;
}
```

## CH32V002-TRAIN - Snake-like hra

Train je hra, kde hráč ovládá vlak sbírající itemy v tile-based bludišti.

**Lokace:** `/home/user/ch32-kit/ch32libsdk/Rvpc/Games/ch32v002-Train/src/main.c`

### 1. Tile-based rendering s texture atlasem

```c
#define TILESIZE    8           // Pixel size jednoho tile
#define MAPW        20          // Šířka mapy v tiles
#define MAPH        12          // Výška mapy v tiles

// Lookup tabulka souřadnic v atlasu
const u16 TileXY[TILESNUM*2] = {
    0*WH, 0*WH,   // Tile 0 na (0,0)
    1*WH, 0*WH,   // Tile 1 na (8,0)
    2*WH, 0*WH,   // Tile 2 na (16,0)
    // ... 50 tiles
};

// Zobrazení jednoho tile
void DispTile(u8 x, u8 y)
{
    u8 tile = Board[x + y*MAPW];
    DrawImgInvFast(ImgTiles,
        x*TILESIZE, y*TILESIZE + MAPY,
        TileXY[tile*2], TileXY[tile*2+1],
        TILESIZE, TILESIZE, 20);
}
```

**Výhody:**
- Všechny tiles v jednom obrázku = efektivní paměť
- LUT tabulka = flexibilní umístění v atlasu
- DrawImgInvFast = optimalizovaná funkce

### 2. Komprese level dat (4-bit packing)

```c
#define TB(tile1, tile2) ((tile1) | ((tile2)<<4))
// Jeden byte = dva tiles (4 bity na tile)

// Level data
const u8 Level1[] = {
    TB(WALL, WALL), TB(WALL, WALL), TB(WALL, WALL), ...
    TB(WALL, EMPTY), TB(EMPTY, EMPTY), TB(ITEM, EMPTY), ...
    // ...
};

// Dekomprese při načítání
void LoadLevel(const u8* level) {
    for (int i = 0; i < MAPSIZE/2; i++) {
        Board[i*2] = level[i] & 0x0F;       // Dolní 4 bity
        Board[i*2+1] = level[i] >> 4;       // Horní 4 bity
    }
}
```

**Výhoda:** -50% velikost level dat (50 levelů × 240 tiles = 6000 bajtů místo 12000)

### 3. Chain-following systém pro vagony

```c
u8 Board[MAPSIZE];   // Co se zobrazuje
u8 Dir[MAPSIZE];     // Kterou stranou jel vlak

void StepLevel()
{
    // 1. Pohyb hlavy vlaku
    s8 x = HeadX, y = HeadY;
    u8 d = CurDir;
    if (d == DIR_L) x--;
    else if (d == DIR_U) y--;
    else if (d == DIR_R) x++;
    else if (d == DIR_D) y++;

    // 2. Kontrola kolize
    if ((x < 0) || (x >= MAPW) || (y < 0) || (y >= MAPH) ||
        (Board[x + y*MAPW] == WALL)) {
        State = S_CRASH;
        return;
    }

    // 3. Zapiš směr na tuto pozici
    Dir[x + y*MAPW] = d;

    // 4. Pohyb vagonů - řetězový systém
    for (i = Length-1; i > 0; i--) {
        d = Dir[x + y*MAPW];  // Směr, kterým tudy prošla hlava

        // Jdi opačným směrem (vagón následuje stopu)
        if (d == DIR_L) x++;
        else if (d == DIR_U) y++;
        else if (d == DIR_R) x--;
        else if (d == DIR_D) y--;

        // Nakresli vagón správným směrem
        if (d == DIR_L) PutTile(xold, yold, WAGON_L);
        else if (d == DIR_U) PutTile(xold, yold, WAGON_U);
        else if (d == DIR_R) PutTile(xold, yold, WAGON_R);
        else if (d == DIR_D) PutTile(xold, yold, WAGON_D);
    }

    // 5. Sběr itemů
    if (Board[HeadX + HeadY*MAPW] == ITEM) {
        Length++;  // Přidej vagón
        ItemNum--;
        Score += 10;
        PlayMelody(CollectSound);

        if (ItemNum == 0) {
            // Otevři bránu
            PutTile(GateX, GateY, GATEMIN+1);
        }
    }
}
```

**Klíčová myšlenka:** Každé pole si pamatuje směr, kterým tudy prošla hlava. Vagony následují tuto stopu opačným směrem.

### 4. Postupná animace brány

```c
#define GATEMIN 30       // První sprite brány
#define GATEMAX 35       // Poslední sprite brány

// V game loop:
if ((GetTile(GateX, GateY) > GATEMIN) && (GetTile(GateX, GateY) < GATEMAX))
{
    PutTile(GateX, GateY, GetTile(GateX, GateY)+1);  // Další sprite
}
```

**Výhoda:** Plynulá animace otevírání brány (6 snímků)

### 5. Dual key buffer pro smooth control

```c
char KeyBuf1, KeyBuf2;  // Dva klávesové buffery

// Během 3 snímků fáze může hráč zadat 2 směry
void ProcessInput() {
    char key = KeyGet();
    if (key != NOKEY) {
        if (KeyBuf1 == NOKEY) {
            KeyBuf1 = key;
        } else if (KeyBuf2 == NOKEY) {
            KeyBuf2 = key;
        }
    }
}

// Na konci fáze aplikuj buffer
if (Phase >= 2) {
    Phase = 0;
    if (KeyBuf1 != NOKEY) {
        ProcessKey(KeyBuf1);
        KeyBuf1 = KeyBuf2;
        KeyBuf2 = NOKEY;
    }
}
```

**Výhoda:** Responzivní ovládání - hráč může zadat směr dopředu

### 6. Password systém s blikáním

```c
void Psw()
{
    char buf[9] = "00000000";
    int pos = 0;

    for (;;) {
        // Blikání pomocí časovače
        if (((Time() >> 23) & 1) == 0) {
            InfoDispText(x, buf, COL_WHITE);  // Celé heslo
        } else {
            PrintInv = 128;  // Inverzní pro aktuální znak
            char buf2[2] = { buf[pos], 0 };
            InfoDispText(x + pos*6, buf2, COL_WHITE);
            PrintInv = 0;
        }

        char c = KeyGet();
        if (c == KEY_UP) {
            buf[pos]++;
            if (buf[pos] > '9') buf[pos] = '0';
        }
        if (c == KEY_DOWN) {
            buf[pos]--;
            if (buf[pos] < '0') buf[pos] = '9';
        }
        if (c == KEY_LEFT) {
            pos--;
            if (pos < 0) pos = 7;
        }
        if (c == KEY_RIGHT) {
            pos++;
            if (pos > 7) pos = 0;
        }
        if (c == KEY_A) {
            // Validace hesla
            for (i = 0; i < MAXLEVEL; i++) {
                if (strcmp(buf, LevelPsw[i]) == 0) {
                    Level = i;
                    return;
                }
            }
        }
    }
}
```

**Technika:** Blikání pomocí vysokého bitu časovače (`Time() >> 23`)

### 7. Efficient tile update

```c
void PutTile(u8 x, u8 y, u8 tile)
{
    if (Board[x + y*MAPW] != tile)  // Pouze při změně
    {
        Board[x + y*MAPW] = tile;
        DispTile(x, y);  // Redraw jen tento tile
    }
}
```

**Výhoda:** Minimální překreslování = vyšší FPS

### 8. Frame-locked timing

```c
#define GAMESPEED   166         // ms per frame

void WaitStep()
{
    u32 t;
    for (;;) {
        t = Time();
        if ((u32)(t - LastTime) >= GAMESPEED*1000*HCLK_PER_US) break;
    }
    LastTime = t;
}

// Main loop:
while (True) {
    if (GameLoop()) break;
    WaitStep();  // Fixní 166ms = ~6 FPS
}
```

**Výhoda:** Konzistentní gameplay na různých MCU

### 9. Crash animace s loop

```c
#define CRASH 40
#define CRASHMAX (CRASH+9)

// Crash animation
State = S_CRASH;
PlayMelody(CrashSound);

u8 b = CRASH;
while (True) {
    PutTile(HeadX, HeadY, b);
    WaitStep();
    b++;
    if (b > CRASHMAX) b = CRASHMAX - 2;  // Loop zpět
    if (KeyGet() != NOKEY) break;
}
```

**Výhoda:** Plynulá animace srážky s loopem

## Shrnutí best practices

### Datové struktury

| Technika | Přínos |
|----------|--------|
| **1D pole místo 2D** | Lineární přístup, cache-friendly |
| **Bitové operace** (`& 3`, `>> 2`) | Rychlejší než `%` a `/` |
| **Lookup tabulky** | Trade-off paměť za rychlost |
| **4-bit packing** | -50% velikost dat |
| **Direction map** | Elegantní chain-following |

### Algoritmy

| Technika | Přínos |
|----------|--------|
| **Inkrementální update** | O(1) místo O(n) |
| **DFS s pruningem** | Efektivní prohledávání |
| **Adaptivní hloubka** | Balancování speed/optimality |
| **Hybridní solver** | Heuristika + exaktní řešení |

### Rendering

| Technika | Přínos |
|----------|--------|
| **Tile-based** | Modularita, snadná správa |
| **Texture atlas** | Efektivní paměť |
| **DrawImgInvFast** | Rychlé renderování |
| **Conditional redraw** | Pouze změněné oblasti |
| **Box-drawing chars** | Estetika bez spritů |

### UX

| Technika | Přínos |
|----------|--------|
| **Dual key buffer** | Responzivní ovládání |
| **Zvuková zpětná vazba** | Validace/kolize feedback |
| **Animace čekání** | Vizuální feedback během výpočtu |
| **Blikání časovačem** | Minimální overhead |
| **Frame-locked timing** | Konzistentní gameplay |

### Paměťová optimalizace

| Technika | Přínos |
|----------|--------|
| **Fixed arrays** | Bez dynamic alokace |
| **Object pooling** | Recyklace objektů |
| **Bitové flagy** | Kompaktní state |
| **Inline funkce** | Zero overhead |

## Odkazy na zdrojové kódy

- **ch32v002-Fifteen:** `/home/user/ch32-kit/ch32libsdk/Rvpc/Games/ch32v002-Fifteen/src/main.c`
- **ch32v002-Train:** `/home/user/ch32-kit/ch32libsdk/Rvpc/Games/ch32v002-Train/src/main.c`

## Související dokumentace

- [Práce s displayem](display.md) - Draw API, fonty, rendering
- [Definice zařízení](devices.md) - Konfigurace hardware
- [Game Patterns](game_patterns.md) - Obecné herní vzory
