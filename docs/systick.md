# SysTick - Systémový časovač

## Úvod

CH32LibSDK poskytuje kompletní API pro SysTick timer - 64-bitový systémový časovač používaný pro timing, delay a time-based operace. SysTick je kritická součást pro hry a real-time aplikace.

## Základní funkce

### Inicializace a ovládání

```c
void SysTick_Init(void);       // Inicializovat SysTick
void SysTick_Reset(void);      // Resetovat SysTick

void SysTick_Enable(void);     // Zapnout čítač
void SysTick_Disable(void);    // Vypnout čítač
```

### Čtení čítače

```c
u32 SysTick_Get(void);         // Přečíst čítač (32-bit)
u64 SysTick_Get64(void);       // Přečíst čítač (64-bit, interrupt-safe)

u32 Time(void);                // Aktuální čas [takty]
u64 Time64(void);              // Aktuální čas 64-bit [takty]
```

### Nastavení

```c
void SysTick_Set(u64 cnt);     // Nastavit čítač (čítač musí být vypnutý)
void SysTick_SetCmp(u64 cmp);  // Nastavit porovnávací hodnotu
u64 SysTick_GetCmp64(void);    // Přečíst compare
```

## Konfigurace

### Konstant

y

```c
// Počet HCLK cyklů na 1 mikrosekundu (děleno 8)
#define HCLK_PER_US    9        // CH32V103: 72 MHz / 8 = 9 MHz

// Počet HCLK cyklů na 1 milisekundu
#define HCLK_PER_MS    (HCLK_PER_US * 1000)  // 9000

// Interval SysTick přerušení v [ms] (0 = bez přerušení)
#define SYSTICK_MS     5        // Přerušení každých 5ms

// Počet HCLK cyklů na jedno přerušení
#define SYSTICK_HCLK   (SYSTICK_MS * HCLK_PER_MS)  // 45000
```

### Hardware struktura

```c
typedef struct {
    io32  CTLR;      // 0x00: Control register
    union {
        io8  CNT8[8];  // Byte access
        struct {
            io32 CNT;   // 0x04: Counter LOW
            io32 CNTH;  // 0x08: Counter HIGH
        };
    };
    union {
        io8  CMP8[8];  // Byte access
        struct {
            io32 CMP;   // 0x0C: Compare LOW
            io32 CMPH;  // 0x10: Compare HIGH
        };
    };
} SysTick_t;

#define SysTick ((SysTick_t*)SYSTICK_BASE)  // 0xE000F000
```

## Delay funkce

```c
void WaitClk(u32 clk);         // Čekat [CPU takty] (max 89s)
void WaitUs(u32 us);           // Čekat [mikrosekundy] (max 89s)
void WaitMs(u32 ms);           // Čekat [milisekundy] (max 89s)
```

### Příklady delay

```c
WaitMs(100);    // Čekat 100ms
WaitUs(1000);   // Čekat 1000us (1ms)
WaitClk(9000);  // Čekat 9000 taktů (1ms na 9MHz)
```

## Globální časové proměnné

```c
extern volatile u32 SysTime;    // Čas od startu [ms], overflow po 49 dnech
extern volatile u32 UnixTime;   // Unix čas [sekundy od 1.1.1970]
extern volatile u16 CurTimeMs;  // Aktuální čas v [ms] (0..999)
```

## SysTick přerušení

### IRQ Handler

```c
#define IRQ_SYSTICK    12       // SysTick interrupt ID

#if SYSTICK_MS > 0
HANDLER void SysTick_Handler()
{
    // Nový čas (64-bitový)
    u64 cnt = SysTick_Get64();

    // Načíst aktuální čas
    u32 sys = SysTime;
    u16 ms = CurTimeMs;
    u64 old = SysTick_OldCnt;

    // Zvýšit čas o intervaly
    int dif = (int)(cnt - old);
    while (dif >= SYSTICK_HCLK)
    {
        dif -= SYSTICK_HCLK;
        old += SYSTICK_HCLK;
        sys += SYSTICK_MS;
        ms += SYSTICK_MS;
        if (ms >= 1000)
        {
            UnixTime++;
            ms -= 1000;
        }
    }

    // Uložit nový čas
    SysTime = sys;
    CurTimeMs = ms;
    SysTick_OldCnt = old;

    // Nastavit příští interrupt
    cnt += SYSTICK_HCLK - dif;
    if (dif >= SYSTICK_HCLK - 100*HCLK_PER_US)
        cnt += SYSTICK_HCLK;
    SysTick_SetCmp(cnt);
}
#endif
```

### Konfigurace priority

```c
// Nastavit prioritu pro SysTick
NVIC_IRQPriority(IRQ_SYSTICK, IRQ_PRIO_HIGH);
NVIC_IRQEnable(IRQ_SYSTICK);
```

## Příklady z her

### Příklad 1: FPS Control

```c
// Z: Tiny-Bomber
u32 MemMillis;

void FPS_Control()
{
    u32 ms = (u32)44 * HCLK_PER_MS;  // 44ms pro ~23 FPS
    u32 start = MemMillis;
    u32 t;
    while ((u32)((t = Time()) - start) < ms) {}  // Aktivní čekání
    MemMillis = t;
}

// V main loop:
void loop()
{
    u32 MemMillis = 0;
    while (1)
    {
        UpdateGame();
        FPS_Control();
    }
}
```

### Příklad 2: Časový delta

