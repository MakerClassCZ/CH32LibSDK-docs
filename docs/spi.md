# SPI - Sériová periferní rozhraní

## Přehled

SPI (Serial Peripheral Interface) modul umožňuje komunikaci s externími zařízeními pomocí synchronního sériového protokolu. SPI používá 4 piny: SCLK (hodiny), MOSI (Master Out Slave In), MISO (Master In Slave Out) a NSS/CS (Chip Select).

## Základní koncepty

### SPI Registry

CH32 mikrokontroléry mají typicky jednu SPI jednotku:
- **SPI1** - Hlavní SPI periférie (adresa 0x40013000)

### SPI Režimy

SPI může pracovat v několika režimech podle kombinace CPOL (Clock Polarity) a CPHA (Clock Phase):
- **Režim 0**: CPOL=0, CPHA=0 (default) - Hodiny klidové Low, vzorkování na první hraně
- **Režim 1**: CPOL=0, CPHA=1 - Hodiny klidové Low, vzorkování na druhé hraně  
- **Režim 2**: CPOL=1, CPHA=0 - Hodiny klidové High, vzorkování na první hraně
- **Režim 3**: CPOL=1, CPHA=1 - Hodiny klidové High, vzorkování na druhé hraně

### Frekvence hodin

```c
#define SPI_BAUD_DIV2    0  // Fhclk/2   (nejrychlejší)
#define SPI_BAUD_DIV4    1  // Fhclk/4
#define SPI_BAUD_DIV8    2  // Fhclk/8
#define SPI_BAUD_DIV16   3  // Fhclk/16
#define SPI_BAUD_DIV32   4  // Fhclk/32
#define SPI_BAUD_DIV64   5  // Fhclk/64
#define SPI_BAUD_DIV128  6  // Fhclk/128
#define SPI_BAUD_DIV256  7  // Fhclk/256 (nejpomalejší)
```

## Základní nastavení

### Inicializace jako Master

```c
// Základní nastavení SPI1 jako master
SPI1_Master();              // Nastavit jako master
SPI1_Baud(SPI_BAUD_DIV16);  // Nastavit frekvenci Fhclk/16
SPI1_Data8();               // 8-bitový datový formát
SPI1_MSB();                 // MSB první (default)
SPI1_ClockPolLow();         // Hodiny klidové Low (default)
SPI1_ClockPhaseFirst();     // Vzorkování na první hraně (default)
SPI1_Enable();              // Povolit SPI
```

### Inicializace jako Slave

```c
// Nastavení SPI1 jako slave
SPI1_Slave();               // Nastavit jako slave
SPI1_Data8();               // 8-bitový datový formát
SPI1_Enable();              // Povolit SPI
```

## Konfigurační funkce

### Režim Master/Slave

```c
void SPI1_Master(void);     // Nastavit jako master
void SPI1_Slave(void);      // Nastavit jako slave
```

### Nastavení hodin

```c
// Polarita hodin (CPOL)
void SPI1_ClockPolLow(void);   // Hodiny klidové Low (default)
void SPI1_ClockPolHigh(void);  // Hodiny klidové High

// Fáze hodin (CPHA)  
void SPI1_ClockPhaseFirst(void);   // Vzorkování na první hraně (default)
void SPI1_ClockPhaseSecond(void);  // Vzorkování na druhé hraně

// Frekvence hodin
void SPI1_Baud(int baud);   // Nastavit dělič frekvence
```

### Datový formát

```c
void SPI1_Data8(void);      // 8-bitový datový formát (default)
void SPI1_Data16(void);     // 16-bitový datový formát

void SPI1_MSB(void);        // MSB první (default)
void SPI1_LSB(void);        // LSB první (pouze master režim)
```

### NSS (Chip Select) řízení

```c
// Software řízení NSS
void SPI1_NSSSw(void);      // Software řízení NSS
void SPI1_NSSHigh(void);    // Nastavit NSS na High
void SPI1_NSSLow(void);     // Nastavit NSS na Low

// Hardware řízení NSS
void SPI1_NSSHw(void);      // Hardware řízení NSS (default)
void SPI1_SSEnable(void);   // Povolit SS výstup
void SPI1_SSDisable(void);  // Zakázat SS výstup
```

### Směr komunikace

