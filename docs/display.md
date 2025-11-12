# Práce s displayem a kreslení

## Úvod

CH32LibSDK poskytuje kompletní API pro práci s monochrom displayi (OLED, VGA) včetně kreslení geometrických tvarů, textu a obrázků. Tato dokumentace pokrývá všechny funkce pro práci s displayem.

## Základní informace

### Poznámka o implementaci

Draw funkce (DrawPoint, DrawLine, atd.) jsou implementovány v jednotlivých zařízeních (`_devices/*/draw.c`), nikoli v centrálním SDK. Důvodem je různá organizace frame bufferu - například BabyBoy ukládá 8 pixelů do 1 bytu s bitovými operacemi, zatímco TweetyBoy používá 1 byte na pixel. API je však sjednocené, takže váš kód funguje na všech zařízeních. Více detailů najdete v [dokumentaci zařízení](devices.md).

### Hardware parametry (typicky pro 128x64 OLED)

```c
#define WIDTH       128         // Šířka v pixelech
#define HEIGHT      64          // Výška v grafických řádcích
#define WIDTHBYTE   16          // Šířka v bajtech (WIDTH/8)
#define FRAMESIZE   1024        // Velikost frame bufferu (WIDTHBYTE × HEIGHT)
#define TEXTWIDTH   16          // Textová šířka v znacích (WIDTH/8)
#define TEXTHEIGHT  8           // Textová výška v řádcích (HEIGHT/8)
```

### Frame buffer

```c
extern u8 FrameBuf[FRAMESIZE];  // Grafický buffer (1024 bajtů)
```

Veškeré kreslení probíhá do tohoto bufferu v paměti. Pro zobrazení na displeji je potřeba zavolat `DispUpdate()`.

### Barvy

```c
#define COL_BLACK   0           // Černá barva (vypnuto)
#define COL_WHITE   1           // Bílá barva (zapnuto)
```

## Inicializace a správa displeje

### Základní funkce

```c
void DispInit(void);            // Inicializuje displej (I2C, nastavení SSD1306)
void DispTerm(void);            // Vypne displej a uvolní GPIO
void DispUpdate();              // Odešle frame buffer do displeje (~15 ms)
void DrawClear();               // Vyčistí displej (všechny pixely na 0)
```

### Typické použití

```c
int main() {
    DispInit();              // Inicializuj displej

    DrawClear();
    DrawText("Hello!", 10, 10, COL_WHITE);
    DispUpdate();            // Odešli do displeje

    // ... nějaký kód ...

    DispTerm();              // Vypni displej a uvolni GPIO
    return 0;
}
```

## Kreslení bodů (pixelů)

### Základní operace s pixely

```c
// Nakreslí/smaže/invertuje pixel s kontrolou hranic
void DrawPoint(int x, int y, u8 col);        // Nakreslí pixel barvou (0/1)
void DrawPointClr(int x, int y);             // Smaže pixel
void DrawPointSet(int x, int y);             // Zapne pixel
void DrawPointInv(int x, int y);             // Invertuje pixel

// Rychlé verze bez kontroly hranic (pouze v rozsahu 0-127 x, 0-63 y)
void DrawPointFast(int x, int y, u8 col);
void DrawPointClrFast(int x, int y);
void DrawPointSetFast(int x, int y);

// Čtení barvy pixelu
u8 DrawGetPoint(int x, int y);               // Vrátí 0 nebo 1
```

### Příklad

```c
DrawClear();
DrawPoint(10, 20, COL_WHITE);   // Nakreslí bílý pixel na [10, 20]
DrawPointInv(50, 30);           // Invertuje pixel na [50, 30]
DispUpdate();
```

## Kreslení čár

### Horizontální a vertikální čáry