```c
// Z: TV Tennis
u32 OldTime;
int TimeDelta;

void GameLoop()
{
    u32 NewTime = Time();
    TimeDelta = (int)(NewTime - OldTime);  // Čas od posledního frame
    OldTime = NewTime;

    // Time-based pohyb
    int BallVelocity = (2 * COORD_MUL * TimeDelta) / HCLK_PER_MS;
    BallX += BallVelocity;
}
```

### Příklad 3: Zvuk s přesným časováním

```c
// Z: TBert
void Sound(uint8_t freq, uint8_t dur)
{
    if (freq == 0)
        WaitMs(dur);  // Ticho
    else
    {
        PlayTone(510 - 2*freq);
        int n = (510 - 2*freq) * dur;
        WaitUs(n);    // Čekání v mikrosekundách
        StopSound();
    }
}
```

### Příklad 4: Random seed

```c
// Z: Train Game
u32 LastTime;

void GameStep()
{
    u32 CurrentTime = Time();

    // Seed random generátoru časem
    RandSeed += Time() + DispFrame + DispLine;

    // Kontrola taktovací frekvence
    if ((u32)(CurrentTime - LastTime) >= HCLK_PER_MS * 50)  // Každých 50ms
    {
        LastTime = CurrentTime;
        UpdateGameState();
    }
}
```

### Příklad 5: Jednoduché aliasy

```c
// V main.h pro hry
INLINE void delay(int ms) { WaitMs(ms); }
INLINE void _delay_ms(int ms) { WaitMs(ms); }

// Použití
delay(100);         // Čekat 100ms
_delay_ms(50);      // Čekat 50ms
```

## API Reference

| Funkce | Typ | Popis |
|--------|-----|-------|
| `SysTick_Init()` | void | Inicializovat SysTick |
| `SysTick_Reset()` | void | Resetovat SysTick |
| `SysTick_Enable()` | inline | Zapnout čítač |
| `SysTick_Disable()` | inline | Vypnout čítač |
| `SysTick_Get()` | u32 | Přečíst čítač (32-bit) |
| `SysTick_Get64()` | u64 | Přečíst čítač (64-bit, safe) |
| `SysTick_Set(u64)` | void | Nastavit čítač |
| `SysTick_SetCmp(u64)` | void | Nastavit compare |
| `Time()` | u32 | Aktuální čas [takty] |
| `Time64()` | u64 | Aktuální čas 64-bit |
| `WaitClk(u32)` | void | Čekat [takty] |
| `WaitUs(u32)` | void | Čekat [mikrosekundy] |
| `WaitMs(u32)` | void | Čekat [milisekundy] |

## Globální proměnné

| Proměnná | Typ | Popis |
|----------|-----|-------|
| `SysTime` | u32 volatile | Čas od startu [ms] |
| `CurTimeMs` | u16 volatile | Čas v [ms] 0-999 |
| `UnixTime` | u32 volatile | Unix čas [sekundy] |

## Praktické tipy

### Overflow správa

```c
// SysTime overflows po 49 dnech - vždy používejte rozdíl!
u32 time1 = SysTime;
u32 time2 = SysTime;
u32 elapsed = time2 - time1;  // SPRÁVNĚ - funguje i přes overflow
```

### Interrupt-safe čtení 64-bitů

```c
// Použijte SysTick_Get64() - je interrupt-safe
u64 time = SysTick_Get64();    // SPRÁVNĚ

// Nepoužívejte:
u64 time = SysTick->CNT | ((u64)SysTick->CNTH << 32);  // ŠPATNĚ
```

### FPS synchronizace

```c
// Pro 60 FPS
#define FPS_60_MS (HCLK_PER_MS * (1000/60))  // ~16ms

u32 last = 0;
while (1)
{
    u32 now = Time();
    if ((u32)(now - last) >= FPS_60_MS)
    {
        last = now;
        UpdateFrame();
    }
}
```

### Non-blocking delay

```c
// Špatně - blocking
void Update() {
    DoSomething();
    WaitMs(100);  // Blokuje!
    DoMore();
}

// Správně - non-blocking
u32 lastTime = 0;
void Update() {
    DoSomething();

    u32 now = Time();
    if ((u32)(now - lastTime) >= HCLK_PER_MS * 100) {
        lastTime = now;
        DoMore();
    }
}
```

## Časové konstanty

### Frekvence podle MCU

| MCU | HSI | HCLK_PER_US | HCLK_PER_MS |
|-----|-----|-------------|-------------|
| **CH32V003** | 24MHz | 3 | 3000 |
| **CH32V103** | 72MHz | 9 | 9000 |
| **CH32V2** | 72MHz | 9 | 9000 |

### Přesnost

- **Rozlišení:** 1/8 HCLK cyklu (WCH hardware divider)
- **Range:** 0 až 2^64 cyklů (~73 let při 72MHz)
- **Overflow:** SysTime po 49 dnech, Time64() nikdy

## Klíčové poznámky

1. **SysTick běží vždy** - I když SYSTICK_MS = 0
2. **64-bit čítač** - Použijte SysTick_Get64() pro long-running aplikace
3. **Interrupt priority** - Nastavte HIGH pro přesný timing
4. **Aktivní čekání** - WaitMs() neusíná, CPU běží na 100%
5. **Frame delta** - Vždy používejte time delta pro smooth animace

## Související dokumentace

- [IRQ](irq.md) - SysTick interrupt handling
- [Timer](timer.md) - Hardware timery pro přesné timing
- [Game Patterns](game_patterns.md) - FPS control a game loops
- [Game Examples](game_examples.md) - Praktické příklady z her
