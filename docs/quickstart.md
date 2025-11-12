# CH32LibSDK - Rychlý start

## Přehled

Tento průvodce vás provede prvními kroky s CH32LibSDK a ukáže praktické použití GPIO, OLED displeje, reproduktoru a I2C senzorů na příkladu aplikace pro měření teploty a vlhkosti.

## Požadavky

### Hardware
- Jedna z podporovaných desek:
  - **BabyBoy** - OLED 128x64, 6 tlačítek, reproduktor, CR2032
  - **PidiBoy** - OLED 128x64, 7 tlačítek, reproduktor, SD karta
  - **TweetyBoy** - LCD 160x80, 8 tlačítek, reproduktor, SD karta
  - **BabyPad/PidiPad** - VGA výstup, tlačítka, audio výstup
- SHT40 senzor teploty/vlhkosti (I2C)
- WCH-LinkE programátor

### Software
- MounRiver Studio v1.92+
- RISC-V Embedded GCC12 kompilátor
- WCH-LinkUtility pro programování

## Vytvoření prvního projektu

### 1. Struktura projektu

```
MonitorTeploty/
├── Makefile
├── config.h
├── include.h
├── main.c
└── sht40.c
```

### 2. Makefile

```makefile
# Název projektu
TARGET = MonitorTeploty

# Cílová deska (babyboy, pidiboy, tweetyboy, babypad, atd.)
DEVCLASS = babyboy

# Zdrojové soubory
CSRC = main.c sht40.c

# Včetně základní Makefile
include ../Makefile1.inc
include ../Makefile2.inc
```

### 3. config.h

```c
#ifndef _CONFIG_H
#define _CONFIG_H

// Základní konfigurace pro BabyBoy
#define DEVICE_NAME	"BabyBoy"
#define BOARD_TYPE	BOARD_BABYBOY

// Použít display driver
#define USE_DISP	1	// 1=software I2C, 2=hardware I2C

// Použít klávesnici
#define USE_KEY		1	// 1=použít klávesnici

// Použít zvuk
#define USE_SOUND	2	// 1=tóny, 2=melodie

// Použít kreslení
#define USE_DRAW	1	// 1=použít kreslicí funkce

// OLED display I2C konfigurace
#define DISP_I2C_ADDR	0x3C	// adresa OLED displeje
#define DISP_SDA_GPIO	PC1	// SDA pin
#define DISP_SCL_GPIO	PC2	// SCL pin
#define DISP_WAIT_CLK	4	// I2C wait cykly

// SHT40 senzor I2C konfigurace  
#define SHT40_I2C_ADDR	0x44	// adresa SHT40 senzoru
#define SHT40_SDA_GPIO	PC1	// SDA pin (sdílený s OLED)
#define SHT40_SCL_GPIO	PC2	// SCL pin (sdílený s OLED)

#endif // _CONFIG_H
```

### 4. include.h

```c
#ifndef _INCLUDE_H
#define _INCLUDE_H

// Základní systémové includes
#include "../_sdk/inc/sdk.h"		// SDK knihovny
#include "../_devices/_include.h"	// Device abstraction
#include "config.h"			// Konfigurace projektu

// Lokální includes
#include "sht40.h"			// SHT40 senzor

#endif // _INCLUDE_H
```

## Praktický příklad: Meteostanice

Vytvoříme jednoduchou meteostnici, která:
- Měří teplotu a vlhkost pomocí SHT40
- Zobrazuje data na OLED displeji
- Tlačítkem přepíná mezi teplotou a vlhkostí
- Pípne při stisknutí tlačítka

### SHT40 driver (sht40.h)

