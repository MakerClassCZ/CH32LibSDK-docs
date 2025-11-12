# FLASH paměť - Práce s interní FLASH pamětí

## Úvod

CH32LibSDK poskytuje jednotné API pro práci s interní FLASH pamětí mikrokontrolérů CH32. FLASH paměť se používá pro ukládání programu a perzistentních dat. Tento dokument pokrývá čtení, zápis, mazání a správu FLASH paměti.

## Základní informace

### Velikosti stránek podle variant

| MCU | Stránka Program | Stránka Erase | FLASH Velikost |
|-----|----------------|---------------|----------------|
| **CH32V002** | 256 B (0x100) | 256 B | 16 KB |
| **CH32V003** | 64 B (0x40) | 1 KB | 16 KB |
| **CH32V103** | 128 B (0x80) | 1 KB | 64 KB |
| **CH32V00x** | 256 B (0x100) | 1 KB | 64 KB |
| **CH32V03x** | 256 B (0x100) | 1 KB | 62 KB |
| **CH32L103** | 128 B (0x80) | 1 KB | 64 KB |

### Paměťová mapa

```c
#define FLASH_BASE      0x08000000      // Začátek FLASH paměti
#define FLASHSIZE       (16*1024)       // Velikost FLASH (příklad pro V002)
```

## Čtení z FLASH

FLASH paměť se čte přímo bez potřeby odemykání nebo speciálních příkazů.

### Přímé čtení

```c
// Čtení z FLASH jako z běžné paměti
void Flash_ReadData(u32 address, u32* buffer, int size) {
    const u32* src = (const u32*)address;
    for (int i = 0; i < size / 4; i++) {
        buffer[i] = src[i];
    }
}

// Nebo jednodušeji s memcpy
void Flash_ReadData_Simple(u32 address, void* buffer, int size) {
    memcpy(buffer, (const void*)address, size);
}
```

### Ověření dat v FLASH

```c
// Flash_Verify() - vrací NULL pokud se data shodují, nebo adresu prvního rozdílu
const u32* Flash_Verify(u32 addr, const u32* src, u32 size);

// Příklad
u32 data[64];
const u32* diff = Flash_Verify(FLASH_ADDR, data, 256);
if (diff == NULL) {
    printf("Data se shodují\n");
} else {
    printf("Rozdíl na adrese: 0x%08X\n", (u32)diff);
}
```

### Hledání volné FLASH paměti

```c
// Flash_Check() - najde začátek nevymazeného prostoru
const u32* Flash_Check(u32 addr, u32 size);

// Příklad
const u32* free = Flash_Check(FLASH_BASE, FLASHSIZE);
if (free == NULL) {
    printf("Veškerá FLASH je vymazána\n");
} else {
    printf("Volné místo od: 0x%08X\n", (u32)free);
}

// Hledání ve všech FLASH
const u32* Flash_CheckAll(void);
```

## Zápis do FLASH

**POZOR: FLASH MUSÍ BÝT NEJDŘÍVE VYMAZÁNA!**

### Hlavní funkce pro zápis

```c
Bool Flash_Program(u32 addr, const u32* src, u32 size, int ms);
```

**Parametry:**
- `addr` - Cílová adresa v FLASH (musí být zarovnána na stránku)
- `src` - Zdrojová data (musí být zarovnána na u32)
- `size` - Velikost dat (musí být násobek stránky)
- `ms` - Timeout v milisekundách na stránku
- **Vrací:** `True` pokud OK, `False` při chybě

### Kompletní příklad zápisu

```c
#include "sdk_flash.h"

void Write_Data_To_Flash(u32 flash_addr, const u8 *data, u32 size) {
    // 1. Zkontrolovat zarovnání a velikost
    if (flash_addr % FLASH_PAGEPG_SIZE != 0) {
        printf("Chyba: Adresa není zarovnána na stránku!\n");
        return;
    }
    if (size % FLASH_PAGEPG_SIZE != 0) {
        printf("Chyba: Velikost není násobek stránky!\n");
        return;
    }

    // 2. Vymazat FLASH (musí být před zápisem!)
    if (!Flash_Erase(flash_addr, size, 1000)) {
        printf("Chyba: Mazání FLASH selhalo!\n");
        return;
    }
    printf("FLASH vymazána\n");

    // 3. Zapsat data
    if (!Flash_Program(flash_addr, (const u32*)data, size, 1000)) {
        printf("Chyba: Zápis do FLASH selhal!\n");
        return;
    }
    printf("Data zapsána do FLASH\n");

    // 4. Ověřit data
    const u32* verify_error = Flash_Verify(flash_addr, (const u32*)data, size);
    if (verify_error == NULL) {
        printf("Ověření OK: Data se shodují\n");
    } else {
        printf("Chyba ověření na adrese: 0x%08X\n", (u32)verify_error);
    }
}
```

## Mazání FLASH

### Hlavní funkce pro mazání

```c
Bool Flash_Erase(u32 addr, u32 size, int ms);
```

