# RCC - Reset and Clock Control

## Úvod

CH32LibSDK poskytuje komplexní API pro Reset and Clock Control (RCC) mikrokontrolérů CH32. RCC modul řídí systémové hodiny, PLL, oscilátory a periferní clock enable/disable. Tento dokument pokrývá všechny funkce pro konfiguraci hodinových signálů.

## Oscilátory

### HSI (High-Speed Internal)

```c
void RCC_HSIEnable(void);      // Povolit HSI
void RCC_HSIDisable(void);     // Zakázat HSI
Bool RCC_HSIIsStable(void);    // Je HSI stabilní?
```

**Frekvence:**
- **CH32V003**: 24 MHz
- **CH32V103/V2**: 8 MHz

### HSE (High-Speed External)

```c
void RCC_HSEEnable(void);      // Povolit HSE
void RCC_HSEDisable(void);     // Zakázat HSE
Bool RCC_HSEIsStable(void);    // Je HSE stabilní?
Bool RCC_HSEStart(Bool wait);  // Spustit HSE s/bez čekání
```

**Frekvence:**
- **CH32V003**: 4-25 MHz (typicky 24 MHz)
- **CH32V103/V2**: 8-24 MHz (typicky 8 nebo 16 MHz)

### LSI (Low-Speed Internal)

```c
void RCC_LSIEnable(void);      // Povolit LSI
void RCC_LSIDisable(void);     // Zakázat LSI
Bool RCC_LSIIsStable(void);    // Je LSI stabilní?
```

**Frekvence:**
- **CH32V003**: 128 kHz
- **CH32V103/V2**: 40 kHz

### LSE (Low-Speed External)

```c
void RCC_LSEEnable(void);      // Povolit LSE (pouze V103/V2)
void RCC_LSEDisable(void);     // Zakázat LSE
Bool RCC_LSEIsStable(void);    // Je LSE stabilní?
```

**Frekvence:** 32768 Hz (32.768 kHz)

## PLL Konfigurace

PLL (Phase-Locked Loop) násobí vstupní frekvenci pro dosažení vyšší systémové frekvence.

### Základní funkce

```c
void RCC_PLLEnable(void);      // Povolit PLL
void RCC_PLLDisable(void);     // Zakázat PLL
Bool RCC_PLLIsReady(void);     // Je PLL připraven?
```

### Nastavení PLL

```c
// Zdroj PLL
void RCC_PLLSrc(u8 src);
// 0 = HSI, 1 = HSE

// Multiplikátor (2-18)
void RCC_PLLMul(u8 mul);

// Dělitel (pouze V103/V2)
void RCC_PLLHSEDiv1(void);     // HSE / 1
void RCC_PLLHSEDiv2(void);     // HSE / 2
void RCC_PLLHSIDiv1(void);     // HSI / 1
void RCC_PLLHSIDiv2(void);     // HSI / 2
```

### Příklad: Konfigurace PLL

```c
// HSE (8MHz) * 9 = 72MHz
RCC_HSEEnable();
while (!RCC_HSEIsStable()) {}

RCC_PLLSrc(1);          // HSE
RCC_PLLHSEDiv1();       // HSE / 1 = 8MHz
RCC_PLLMul(9);          // 8MHz * 9 = 72MHz
RCC_PLLEnable();

while (!RCC_PLLIsReady()) {}

RCC_SysClkSrc(RCC_SYSCLKSRC_PLL);
```

## System Clock

### Výběr zdroje

```c
#define RCC_SYSCLKSRC_HSI  0    // HSI přímý
#define RCC_SYSCLKSRC_HSE  1    // HSE přímý
#define RCC_SYSCLKSRC_PLL  2    // PLL násobená frekvence

void RCC_SysClkSrc(u8 src);    // Nastavit zdroj
u8 RCC_SysClk(void);           // Získat aktuální zdroj
```

### AHB Divider

```c
void RCC_AHBDiv(u8 div);       // Nastavit AHB dělitel (1-512)
```

**Hodnoty:** 1, 2, 3, 4, 8, 16, 64, 128, 256, 512

### APB Dividers

```c
void RCC_APB1Div(u8 div);      // APB1 dělitel (1-16)
void RCC_APB2Div(u8 div);      // APB2 dělitel (1-16)
```

**Hodnoty:** 1, 2, 4, 8, 16

## Periferní Clock Enable/Disable

### Jednotlivé periferie

```c
// AHB
void RCC_DMA1ClkEnable(void);
void RCC_SRAMClkEnable(void);

// APB2
void RCC_PAClkEnable(void);    // Port A
void RCC_PBClkEnable(void);    // Port B
void RCC_PCClkEnable(void);    // Port C
void RCC_PDClkEnable(void);    // Port D
void RCC_TIM1ClkEnable(void);
void RCC_ADC1ClkEnable(void);
void RCC_SPI1ClkEnable(void);
void RCC_USART1ClkEnable(void);

// APB1
void RCC_TIM2ClkEnable(void);
void RCC_I2C1ClkEnable(void);
void RCC_USBFSClkEnable(void);
void RCC_PWRClkEnable(void);
```

### Skupinové operace

```c
// Enable více periférií najednou
void RCC_APB1ClkEnable(u32 mask);
void RCC_APB2ClkEnable(u32 mask);
void RCC_AHBClkEnable(u32 mask);

// Příklad
RCC_APB2ClkEnable(RCC_APB2CLK_PA | RCC_APB2CLK_USART1);
```

## ADC Clock Divider