```c
#ifndef _SHT40_H
#define _SHT40_H

#include "include.h"

// SHT40 I2C příkazy
#define SHT40_CMD_MEASURE_HI_PRECISION	0xFD	// Vysoká přesnost
#define SHT40_CMD_MEASURE_MED_PRECISION	0xF6	// Střední přesnost  
#define SHT40_CMD_MEASURE_LOW_PRECISION	0xE0	// Nízká přesnost
#define SHT40_CMD_READ_SERIAL		0x89	// Čtení sériového čísla
#define SHT40_CMD_SOFT_RESET		0x94	// Software reset

// Struktura pro data ze senzoru
typedef struct {
    float temperature;	// Teplota v °C
    float humidity;	// Vlhkost v %RH
    Bool valid;		// Platnost dat
} SHT40_Data_t;

// Funkce pro práci se SHT40
Bool SHT40_Init(void);
Bool SHT40_Reset(void);
Bool SHT40_ReadData(SHT40_Data_t* data);
uint32_t SHT40_ReadSerial(void);

#endif // _SHT40_H
```

### SHT40 driver (sht40.c)

```c
#include "include.h"

// I2C komunikace se SHT40
static void SHT40_I2C_Start(void) {
    GPIO_Out1(SHT40_SCL_GPIO);
    GPIO_Out1(SHT40_SDA_GPIO);
    WaitUs(5);
    GPIO_Out0(SHT40_SDA_GPIO);
    WaitUs(5);
    GPIO_Out0(SHT40_SCL_GPIO);
    WaitUs(5);
}

static void SHT40_I2C_Stop(void) {
    GPIO_Out0(SHT40_SDA_GPIO);
    WaitUs(5);
    GPIO_Out1(SHT40_SCL_GPIO);
    WaitUs(5);
    GPIO_Out1(SHT40_SDA_GPIO);
    WaitUs(5);
}

static void SHT40_I2C_WriteByte(uint8_t data) {
    for (int i = 7; i >= 0; i--) {
        GPIO_Out(SHT40_SDA_GPIO, (data >> i) & 1);
        WaitUs(2);
        GPIO_Out1(SHT40_SCL_GPIO);
        WaitUs(5);
        GPIO_Out0(SHT40_SCL_GPIO);
        WaitUs(2);
    }
    
    // ACK
    GPIO_Mode(SHT40_SDA_GPIO, GPIO_MODE_IN_PU);
    WaitUs(2);
    GPIO_Out1(SHT40_SCL_GPIO);
    WaitUs(5);
    GPIO_Out0(SHT40_SCL_GPIO);
    GPIO_Mode(SHT40_SDA_GPIO, GPIO_MODE_OUT);
    WaitUs(2);
}

static uint8_t SHT40_I2C_ReadByte(Bool ack) {
    uint8_t data = 0;
    
    GPIO_Mode(SHT40_SDA_GPIO, GPIO_MODE_IN_PU);
    
    for (int i = 7; i >= 0; i--) {
        GPIO_Out1(SHT40_SCL_GPIO);
        WaitUs(5);
        if (GPIO_In(SHT40_SDA_GPIO)) {
            data |= (1 << i);
        }
        GPIO_Out0(SHT40_SCL_GPIO);
        WaitUs(5);
    }
    
    // ACK/NACK
    GPIO_Mode(SHT40_SDA_GPIO, GPIO_MODE_OUT);
    GPIO_Out(SHT40_SDA_GPIO, ack ? 0 : 1);
    WaitUs(2);
    GPIO_Out1(SHT40_SCL_GPIO);
    WaitUs(5);
    GPIO_Out0(SHT40_SCL_GPIO);
    WaitUs(2);
    
    return data;
}

Bool SHT40_Init(void) {
    // Konfigurace I2C pinů
    GPIO_Mode(SHT40_SDA_GPIO, GPIO_MODE_OUT);
    GPIO_Mode(SHT40_SCL_GPIO, GPIO_MODE_OUT);
    
    // Reset senzoru
    return SHT40_Reset();
}

Bool SHT40_Reset(void) {
    SHT40_I2C_Start();
    SHT40_I2C_WriteByte(SHT40_I2C_ADDR << 1);  // Write address
    SHT40_I2C_WriteByte(SHT40_CMD_SOFT_RESET);
    SHT40_I2C_Stop();
    
    WaitMs(1);  // Čekání na reset
    return True;
}

Bool SHT40_ReadData(SHT40_Data_t* data) {
    uint8_t buffer[6];
    
    // Poslat příkaz pro měření
    SHT40_I2C_Start();
    SHT40_I2C_WriteByte(SHT40_I2C_ADDR << 1);  // Write address
    SHT40_I2C_WriteByte(SHT40_CMD_MEASURE_HI_PRECISION);
    SHT40_I2C_Stop();
    
    // Čekat na dokončení měření (10ms pro vysokou přesnost)
    WaitMs(10);
    
    // Číst data
    SHT40_I2C_Start();
    SHT40_I2C_WriteByte((SHT40_I2C_ADDR << 1) | 1);  // Read address
    
    for (int i = 0; i < 6; i++) {
        buffer[i] = SHT40_I2C_ReadByte(i < 5);  // ACK pro všechny kromě posledního
    }
    
    SHT40_I2C_Stop();
    
    // Převod raw dat na teplotu a vlhkost
    uint16_t temp_raw = (buffer[0] << 8) | buffer[1];
    uint16_t humi_raw = (buffer[3] << 8) | buffer[4];
    
    // TODO: Kontrola CRC (buffer[2] a buffer[5])
    
    // Konverze podle datasheetu SHT40
    data->temperature = -45.0f + 175.0f * (float)temp_raw / 65535.0f;
    data->humidity = -6.0f + 125.0f * (float)humi_raw / 65535.0f;
    data->valid = True;
    
    return True;
}

uint32_t SHT40_ReadSerial(void) {
    uint8_t buffer[6];
    
    SHT40_I2C_Start();
    SHT40_I2C_WriteByte(SHT40_I2C_ADDR << 1);
    SHT40_I2C_WriteByte(SHT40_CMD_READ_SERIAL);
    SHT40_I2C_Stop();
    
    WaitMs(1);
    
    SHT40_I2C_Start();
    SHT40_I2C_WriteByte((SHT40_I2C_ADDR << 1) | 1);
    
    for (int i = 0; i < 6; i++) {
        buffer[i] = SHT40_I2C_ReadByte(i < 5);
    }
    
    SHT40_I2C_Stop();
    
    return ((uint32_t)buffer[0] << 24) | ((uint32_t)buffer[1] << 16) | 
           ((uint32_t)buffer[3] << 8) | buffer[4];
}
```

