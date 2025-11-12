# Definice periferních zařízení (_devices)

## Úvod

CH32LibSDK používá modulární systém definic zařízení umístěný v adresáři `_devices/`. Každé zařízení (např. BabyBoy, TinyBoy, TweetyBoy) má vlastní konfiguraci hardwaru a ovladače pro periferie jako displej, klávesnici a zvuk.

## Struktura _devices

Adresář `ch32libsdk/_devices/` obsahuje 9 předdefinovaných zařízení:

```
_devices/
├── babyboy/     - Malá 128x64 OLED konzole
├── babypad/     - VGA konzole (160x120 nebo 320x240)
├── babypc/      - Větší VGA konzole
├── ch32base/    - Základní CH32 MCU bez specifického hardware
├── pidiboy/     - Rozšíření BabyBoy s SD kartou
├── pidipad/     - Rozšíření BabyPad s SD kartou
├── rvpc/        - Port klasického RVPC počítače
├── tinyboy/     - Kompaktní konzole s baterií
└── tweetyboy/   - Barevný displej (160x80 ST7735S)
```

## Struktura každého zařízení

Každé zařízení má sjednocenou strukturu souborů:

### Základní soubory

- **`_config.h`** - Konfigurace zařízení (hardwarové parametry, piny GPIO)
- **`_include.h`** - Include file s forward deklaracemi
- **`_makefile.inc`** - Makefile dodatek (MCU typ, zdrojové soubory)
- **`[devicename].c`** - Hlavní agregační soubor

### Modulární součásti (dle USE_* maker)

- **`*_disp.c/.h`** - Display/VGA ovladač (USE_DISP)
- **`*_draw.c/.h`** - Grafické kreslící funkce (USE_DRAW)
- **`*_key.c/.h`** - Klávesnicový ovladač (USE_KEY)
- **`*_snd.c/.h`** - Zvukový ovladač (USE_SOUND)
- **`*_init.c/.h`** - Inicializace zařízení
- **`*_vga.c/.h`** - VGA/SPI video výstup
- **`*_bat.c/.h`** - Monitor baterie (TinyBoy)
- **`*_ss.c/.h`** - Screenshot do SD karty

Draw funkce jsou implementovány odděleně v každém zařízení kvůli různé organizaci frame bufferu. Například BabyBoy ukládá 8 pixelů do 1 bytu (bitové operace), TweetyBoy používá 1 byte na pixel (přímé přiřazení), a RVPC M8 funguje pouze v textovém módu. API je však sjednocené napříč všemi zařízeními.

## Konfigurace displeje

### I2C OLED displej (BabyBoy, TinyBoy, PidiBoy)

```c
// _config.h pro OLED SSD1306
#define DISP_I2C_ADDR       0x3C        // I2C adresa displeje
#define DISP_SDA_GPIO       PC1         // SDA pin
#define DISP_SCL_GPIO       PC2         // SCL pin
#define DISP_I2C_MAP        0           // I2C mapping (0-1)
#define DISP_WAIT_CLK       3           // Software I2C zpoždění
#define DISP_SPEED_HZ       750000      // Hardware I2C rychlost

#define USE_DISP            1           // 0=žádný, 1=software, 2=hardware

// Rozlišení displeje
#define WIDTH               128
#define HEIGHT              64
```

### SPI barevný displej (TweetyBoy)

```c
// _config.h pro ST7735S LCD
#define DISP_SCK_GPIO       PA5         // SPI hodinový pin
#define DISP_MOSI_GPIO      PA7         // SPI data pin
#define DISP_CS_GPIO        PA4         // Chip Select
#define DISP_DC_GPIO        PA13        // Data/Command pin
#define DISP_RES_GPIO       PA12        // Reset pin

#define USE_DISP            2           // Hardware SPI

// Rozlišení
#define WIDTH               160
#define HEIGHT              80
```

### Porovnání zařízení