```c
void RCC_ADCDiv(u8 div);       // Nastavit ADC dělitel
u8 RCC_GetADCDivVal(void);     // Získat aktuální hodnotu
```

**CH32V003:** AHBCLK / {2, 4, 6, 8, 12, 16, 24, 32, 48, 64, 96, 128}
**CH32V103/V2:** PCLK2 / {2, 4, 6, 8} (max 14MHz)

## Automatická inicializace

SDK automaticky inicializuje clock při startu pomocí `SystemInit()`.

### Konfigurace přes define

```c
// V config.h nebo Makefile
#define SYSCLK_SRC  5          // PLL_HSE
#define PLLCLK_MUL  2          // HSE * 2
#define SYSCLK_DIV  1          // AHB / 1
#define APB1CLK_DIV 1          // APB1 / 1
#define APB2CLK_DIV 1          // APB2 / 1
#define ADCCLK_DIV  2          // ADC / 2
```

**SYSCLK_SRC hodnoty:**
- `1` = HSI
- `2` = HSE
- `3` = HSE_Bypass
- `4` = PLL_HSI
- `5` = PLL_HSE

## Získání frekvencí

```c
#define CLK_SYSCLK  0          // System clock
#define CLK_HCLK    1          // AHB clock
#define CLK_PCLK1   2          // APB1 clock
#define CLK_PCLK2   3          // APB2 clock
#define CLK_ADCCLK  4          // ADC clock

u32 RCC_GetFreq(int clk);      // Získat frekvenci v Hz

// Příklad
u32 sysclk = RCC_GetFreq(CLK_SYSCLK);
u32 apb1clk = RCC_GetFreq(CLK_PCLK1);
```

## Reset periférií

```c
void RCC_APB1Reset(u32 mask);  // Reset periferie na APB1
void RCC_APB2Reset(u32 mask);  // Reset periferie na APB2

// Příklad
RCC_APB2Reset(RCC_APB2CLK_USART1);
```

## Příklady použití

### Příklad 1: Výchozí konfigurace (72MHz)

```c
// Automatická inicializace v SystemInit():
// HSI(8MHz) -> PLL x9 -> 72MHz SYSCLK

int main(void) {
    u32 freq = RCC_GetFreq(CLK_HCLK);
    // freq = 72000000
}
```

### Příklad 2: Vlastní clock (48MHz z HSE)

```c
// config.h
#define SYSCLK_SRC  5          // PLL_HSE
#define PLLCLK_MUL  2          // 24MHz HSE * 2 = 48MHz

// main.c
int main(void) {
    // RCC_ClockInit() již zavolána v startup
    u32 freq = RCC_GetFreq(CLK_SYSCLK);
    // freq = 48000000
}
```

### Příklad 3: Dynamická změna na HSE

```c
int main(void) {
    // Spustit HSE
    if (RCC_HSEStart(False)) {
        // Přepnout na HSE
        RCC_SysClkSrc(RCC_SYSCLKSRC_HSE);

        // Čekat na přepnutí
        while (RCC_SysClk() != RCC_SYSCLKSRC_HSE) {}

        // Aktualizovat frekvence
        RCC_UpdateFreq();
    }
}
```

### Příklad 4: Šetření energie

```c
void LowPower_Mode(void) {
    // Vypnout nepoužívané periferie
    RCC_APB1ClkDisable(~0);  // Vypnout všechny APB1
    RCC_APB2ClkDisable(RCC_APB2CLK_TIM1 | RCC_APB2CLK_ADC1);

    // Snížit frekvenci
    RCC_AHBDiv(8);           // Snížit HCLK na 1/8

    // Aktualizovat timing
    RCC_UpdateFreq();
}
```

### Příklad 5: USB clock (48MHz)

```c
void USB_Init(void) {
    // USB vyžaduje 48MHz
    // 72MHz SYSCLK / 1.5 = 48MHz

    RCC_USBDiv(RCC_USBDIV_1_5);
    RCC_USBFSClkEnable();
}
```

## Clock Tree

```
HSI (8/24MHz) ───┐
                 ├──→ PLL ×2-18 ──→ SYSCLK ──→ AHB /1-512 ──→ HCLK
HSE (8-24MHz) ───┘                     │
                                       ├──→ APB1 /1-16 ──→ PCLK1
                                       ├──→ APB2 /1-16 ──→ PCLK2
                                       └──→ ADC /2-128 ──→ ADCCLK

LSI (40kHz/128kHz) ──→ RTC, IWDG
LSE (32.768kHz) ─────→ RTC
```

## Důležité poznámky

1. **PLL konfigurace** - Může být nastavena jen když je PLL vypnutý
2. **HSE startup** - Může trvat několik ms, použijte `RCC_HSEStart()`
3. **Flash latence** - Musí být nastavena podle SYSCLK frekvence
4. **USB clock** - Vyžaduje přesně 48MHz
5. **ADC clock** - Max 14MHz na V103/V2

## Časté konfigurace

| MCU | HSE | PLL | SYSCLK | HCLK | APB1 | APB2 |
|-----|-----|-----|--------|------|------|------|
| **V003** | 24MHz | ×1 | 24MHz | 24MHz | 24MHz | 24MHz |
| **V103** | 8MHz | ×9 | 72MHz | 72MHz | 36MHz | 72MHz |
| **V2** | 8MHz | ×9 | 72MHz | 72MHz | 36MHz | 72MHz |

## Související dokumentace

- [PWR](pwr.md) - Power management
- [FLASH](flash.md) - Flash latence
- [Timer](timer.md) - Timer clock source
