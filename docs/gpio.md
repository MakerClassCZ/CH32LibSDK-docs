# GPIO - Práce s digitálními piny

## Přehled

GPIO (General Purpose Input/Output) modul poskytuje funkce pro práci s digitálními piny mikrokontroléru. Umožňuje nastavení pinů jako vstup nebo výstup, čtení a zápis hodnot a konfiguraci různých režimů.

## Základní koncepty

### Porty a piny

CH32 mikrokontroléry mají typicky 3 GPIO porty:
- **GPIOA** - Port A (adresa 0x40010800)
- **GPIOC** - Port C (adresa 0x40011000)  
- **GPIOD** - Port D (adresa 0x40011400)

Každý port má až 8 pinů (0-7).

### Identifikace pinů

Piny jsou identifikovány pomocí makra `PA0`, `PC1`, `PD7` atd., které kombinují port a číslo pinu.

## Režimy pinů

### Vstupní režimy

```c
#define GPIO_MODE_AIN     0x00  // Analogový vstup (pro ADC)
#define GPIO_MODE_IN      0x04  // Plovoucí vstup (bez pull rezistorů)  
#define GPIO_MODE_IN_PD   0x28  // Vstup s pull-down rezistorem
#define GPIO_MODE_IN_PU   0x48  // Vstup s pull-up rezistorem
```

### Výstupní režimy

```c
#define GPIO_MODE_OUT      (0+1)  // Push-pull výstup, 10 MHz
#define GPIO_MODE_OUT_SLOW (0+2)  // Push-pull výstup, 2 MHz
#define GPIO_MODE_OUT_FAST (0+3)  // Push-pull výstup, 30 MHz

#define GPIO_MODE_OD       (4+1)  // Open-drain výstup, 10 MHz
#define GPIO_MODE_OD_SLOW  (4+2)  // Open-drain výstup, 2 MHz  
#define GPIO_MODE_OD_FAST  (4+3)  // Open-drain výstup, 30 MHz
```

### Alternativní funkce

```c
#define GPIO_MODE_AF       (8+1)  // Alternativní funkce, 10 MHz
#define GPIO_MODE_AFOD     (12+1) // Alternativní funkce open-drain (otevřený kolektor), 10 MHz
```

## Základní funkce

### Nastavení režimu pinu

```c
void GPIO_Mode(int gpio, int mode);
```

Nastaví režim pinu.

**Parametry:**
- `gpio` - identifikátor pinu (např. `PA0`)
- `mode` - režim pinu (např. `GPIO_MODE_OUT`)

**Příklad:**
```c
// Nastavení PA0 jako výstup
GPIO_Mode(PA0, GPIO_MODE_OUT);

// Nastavení PC1 jako vstup s pull-up
GPIO_Mode(PC1, GPIO_MODE_IN_PU);
```

### Čtení vstupního pinu

```c
u8 GPIO_In(int gpio);
```

Přečte hodnotu vstupního pinu (0 nebo 1).

**Příklad:**
```c
if (GPIO_In(PA0) == 1) {
    // Pin je v log. 1
}
```

### Zápis na výstupní pin

```c
void GPIO_Out(int gpio, int val);
void GPIO_Out0(int gpio);  // Rychlý zápis 0
void GPIO_Out1(int gpio);  // Rychlý zápis 1
```

Zapíše hodnotu na výstupní pin.

**Příklad:**
```c
// Nastavení pinu do log. 1
GPIO_Out1(PA0);

// Nastavení pinu do log. 0  
GPIO_Out0(PA0);

// Nastavení podle proměnné
GPIO_Out(PA0, value);
```

### Přepnutí výstupu

```c
void GPIO_Flip(int gpio);
```

Invertuje stav výstupního pinu.

**Příklad:**
```c
// Přepnutí LED
GPIO_Flip(PA0);
```

## Pokročilé funkce

### Práce s celým portem

```c
u8 GPIO_InAll(int gpio);           // Čtení všech pinů portu
void GPIO_OutAll(int gpio, int value); // Zápis na všechny piny portu
```

### Získání informací o pinu

```c
int GPIO_GetMode(int gpio);        // Získání aktuálního režimu
u8 GPIO_GetOut(int gpio);          // Získání výstupní hodnoty
```

### Zamknutí konfigurace

```c
void GPIO_Lock(int gpio);
```

Zamkne konfiguraci pinu - nelze ji změnit až do resetu MCU.

## Alternativní funkce a remapování

### Externí přerušení

```c
void GPIO_EXTILine(int port, int pin);
```

Nastaví, který pin portu bude zdrojem externího přerušení.

### Remapování periferií

```c
GPIO_Remap_SPI1(int map);   // Remapování SPI1
GPIO_Remap_I2C1(int map);   // Remapování I2C1
GPIO_Remap_USART1(int map); // Remapování USART1
GPIO_Remap_TIM1(int map);   // Remapování Timer1
GPIO_Remap_TIM2(int map);   // Remapování Timer2
```

## Praktické příklady

### Blikání LED

```c
// Inicializace
GPIO_Mode(PA0, GPIO_MODE_OUT);

// Hlavní smyčka
while(1) {
    GPIO_Out1(PA0);     // LED zapnout
    DelayMs(500);       // Čekat 500ms
    GPIO_Out0(PA0);     // LED vypnout
    DelayMs(500);       // Čekat 500ms
}
```

### Čtení tlačítka

```c
// Inicializace - tlačítko s pull-up
GPIO_Mode(PC0, GPIO_MODE_IN_PU);

// Čtení (tlačítko spojuje pin na GND)
if (GPIO_In(PC0) == 0) {
    // Tlačítko je stisknuto
}
```

