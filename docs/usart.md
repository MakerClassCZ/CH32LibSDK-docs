# USART - Univerzální sériový přijímač/vysílač

## Přehled

USART (Universal Synchronous/Asynchronous Receiver Transmitter) umožňuje sériovou komunikaci s ostatními zařízeními. Podporuje asynchronní i synchronní režimy, různé baudraty, paritní kontrolu a řízení toku.

## Základní koncepty

### Dostupné USART jednotky

CH32 mikrokontroléry mají typicky:
- **USART1** - Hlavní USART periférie (adresa 0x40013800)

### Komunikační parametry

#### Přenosová rychlost (baud rate)
Typické rychlosti: 9600, 19200, 38400, 57600, 115200, 230400, 460800, 921600 Bd

#### Parita
```c
#define USART_PARITY_NONE   0   // Bez parity (default)
#define USART_PARITY_EVEN   2   // Sudá parita
#define USART_PARITY_ODD    3   // Lichá parita
```

#### Stop bity
```c
#define USART_STOP_1      0   // 1 stop bit (default)
#define USART_STOP_05     1   // 0.5 stop bit
#define USART_STOP_2      2   // 2 stop bity
#define USART_STOP_15     3   // 1.5 stop bit
```

#### Délka slova
```c
#define USART_WORDLEN_8   0   // 8 datových bitů (default)
#define USART_WORDLEN_9   1   // 9 datových bitů
```

## Základní nastavení

### Inicializace GPIO

```c
// Nastavení GPIO pinů pro USART
void USART_GPIO_Init(void) {
    // TX pin jako alternativní funkce push-pull
    GPIO_Mode(PA9, GPIO_MODE_AF);       // USART1_TX
    
    // RX pin jako vstup s pull-up
    GPIO_Mode(PA10, GPIO_MODE_IN_PU);   // USART1_RX
    
    // Remapování (pokud je potřeba)
    // GPIO_Remap_USART1(1);           // Alternativní piny
}
```

### Základní inicializace

```c
// Kompletní inicializace USART1
void USART1_BasicInit(void) {
    // GPIO nastavení
    USART_GPIO_Init();
    
    // Povolit hodiny USART1
    USART1_RCCClkEnable();
    
    // Reset USART1
    USART1_Reset();
    
    // Nastavení baudráty
    USART1_Baud(115200);
    
    // Nastavení formátu (8N1 - default)
    USART1_WordLen(USART_WORDLEN_8);
    USART1_Parity(USART_PARITY_NONE);
    USART1_StopBits(USART_STOP_1);
    
    // Povolit TX a RX
    USART1_TxEnable();
    USART1_RxEnable();
    
    // Povolit USART
    USART1_Enable();
}
```

## Konfigurace

### Přenosová rychlost

```c
// Nastavení přenosové rychlosti (baud rate)
void USART1_Baud(int baud);         // Nastavit rychlost v Bd
int USART1_GetBaud(void);           // Přečíst rychlost

// Přímé nastavení děliče
void USART1_IBaud(int div);         // Nastavit dělič
int USART1_GetIBaud(void);          // Přečíst dělič
```

### Formát dat

```c
// Délka slova
void USART1_WordLen(int len);       // 8 nebo 9 bitů

// Parita
void USART1_Parity(int parity);     // USART_PARITY_*

// Stop bity
void USART1_StopBits(int stop);     // USART_STOP_*
```

### Povolit/zakázat funkce

```c
// Základní funkce
void USART1_Enable(void);           // Povolit USART
void USART1_Disable(void);          // Zakázat USART

void USART1_TxEnable(void);         // Povolit vysílání
void USART1_TxDisable(void);        // Zakázat vysílání

void USART1_RxEnable(void);         // Povolit příjem
void USART1_RxDisable(void);        // Zakázat příjem
```

## Základní komunikace

### Zápis a čtení dat