```c
void SPI1_Duplex(void);     // Full-duplex (default)
void SPI1_RxOnly(void);     // Pouze příjem

// Jednosměrný režim
void SPI1_Bidi1(void);      // 1-vodičový bidirectional
void SPI1_Bidi2(void);      // 2-vodičový (default)
void SPI1_TxBidi(void);     // Vysílání v 1-vodičovém režimu
void SPI1_RxBidi(void);     // Příjem v 1-vodičovém režimu
```

## Přenos dat

### Základní funkce

```c
// Okamžitý zápis/čtení
void SPI1_Write(u16 data);  // Zapsat data do TX bufferu
u16 SPI1_Read(void);        // Přečíst data z RX bufferu

// Zápis/čtení s čekáním
void SPI1_SendWait(u16 data);  // Zapsat a čekat na dokončení
u16 SPI1_RecvWait(void);       // Čekat a přečíst data
```

### Kontrola stavu

```c
Bool SPI1_TxEmpty(void);    // TX buffer je prázdný
Bool SPI1_RxReady(void);    // RX buffer obsahuje data
Bool SPI1_Busy(void);       // SPI je zaneprázdněno
```

## Přerušení

### Povolení přerušení

```c
void SPI1_TxEmptyIntEnable(void);   // Přerušení při prázdném TX bufferu
void SPI1_RxReadyIntEnable(void);   // Přerušení při připravených RX datech
void SPI1_ErrIntEnable(void);       // Přerušení při chybě
```

### Zakázání přerušení

```c
void SPI1_TxEmptyIntDisable(void);
void SPI1_RxReadyIntDisable(void);
void SPI1_ErrIntDisable(void);
```

## Detekce a řešení chyb

### Kontrola chyb

```c
Bool SPI1_Over(void);       // Overrun chyba
Bool SPI1_Under(void);      // Underrun chyba
Bool SPI1_ModeErr(void);    // Chyba režimu
Bool SPI1_CRCErr(void);     // CRC chyba
```

### Vymazání chyb

```c
void SPI1_OverClr(void);    // Vymazat overrun
void SPI1_UnderClr(void);   // Vymazat underrun
void SPI1_ModeErrClr(void); // Vymazat chybu režimu
void SPI1_CRCErrClr(void);  // Vymazat CRC chybu
```

## CRC kontrola

### Nastavení CRC

```c
void SPI1_CRCEnable(void);      // Povolit hardware CRC
void SPI1_CRCDisable(void);     // Zakázat CRC (default)
void SPI1_Crc(u16 polynomial); // Nastavit CRC polynom
void SPI1_CRCNextEnable(void);  // Poslat CRC po dalším přenosu
```

### Čtení CRC

```c
u16 SPI1_GetCrc(void);      // Aktuální CRC polynom
u16 SPI1_RxCrc(void);       // Přijatý CRC
u16 SPI1_TxCrc(void);       // Vysílaný CRC
```

## DMA podpora

```c
void SPI1_TxDMAEnable(void);    // Povolit TX DMA
void SPI1_RxDMAEnable(void);    // Povolit RX DMA
void SPI1_TxDMADisable(void);   // Zakázat TX DMA
void SPI1_RxDMADisable(void);   // Zakázat RX DMA
```

## Vysokorychlostní režim

```c
void SPI1_HSEnable(void);   // Povolit high-speed (dělič 2)
void SPI1_HSDisable(void);  // Zakázat high-speed (default)
```

## Praktické příklady

### Komunikace s SD kartou

```c
// Inicializace pro SD kartu (SPI Mode 0, pomalá rychlost)
void SD_SPI_Init(void) {
    // Nastavení GPIO pinů pro SPI
    GPIO_Mode(PA5, GPIO_MODE_AF);     // SCLK
    GPIO_Mode(PA6, GPIO_MODE_IN);     // MISO
    GPIO_Mode(PA7, GPIO_MODE_AF);     // MOSI
    GPIO_Mode(PA4, GPIO_MODE_OUT);    // CS (software řízení)
    
    // Nastavení SPI
    SPI1_Master();
    SPI1_Baud(SPI_BAUD_DIV256);      // Pomalá rychlost pro inicializaci
    SPI1_Data8();
    SPI1_ClockPolLow();              // CPOL = 0
    SPI1_ClockPhaseFirst();          // CPHA = 0 (Mode 0)
    SPI1_NSSSw();                    // Software NSS
    SPI1_NSSHigh();                  // CS neaktivní
    SPI1_Enable();
}

// Poslání příkazu SD kartě
u8 SD_SendCommand(u8 cmd, u32 arg) {
    // CS aktivní
    GPIO_Out0(PA4);
    
    // Poslání příkazu (6 bytů)
    SPI1_SendWait(0x40 | cmd);       // Start bit + command
    SPI1_SendWait((arg >> 24) & 0xFF); // Argument byte 3
    SPI1_SendWait((arg >> 16) & 0xFF); // Argument byte 2
    SPI1_SendWait((arg >> 8) & 0xFF);  // Argument byte 1
    SPI1_SendWait(arg & 0xFF);         // Argument byte 0
    SPI1_SendWait(0x95);               // CRC (pro CMD0)
    
    // Čekání na odpověď
    u8 response;
    for (int i = 0; i < 8; i++) {
        response = SPI1_RecvWait();
        if (response != 0xFF) break;
    }
    
    return response;
}
```