```c
// HORIZONTÁLNÍ ČÁRA (y je fixní, pohybuje se x)
void DrawHLine(int x, int y, int w, u8 col);   // Nakreslí h. čáru
void DrawHLineClr(int x, int y, int w);        // Smaže h. čáru
void DrawHLineInv(int x, int y, int w);        // Invertuje h. čáru

// VERTIKÁLNÍ ČÁRA (x je fixní, pohybuje se y)
void DrawVLine(int x, int y, int h, u8 col);   // Nakreslí v. čáru
void DrawVLineClr(int x, int y, int h);        // Smaže v. čáru
void DrawVLineInv(int x, int y, int h);        // Invertuje v. čáru
```

### Obecné čáry (Bresenham algoritmus)

```c
void DrawLine(int x1, int y1, int x2, int y2, u8 col);   // Čára libovolný směr
void DrawLineClr(int x1, int y1, int x2, int y2);        // Smaže čáru
void DrawLineInv(int x1, int y1, int x2, int y2);        // Invertuje čáru
```

## Kreslení obdélníků

### Vyplněné obdélníky

```c
void DrawRect(int x, int y, int w, int h, u8 col);   // Vyplní obdélník
void DrawRectClr(int x, int y, int w, int h);        // Smaže obdélník
void DrawRectInv(int x, int y, int w, int h);        // Invertuje obdélník
```

### Rámečky (pouze okraj)

```c
void DrawFrame(int x, int y, int w, int h, u8 col);  // Nakreslí rámec
void DrawFrameClr(int x, int y, int w, int h);       // Smaže rámec
void DrawFrameInv(int x, int y, int w, int h);       // Invertuje rámec
```

### Příklad

```c
DrawClear();
DrawFrame(10, 10, 50, 30, COL_WHITE);      // Rámec
DrawLine(10, 10, 60, 40, COL_WHITE);       // Diagonální čára
DispUpdate();
```

## Kreslení kruhů

### Vyplněné kruhy

```c
void DrawRound(int x0, int y0, int r, u8 col);      // Vyplní kruh (r = poloměr)
void DrawRoundClr(int x0, int y0, int r);           // Smaže kruh
void DrawRoundInv(int x0, int y0, int r);           // Invertuje kruh
```

### Kruhy (pouze obrys)

```c
void DrawCircle(int x0, int y0, int r, u8 col);     // Kruh obrys
void DrawCircleClr(int x0, int y0, int r);          // Smaže kruh obrys
void DrawCircleInv(int x0, int y0, int r);          // Invertuje kruh obrys
```

### Prstence (mezikruží)

```c
void DrawRing(int x0, int y0, int rin, int rout, u8 col);  // rin = vnitřní, rout = vnější
void DrawRingClr(int x0, int y0, int rin, int rout);
void DrawRingInv(int x0, int y0, int rin, int rout);
```

### Příklad

```c
DrawClear();
DrawRound(64, 32, 10, COL_WHITE);         // Vyplněný kruh uprostřed
DrawCircle(64, 32, 20, COL_WHITE);        // Obrys kolem něj
DispUpdate();
```

## Kreslení trojúhelníků

```c
void DrawTriangle(int x1, int y1, int x2, int y2, int x3, int y3, u8 col);  // Vyplní trojúhel
void DrawTriangleClr(int x1, int y1, int x2, int y2, int x3, int y3);       // Smaže trojúhel
void DrawTriangleInv(int x1, int y1, int x2, int y2, int x3, int y3);       // Invertuje trojúhel
```

## Fonty

### Dostupné fonty (všechny jsou 8×8 pixelů pokud není uvedeno)

| Název fontu | Velikost | Popis |
|------------|----------|-------|
| `FontBold8x8` | 8×8 | Tučný font |
| `FontGame8x8` | 8×8 | Herní font |
| `FontIbm8x8` | 8×8 | IBM standardní |
| `FontIbmTiny8x8` | 8×8 | IBM zmenšený |
| `FontItalic8x8` | 8×8 | Kurzívní |
| `FontThin8x8` | 8×8 | Tenký font |
| `FontThin8x8Inv` | 8×8 | Tenký invertovaný |
| `FontCond6x8` | 6×8 | Zkondenzovaný (ušetří místo) |
| `FontCond6x6` | 6×6 | Velmi zkondenzovaný |
| `FontZx` | ? | ZX Spectrum font |
| `Font80` | ? | CP 437 font |
| `Font81` | ? | CP 437 font |