| Zařízení  | Typ displeje    | Komunikace | Rozlišení | MCU       |
|-----------|----------------|-----------|-----------|-----------|
| BabyBoy   | OLED SSD1306   | I2C soft  | 128x64    | CH32V002  |
| TinyBoy   | OLED SSD1306   | I2C soft  | 128x64    | CH32V002  |
| PidiBoy   | OLED SSD1306   | I2C soft  | 128x64    | CH32V003  |
| TweetyBoy | ST7735S LCD    | SPI hard  | 160x80    | CH32V003  |
| BabyPad   | VGA (vlastní)  | Own drv   | 160x120   | CH32V002  |
| BabyPC    | VGA (vlastní)  | Own drv   | 640x480   | CH32V003  |
| RVPC      | VGA (vlastní)  | Own drv   | 128x64    | CH32V003  |

## Konfigurace klávesnice

### Definice tlačítek

```c
// _config.h
#define USE_KEY             1           // Podpora klávesnice
#define KEYCNT_REL          50          // Release interval [ms]
#define KEYCNT_PRESS        400         // První opakování [ms]
#define KEYCNT_REPEAT       100         // Další opakování [ms]
#define SYSTICK_KEYSCAN     1           // Skenování z SysTick
```

### Mapování tlačítek (příklad BabyBoy)

```c
// babyboy_key.c
#define KEY_RIGHT   1   // PC3
#define KEY_UP      2   // PC4
#define KEY_LEFT    3   // PC6
#define KEY_DOWN    4   // PC7
#define KEY_A       5   // PC0
#define KEY_B       6   // PD7

const u8 KeyGpioTab[KEY_NUM] = {
    PC3,    // KEY_RIGHT
    PC4,    // KEY_UP
    PC6,    // KEY_LEFT
    PC7,    // KEY_DOWN
    PC0,    // KEY_A
    PD7,    // KEY_B
};
```

### Funkční rozhraní

```c
// Přímé dotazy
Bool KeyPressed(u8 key);            // Je tlačítko právě stisknuté?
Bool KeyNoPressed();                // Jsou všechna tlačítka uvolněná?

// Buffer operace
u8 KeyGet();                        // Přečti tlačítko z bufferu
void KeyFlush();                    // Vyprázdni buffer
void KeyWaitNoPressed();            // Čekej na uvolnění všech tlačítek

// Lifecycle
void KeyInit();                     // Inicializace
void KeyTerm();                     // Ukončení
```

### Časování opakování

```
0ms        50ms      100ms     400ms     500ms     600ms
│          │         │         │         │         │
Stisk      └─Release─┘         │         │         │
                                └─Repeat─┘         │
                                          └─Repeat─┘
```

## Konfigurace zvuku

### Definice not

```c
// _config.h
#define USE_SOUND           1           // 0=žádný, 1=tone, 2=melody
#define SYSTICK_SOUNDSCAN   1           // Skenování z SysTick

// Nota C4 má frekvenci 261.626 Hz
#define NOTE_C4     SND_GET_DIV(26163)
#define NOTE_CS4    SND_GET_DIV(27718)
#define NOTE_D4     SND_GET_DIV(29367)
// ... pokračuje až do NOTE_C9
```

### Struktura melodie

```c
typedef struct {
    u16 len;    // Délka v 1/60 sec
    u16 div;    // Divider frekvence nebo 0 pro pauzu
} sMelodyNote;

// Délky not
#define NOTE_LEN1DOT    96      // Celá s tečkou
#define NOTE_LEN1       64      // Celá
#define NOTE_LEN2DOT    48      // Půl s tečkou
#define NOTE_LEN2       32      // Půl
#define NOTE_LEN4       16      // Čtvrtina
#define NOTE_LEN8       8       // Osmina
#define NOTE_LEN16      4       // Šestnáctina
#define NOTE_STOP       0       // Konec melodie
```

### Příklad melodie

```c
const sMelodyNote StartMelody[] = {
    { NOTE_LEN4, NOTE_C5 },     // Čtvrtina C5
    { NOTE_LEN4, NOTE_E5 },     // Čtvrtina E5
    { NOTE_LEN4, NOTE_G5 },     // Čtvrtina G5
    { NOTE_LEN2, NOTE_C6 },     // Půl C6
    { NOTE_LEN4, 0 },           // Čtvrtina pauza
    { 0, 0 },                   // Konec
};

// Přehrávání
PlayMelody(StartMelody);
```

### Funkční rozhraní