### VGA generování pomocí SPI

```c
// Reálný příklad z BabyPad VGA driveru
void VGA_SPI_Init(void) {
    // SPI pro pixel output
    RCC_SPI1ClkEnable();            // Povolit SPI hodiny
    RCC_SPI1Reset();                // Reset na výchozí stav
    
    SPI1_Baud(SPI_HDIV);            // Horizontální dělič pro pixel timing
    SPI1_Data8();                   // 8-bitový režim
    SPI1_MSB();                     // MSB první (bit 7 první)
    SPI1_NSSSw();                   // Software NSS řízení
    SPI1_NSSHigh();                 // NSS HIGH pro master výstup
    SPI1_SSDisable();               // Zakázat SS výstup
    SPI1_Bidi1();                   // 1-vodičový bidirectional režim
    SPI1_TxBidi();                  // Pouze transmit
    SPI1_Master();                  // Master režim
    SPI1_Enable();                  // Povolit SPI
    
    // GPIO nastavení pro VGA výstup
    GPIO_Remap_SPI1(0);             // Remap MOSI na PC6
    GPIO_Mode(PC6, GPIO_MODE_AF_FAST); // MOSI jako fast AF output
}
```

### Komunikace s OLED displejem (SSD1306)

```c
// Inicializace pro OLED displej
void OLED_SPI_Init(void) {
    // GPIO nastavení
    GPIO_Mode(PA5, GPIO_MODE_AF);     // SCLK
    GPIO_Mode(PA7, GPIO_MODE_AF);     // MOSI (pouze TX)
    GPIO_Mode(PA4, GPIO_MODE_OUT);    // CS
    GPIO_Mode(PA3, GPIO_MODE_OUT);    // DC (Data/Command)
    GPIO_Mode(PA2, GPIO_MODE_OUT);    // RST
    
    // SPI nastavení
    SPI1_Master();
    SPI1_Baud(SPI_BAUD_DIV4);        // Rychlá komunikace
    SPI1_Data8();
    SPI1_ClockPolLow();              // CPOL = 0
    SPI1_ClockPhaseFirst();          // CPHA = 0
    SPI1_NSSSw();
    SPI1_Enable();
}

// Poslání příkazu
void OLED_Command(u8 cmd) {
    GPIO_Out0(PA3);     // DC = 0 (command)
    GPIO_Out0(PA4);     // CS aktivní
    SPI1_SendWait(cmd);
    GPIO_Out1(PA4);     // CS neaktivní
}

// Poslání dat
void OLED_Data(u8 data) {
    GPIO_Out1(PA3);     // DC = 1 (data)
    GPIO_Out0(PA4);     // CS aktivní
    SPI1_SendWait(data);
    GPIO_Out1(PA4);     // CS neaktivní
}
```

### Čtení z ADC přes SPI (MCP3008)

```c
// Čtení z 8-kanálového ADC MCP3008
u16 MCP3008_Read(u8 channel) {
    // CS aktivní
    GPIO_Out0(PA4);
    
    // Start bit + single-ended + channel
    u8 cmd1 = 0x01;                    // Start bit
    u8 cmd2 = (0x08 | channel) << 4;  // Single-ended + channel
    u8 cmd3 = 0x00;                   // Dummy byte
    
    SPI1_SendWait(cmd1);
    u8 result_high = SPI1_RecvWait();
    
    SPI1_SendWait(cmd2);
    result_high = SPI1_RecvWait();
    
    SPI1_SendWait(cmd3);
    u8 result_low = SPI1_RecvWait();
    
    // CS neaktivní
    GPIO_Out1(PA4);
    
    // Sestavení 10-bitového výsledku
    u16 result = ((result_high & 0x03) << 8) | result_low;
    return result;
}
```

### Rychlé graphics rendering pro hry