### Deklarace fontů

```c
extern const ALIGNED u8 FontBold8x8[2048];      // 256 znaků × 8 bajtů
extern const ALIGNED u8 FontCond6x8[2048];
extern const ALIGNED u8 FontCond6x6[1536];
// ... další fonty
```

### Jak fonty fungují

Každý font obsahuje bitmap pro všech 256 ASCII znaků. Znaky jsou organizovány po řádcích:
- Pro znaky 8×8: 256 znaků × 8 bajtů = 2048 bajtů
- Adresa znaku: `font_base + (char_code) + (row * 256)`
- Každý bajt = jeden řádek znaku (8 pixelů = 8 bitů)

```c
// Příklad: jak se čte font data
const u8* src = &FontBold8x8[(u8)'A'];  // Adresa znaku 'A'
u8 byte0 = src[0*256];                   // Řádek 0 znaku 'A'
u8 byte1 = src[1*256];                   // Řádek 1 znaku 'A'
// ... až byte7 pro řádek 7
```

### Nastavení fontu

```c
extern const u8* DrawFont;       // Aktuální font pro kreslení
extern const u8* DrawFontCond;   // Aktuální zkondenzovaný font

// Změna aktuálního fontu
void SetFont(const char* font);

// Příklad:
SetFont(FontBold8x8);   // Použij tučný font
SetFont(FontCond6x8);   // Přepni na zkondenzovaný
```

## Kreslení znaků

### Normální velikost (8×8)

```c
void DrawChar(char ch, int x, int y, u8 col);       // Bez pozadí
void DrawCharBg(char ch, int x, int y);             // S černým pozadím
void DrawCharClr(char ch, int x, int y);            // Smaže znak
void DrawCharInv(char ch, int x, int y);            // Invertuje znak
```

### Double-width (16×8)

```c
void DrawCharW(char ch, int x, int y, u8 col);      // Dvojnásobná šířka
void DrawCharWClr(char ch, int x, int y);
void DrawCharWInv(char ch, int x, int y);
```

### Double-height (8×16)

```c
void DrawCharH(char ch, int x, int y, u8 col);      // Dvojnásobná výška
void DrawCharHClr(char ch, int x, int y);
void DrawCharHInv(char ch, int x, int y);
```

### Double-size (16×16)

```c
void DrawChar2(char ch, int x, int y, u8 col);      // Obě dimenze dvojité
void DrawChar2Clr(char ch, int x, int y);
void DrawChar2Inv(char ch, int x, int y);
```

### Zkondenzované fonty (6×8 a 6×6)

```c
void DrawCharCond(char ch, int x, int y, u8 col);   // 6×8
void DrawCharCondBg(char ch, int x, int y);
void DrawCharCond6(char ch, int x, int y, u8 col);  // 6×6
void DrawCharCond6Bg(char ch, int x, int y);
```

## Kreslení textu

### Normální velikost

```c
void DrawText(const char* text, int x, int y, u8 col);    // Bez pozadí
void DrawTextClr(const char* text, int x, int y);         // Smaže text
void DrawTextInv(const char* text, int x, int y);         // Invertuje text
```

### Různé velikosti

```c
// Double-width
void DrawTextW(const char* text, int x, int y, u8 col);
void DrawTextWClr(const char* text, int x, int y);
void DrawTextWInv(const char* text, int x, int y);

// Double-height
void DrawTextH(const char* text, int x, int y, u8 col);
void DrawTextHClr(const char* text, int x, int y);
void DrawTextHInv(const char* text, int x, int y);

// Double-size
void DrawText2(const char* text, int x, int y, u8 col);
void DrawText2Clr(const char* text, int x, int y);
void DrawText2Inv(const char* text, int x, int y);
```

### Zkondenzované texty

```c
void DrawTextCond(const char* text, int x, int y, u8 col);     // 6×8
void DrawTextCondBg(const char* text, int x, int y);
void DrawTextCond6(const char* text, int x, int y, u8 col);    // 6×6
void DrawTextCond6Bg(const char* text, int x, int y);
```

