# I2C - Inter-Integrated Circuit

## Přehled

I2C (Inter-Integrated Circuit) je sériový komunikační protokol pro připojení různých periferií a senzorů. Používá pouze 2 vodiče - SDA (data) a SCL (hodiny) a podporuje více zařízení na jedné sběrnici.

**Poznámka:** Některé příklady v této dokumentaci používají `printf()` pro debugging výstup. Printf není v SDK defaultně implementován - pro jeho použití je potřeba inicializovat USART a implementovat `_write()` funkci (viz [USART dokumentace](usart.md)). Na zařízeních s displejem můžete místo toho použít `DrawText()`.

## Základní koncepty

### I2C Registry

CH32 mikrokontroléry mají typicky jednu I2C jednotku:
- **I2C1** - Hlavní I2C periférie (adresa 0x40005400)

### Rychlosti komunikace

```c
#define I2C_SPEED_SLOW  100000  // Standardní rychlost (100 kHz)
#define I2C_SPEED_FAST  400000  // Rychlá komunikace (400 kHz)
```

### Režimy práce

- **Standard mode**: max. 100 kHz (Tlow ≥ 4.7μs, Thigh ≥ 4.0μs)
- **Fast mode**: max. 400 kHz (Tlow ≥ 1.3μs, Thigh ≥ 0.6μs)
- **Fast mode Plus**: max. 1 MHz

### Směr komunikace

```c
#define I2C_DIR_WRITE   0   // Zápis do zařízení
#define I2C_DIR_READ    1   // Čtení ze zařízení
```

## Základní inicializace

### Nastavení GPIO pinů

```c
// Nejdříve nastavit GPIO piny pro I2C
GPIO_Mode(PA9, GPIO_MODE_AFOD);   // SCL - Alternativní funkce Open-Drain
GPIO_Mode(PA10, GPIO_MODE_AFOD);  // SDA - Alternativní funkce Open-Drain

// Remapování I2C1 pinů (pokud je potřeba)
GPIO_Remap_I2C1(1);               // Alternativní mapování
```

### Inicializace I2C jako Master

```c
// Kompletní inicializace I2C1
void I2C_Master_Init(void) {
    // GPIO nastavení
    GPIO_Mode(PA9, GPIO_MODE_AFOD);   // SCL
    GPIO_Mode(PA10, GPIO_MODE_AFOD);  // SDA
    
    // I2C inicializace (adresa 0, protože je master)
    I2C1_Init(0, I2C_SPEED_FAST);    // 400 kHz
}
```

### Inicializace I2C jako Slave

```c
// I2C jako slave s 7-bitovou adresou 0x42
void I2C_Slave_Init(void) {
    // GPIO nastavení
    GPIO_Mode(PA9, GPIO_MODE_AFOD);   // SCL
    GPIO_Mode(PA10, GPIO_MODE_AFOD);  // SDA
    
    // I2C inicializace jako slave
    I2C1_Init(0x42, I2C_SPEED_FAST); // Adresa 0x42
}
```

## Pokročilé nastavení

### Ruční konfigurace

```c
// Ruční nastavení I2C parametrů
void I2C_Manual_Setup(void) {
    I2C1_ClkEnable();          // Povolit hodiny I2C
    I2C1_Reset();              // Reset I2C modulu
    
    I2C1_Freq(HCLK_PER_US);    // Nastavit frekvenci pro timeouty
    I2C1_SetSpeed(400000);     // 400 kHz
    I2C1_Addr7(0x42);          // 7-bitová adresa slave
    
    I2C1_AckEnable();          // Povolit ACK odpovědi
    I2C1_Enable();             // Povolit I2C modul
}
```

### Nastavení rychlosti

```c
// Ruční nastavení rychlosti
void I2C1_Slow(void);          // Standardní režim (≤100 kHz)
void I2C1_Fast(void);          // Rychlý režim (≤400 kHz)
void I2C1_Duty21(void);        // Poměr Tlow:Thigh = 2:1
void I2C1_Duty169(void);       // Poměr Tlow:Thigh = 16:9
void I2C1_SetSpeed(int speed); // Automatické nastavení
```

### Adresování

```c
// 7-bitové adresy (0-127)
void I2C1_Addr7(int addr);         // Hlavní adresa
void I2C1_Addr7Dual(int addr);     // Druhá adresa (dual mode)
void I2C1_AddrDualDisable(void);   // Zakázat dual mode

// 10-bitové adresy (0-1023)
void I2C1_Addr10(int addr);        // 10-bitová adresa
```

## Master komunikace

### Základní sekvence pro Master