```c
// Reálné příklady z her - optimalizované graphics přes SPI
// Typicky pro OLED displeje nebo VGA timing

// Rychlé vykreslování spritů s DMA
void DrawSprite_Fast(const u8* sprite_data, int x, int y, int width, int height) {
    // Nastavení display okna (závislé na displeji)
    OLED_SetWindow(x, y, width, height);
    
    // Spustit DMA přenos místo pomalého polling
    SPI1_DMAEnable();
    DMA_Start(sprite_data, width * height);
    
    // Během DMA přenosu můžeme dělat jiné věci
    while (!DMA_Complete()) {
        // Zpracovat další game logic
        Update_Game_Physics();
    }
}

// Optimalizované vykreslování tile map (reálný kód z her)
void DrawTileMap_Optimized(void) {
    // Batch SPI operace pro lepší výkon
    GPIO_Out0(PA4);  // CS aktivní na začátku
    
    for (int y = 0; y < MAP_HEIGHT; y++) {
        for (int x = 0; x < MAP_WIDTH; x++) {
            u8 tile = GetTile(x, y);
            
            // Rychlé vykreslení tile bez CS toggle
            if (tile != TILE_EMPTY) {
                const u8* tile_data = &ImgTiles[tile * TILE_SIZE];
                
                // Použít DMA pro celý řádek tiles
                SPI1_SendBlock(tile_data, TILE_SIZE);
            }
        }
    }
    
    GPIO_Out1(PA4);  // CS neaktivní na konci
}

// Frame buffer optimalizace pro VGA (reálný kód z BabyPad)
volatile u8 frame_buffer[160*20];  // Line buffer pro VGA
volatile Bool line_ready = False;

// DMA callback pro další řádek
void DMA_LineComplete_Handler(void) {
    line_ready = True;
    PrepareNextLine();  // Připravit další řádek v background
}

// VGA line rendering s double buffering
void VGA_RenderLine_Fast(int line_num) {
    // Použít hardware SPI pro pixel timing
    SPI1_Baud(SPI_HDIV);            // Maximální rychlost pro VGA timing
    
    // DMA setup pro celý řádek najednou
    SPI1_DMASetup(&frame_buffer[line_num * 160], 160);
    
    // Spustit při HSYNC signálu (timer interrupt)
    SPI1_DMAStart();
}
```

### Graphics optimalizace techniky

```c
// Sprite clipping a culling (z reálných her)
Bool SpriteVisible(int x, int y, int w, int h) {
    // Rychlý visibility test před expensive rendering
    return (x + w >= 0 && x < SCREEN_WIDTH && 
            y + h >= 0 && y < SCREEN_HEIGHT);
}

// Dirty rectangle tracking pro částečné updaty
typedef struct {
    int x, y, w, h;
    Bool dirty;
} DirtyRect_t;

DirtyRect_t dirty_regions[MAX_DIRTY];
int dirty_count = 0;

void MarkDirty(int x, int y, int w, int h) {
    if (dirty_count < MAX_DIRTY) {
        dirty_regions[dirty_count] = {x, y, w, h, True};
        dirty_count++;
    }
}

// Překreslit pouze změněné oblasti
void UpdateDisplay_Dirty(void) {
    for (int i = 0; i < dirty_count; i++) {
        if (dirty_regions[i].dirty) {
            DrawRegion_Fast(dirty_regions[i].x, dirty_regions[i].y,
                          dirty_regions[i].w, dirty_regions[i].h);
            dirty_regions[i].dirty = False;
        }
    }
    dirty_count = 0;
}

// Fast pixel manipulation pro collision detection
Bool CheckPixelCollision(int x1, int y1, const u8* sprite1,
                        int x2, int y2, const u8* sprite2,
                        int width, int height) {
    // Per-pixel collision s bit masking
    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width/8; x++) {  // 8 pixelů per byte
            u8 mask1 = sprite1[y * (width/8) + x];
            u8 mask2 = sprite2[y * (width/8) + x];
            
            if (mask1 & mask2) {  // Collision detected
                return True;
            }
        }
    }
    return False;
}
```

### SPI Graphics pipeline pro hry