```c
// Přímý přístup k registrům
void USART1_TxWrite(int data);      // Zapsat do TX registru
int USART1_RxRead(void);            // Přečíst z RX registru

// Zápis/čtení s čekáním
void USART1_SendChar(int ch);       // Poslat znak s čekáním
int USART1_RecvChar(void);          // Přijmout znak s čekáním
```

### Kontrola stavu

```c
// TX status
Bool USART1_TxEmpty(void);          // TX buffer je prázdný
Bool USART1_TxSent(void);           // Přenos je dokončen

// RX status
Bool USART1_RxReady(void);          // Data jsou připravena k čtení

// Vymazání flagů
void USART1_TxSentClr(void);        // Vymazat TX dokončeno
void USART1_RxReadyClr(void);       // Vymazat RX připraveno
```

### Blokové přenosy

```c
// Poslání bloku dat
void USART1_SendBuf(const u8* buf, int len);

// Příjem bloku dat s timeoutem
int USART1_RecvBuf(u8* buf, int len, u32 timeout_us);
```

## Detekce chyb

### Chybové flagy

```c
// Kontrola chyb
Bool USART1_ParityError(void);      // Chyba parity
Bool USART1_FrameError(void);       // Chyba rámce
Bool USART1_NoiseError(void);       // Chyba šumu
Bool USART1_OverloadError(void);    // Přetečení bufferu
Bool USART1_IdleError(void);        // Nečinnost linky
Bool USART1_BreakError(void);       // Break detekce

// Vymazání chyb
void USART1_BreakErrorClr(void);    // Vymazat break chybu
```

### Obsluha chyb

```c
// Kontrola všech chyb
void USART_CheckErrors(void) {
    if (USART1_ParityError()) {
        printf("Parity error!\n");
        // Přečíst data registr pro vymazání
        USART1_RxRead();
    }
    
    if (USART1_FrameError()) {
        printf("Frame error!\n");
        USART1_RxRead();
    }
    
    if (USART1_OverloadError()) {
        printf("Overrun error!\n");
        USART1_RxRead();
    }
}
```

## Přerušení

### Povolení přerušení

```c
// RX přerušení
void USART1_RxIntEnable(void);      // Přerušení při přijetí
void USART1_RxIntDisable(void);

// TX přerušení
void USART1_TxEmptyIntEnable(void); // Přerušení při prázdném TX
void USART1_TxEmptyIntDisable(void);

void USART1_TxSentIntEnable(void);  // Přerušení při dokončení TX
void USART1_TxSentIntDisable(void);

// Chybová přerušení
void USART1_ErrorIntEnable(void);   // Přerušení při chybách
void USART1_ErrorIntDisable(void);

// Další přerušení
void USART1_IdleIntEnable(void);    // Přerušení při nečinnosti
void USART1_CTSIntEnable(void);     // Přerušení při změně CTS
```

### Obsluha přerušení

```c
// Buffer pro příjem
#define RX_BUFFER_SIZE 256
volatile char rx_buffer[RX_BUFFER_SIZE];
volatile int rx_head = 0, rx_tail = 0;

// Obsluha USART přerušení
void USART1_Handler(void) {
    // Příjem dat
    if (USART1_RxReady()) {
        char ch = USART1_RxRead();
        
        int next_head = (rx_head + 1) % RX_BUFFER_SIZE;
        if (next_head != rx_tail) {
            rx_buffer[rx_head] = ch;
            rx_head = next_head;
        }
    }
    
    // Kontrola chyb
    if (USART1_OverloadError()) {
        USART1_RxRead();  // Vymazat chybu
    }
}

// Čtení z bufferu
int USART_GetChar(void) {
    if (rx_head == rx_tail) {
        return -1;  // Buffer prázdný
    }
    
    char ch = rx_buffer[rx_tail];
    rx_tail = (rx_tail + 1) % RX_BUFFER_SIZE;
    return ch;
}
```

## DMA podpora