### Hlavní aplikace (main.c)

```c
#include "include.h"

// Aplikační stavy
typedef enum {
    DISPLAY_TEMPERATURE = 0,
    DISPLAY_HUMIDITY = 1
} DisplayMode_t;

// Globální proměnné
static DisplayMode_t display_mode = DISPLAY_TEMPERATURE;
static SHT40_Data_t sensor_data = {0};
static uint32_t last_measurement = 0;
static uint32_t last_key_press = 0;

// Zvukové efekty
static const sMelodyNote beep_sound[] = {
    {NOTE_LEN16, NOTE_C5},  // Krátký beep
    {NOTE_STOP, 0}
};

static const sMelodyNote startup_sound[] = {
    {NOTE_LEN8, NOTE_C4},
    {NOTE_LEN8, NOTE_E4},
    {NOTE_LEN4, NOTE_G4},
    {NOTE_STOP, 0}
};

void InitSystem(void) {
    // Inicializace device layer
    DeviceInit();
    
    // Inicializace displeje
    DispInit();
    DrawClear();
    
    // Inicializace klávesnice
    KeyInit();
    
    // Inicializace zvuku
    SoundInit();
    
    // Inicializace SHT40 senzoru
    if (!SHT40_Init()) {
        DrawText("SHT40 Error!", 0, 0, COL_WHITE);
        DispUpdate();
        while(1) WaitMs(1000);  // Zastavit při chybě
    }
    
    // Uvítací zvuk
    PlayMelody(startup_sound);
    
    // Uvítací zpráva
    DrawText("Meteostanice", 2, 1, COL_WHITE);
    DrawText("Inicializace...", 1, 3, COL_WHITE);
    DispUpdate();
    WaitMs(2000);
    
    DrawClear();
}

void UpdateSensorData(void) {
    uint32_t now = Time();
    
    // Měření každé 2 sekundy
    if (now - last_measurement > 2000) {
        last_measurement = now;
        
        if (SHT40_ReadData(&sensor_data)) {
            // Data úspěšně načtena
        } else {
            sensor_data.valid = False;
        }
    }
}

void HandleInput(void) {
    uint8_t key = KeyGet();
    uint32_t now = Time();
    
    if (key != NOKEY && (now - last_key_press > 200)) {  // Debouncing
        last_key_press = now;
        
        switch (key) {
            case KEY_A:
                // Přepnutí mezi teplotou a vlhkostí
                display_mode = (display_mode == DISPLAY_TEMPERATURE) ? 
                              DISPLAY_HUMIDITY : DISPLAY_TEMPERATURE;
                PlayMelody(beep_sound);
                break;
                
            case KEY_B:
                // Force refresh měření
                last_measurement = 0;
                PlayMelody(beep_sound);
                break;
                
            case KEY_UP:
                // Možnost pro další funkce
                break;
        }
    }
}

void UpdateDisplay(void) {
    DrawClear();
    
    // Hlavička
    DrawText("== METEOSTANICE ==", 0, 0, COL_WHITE);
    
    if (!sensor_data.valid) {
        DrawText("Chyba senzoru!", 2, 3, COL_WHITE);
        DrawText("Stiskni B pro", 2, 4, COL_WHITE);
        DrawText("opakovani", 4, 5, COL_WHITE);
    } else {
        char buffer[20];
        
        if (display_mode == DISPLAY_TEMPERATURE) {
            DrawText("TEPLOTA:", 4, 2, COL_WHITE);
            
            // Formátování teploty s 1 desetinným místem
            int temp_int = (int)sensor_data.temperature;
            int temp_dec = (int)((sensor_data.temperature - temp_int) * 10);
            snprintf(buffer, sizeof(buffer), "%d.%d C", temp_int, temp_dec);
            
            DrawText(buffer, 3, 4, COL_WHITE);
            DrawText("[A] -> Vlhkost", 1, 6, COL_WHITE);
        } else {
            DrawText("VLHKOST:", 4, 2, COL_WHITE);
            
            // Formátování vlhkosti
            int humi_int = (int)sensor_data.humidity;
            snprintf(buffer, sizeof(buffer), "%d %%RH", humi_int);
            
            DrawText(buffer, 3, 4, COL_WHITE);
            DrawText("[A] -> Teplota", 1, 6, COL_WHITE);
        }
    }
    
    DrawText("[B] Refresh", 7, 7, COL_WHITE);
    DispUpdate();
}

int main(void) {
    InitSystem();
    
    // Hlavní smyčka
    while (1) {
        UpdateSensorData();
        HandleInput();
        UpdateDisplay();
        
        WaitMs(50);  // 20 FPS refresh rate
    }
    
    return 0;
}
```