```c
void SoundInit();                               // Init zvuku
void SoundTerm();                               // Ukončení zvuku
void PlayTone(u32 div);                        // Přehrát tón
void StopSound();                              // Zastavit zvuk
void PlayMelody(const sMelodyNote* melody);   // Přehrát melodii (USE_SOUND==2)
Bool PlayingSound();                           // Je zvuk právě přehrávám?
```

## Inicializační sekvence

### DevInit() - inicializace zařízení

```c
void DevInit()
{
    #if USE_KEY
    KeyInit();              // Inicializace klávesnice
    #endif

    #if USE_SOUND
    SoundInit();            // Inicializace zvuku
    #endif

    #if USE_SD
    SD_Init();              // Inicializace SD karty
    #endif

    #if USE_DISP
    DispInit();             // Inicializace displeje
    #endif

    #if USE_DRAW || USE_PRINT
    DrawClear();            // Vymažení displeje
    #endif
}
```

### DevTerm() - ukončení zařízení

```c
void DevTerm()
{
    #if USE_KEY
    KeyWaitNoPressed();     // Čekej na uvolnění tlačítek
    #endif

    #if USE_DISP
    DispTerm();             // Vypnutí displeje
    #endif

    #if USE_SD
    SD_Term();              // Deaktivace SD
    #endif

    #if USE_SOUND
    SoundTerm();            // Vypnutí zvuku
    #endif

    #if USE_KEY
    KeyTerm();              // Vypnutí klávesnice
    #endif
}
```

## Použití v projektu

### Include v hlavním kódu

```c
#include "ch32libsdk/includes.h"
```

Soubor `includes.h` automaticky vybírá správné zařízení podle `USE_*` maker:

```c
#if USE_BABYBOY
#include "_devices/babyboy/_include.h"
#endif

#if USE_TINYBOY
#include "_devices/tinyboy/_include.h"
#endif
// ... další zařízení
```

### Makefile integrace

```makefile
# Příklad pro BabyBoy
CSRC += ${CH32LIBSDK_DEVICES_DIR}/babyboy/babyboy_disp.c
CSRC += ${CH32LIBSDK_DEVICES_DIR}/babyboy/babyboy_draw.c
CSRC += ${CH32LIBSDK_DEVICES_DIR}/babyboy/babyboy_key.c
CSRC += ${CH32LIBSDK_DEVICES_DIR}/babyboy/babyboy_snd.c
CSRC += ${CH32LIBSDK_DEVICES_DIR}/babyboy/babyboy_init.c

DEFINE += -D USE_BABYBOY=1
MCU=CH32V002x4
```

### Typická aplikace

```c
#include "ch32libsdk/includes.h"

int main()
{
    DevInit();              // Inicializace zařízení
    DrawClear();            // Vymažení displeje
    DrawText(0, 0, "Hello!");
    DispUpdate();           // Odeslání na displej

    while (1)
    {
        u8 key = KeyGet();  // Přečti klávesnici
        if (key == KEY_A)
        {
            PlayTone(NOTE_C4);
        }
    }

    DevTerm();              // Ukončení
    return 0;
}
```

## Vlastní zařízení

Můžete vytvořit vlastní zařízení v adresáři `custom_device/custom_devices/`:

```c
// custom_device/custom_devices/mydevice/_config.h

#define DISP_I2C_ADDR       0x3D        // Jiná I2C adresa
#define DISP_SDA_GPIO       PA9         // Vlastní piny
#define DISP_SCL_GPIO       PA10
#define USE_DISP            2           // Hardware driver místo software
#define USE_SD              1           // Přidáno SD
#define USE_BAT             1           // Přidáno měření baterie
#define DEVICE_NAME         "MyDevice"
#define DEVICE_VERSION      "1.0"

// Nové vlastní piny
#define LED_GPIO            PC3         // Status LED
#define BUTTON_GPIO         PD2         // Vlastní tlačítko
#define SD_CS_GPIO          PB1         // SD ChipSelect
```

## Další informace

- [Práce s displayem](display.md) - Podrobný popis grafických funkcí
- [GPIO](gpio.md) - Konfigurace GPIO pinů
- [I2C](i2c.md) - I2C komunikace pro displeje
- [SPI](spi.md) - SPI komunikace pro barevné displeje
- [Quick Start](quickstart.md) - Rychlý start s CH32LibSDK