```c
// DMA pro USART
void USART1_TxDMAEnable(void);      // Povolit TX DMA
void USART1_RxDMAEnable(void);      // Povolit RX DMA
void USART1_TxDMADisable(void);     // Zakázat TX DMA
void USART1_RxDMADisable(void);     // Zakázat RX DMA
```

## Praktické příklady

### Printf implementace

```c
// Přesměrování printf na USART
int _write(int fd, char* ptr, int len) {
    for (int i = 0; i < len; i++) {
        USART1_SendChar(ptr[i]);
    }
    return len;
}

// Použití
void USART_Printf_Example(void) {
    USART1_BasicInit();
    
    printf("Hello World!\n");
    printf("Number: %d\n", 42);
    printf("Float: %.2f\n", 3.14f);
}
```

### Příkazový řádek

```c
#define CMD_BUFFER_SIZE 64
char cmd_buffer[CMD_BUFFER_SIZE];
int cmd_index = 0;

// Zpracování příkazů
void USART_CommandLine(void) {
    int ch = USART_GetChar();
    if (ch == -1) return;
    
    if (ch == '\r' || ch == '\n') {
        // Konec příkazu
        cmd_buffer[cmd_index] = '\0';
        
        if (cmd_index > 0) {
            ProcessCommand(cmd_buffer);
            cmd_index = 0;
        }
        
        printf("$ ");  // Prompt
    }
    else if (ch == '\b' || ch == 127) {
        // Backspace
        if (cmd_index > 0) {
            cmd_index--;
            printf("\b \b");  // Vymazat znak
        }
    }
    else if (cmd_index < CMD_BUFFER_SIZE - 1) {
        // Přidat znak
        cmd_buffer[cmd_index++] = ch;
        USART1_SendChar(ch);  // Echo
    }
}

// Zpracování příkazu
void ProcessCommand(const char* cmd) {
    printf("\nCommand: %s\n", cmd);
    
    if (strcmp(cmd, "help") == 0) {
        printf("Available commands:\n");
        printf("  help - show this help\n");
        printf("  led on/off - control LED\n");
        printf("  status - show system status\n");
    }
    else if (strcmp(cmd, "led on") == 0) {
        GPIO_Out1(PA0);
        printf("LED ON\n");
    }
    else if (strcmp(cmd, "led off") == 0) {
        GPIO_Out0(PA0);
        printf("LED OFF\n");
    }
    else if (strcmp(cmd, "status") == 0) {
        printf("System OK, Uptime: %lu ms\n", Time() / HCLK_PER_MS);
    }
    else {
        printf("Unknown command: %s\n", cmd);
    }
}
```

### Modbus RTU komunikace

```c
#define MODBUS_BUFFER_SIZE 256
u8 modbus_buffer[MODBUS_BUFFER_SIZE];
int modbus_index = 0;
u32 modbus_timeout = 0;

// Modbus RTU frame detector
void USART_Modbus_Process(void) {
    // Kontrola timeoutu (3.5 char time)
    u32 char_time = (11 * 1000000) / USART1_GetBaud(); // us per char
    u32 frame_timeout = char_time * 4; // 3.5 char times
    
    if (modbus_index > 0 && 
        (Time() - modbus_timeout) > frame_timeout * HCLK_PER_US) {
        // Frame timeout - zpracovat rámec
        ProcessModbusFrame(modbus_buffer, modbus_index);
        modbus_index = 0;
    }
    
    // Příjem nových dat
    int ch = USART_GetChar();
    if (ch != -1) {
        if (modbus_index < MODBUS_BUFFER_SIZE) {
            modbus_buffer[modbus_index++] = ch;
            modbus_timeout = Time();
        }
    }
}

// CRC16 pro Modbus
u16 ModbusCRC16(const u8* data, int len) {
    u16 crc = 0xFFFF;
    
    for (int i = 0; i < len; i++) {
        crc ^= data[i];
        for (int j = 0; j < 8; j++) {
            if (crc & 1) {
                crc = (crc >> 1) ^ 0xA001;
            } else {
                crc >>= 1;
            }
        }
    }
    
    return crc;
}

// Zpracování Modbus rámce
void ProcessModbusFrame(const u8* frame, int len) {
    if (len < 4) return; // Minimální délka
    
    // Kontrola CRC
    u16 crc_received = (frame[len-1] << 8) | frame[len-2];
    u16 crc_calculated = ModbusCRC16(frame, len - 2);
    
    if (crc_received != crc_calculated) {
        printf("Modbus CRC error\n");
        return;
    }
    
    u8 slave_addr = frame[0];
    u8 function = frame[1];
    
    printf("Modbus: Addr=%d, Func=%d\n", slave_addr, function);
    
    // Zpracování podle funkce...
}
```