## Kompilace a programování

### 1. Kompilace
```bash
# Z adresáře projektu
c.bat
```

### 2. Nahrání do MCU
```bash
# Pomocí WCH-LinkE
e.bat
```

### 3. Vyčištění
```bash
# Smazání objektových souborů
d.bat
```

## Výstup aplikace

Po spuštění aplikace uvidíte:

1. **Úvodní obrazovka**: "Meteostanice Inicializace..."
2. **Hlavní obrazovka teploty**:
   ```
   == METEOSTANICE ==
   
       TEPLOTA:
   
       23.5 C
   
   [A] -> Vlhkost
                [B] Refresh
   ```

3. **Hlavní obrazovka vlhkosti** (po stisknutí A):
   ```
   == METEOSTANICE ==
   
       VLHKOST:
   
       45 %RH
   
   [A] -> Teplota
                [B] Refresh
   ```

## Přizpůsobení pro jiné desky

### PidiBoy (7 tlačítek, SD karta)
```c
// V config.h změnit:
#define DEVICE_NAME	"PidiBoy"
#define BOARD_TYPE	BOARD_PIDIBOY

// V Makefile změnit:
DEVCLASS = pidiboy

// Přidat podporu SD karty pro logování:
#define USE_SD		1
```

### TweetyBoy (barevný LCD)
```c
// V config.h změnit:
#define DEVICE_NAME	"TweetyBoy"  
#define BOARD_TYPE	BOARD_TWEETYBOY

// Barevný displej 160x80
#define WIDTH		160
#define HEIGHT		80

// V Makefile změnit:
DEVCLASS = tweetyboy
```