### Příklad

```c
DrawClear();
SetFont(FontBold8x8);
DrawText("HELLO", 10, 10, COL_WHITE);      // Normální text
DrawText2("BIG", 10, 30, COL_WHITE);       // Velký text (16×16)

SetFont(FontCond6x8);
DrawTextCond("Small", 10, 50, COL_WHITE);  // Malý text

DispUpdate();
```

## Textový režim (Print API)

Print API poskytuje režim, kde se text automaticky umisťuje do textové mřížky (16 znaků × 8 řádků).

### Stavy tisku

```c
extern int PrintPos;       // Aktuální pozice v řádku (0-15)
extern int PrintRow;       // Aktuální řádek (0-7)
extern u8 PrintInv;        // Inverze textu (0 = normální, 128 = invertovaný)
```

### Základní funkce

```c
void PrintClear();                                 // Vyčistí obrazovku
void PrintHome(void);                              // Pozice [0, 0]
void PrintNewLine(void);                           // Nový řádek
void PrintScroll(void);                            // Scrolluj o řádek nahoru

// Tisk na konkrétní pozici
void PrintCharAt(char ch, int x, int y);
void PrintTextAt(const char* text, int x, int y);

// Tisk na aktuální pozici s ovládacími znaky
void PrintChar(char ch);
void PrintText(const char* text);

// Tisk bez ovládacích znaků (RAW)
void PrintCharRaw(char ch);
void PrintTextRaw(const char* text);

// Pomocné funkce
void PrintSpc(void);                               // Tisk mezery
void PrintSpcRep(int num);                         // Tisk N mezer
```

### Ovládací znaky

```
0x01 '\1' ^A - Normální text (zrušit inverzi)
0x02 '\2' ^B - Invertovaný text
0x07 '\a' ^G - Zvonek (posuň kurzor doprava)
0x08 '\b' ^H - Backspace
0x09 '\t' ^I - Tabulator (příští 8-znaková pozice)
0x0A '\n' ^J - Nový řádek
0x0B '\v' ^K - Vertikální tabulator (předchozí řádek)
0x0C '\f' ^L - Form feed (vyčistit, reset)
0x0D '\r' ^M - Carriage return (začátek řádku)
```

### Příklad

```c
PrintClear();
PrintText("Hello\nWorld");      // Tiskne na 2 řádky

PrintHome();
PrintChar('A');
PrintChar('B');                 // A na [0,0], B na [1,0]

PrintChar('\2');                // Zapni inverzi
PrintText("INVERTED");          // Invertovaný text
PrintChar('\1');                // Vypni inverzi
```

## Kreslení obrázků

### Monochromatické obrázky

```c
// Kreslení obrázku
void DrawImg(const u8* img, int x, int y, int w, int h, int wsb, u8 col);
// img = ukazatel na data obrázku
// x, y = pozice na displeji
// w, h = šířka a výška obrázku v pixelech
// wsb = šířka obrázku v bajtech (= w/8)
// col = barva (0/1)

void DrawImgBg(const u8* img, int x, int y, int w, int h, int wsb);  // S černým pozadím
void DrawImgClr(const u8* img, int x, int y, int w, int h, int wsb); // Smaže obrázek
void DrawImgInv(const u8* img, int x, int y, int w, int h, int wsb); // Invertuje obrázek

// Rychlá verze (vyžaduje zarovnání bytů)
void DrawImgFast(const u8* img, int x, int y, int xs, int ys, int w, int h, int wsb);
void DrawImgInvFast(const u8* img, int x, int y, int xs, int ys, int w, int h, int wsb);
```

### Příklad

```c
// Definice obrázku 16×16 (2 bajty × 16 řádků = 32 bajtů)
const u8 sprite[] = {
    0b00111100, 0b00000000,
    0b01000010, 0b00000000,
    0b10000001, 0b00000000,
    // ... další řádky
};

DrawClear();
DrawImg(sprite, 50, 20, 16, 16, 2, COL_WHITE);
DispUpdate();
```