### Bluetooth AT příkazy

```c
// AT příkazy pro Bluetooth modul
void Bluetooth_SendAT(const char* cmd) {
    printf("AT+%s\r\n", cmd);
    DelayMs(100);
}

void Bluetooth_Init(void) {
    USART1_BasicInit();
    
    printf("Initializing Bluetooth...\n");
    
    // Test komunikace
    printf("AT\r\n");
    DelayMs(1000);
    
    // Nastavení názvu
    Bluetooth_SendAT("NAME=CH32_Device");
    
    // Nastavení PIN
    Bluetooth_SendAT("PIN=1234");
    
    // Nastavení baudráty
    Bluetooth_SendAT("BAUD=4"); // 9600 baud
    
    printf("Bluetooth ready\n");
}

// Zpracování Bluetooth dat
void Bluetooth_Process(void) {
    static char bt_buffer[128];
    static int bt_index = 0;
    
    int ch = USART_GetChar();
    if (ch == -1) return;
    
    if (ch == '\n') {
        bt_buffer[bt_index] = '\0';
        
        if (strncmp(bt_buffer, "+CONNECTED", 10) == 0) {
            printf("Bluetooth connected\n");
        }
        else if (strncmp(bt_buffer, "+DISCONNECTED", 13) == 0) {
            printf("Bluetooth disconnected\n");
        }
        else if (bt_index > 0) {
            printf("BT Data: %s\n", bt_buffer);
        }
        
        bt_index = 0;
    }
    else if (ch != '\r' && bt_index < sizeof(bt_buffer) - 1) {
        bt_buffer[bt_index++] = ch;
    }
}
```

### GPS NMEA parser

```c
// GPS NMEA zpracování
typedef struct {
    float latitude;
    float longitude;
    int satellites;
    Bool fix_valid;
} GPS_Data_t;

GPS_Data_t gps_data = {0};

// Zpracování NMEA dat
void GPS_ProcessNMEA(const char* line) {
    if (strncmp(line, "$GPGGA", 6) == 0) {
        // GGA - Global Positioning System Fix Data
        char* tokens[15];
        int token_count = 0;
        
        // Rozdělení na tokeny
        char* line_copy = strdup(line);
        char* token = strtok(line_copy, ",");
        while (token && token_count < 15) {
            tokens[token_count++] = token;
            token = strtok(NULL, ",");
        }
        
        if (token_count >= 8) {
            // Kontrola fix quality
            int fix_quality = atoi(tokens[6]);
            gps_data.fix_valid = (fix_quality > 0);
            
            if (gps_data.fix_valid) {
                // Latitude
                float lat = atof(tokens[2]);
                if (strlen(tokens[2]) > 4) {
                    int degrees = (int)(lat / 100);
                    float minutes = lat - (degrees * 100);
                    gps_data.latitude = degrees + minutes / 60.0f;
                    if (tokens[3][0] == 'S') gps_data.latitude = -gps_data.latitude;
                }
                
                // Longitude
                float lon = atof(tokens[4]);
                if (strlen(tokens[4]) > 4) {
                    int degrees = (int)(lon / 100);
                    float minutes = lon - (degrees * 100);
                    gps_data.longitude = degrees + minutes / 60.0f;
                    if (tokens[5][0] == 'W') gps_data.longitude = -gps_data.longitude;
                }
                
                // Počet satelitů
                gps_data.satellites = atoi(tokens[7]);
            }
        }
        
        free(line_copy);
    }
}

// Hlavní GPS smyčka
void GPS_Process(void) {
    static char gps_buffer[128];
    static int gps_index = 0;
    
    int ch = USART_GetChar();
    if (ch == -1) return;
    
    if (ch == '\n') {
        gps_buffer[gps_index] = '\0';
        
        if (gps_index > 0) {
            GPS_ProcessNMEA(gps_buffer);
        }
        
        gps_index = 0;
    }
    else if (ch != '\r' && gps_index < sizeof(gps_buffer) - 1) {
        gps_buffer[gps_index++] = ch;
    }
}
```