### BabyPad (VGA výstup)
```c
// V config.h změnir:
#define DEVICE_NAME	"BabyPad"
#define BOARD_TYPE	BOARD_BABYPAD

// VGA výstup místo OLED
#define USE_VGA		1

// V Makefile změnit:
DEVCLASS = babypad
```

## Rozšíření projektu

### 1. Přidání dalších senzorů
- Barometr (BMP280)
- Světelný senzor (BH1750)
- Kvalita vzduchu (CCS811)

### 2. Datové logování
```c
// Pro desky s SD kartou (PidiBoy, TweetyBoy)
void LogData(SHT40_Data_t* data) {
    char log_line[64];
    snprintf(log_line, sizeof(log_line), 
             "%lu,%.1f,%.1f\n", Time(), 
             data->temperature, data->humidity);
    
    // Zápis na SD kartu
    SD_WriteLog("weather.csv", log_line);
}
```

### 3. Alarm funkce
```c
// Teplotní alarm
void CheckAlarms(SHT40_Data_t* data) {
    if (data->temperature > 30.0f) {
        // Přehřátí - rychlé pípání
        for (int i = 0; i < 5; i++) {
            PlayTone(NOTE_C6);
            WaitMs(100);
            StopSound();
            WaitMs(100);
        }
    }
}
```

### 4. Grafické zobrazení
```c
// Graf teploty za posledních 60 měření
void DrawTemperatureGraph(void) {
    static float temp_history[60];
    static int history_index = 0;
    
    // Uložit novou hodnotu
    temp_history[history_index] = sensor_data.temperature;
    history_index = (history_index + 1) % 60;
    
    // Vykreslit graf
    for (int i = 0; i < 59; i++) {
        int x1 = i;
        int x2 = i + 1;
        int y1 = HEIGHT - (int)((temp_history[i] - 10) * 2);
        int y2 = HEIGHT - (int)((temp_history[i+1] - 10) * 2);
        
        DrawLine(x1, y1, x2, y2, COL_WHITE);
    }
}
```

## Tipy a triky

1. **I2C sharing**: SHT40 a OLED můžou sdílet stejný I2C bus
2. **Power management**: Použijte sleep módy mezi měřeními pro delší výdrž baterie
3. **Error handling**: Vždy kontrolujte návratové hodnoty I2C komunikace
4. **Floating point**: Na CH32V002/003 je float pomalý - používejte fixed-point aritmetiku
5. **Memory management**: Minimalizujte použití RAM - OLED buffer zabírá 1KB

## Debugging

### Sériový výstup pro debug
```c
// V main.c přidat:
void DebugPrint(const char* msg) {
    #ifdef DEBUG
    printf("%s\n", msg);
    #endif
}

// Použití:
DebugPrint("SHT40 initialized");
```

### LED indikátory
```c
// Použití vestavěných LED pro signalizaci
void SetStatusLED(Bool on) {
    // Závisí na desce - např. pro BabyBoy
    GPIO_Out(PA0, on ? 1 : 0);
}
```

Tento quickstart vás provede základy CH32LibSDK a ukáže praktické použití všech hlavních komponent. Projekt lze snadno rozšířit a přizpůsobit vašim potřebám!