**Parametry:**
- `addr` - Počáteční adresa (musí být zarovnána na stránku)
- `size` - Velikost k vymazání (musí být násobek stránky)
- `ms` - Timeout v milisekundách na stránku
- **Vrací:** `True` pokud OK, `False` při chybě

### Příklad mazání

```c
void Erase_Flash_Sector(u32 flash_addr, u32 sector_size) {
    // 1. Zkontrolovat zarovnání
    if (flash_addr % FLASH_PAGEER_SIZE != 0) {
        printf("Chyba: Adresa není zarovnána!\n");
        return;
    }

    // 2. Vymazat sektor
    printf("Vmazání FLASH od 0x%08X, velikost: %d bajtů\n",
           flash_addr, sector_size);

    if (!Flash_Erase(flash_addr, sector_size, 1000)) {
        printf("Chyba: Mazání FLASH selhalo!\n");
        return;
    }

    printf("Mazání FLASH OK\n");

    // 3. Ověřit, že je oblast vymazána
    const u32* non_empty = Flash_Check(flash_addr, sector_size);
    if (non_empty == NULL) {
        printf("Ověření OK: Oblast je zcela vymazána\n");
    } else {
        printf("Chyba: Nevymazaná data v FLASH na 0x%08X\n", (u32)non_empty);
    }
}

// Příklady
void Test_Erase() {
    // Vymazat jednu stránku (256B na V002)
    Flash_Erase(0x08010000, 256, 1000);

    // Vymazat 4 stránky (1KB)
    Flash_Erase(0x08010000, 1024, 1000);

    // Vymazat 4KB sektor
    Flash_Erase(0x08010000, 4096, 1000);
}
```

## Zámky a odemykání

### Základní funkce

```c
// Klíče pro odemykání
#define FLASH_KEY1  0x45670123
#define FLASH_KEY2  0xCDEF89AB

// Odemykání a zamykání
void Flash_Unlock(void);           // Odemknout FLASH řadič
INLINE void Flash_Lock(void);      // Zamknout FLASH řadič
INLINE Bool Flash_Locked(void);    // Zkontrolovat, zda je zamčeno

void Flash_FastUnlock(void);       // Odemknout rychlý FLASH
INLINE void Flash_FastLock(void);  // Zamknout rychlý FLASH
```

**Poznámka:** Funkce `Flash_Program()` a `Flash_Erase()` automaticky odemykají a zamykají FLASH.

## Option Bytes

Option bytes konfigurují hardware MCU (ochrana, watchdog, reset).

### Struktura Option Bytes

```c
typedef struct {
    u16 RDP;        // Read protection
    u16 USER;       // User options
    u16 Data0;      // User data byte 0
    u16 Data1;      // User data byte 1
    u16 WRP0;       // Write protection 0
    u16 WRP1;       // Write protection 1
    u16 WRP2;       // Write protection 2
    u16 WRP3;       // Write protection 3
} OB_t;
```

### Operace s Option Bytes

```c
// Přečíst Option Bytes
void Flash_OBRead(OB_t *ob);

// Zapsat Option Bytes (automaticky smaže a zapíše)
void Flash_OBWrite(const OB_t *ob);

// Vymazat Option Bytes
void Flash_OBErase(void);

// Ověřit Option Bytes
Bool Flash_OBVerify(OB_t *ob);
```

### Kontrolní funkce

```c
INLINE Bool Flash_GetReadProt(void);    // Čtení chráněno?
INLINE Bool Flash_GetIWDGEnabled(void); // IWDG povolen?
INLINE Bool Flash_GetStop(void);        // Reset v STOP režimu?
INLINE Bool Flash_GetStandby(void);     // Reset v STANDBY režimu?
INLINE u8 Flash_GetData0(void);         // Uživatelská data byte 0
INLINE u8 Flash_GetData1(void);         // Uživatelská data byte 1
INLINE u32 Flash_GetWProt(void);        // Ochrana před zápisem
```

### Příklad: Konfigurace GPIO z Option Bytes

```c
// Z babypad_key.c - Přepnout PD7 na GPIO režim
void KeyInit()
{
    // Přepnout PD7 na GPIO režim, pokud ještě není
    if (((OB->USER >> 3) & 3) != 3) {
        OB_t ob;

        // Přečíst aktuální option bytes
        Flash_OBRead(&ob);

        // Zakázat RESET funkci na PD7
        ob.USER |= B3|B4;

        // Zapsat nové option bytes
        Flash_OBWrite(&ob);
    }

    // Inicializovat GPIO piny
    GPIO_Mode(PD7, GPIO_MODE_IN_PU);
}
```

## Kontrola stavu

```c
// Kontrola stavů operací
INLINE Bool Flash_Busy(void);           // Běží operace?
INLINE Bool Flash_WProtErr(void);       // Chyba ochrany proti zápisu?
INLINE void Flash_WProtErrClr(void);    // Vyčistit chybu ochrany
INLINE Bool Flash_EndOp(void);          // Operace skončena?
INLINE void Flash_EndOpClr(void);       // Vyčistit příznak konce
```