```c
// Kompletní transakce pro zápis
void I2C_Write_Example(u8 device_addr, u8 reg_addr, u8 value) {
    I2C1_SendAddr(device_addr, I2C_DIR_WRITE);  // Start + adresa + W
    I2C1_WriteWait(reg_addr);                   // Adresa registru
    I2C1_WriteWait(value);                      // Data
    I2C1_StopEnable();                          // Stop
}

// Kompletní transakce pro čtení
u8 I2C_Read_Example(u8 device_addr, u8 reg_addr) {
    // Zápis adresy registru
    I2C1_SendAddr(device_addr, I2C_DIR_WRITE);
    I2C1_WriteWait(reg_addr);
    
    // Restart a čtení
    I2C1_SendAddr(device_addr, I2C_DIR_READ);
    u8 value = I2C1_ReadWait(100, 0xFF);       // Timeout 100ms
    I2C1_StopEnable();
    
    return value;
}
```

### Vysokoúrovňové funkce

```c
// Zápis do 8-bitového registru
void I2C1_WriteReg8(int addr, u8 cmd, u8 val);

// Zápis do 16-bitového registru  
void I2C1_WriteReg16(int addr, u8 cmd, u16 val);

// Čtení z 8-bitového registru
u8 I2C1_ReadReg8(int addr, u8 cmd, int ms, u8 def);

// Čtení z 16-bitového registru
u16 I2C1_ReadReg16(int addr, u8 cmd, int ms, u16 def);
```

### Přenos více dat

```c
// Poslání bloku dat
void I2C1_SendData(const void* buf, int num);

// Příjem bloku dat (vrací počet přijatých bytů)
int I2C1_RecvData(void* buf, int num, int ms);
```

## Slave komunikace

### Základní sekvence pro Slave

```c
void I2C_Slave_Handle(void) {
    // Čekání na adresování
    while (!I2C1_AddrOk()) {
        // Možnost dělat jiné věci
    }
    
    // Kontrola směru komunikace
    if (I2C1_IsTrans()) {
        // Master čte ze slave - pošleme data
        I2C1_WriteWait(sensor_value);
    } else {
        // Master píše do slave - přijmeme data
        u8 received = I2C1_ReadWait(1000, 0);
        process_received_data(received);
    }
    
    // Čekání na stop
    while (!I2C1_StopEvt()) {}
    I2C1_StatusClr();  // Vymazat status
}
```

## Kontrola stavu a chyb

### Základní status

```c
Bool I2C1_StartSent(void);     // Start byl odeslán
Bool I2C1_AddrOk(void);        // Adresa byla potvrzena
Bool I2C1_TransEnd(void);      // Konec přenosu bytu
Bool I2C1_TxEmpty(void);       // TX buffer je prázdný
Bool I2C1_RxReady(void);       // RX buffer obsahuje data
Bool I2C1_Busy(void);          // Sběrnice je zaneprázdněna
```

### Pokročilé status flagy

```c
Bool I2C1_IsMaster(void);      // Režim master
Bool I2C1_IsTrans(void);       // Směr: transmit (vs. receive)
Bool I2C1_IsBroad(void);       // Broadcast adresa přijata
Bool I2C1_IsDual(void);        // Použita druhá adresa
```

### Detekce chyb

```c
Bool I2C1_BusErr(void);        // Chyba sběrnice
Bool I2C1_ArbLos(void);        // Ztráta arbitráže
Bool I2C1_AnsErr(void);        // Chyba ACK (NACK)
Bool I2C1_OvrErr(void);        // Overrun/Underrun
Bool I2C1_CrcErr(void);        // CRC chyba
```

### Vymazání chyb

```c
void I2C1_BusErrClr(void);     // Vymazat chybu sběrnice
void I2C1_ArbLosClr(void);     // Vymazat ztrátu arbitráže
void I2C1_AnsErrClr(void);     // Vymazat chybu ACK
void I2C1_OvrErrClr(void);     // Vymazat overrun/underrun
void I2C1_CrcErrClr(void);     // Vymazat CRC chybu
void I2C1_StatusClr(void);     // Vymazat všechny chyby
```

## Přerušení

### Povolení přerušení

```c
void I2C1_IntEvtEnable(void);   // Přerušení při událostech
void I2C1_IntBufEnable(void);   // Přerušení při buffer events
void I2C1_IntErrEnable(void);   // Přerušení při chybách
```

### Zakázání přerušení

```c
void I2C1_IntEvtDisable(void);
void I2C1_IntBufDisable(void);
void I2C1_IntErrDisable(void);
```

## CRC kontrola

```c
// Povolení CRC kalkulace
void I2C1_CrcCalcEnable(void);
void I2C1_CrcEnable(void);      // Poslat CRC

// Čtení CRC
u8 I2C1_GetCrc(void);           // Aktuální CRC hodnota
```

## DMA podpora