## Flow Control

### Hardware flow control

```c
// RTS/CTS řízení toku
void USART1_RTSEnable(void);        // Povolit RTS
void USART1_RTSDisable(void);       // Zakázat RTS
void USART1_CTSEnable(void);        // Povolit CTS
void USART1_CTSDisable(void);       // Zakázat CTS

// Kontrola CTS stavu
Bool USART1_CTSActive(void);        // CTS je aktivní
```

### Software flow control

```c
// XON/XOFF implementace
#define XON  0x11
#define XOFF 0x13

volatile Bool flow_paused = False;

void USART_ProcessFlowControl(int ch) {
    if (ch == XOFF) {
        flow_paused = True;
    }
    else if (ch == XON) {
        flow_paused = False;
    }
}

void USART_SendWithFlow(const char* data, int len) {
    for (int i = 0; i < len; i++) {
        while (flow_paused) {
            // Čekat na XON
            DelayMs(1);
        }
        USART1_SendChar(data[i]);
    }
}
```

## Synchronní režim

```c
// Synchronní USART
void USART1_SyncMode(void) {
    USART1_ClkEnable();             // Povolit výstup hodin
    USART1_ClkPol(0);               // Polarita hodin
    USART1_ClkPhase(0);             // Fáze hodin
    USART1_LastBitClk(True);        // Hodiny pro poslední bit
}
```

## Tipy a triky

1. **Přenosová rychlost**: Ověřte, že zvolená rychlost je dosažitelná s danou frekvencí systému
2. **Buffery**: Používejte circular buffery pro přerušovací obsluhu
3. **Timeouty**: Vždy implementujte timeouty pro komunikaci
4. **Chyby**: Pravidelně kontrolujte chybové flagy
5. **DMA**: Pro vysoké přenosové rychlosti používejte DMA místo přerušení

## Výpočty

### Přenosová rychlost
```
Baudrate = PCLK / (16 × USARTDIV)
USARTDIV = PCLK / (16 × Baudrate)
```

### Chyba přenosové rychlosti
```
Error = |(Calculated_Baudrate - Desired_Baudrate)| / Desired_Baudrate × 100%
```

## Poznámky pro různé MCU

- **CH32V003**: USART1 s DMA podporou, základní funkcionalita (2KB SRAM)
- **CH32V002**: Vylepšená varianta s 4KB SRAM, 12-bit ADC, touch sensing
- **CH32V006**: Pokročilá varianta s více možnostmi remapování
- **CH32X035**: Nejpokročilejší s více USART jednotkami a funkcemi

## Mapování pinů

### USART1 standardní mapování:
- **TX**: PA9
- **RX**: PA10
- **CK**: PA8 (synchronní režim)
- **CTS**: PA11
- **RTS**: PA12

### Alternativní mapování:
- **TX**: PB6 (remap)
- **RX**: PB7 (remap)

### Remapování:
```c
GPIO_Remap_USART1(1);  // Přepnout na alternativní piny
```