## Praktické příklady

### Příklad 1: Základní kreslení

```c
#include "ch32libsdk/includes.h"

int main() {
    DispInit();
    DrawClear();

    // Nakreslíme rámec
    DrawFrame(5, 5, 118, 54, COL_WHITE);

    // Nakreslíme kruh uprostřed
    DrawRound(64, 32, 15, COL_WHITE);

    // Napíšeme text
    SetFont(FontBold8x8);
    DrawText("TEST", 45, 10, COL_WHITE);

    // Odešleme do displeje
    DispUpdate();

    // Počkej a smaž
    WaitMs(3000);
    DrawClear();
    DispUpdate();

    DispTerm();
    return 0;
}
```

### Příklad 2: Animace

```c
int main() {
    DispInit();

    for (int i = 0; i < 30; i++) {
        DrawClear();
        int x = (i * 4) % 128;
        int y = (i * 2) % 64;
        DrawRound(x, y, 5, COL_WHITE);
        DispUpdate();
        WaitMs(100);
    }

    DispTerm();
    return 0;
}
```

### Příklad 3: Různé velikosti textů

```c
int main() {
    DispInit();
    DrawClear();

    SetFont(FontBold8x8);
    DrawText("Normal", 0, 0, COL_WHITE);
    DrawTextW("Wide", 0, 10, COL_WHITE);
    DrawTextH("Tall", 0, 20, COL_WHITE);
    DrawText2("Big", 0, 40, COL_WHITE);

    DispUpdate();

    while(1);  // Zobraz navždy

    return 0;
}
```

### Příklad 4: Herní UI

```c
void DrawGameUI(int score, int lives) {
    DrawClear();

    // Status bar nahoře
    DrawRect(0, 0, WIDTH, 10, COL_WHITE);
    SetFont(FontThin8x8);
    PrintInv = 128;  // Invertovaný text
    PrintTextAt("SCORE:", 0, 0);
    PrintInv = 0;

    // Skóre
    char buf[16];
    sprintf(buf, "%d", score);
    DrawText(buf, 50, 2, COL_WHITE);

    // Životy
    for (int i = 0; i < lives; i++) {
        DrawRound(100 + i*10, 5, 3, COL_WHITE);
    }

    // Herní oblast
    DrawFrame(2, 12, WIDTH-4, HEIGHT-14, COL_WHITE);

    DispUpdate();
}
```

## Optimalizace výkonu

### Tipy pro rychlejší kreslení

1. **Používejte Fast varianty** pokud víte, že souřadnice jsou v rozsahu
2. **Minimalizujte DispUpdate()** - volání trvá ~15 ms
3. **Kreslete pouze změněné oblasti** - nemazávejte celý displej každý frame
4. **Používejte DrawRect místo DrawFrame** pro vyplněné plochy
5. **Zkondenzované fonty** šetří místo a jsou rychlejší

### Příklad optimalizovaného kreslení

```c
// Špatně - pomalé
while(1) {
    DrawClear();  // 15 ms
    DrawText("Score: 100", 0, 0, COL_WHITE);
    DispUpdate(); // 15 ms
    // Celkem ~30 ms per frame = ~33 FPS
}

// Správně - rychlé
DrawClear();
DrawText("Score: ", 0, 0, COL_WHITE);
DispUpdate();

int last_score = 0;
while(1) {
    if (score != last_score) {
        // Vymaž pouze oblast čísla
        DrawRect(48, 0, 30, 8, COL_BLACK);
        // Překresli pouze číslo
        char buf[8];
        sprintf(buf, "%d", score);
        DrawText(buf, 48, 0, COL_WHITE);
        DispUpdate();  // Jen když je změna
        last_score = score;
    }
}
```

## Souvislosti

- [Definice periferních zařízení](devices.md) - Konfigurace displeje
- [Příklady z her](game_examples.md) - Pokročilé techniky kreslení
- [Game Patterns](game_patterns.md) - Herní programovací vzory