```c
void I2C1_DMAEnable(void);      // Povolit DMA
void I2C1_LastEnable(void);     // Označit jako poslední transfer
```

## Praktické příklady

### Komunikace s OLED displejem (SSD1306)

```c
#define OLED_ADDR 0x3C

// Inicializace OLED
void OLED_Init(void) {
    // I2C inicializace
    GPIO_Mode(PA9, GPIO_MODE_AFOD);    // SCL
    GPIO_Mode(PA10, GPIO_MODE_AFOD);   // SDA
    I2C1_Init(0, I2C_SPEED_FAST);
    
    // OLED inicializační sekvence
    OLED_Command(0xAE);  // Display OFF
    OLED_Command(0xD5);  // Set display clock
    OLED_Command(0x80);  // Suggested value
    // ... další inicializační příkazy
}

// Poslání příkazu
void OLED_Command(u8 cmd) {
    I2C1_SendAddr(OLED_ADDR, I2C_DIR_WRITE);
    I2C1_WriteWait(0x00);  // Command mode
    I2C1_WriteWait(cmd);   // Příkaz
    I2C1_StopEnable();
}

// Poslání dat
void OLED_Data(u8 data) {
    I2C1_SendAddr(OLED_ADDR, I2C_DIR_WRITE);
    I2C1_WriteWait(0x40);  // Data mode
    I2C1_WriteWait(data);  // Data
    I2C1_StopEnable();
}
```

### Čtení ze senzoru teploty a vlhkosti (SHT40)

```c
#define SHT40_ADDR 0x44
#define SHT40_CMD_MEASURE_HIGH 0xFD  // Vysoká přesnost, ~10ms

// Inicializace senzoru
void SHT40_Init(void) {
    // I2C inicializace
    GPIO_Mode(PA9, GPIO_MODE_AFOD);    // SCL
    GPIO_Mode(PA10, GPIO_MODE_AFOD);   // SDA
    I2C1_Init(0, I2C_SPEED_FAST);
}

// Čtení teploty a vlhkosti
Bool SHT40_Read(float* temperature, float* humidity) {
    u8 data[6];

    // Poslání příkazu pro měření
    I2C1_SendAddr(SHT40_ADDR, I2C_DIR_WRITE);
    I2C1_WriteWait(SHT40_CMD_MEASURE_HIGH);
    I2C1_StopEnable();

    // Čekání na dokončení měření
    WaitMs(10);

    // Čtení výsledků (6 bytů: temp_H, temp_L, CRC, hum_H, hum_L, CRC)
    I2C1_SendAddr(SHT40_ADDR, I2C_DIR_READ);
    for (int i = 0; i < 6; i++) {
        data[i] = I2C1_ReadWait(100, 0xFF);
    }
    I2C1_StopEnable();

    // Kontrola chyby komunikace
    if (data[0] == 0xFF && data[1] == 0xFF) {
        return False;
    }

    // Konverze teploty (big-endian)
    u16 temp_raw = ((u16)data[0] << 8) | data[1];
    *temperature = -45.0f + 175.0f * (float)temp_raw / 65535.0f;

    // Konverze vlhkosti (big-endian)
    u16 hum_raw = ((u16)data[3] << 8) | data[4];
    *humidity = -6.0f + 125.0f * (float)hum_raw / 65535.0f;

    // Omezení vlhkosti na 0-100%
    if (*humidity < 0.0f) *humidity = 0.0f;
    if (*humidity > 100.0f) *humidity = 100.0f;

    return True;
}

// Příklad použití
void Example_SHT40(void) {
    float temp, hum;

    SHT40_Init();

    if (SHT40_Read(&temp, &hum)) {
        // temp obsahuje teplotu v °C (např. 23.5)
        // hum obsahuje vlhkost v % (např. 45.2)

        // Dále můžete hodnoty zobrazit na displeji pomocí DrawText(),
        // odeslat přes USART, nebo jinak zpracovat
    }
}
```

### Komunikace s EEPROM (24C02)

```c
#define EEPROM_ADDR 0x50

// Zápis bytu do EEPROM
void EEPROM_WriteByte(u8 addr, u8 data) {
    I2C1_SendAddr(EEPROM_ADDR, I2C_DIR_WRITE);
    I2C1_WriteWait(addr);    // Adresa v EEPROM
    I2C1_WriteWait(data);    // Data
    I2C1_StopEnable();
    
    DelayMs(5);  // Čekání na dokončení zápisu
}

// Čtení bytu z EEPROM
u8 EEPROM_ReadByte(u8 addr) {
    // Nastavení adresy
    I2C1_SendAddr(EEPROM_ADDR, I2C_DIR_WRITE);
    I2C1_WriteWait(addr);
    
    // Čtení dat
    I2C1_SendAddr(EEPROM_ADDR, I2C_DIR_READ);
    u8 data = I2C1_ReadWait(100, 0xFF);
    I2C1_StopEnable();
    
    return data;
}

// Zápis bloku dat
void EEPROM_WriteBlock(u8 addr, const u8* data, int len) {
    for (int i = 0; i < len; i++) {
        EEPROM_WriteByte(addr + i, data[i]);
    }
}
```