```c
// Multi-layer rendering systém (reálný pattern z her)
typedef enum {
    LAYER_BACKGROUND = 0,
    LAYER_TILES = 1,
    LAYER_SPRITES = 2,
    LAYER_UI = 3,
    LAYER_MAX = 4
} RenderLayer_t;

typedef struct {
    const u8* data;
    int x, y, w, h;
    u8 layer;
    Bool visible;
} RenderObject_t;

RenderObject_t render_queue[MAX_OBJECTS];
int render_count = 0;

// Přidat objekt do render fronty
void AddToRenderQueue(const u8* data, int x, int y, int w, int h, RenderLayer_t layer) {
    if (render_count < MAX_OBJECTS && SpriteVisible(x, y, w, h)) {
        render_queue[render_count] = {data, x, y, w, h, layer, True};
        render_count++;
    }
}

// Seřadit podle layers a vykreslit
void FlushRenderQueue(void) {
    // Sort by layer (bubble sort pro malé počty)
    for (int i = 0; i < render_count-1; i++) {
        for (int j = i+1; j < render_count; j++) {
            if (render_queue[i].layer > render_queue[j].layer) {
                RenderObject_t temp = render_queue[i];
                render_queue[i] = render_queue[j];
                render_queue[j] = temp;
            }
        }
    }
    
    // Batch SPI operace podle layers
    GPIO_Out0(PA4);  // CS aktivní
    for (int i = 0; i < render_count; i++) {
        if (render_queue[i].visible) {
            SPI_DrawSprite_Fast(&render_queue[i]);
        }
    }
    GPIO_Out1(PA4);  // CS neaktivní
    
    render_count = 0;  // Clear queue
}

// Herní render loop
void Game_Render_Frame(void) {
    // Layer 0: Background
    AddToRenderQueue(BackgroundImg, 0, 0, SCREEN_WIDTH, SCREEN_HEIGHT, LAYER_BACKGROUND);
    
    // Layer 1: Game tiles
    for (int y = 0; y < MAP_HEIGHT; y++) {
        for (int x = 0; x < MAP_WIDTH; x++) {
            u8 tile = GetTile(x, y);
            if (tile != TILE_EMPTY) {
                AddToRenderQueue(&TileData[tile], x*TILE_SIZE, y*TILE_SIZE, 
                               TILE_SIZE, TILE_SIZE, LAYER_TILES);
            }
        }
    }
    
    // Layer 2: Game objects (enemies, bullets, player)
    for (int i = 0; i < enemy_count; i++) {
        AddToRenderQueue(EnemySprite, enemies[i].x, enemies[i].y, 
                       ENEMY_WIDTH, ENEMY_HEIGHT, LAYER_SPRITES);
    }
    
    AddToRenderQueue(PlayerSprite, player.x, player.y, 
                   PLAYER_WIDTH, PLAYER_HEIGHT, LAYER_SPRITES);
    
    // Layer 3: UI elements
    AddToRenderQueue(ScoreDisplay, SCORE_X, SCORE_Y, 
                   SCORE_WIDTH, SCORE_HEIGHT, LAYER_UI);
    
    // Vykreslit všechno v jednom SPI batch
    FlushRenderQueue();
}
```

## Tipy a triky

1. **Rychlost komunikace**: Začněte s pomalou rychlostí a postupně zvyšujte
2. **Timing**: Různá zařízení vyžadují různé SPI režimy - zkontrolujte datasheet
3. **CS řízení**: Pro více zařízení použijte software řízení CS s různými GPIO piny
4. **DMA**: Pro velké přenosy používejte DMA pro zvýšení výkonu
5. **Chyby**: Vždy kontrolujte chybové flagy, zejména při vysokých rychlostech
6. **Graphics batching**: Minimalizujte CS toggles - seskupte více SPI operací do jednoho CS cyklu
7. **Frame rate**: Pro hry použijte VGA timing nebo fixed 60 FPS timer pro smooth graphics
8. **Sprite culling**: Testujte viditelnost před expensive rendering operacemi
9. **Layer ordering**: Seřaďte objekty podle render priority pro správné překrývání

## Poznámky pro různé MCU

- **CH32V003**: Základní SPI1 s DMA podporou (2KB SRAM, 10-bit ADC)
- **CH32V002**: Vylepšená varianta s 4KB SRAM, 12-bit ADC, touch sensing
- **CH32V006**: Pokročilá varianta s více možnostmi remapování
- **CH32X035**: Nejpokročilejší s více SPI jednotkami a funkcemi

## Mapování pinů

### Standardní mapování SPI1:
- **NSS**: PA4
- **SCK**: PA5  
- **MISO**: PA6
- **MOSI**: PA7

### Alternativní mapování (pokud je podporováno):
- Použijte `GPIO_Remap_SPI1()` pro změnu mapování pinů