### Ovládání více LED najednou

```c
// Inicializace celého portu jako výstup
for (int i = 0; i < 8; i++) {
    GPIO_Mode(GPIO_GPIO(PORTA_INX, i), GPIO_MODE_OUT);
}

// Rozsvítit LED podle vzoru
GPIO_OutAll(PA0, 0b10101010);
```

### Herní ovládání s debouncing

```c
// Reálný příklad z her - debouncing pro herní ovládání
volatile uint8_t key_state = 0;
volatile uint8_t key_prev = 0;
volatile uint8_t frame_counter = 0;

// Inicializace herních tlačítek
void Game_Input_Init(void) {
    GPIO_Mode(PC0, GPIO_MODE_IN_PU);  // KEY_LEFT
    GPIO_Mode(PC1, GPIO_MODE_IN_PU);  // KEY_RIGHT  
    GPIO_Mode(PC2, GPIO_MODE_IN_PU);  // KEY_UP
    GPIO_Mode(PC3, GPIO_MODE_IN_PU);  // KEY_DOWN
    GPIO_Mode(PC4, GPIO_MODE_IN_PU);  // KEY_A (akční tlačítko)
    GPIO_Mode(PC5, GPIO_MODE_IN_PU);  // KEY_B
}

// Čtení herního vstupu s frame-based sampling (60 FPS)
uint8_t Game_ReadInput(void) {
    uint8_t keys = 0;
    
    // Čtení všech tlačítek najednou
    if (GPIO_In(PC0) == 0) keys |= 0x01;  // LEFT
    if (GPIO_In(PC1) == 0) keys |= 0x02;  // RIGHT
    if (GPIO_In(PC2) == 0) keys |= 0x04;  // UP
    if (GPIO_In(PC3) == 0) keys |= 0x08;  // DOWN
    if (GPIO_In(PC4) == 0) keys |= 0x10;  // A
    if (GPIO_In(PC5) == 0) keys |= 0x20;  // B
    
    return keys;
}

// Herní smyčka s frame-based input handling
void Game_Loop(void) {
    while(1) {
        frame_counter++;
        
        // Čtení vstupů pouze každý 8. frame pro debouncing
        if (frame_counter % 8 == 0) {
            key_prev = key_state;
            key_state = Game_ReadInput();
            
            // Detekce stisknutí (edge detection)
            uint8_t key_pressed = key_state & ~key_prev;
            
            // Herní logika podle stisknutých kláves
            if (key_state & 0x01) {  // LEFT drženo
                // Kontinuální pohyb vlevo
                player_x--;
            }
            if (key_pressed & 0x10) {  // A právě stisknuto
                // Jednorázová akce (střelba, skok)
                FireBullet();
            }
        }
        
        // Aktualizace hry 60 FPS
        Game_Update();
        Wait_VSync();  // Čekat na vertikální synchronizaci
    }
}
```

### Multi-directional input handling

```c
// Pokročilé směrové ovládání s podporou diagonál
typedef enum {
    DIR_NONE = 0,
    DIR_UP = 1,
    DIR_DOWN = 2, 
    DIR_LEFT = 4,
    DIR_RIGHT = 8,
    DIR_UP_LEFT = DIR_UP | DIR_LEFT,
    DIR_UP_RIGHT = DIR_UP | DIR_RIGHT,
    DIR_DOWN_LEFT = DIR_DOWN | DIR_LEFT,
    DIR_DOWN_RIGHT = DIR_DOWN | DIR_RIGHT
} Direction_t;

Direction_t Get_Direction(void) {
    Direction_t dir = DIR_NONE;
    
    if (GPIO_In(PC2) == 0) dir |= DIR_UP;     // UP
    if (GPIO_In(PC3) == 0) dir |= DIR_DOWN;   // DOWN  
    if (GPIO_In(PC0) == 0) dir |= DIR_LEFT;   // LEFT
    if (GPIO_In(PC1) == 0) dir |= DIR_RIGHT;  // RIGHT
    
    return dir;
}

// Použití v herní logice
void Update_Player_Movement(void) {
    Direction_t dir = Get_Direction();
    
    switch(dir) {
        case DIR_UP:
            player_y--;
            break;
        case DIR_DOWN:
            player_y++;
            break;
        case DIR_LEFT:
            player_x--;
            break;
        case DIR_RIGHT:
            player_x++;
            break;
        case DIR_UP_LEFT:
            player_x--; player_y--;  // Diagonální pohyb
            break;
        // ... další směry
    }
}

## Tipy a triky

1. **Rychlost pinů**: Pro LED a tlačítka stačí `GPIO_MODE_OUT_SLOW` (2 MHz), šetří energii
2. **Pull rezistory**: Vždy používejte pull-up/down pro tlačítka, jinak bude pin plovoucí
3. **Open-drain**: Vhodné pro I2C, společné sběrnice nebo LED s externím pull-up
4. **Zamknutí**: Použijte `GPIO_Lock()` pro kritické piny, aby se zabránilo náhodné změně

## Poznámky pro různé MCU

- **CH32V003**: Základní varianta s porty A, C, D a standardními funkcemi
- **CH32V002**: Nejnovější varianta s vylepšeními - 4KB SRAM vs 2KB, 12-bit ADC, touch sensing
- **CH32V006**: Pokročilá varianta s více I/O piny a pamětí
- **CH32X035**: Nejbohatší na funkce, více alternativních funkcí