### Skenování I2C sběrnice

```c
// Najde všechna připojená I2C zařízení
void I2C_Scan(void) {
    printf("Scanning I2C bus...\n");
    
    for (int addr = 1; addr < 127; addr++) {
        I2C1_SendAddr(addr, I2C_DIR_WRITE);
        
        // Čekání na ACK nebo NACK
        int timeout = 1000;
        while (--timeout > 0 && !I2C1_AddrOk() && !I2C1_AnsErr()) {
            DelayUs(1);
        }
        
        if (I2C1_AddrOk()) {
            printf("Device found at 0x%02X\n", addr);
        }
        
        I2C1_StopEnable();
        I2C1_StatusClr();
        DelayMs(1);
    }
    
    printf("Scan complete.\n");
}
```

## Pokročilé techniky

### Obsluha s přerušením

```c
volatile u8 i2c_rx_buffer[32];
volatile int i2c_rx_index = 0;
volatile Bool i2c_transfer_complete = False;

void I2C1_Handler(void) {
    u32 events = I2C1_Evt();
    
    if (events & I2C_EVT_ADDR) {
        // Adresa byla rozpoznána
        i2c_rx_index = 0;
        I2C1_StatusClr();
    }
    
    if (events & I2C_EVT_RXREADY) {
        // Data k přečtení
        i2c_rx_buffer[i2c_rx_index++] = I2C1_Read();
    }
    
    if (events & I2C_EVT_STOP) {
        // Konec transakce
        i2c_transfer_complete = True;
        I2C1_StatusClr();
    }
}
```

### Robustní komunikace s retry

```c
// Robustní čtení s opakováním
u8 I2C_ReadReg_Robust(u8 device_addr, u8 reg_addr) {
    for (int retry = 0; retry < 3; retry++) {
        // Vymazat všechny chyby
        I2C1_StatusClr();
        
        // Pokus o komunikaci
        I2C1_SendAddr(device_addr, I2C_DIR_WRITE);
        
        // Kontrola ACK
        int timeout = 1000;
        while (--timeout > 0 && !I2C1_AddrOk() && !I2C1_AnsErr()) {
            DelayUs(1);
        }
        
        if (I2C1_AnsErr()) {
            I2C1_StopEnable();
            DelayMs(10);
            continue;  // Opakovat
        }
        
        I2C1_WriteWait(reg_addr);
        I2C1_SendAddr(device_addr, I2C_DIR_READ);
        u8 value = I2C1_ReadWait(100, 0xFF);
        I2C1_StopEnable();
        
        return value;
    }
    
    return 0xFF;  // Chyba po všech pokusech
}
```

## Tipy a triky

1. **Pull-up rezistory**: I2C vyžaduje pull-up rezistory na SDA a SCL (typicky 4.7kΩ)
2. **Open-drain**: Vždy používejte `GPIO_MODE_AFOD` pro I2C piny
3. **Timeouty**: Vždy používejte timeouty při čekání na odpověď
4. **Chyby**: Kontrolujte chybové flagy a reagujte na ně
5. **Reset**: Při zablokování pomůže software reset `I2C1_SwRstEnable()`
6. **Rychlost**: Začněte s pomalou rychlostí a postupně zvyšujte

## Debugging

### Kontrola stavu sběrnice

```c
void I2C_Debug_Status(void) {
    printf("I2C Status:\n");
    printf("  Busy: %d\n", I2C1_Busy());
    printf("  Master: %d\n", I2C1_IsMaster());
    printf("  Bus Error: %d\n", I2C1_BusErr());
    printf("  Arbitration Loss: %d\n", I2C1_ArbLos());
    printf("  ACK Failure: %d\n", I2C1_AnsErr());
}
```

## Poznámky pro různé MCU

- **CH32V003**: Základní I2C funkcionalita s DMA podporou (2KB SRAM)
- **CH32V002**: Vylepšená varianta s 4KB SRAM, 12-bit ADC, touch sensing
- **CH32V006**: Pokročilá varianta s více možnostmi remapování pinů
- **CH32X035**: Nejpokročilejší s více I2C jednotkami a funkcemi

## Mapování pinů

### Standardní mapování I2C1:
- **SCL**: PB6 nebo PA9 (remap)
- **SDA**: PB7 nebo PA10 (remap)

### Remapování:
```c
GPIO_Remap_I2C1(1);  // Přepnout na alternativní piny
```