## Latence (Wait States)

Pro správnou funkci FLASH při různých frekvencích:

```c
// Nastavit počet čekacích stavů (závisí na taktovací frekvenci)
INLINE void Flash_Latency(int wait);

// CH32V2:
//   0 wait (<=40MHz)
//   1 wait (>40MHz, <=72MHz)
//   2 wait (>72MHz)

// CH32V103:
//   0 wait (<=24MHz)
//   1 wait (>24MHz, <=48MHz)
//   2 wait (>48MHz)

// CH32V003:
//   0 wait (<=24MHz)
//   1 wait (>=24MHz)
```

## Přerušení

```c
// Enable/Disable přerušení
INLINE void Flash_ErrIntEnable(void);   // Přerušení chyby
INLINE void Flash_ErrIntDisable(void);

INLINE void Flash_EndOpIntEnable(void); // Přerušení konce operace
INLINE void Flash_EndOpIntDisable(void);
```

## Kompletní příklad - Ukládání konfigurace

```c
typedef struct {
    u32 magic;      // 0x12345678
    u8 version;     // Verze
    u8 reserved[3];
    u32 checksum;
} FLASH_CONFIG;

// Adresa v FLASH (zarovnána na 256 bajtů)
#define CONFIG_FLASH_ADDR   0x0800F000

void ConfigSave(const FLASH_CONFIG *cfg) {
    u32 buffer[64];  // 256 bajtů = 64 slova

    // 1. Příprava dat (zarovnání paměti)
    memcpy(buffer, cfg, sizeof(FLASH_CONFIG));
    memset((u8*)buffer + sizeof(FLASH_CONFIG), 0xFF,
           256 - sizeof(FLASH_CONFIG));

    // 2. Vymazat FLASH
    printf("1. Vmazání FLASH...\n");
    if (!Flash_Erase(CONFIG_FLASH_ADDR, 256, 1000)) {
        printf("  Chyba: Mazání selhalo!\n");
        return;
    }

    // 3. Zapsat konfiguraci
    printf("2. Zápis konfigurace...\n");
    if (!Flash_Program(CONFIG_FLASH_ADDR, buffer, 256, 1000)) {
        printf("  Chyba: Zápis selhal!\n");
        return;
    }

    // 4. Ověřit
    printf("3. Ověření dat...\n");
    const u32* err = Flash_Verify(CONFIG_FLASH_ADDR, buffer, 256);
    if (err != NULL) {
        printf("  Chyba: Data se neshodují na 0x%08X\n", (u32)err);
        return;
    }

    printf("Konfigurace úspěšně uložena!\n");
}

void ConfigLoad(FLASH_CONFIG *cfg) {
    // Přímé čtení z FLASH
    const FLASH_CONFIG *flash_cfg = (const FLASH_CONFIG *)CONFIG_FLASH_ADDR;

    // Ověřit magic number
    if (flash_cfg->magic != 0x12345678) {
        printf("Chyba: Neplatná konfigurace v FLASH!\n");
        memset(cfg, 0, sizeof(FLASH_CONFIG));
        return;
    }

    // Zkopírovat data
    memcpy(cfg, flash_cfg, sizeof(FLASH_CONFIG));
    printf("Konfigurace nahrána ze FLASH\n");
}
```

## Důležité poznámky

### Zarovnání

- **Adresy** musí být zarovnány na stránku (FLASH_PAGEPG_SIZE)
- **Velikosti** musí být násobek velikosti stránky
- **Data** musí být zarovnána na u32 (4 bajty)

### Pracovní postup

1. **Odemknout** (automaticky v funkcích)
2. **Vymazat** FLASH
3. **Zapsat** data
4. **Ověřit** zápis
5. **Zamknout** (automaticky v funkcích)

### Timeout

- Typicky 1000 ms je dostatečné pro většinu operací
- Při nižších frekvencích může být potřeba delší timeout

### Ochrana

- **Read Protection (RDP)** - Ochrana čtení z FLASH přes debugger
- **Write Protection (WRP)** - Ochrana proti zápisu vybraných stránek

## Tabulka funkcí

| Funkce | Popis | Zarovnání |
|--------|-------|-----------|
| `Flash_Erase()` | Vymazat FLASH | Stránka |
| `Flash_Program()` | Zapsat do FLASH | Stránka |
| `Flash_Verify()` | Ověřit data | u32 slovo |
| `Flash_Check()` | Najít volné místo | u32 slovo |
| `Flash_OBRead()` | Čtení Option Bytes | N/A |
| `Flash_OBWrite()` | Zápis Option Bytes | u16 slovo |

## Související dokumentace

- [GPIO](gpio.md) - Konfigurace pinů přes Option Bytes
- [PWR](pwr.md) - Flash low-power režimy
- [RCC](rcc.md) - Flash latence podle frekvence
