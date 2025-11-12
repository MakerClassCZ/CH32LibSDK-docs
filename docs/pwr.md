# PWR - Řízení napájení

## Úvod

CH32LibSDK poskytuje API pro power management (PWR) mikrokontrolérů CH32. PWR modul řídí napájecí režimy, detektory napětí, a nízkoenergetické režimy (Sleep, Stop, Standby). Tento dokument pokrývá všechny funkce pro optimalizaci spotřeby energie.

## Power Management - Režimy

### Režimy napájecího regulátoru

**CH32V2 a CH32V103:**
```c
void PWR_LPDSNorm(void);     // Normální mód (výchozí)
void PWR_LPDSLow(void);      // Režim nízké spotřeby
```

**CH32V00X:**
```c
#define PWR_LDOMODE_LOW     1  // Low-power, LDO 1.0V
#define PWR_LDOMODE_NORMAL  2  // Normal, LDO 1.2V
#define PWR_LDOMODE_SAVE    3  // Energy saving, LDO 1.0V

void PWR_LDOMode(int mode);
```

**CH32L103:**
```c
void PWR_LDONormal(void);   // Normální režim
void PWR_LDOAuto(void);     // Auto-saving režim
void PWR_LDOEnable(void);   // Povolit úsporu energie
void PWR_LDODisable(void);  // Zakázat
```

### Režim hluboké spánku

```c
void PWR_PDDSSleep(void);     // Vyber režim Stop (výchozí)
void PWR_PDDSStandby(void);   // Vyber režim Standby

// Vstup do režimu Standby
void PWR_EnterStandby(void);
```

### Příznak probuzení

```c
void PWR_WUFClr(void);        // Vymazat příznak probuzení
Bool PWR_WUFIsSet(void);      // Zkontrolovat příznak

void PWR_SBFClr(void);        // Vymazat příznak standby
Bool PWR_SBFIsSet(void);      // Zkontrolovat standby
```

## Detektory napětí (PVD)

PVD (Power Voltage Detector) monitoruje napájecí napětí a generuje přerušení při poklesu pod práh.

### Prahové hodnoty

**CH32V003:**
```c
#define PWR_PVDLEVEL_2V9   0  // 2.85V ↑, 2.70V ↓
#define PWR_PVDLEVEL_3V1   1  // 3.05V ↑, 2.90V ↓
#define PWR_PVDLEVEL_3V3   2  // 3.30V ↑, 3.15V ↓
#define PWR_PVDLEVEL_3V5   3  // 3.50V ↑, 3.30V ↓
#define PWR_PVDLEVEL_3V7   4  // 3.70V ↑, 3.50V ↓
#define PWR_PVDLEVEL_3V9   5  // 3.90V ↑, 3.70V ↓
#define PWR_PVDLEVEL_4V1   6  // 4.10V ↑, 3.90V ↓
#define PWR_PVDLEVEL_4V4   7  // 4.40V ↑, 4.20V ↓
```

**CH32V103:**
```c
#define PWR_PVDLEVEL_2V7   0  // 2.65V ↑, 2.50V ↓
#define PWR_PVDLEVEL_2V9   1  // 2.87V ↑, 2.70V ↓
#define PWR_PVDLEVEL_3V1   2  // 3.07V ↑, 2.89V ↓
#define PWR_PVDLEVEL_3V3   3  // 3.27V ↑, 3.08V ↓
#define PWR_PVDLEVEL_3V5   4  // 3.46V ↑, 3.27V ↓
#define PWR_PVDLEVEL_3V8   5  // 3.76V ↑, 3.55V ↓
#define PWR_PVDLEVEL_4V1   6  // 4.07V ↑, 3.84V ↓
#define PWR_PVDLEVEL_4V4   7  // 4.43V ↑, 4.13V ↓
```

### Funkce PVD

```c
void PWR_PVDEnable(void);      // Povolit detektor
void PWR_PVDDisable(void);     // Zakázat detektor
void PWR_PVDLevel(int level);  // Nastavit prahovou hladinu
Bool PWR_PVDLow(void);         // Je napětí pod prahem?
```

### Příklad: Detekce nízkého napětí

```c
void PWR_Init(void) {
    PWR_PVDEnable();
    PWR_PVDLevel(PWR_PVDLEVEL_3V3);  // Práh 3.3V
}

int main(void) {
    PWR_Init();

    while (1) {
        if (PWR_PVDLow()) {
            // Napětí kleslo pod práh!
            // Vypnout periferie
            // Zavolat backup proceduru
        }
        delay_ms(100);
    }
}
```

## Flash režim (CH32L103, CH32V00X, CH32V003)

```c
void PWR_FlashLPEnable(void);   // Povolit Flash nízkovýkonný režim
void PWR_FlashLPDisable(void);  // Zakázat
void PWR_FlashIdle(void);       // Flash na režim nečinnosti
void PWR_FlashSleep(void);      // Flash na režim spánku
Bool PWR_FlashIsLP(void);       // Je Flash v nízkovýkonném režimu?
```

## RTC a Backup registry

```c
void PWR_WRTCEnable(void);     // Povolit zápis do RTC a backup
void PWR_WRTCDisable(void);    // Zakázat zápis
```

## Wake-up pin

```c
void PWR_WKUPEnable(void);     // Povolit WKUP pin pro probuzení
void PWR_WKUPDisable(void);    // Zakázat WKUP pin
```

## Auto-wakeup (CH32V003, CH32L103, CH32V00X)

Auto-wakeup automaticky probouzí MCU po zadané době.

```c
void PWR_AWUEnable(void);      // Povolit auto-wakeup
void PWR_AWUDisable(void);     // Zakázat
void PWR_AWUCmp(int comp);     // Nastavit okno srovnání (0..63)
void PWR_AWUPsc(int psc);      // Nastavit prescaler
```

### Prescaler hodnoty

```c
#define PWR_AWUPSC_OFF      0    // Vypnutý
#define PWR_AWUPSC_1        1    // Bez dělení
#define PWR_AWUPSC_2        2    // Děleno 2
#define PWR_AWUPSC_4        3    // Děleno 4
// ... až ...
#define PWR_AWUPSC_61440    15   // Děleno 61440
```

### Příklad: Auto-wakeup timer

```c
void PWR_Setup_AutoWakeup(uint16_t seconds) {
    // LSI frekvence = 40kHz (typ)
    // Nastavit prescaler (např. děleno 256)
    PWR_AWUPsc(PWR_AWUPSC_256);

    // Frekvence po prescaleru: 40kHz / 256 = 156.25 Hz
    // Čas jednoho ticku: 6.4 ms

    // Počet ticků pro N sekund
    int ticks = (seconds * 1000) / 6.4;
    if (ticks > 63) ticks = 63;

    // Nastavit porovnávací hodnotu
    PWR_AWUCmp(ticks);

    // Povolit auto-wakeup
    PWR_AWUEnable();

    // Vstoupit do Standby
    PWR_EnterStandby();
}
```

## RAM standby napájení (CH32L103)

```c
// 2K RAM
void PWR_R2KSTYEnable(void);   // Povolit standby napájení
void PWR_R2KVBATEnable(void);  // Povolit VBAT napájení

// 18K RAM
void PWR_R18KSTYEnable(void);
void PWR_R18KVBATEnable(void);

// RAM low voltage
void PWR_RAMLVEnable(void);    // Povolit nízké napětí
void PWR_RAMLVDisable(void);
```

## Režimy spánku

### Stop režim

```c
void Enter_Stop_Mode(void) {
    PWR_PDDSSleep();           // Nastavit režim Stop
    PWR_LPDSLow();             // Nízká spotřeba regulátoru
    WFI();                     // Wait For Interrupt
}
```

### Standby režim

```c
void Enter_Standby_Mode(void) {
    PWR_PDDSStandby();         // Nastavit režim Standby
    PWR_WKUPEnable();          // Povolit probuzení z WKUP pinu
    PWR_EnterStandby();        // Vstoupit do Standby

    // MCU se probere zde
    if (PWR_SBFIsSet()) {
        PWR_SBFClr();
        // Inicializace po probuzení
    }
}
```

## Spotřeba energie - Porovnání

| Režim | Spotřeba | Probuzení | RAM | Periferie |
|-------|----------|-----------|-----|-----------|
| **Run** | ~10mA | - | ✓ | ✓ |
| **Sleep** | ~5mA | Any IRQ | ✓ | ✓ |
| **Stop** | ~50µA | EXTI, RTC | ✓ | ✗ |
| **Standby** | ~2µA | WKUP, RTC | ✗ | ✗ |

## Porovnání variant

| Funkce | CH32V2 | CH32V103 | CH32V003 | CH32L103 | CH32V00X |
|--------|--------|----------|----------|----------|----------|
| PWR_LPDSLow | ✓ | ✓ | ✗ | ✓ | ✗ |
| PWR_PVDEnable | ✓ | ✓ | ✓ | ✓ | ✓ |
| PWR_FlashLPEnable | ✗ | ✗ | ✓ | ✓ | ✓ |
| PWR_AWUEnable | ✗ | ✗ | ✓ | ✓ | ✓ |
| PWR_RAMxxx | ✗ | ✗ | ✗ | ✓ | ✗ |
| PWR_LDOMode | ✗ | ✗ | ✗ | ✗ | ✓ |

## Příklady použití

### Příklad 1: Bateriový režim

```c
void Battery_Mode(void) {
    // Nastavit nízkovýkonný režim
    PWR_LPDSLow();

    // Povolit PVD monitoring
    PWR_PVDEnable();
    PWR_PVDLevel(PWR_PVDLEVEL_2V9);

    // Flash low-power
    PWR_FlashLPEnable();
    PWR_FlashSleep();

    // Vypnout nepoužívané periferie
    RCC_APB1ClkDisable(~0);  // Vypnout všechno na APB1
    RCC_APB2ClkDisable(~0);  // Vypnout všechno na APB2

    // Vstoupit do Stop režimu
    PWR_PDDSSleep();
    WFI();
}
```

### Příklad 2: Periodické probouzení

```c
void Periodic_Wakeup(void) {
    // Nastavit auto-wakeup na 10 sekund
    PWR_AWUPsc(PWR_AWUPSC_1024);
    PWR_AWUCmp(30);  // ~10 sekund
    PWR_AWUEnable();

    // Vstoupit do Standby
    PWR_EnterStandby();

    // Po probuzení
    if (PWR_WUFIsSet()) {
        PWR_WUFClr();
        // Provést úlohu
        Do_Task();
        // Zpět do sleep
        Periodic_Wakeup();
    }
}
```

## Klíčové poznámky

1. **Standby režim** - Nejnižší spotřeba, vymazání RAM
2. **Stop režim** - Střední spotřeba, zachování RAM
3. **PVD monitoring** - Důležité pro bateriové aplikace
4. **Flash low-power** - Snižuje spotřebu v sleep režimech
5. **Auto-wakeup** - Periodické probouzení bez external interrupt

## Související dokumentace

- [IRQ](irq.md) - Probuzení přes přerušení
- [RCC](rcc.md) - Clock management pro úsporu energie
- [GPIO](gpio.md) - WKUP pin konfigurace
