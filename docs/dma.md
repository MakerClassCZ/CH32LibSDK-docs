# DMA - Direct Memory Access

## Přehled

DMA (Direct Memory Access) umožňuje přenos dat mezi pamětí a perifériemi nebo mezi různými oblastmi paměti bez zatížení CPU. CH32 obsahuje DMA1 s 7 kanály podporujícími různé periférie a režimy přenosu.

## Základní koncepty

### DMA1 kanály a připojení

```
Kanál 1: ADC1, TIM2_CH3
Kanál 2: SPI1_RX, TIM1_CH1, TIM2_UP
Kanál 3: SPI1_TX, TIM1_CH2
Kanál 4: USART1_TX, TIM1_CH4, TIM1_TRIG, TIM1_COM
Kanál 5: USART1_RX, TIM1_UP, TIM2_CH1
Kanál 6: I2C1_TX, TIM1_CH3
Kanál 7: I2C1_RX, TIM2_CH2, TIM2_CH4
```

### Režimy přenosu

- **Peripheral-to-Memory**: Periférie → paměť (např. ADC, SPI RX)
- **Memory-to-Peripheral**: Paměť → periférie (např. SPI TX, USART TX)
- **Memory-to-Memory**: Paměť → paměť (kopírování dat)

### Velikosti dat

```c
#define DMA_SIZE_8    0   // 8 bitů (1 byte)
#define DMA_SIZE_16   1   // 16 bitů (2 bytes)
#define DMA_SIZE_32   2   // 32 bitů (4 bytes)
```

### Priority kanálů

```c
#define DMA_CFG_PRIOR_LOW       0   // Nízká priorita
#define DMA_CFG_PRIOR_MED       1   // Střední priorita
#define DMA_CFG_PRIOR_HIGH      2   // Vysoká priorita
#define DMA_CFG_PRIOR_VERYHIGH  3   // Velmi vysoká priorita
```

## Základní nastavení

### Inicializace DMA kanálu

```c
// Kompletní nastavení DMA kanálu
void DMA_ChannelInit(int channel, const volatile void* per_addr, 
                     volatile void* mem_addr, int count, u32 config) {
    // Povolit DMA hodiny
    RCC_DMA1ClkEnable();
    
    // Získat ukazatel na kanál
    DMAchan_t* ch = DMA1_Chan(channel);
    
    // Zakázat kanál pro konfiguraci
    DMA_ChanDisable(ch);
    
    // Nastavit adresy
    DMA_PerAddr(ch, per_addr);          // Adresa periférie
    DMA_MemAddr(ch, mem_addr);          // Adresa paměti
    
    // Nastavit počet přenosů
    DMA_Cnt(ch, count);
    
    // Nastavit konfiguraci
    DMA_Cfg(ch, config);
}

// Spustit DMA přenos
void DMA_Start(int channel) {
    DMAchan_t* ch = DMA1_Chan(channel);
    DMA_ChanEnable(ch);
}

// Zastavit DMA přenos
void DMA_Stop(int channel) {
    DMAchan_t* ch = DMA1_Chan(channel);
    DMA_ChanDisable(ch);
}
```

### Konfigurační flagy

```c
// Základní konfigurační flagy
#define DMA_CFG_COMPINT     B1    // Přerušení při dokončení
#define DMA_CFG_HALFINT     B2    // Přerušení při polovině
#define DMA_CFG_TRANSERRINT B3    // Přerušení při chybě
#define DMA_CFG_DIRFROMMEM  B4    // Směr: paměť → periférie
#define DMA_CFG_CIRC        B5    // Cyklický režim
#define DMA_CFG_PERINC      B6    // Increment periferní adresy
#define DMA_CFG_MEMINC      B7    // Increment paměťové adresy

// Kombinované konfigurace
u32 DMA_Config_M2P = DMA_CFG_DIRFROMMEM | DMA_CFG_MEMINC | 
                     DMA_CFG_COMPINT | DMA_CFG_MSIZE_8 | DMA_CFG_PSIZE_8;

u32 DMA_Config_P2M = DMA_CFG_MEMINC | DMA_CFG_COMPINT | 
                     DMA_CFG_MSIZE_8 | DMA_CFG_PSIZE_8;

u32 DMA_Config_M2M = DMA_CFG_MEMINC | DMA_CFG_PERINC | 
                     DMA_CFG_COMPINT | DMA_CFG_MSIZE_8 | DMA_CFG_PSIZE_8;
```

## Periferní DMA

### ADC s DMA

```c
#define ADC_BUFFER_SIZE 100
u16 adc_buffer[ADC_BUFFER_SIZE];

// Inicializace ADC s DMA
void ADC_DMA_Init(void) {
    // ADC inicializace
    ADC1_InitSingle();
    ADC1_DMAEnable();                   // Povolit ADC DMA požadavky
    
    // DMA konfigurace pro ADC (kanál 1)
    u32 config = DMA_CFG_MEMINC |       // Increment paměťové adresy
                 DMA_CFG_CIRC |          // Cyklický režim
                 DMA_CFG_COMPINT |       // Přerušení při dokončení
                 DMA_CFG_MSIZE_16 |      // 16-bit paměť
                 DMA_CFG_PSIZE_16 |      // 16-bit periférie
                 DMA_CFG_PRIOR_HIGH;     // Vysoká priorita
    
    DMA_ChannelInit(1, &ADC1->RDATAR, adc_buffer, ADC_BUFFER_SIZE, config);
    
    // Povolit přerušení DMA
    DMA_CompIntEnable(DMA1_Chan(1));
    
    // Spustit kontinuální ADC + DMA
    ADC1_ContEnable();
    DMA_Start(1);
    ADC1_SwStart();
}

// Obsluha DMA přerušení
void DMA1_Channel1_Handler(void) {
    if (DMA1_Comp(1)) {
        // Buffer je plný - zpracovat data
        ProcessADCData(adc_buffer, ADC_BUFFER_SIZE);
        DMA1_CompClr(1);
    }
}
```

### USART TX s DMA

```c
char tx_buffer[256];
volatile Bool tx_complete = False;

// USART TX s DMA
void USART_DMA_TX_Init(void) {
    // USART inicializace
    USART1_BasicInit();
    USART1_TxDMAEnable();               // Povolit USART TX DMA
    
    // DMA konfigurace pro USART TX (kanál 4)
    u32 config = DMA_CFG_DIRFROMMEM |   // Paměť → periférie
                 DMA_CFG_MEMINC |       // Increment paměťové adresy
                 DMA_CFG_COMPINT |      // Přerušení při dokončení
                 DMA_CFG_MSIZE_8 |      // 8-bit paměť
                 DMA_CFG_PSIZE_8 |      // 8-bit periférie
                 DMA_CFG_PRIOR_MED;     // Střední priorita
    
    DMA_ChannelInit(4, &USART1->DATAR, tx_buffer, 0, config);
    DMA_CompIntEnable(DMA1_Chan(4));
}

// Poslání dat přes DMA
void USART_DMA_Send(const char* data, int len) {
    // Čekat na dokončení předchozího přenosu
    while (!tx_complete) {}
    
    // Kopírovat data do bufferu
    memcpy(tx_buffer, data, len);
    
    // Nastavit DMA
    DMAchan_t* ch = DMA1_Chan(4);
    DMA_ChanDisable(ch);
    DMA_MemAddr(ch, tx_buffer);
    DMA_Cnt(ch, len);
    
    tx_complete = False;
    DMA_ChanEnable(ch);
}

// Obsluha DMA TX
void DMA1_Channel4_Handler(void) {
    if (DMA1_Comp(4)) {
        tx_complete = True;
        DMA1_CompClr(4);
    }
}
```

### USART RX s DMA

```c
#define RX_BUFFER_SIZE 128
char rx_buffer[RX_BUFFER_SIZE];
volatile int rx_count = 0;

// USART RX s DMA
void USART_DMA_RX_Init(void) {
    // USART inicializace
    USART1_BasicInit();
    USART1_RxDMAEnable();               // Povolit USART RX DMA
    
    // DMA konfigurace pro USART RX (kanál 5)
    u32 config = DMA_CFG_MEMINC |       // Increment paměťové adresy
                 DMA_CFG_CIRC |          // Cyklický režim
                 DMA_CFG_HALFINT |       // Přerušení při polovině
                 DMA_CFG_COMPINT |       // Přerušení při dokončení
                 DMA_CFG_MSIZE_8 |       // 8-bit paměť
                 DMA_CFG_PSIZE_8 |       // 8-bit periférie
                 DMA_CFG_PRIOR_HIGH;     // Vysoká priorita
    
    DMA_ChannelInit(5, &USART1->DATAR, rx_buffer, RX_BUFFER_SIZE, config);
    DMA_HalfIntEnable(DMA1_Chan(5));
    DMA_CompIntEnable(DMA1_Chan(5));
    DMA_Start(5);
}

// Obsluha DMA RX
void DMA1_Channel5_Handler(void) {
    if (DMA1_Half(5)) {
        // První polovina bufferu naplněna
        ProcessRXData(rx_buffer, RX_BUFFER_SIZE / 2);
        DMA1_HalfClr(5);
    }
    
    if (DMA1_Comp(5)) {
        // Druhá polovina bufferu naplněna
        ProcessRXData(&rx_buffer[RX_BUFFER_SIZE / 2], RX_BUFFER_SIZE / 2);
        DMA1_CompClr(5);
    }
}
```

### SPI s DMA

```c
u8 spi_tx_buffer[64];
u8 spi_rx_buffer[64];

// SPI full-duplex s DMA
void SPI_DMA_Init(void) {
    // SPI inicializace
    SPI1_Master();
    SPI1_Baud(SPI_BAUD_DIV16);
    SPI1_Enable();
    
    // Povolit SPI DMA
    SPI1_TxDMAEnable();
    SPI1_RxDMAEnable();
    
    // DMA TX konfigurace (kanál 3)
    u32 tx_config = DMA_CFG_DIRFROMMEM | DMA_CFG_MEMINC | 
                    DMA_CFG_MSIZE_8 | DMA_CFG_PSIZE_8 | DMA_CFG_PRIOR_HIGH;
    
    DMA_ChannelInit(3, &SPI1->DATAR, spi_tx_buffer, 0, tx_config);
    
    // DMA RX konfigurace (kanál 2)
    u32 rx_config = DMA_CFG_MEMINC | DMA_CFG_COMPINT |
                    DMA_CFG_MSIZE_8 | DMA_CFG_PSIZE_8 | DMA_CFG_PRIOR_HIGH;
    
    DMA_ChannelInit(2, &SPI1->DATAR, spi_rx_buffer, 0, rx_config);
    DMA_CompIntEnable(DMA1_Chan(2));
}

// SPI přenos s DMA
void SPI_DMA_Transfer(const u8* tx_data, u8* rx_data, int len) {
    // Kopírovat TX data
    memcpy(spi_tx_buffer, tx_data, len);
    
    // Nastavit DMA kanály
    DMAchan_t* tx_ch = DMA1_Chan(3);
    DMAchan_t* rx_ch = DMA1_Chan(2);
    
    DMA_ChanDisable(tx_ch);
    DMA_ChanDisable(rx_ch);
    
    DMA_Cnt(tx_ch, len);
    DMA_Cnt(rx_ch, len);
    
    // Spustit RX před TX
    DMA_ChanEnable(rx_ch);
    DMA_ChanEnable(tx_ch);
}

// Obsluha SPI RX DMA
void DMA1_Channel2_Handler(void) {
    if (DMA1_Comp(2)) {
        // SPI přenos dokončen - data v spi_rx_buffer
        ProcessSPIData(spi_rx_buffer);
        DMA1_CompClr(2);
    }
}
```

## Memory-to-Memory DMA

### Rychlé kopírování dat

```c
// Kopírování dat pomocí DMA
void DMA_MemCopy(const void* src, void* dst, int count, int data_size) {
    // DMA konfigurace pro memory-to-memory
    u32 config = DMA_CFG_MEMINC |       // Increment memory adresy
                 DMA_CFG_PERINC |       // Increment peripheral adresy
                 DMA_CFG_COMPINT |      // Přerušení při dokončení
                 DMA_CFG_MERM2MEM |     // Memory-to-memory režim
                 DMA_CFG_PRIOR_VERYHIGH;
    
    // Nastavit velikost dat
    switch (data_size) {
        case 1: config |= DMA_CFG_MSIZE_8 | DMA_CFG_PSIZE_8; break;
        case 2: config |= DMA_CFG_MSIZE_16 | DMA_CFG_PSIZE_16; break;
        case 4: config |= DMA_CFG_MSIZE_32 | DMA_CFG_PSIZE_32; break;
    }
    
    // Použít volný kanál (např. 1)
    DMA_ChannelInit(1, src, dst, count, config);
    DMA_Start(1);
    
    // Čekat na dokončení
    while (!DMA1_Comp(1)) {}
    DMA1_CompClr(1);
    DMA_Stop(1);
}

// Rychlé vymazání paměti
void DMA_MemSet(void* dst, u32 value, int count) {
    static u32 fill_value;
    fill_value = value;
    
    u32 config = DMA_CFG_MEMINC | DMA_CFG_COMPINT | DMA_CFG_MERM2MEM |
                 DMA_CFG_MSIZE_32 | DMA_CFG_PSIZE_32 | DMA_CFG_PRIOR_VERYHIGH;
    
    DMA_ChannelInit(1, &fill_value, dst, count / 4, config);
    DMA_Start(1);
    
    while (!DMA1_Comp(1)) {}
    DMA1_CompClr(1);
    DMA_Stop(1);
}
```

## Pokročilé techniky

### Double Buffering

```c
#define BUFFER_SIZE 256
u16 buffer_a[BUFFER_SIZE];
u16 buffer_b[BUFFER_SIZE];
volatile u16* active_buffer = buffer_a;
volatile u16* processing_buffer = buffer_b;

// Double buffering pro kontinuální zpracování
void ADC_DoubleBuffer_Init(void) {
    ADC1_InitSingle();
    ADC1_DMAEnable();
    
    u32 config = DMA_CFG_MEMINC | DMA_CFG_HALFINT | DMA_CFG_COMPINT |
                 DMA_CFG_MSIZE_16 | DMA_CFG_PSIZE_16 | DMA_CFG_PRIOR_HIGH;
    
    DMA_ChannelInit(1, &ADC1->RDATAR, buffer_a, BUFFER_SIZE * 2, config);
    DMA_HalfIntEnable(DMA1_Chan(1));
    DMA_CompIntEnable(DMA1_Chan(1));
    
    DMA_Start(1);
    ADC1_ContEnable();
    ADC1_SwStart();
}

void DMA1_Channel1_Handler(void) {
    if (DMA1_Half(1)) {
        // První polovina dokončena - zpracovat buffer A
        processing_buffer = buffer_a;
        active_buffer = buffer_b;
        ProcessBuffer(processing_buffer, BUFFER_SIZE);
        DMA1_HalfClr(1);
    }
    
    if (DMA1_Comp(1)) {
        // Druhá polovina dokončena - zpracovat buffer B
        processing_buffer = buffer_b;
        active_buffer = buffer_a;
        ProcessBuffer(processing_buffer, BUFFER_SIZE);
        DMA1_CompClr(1);
    }
}
```

### Linked List DMA (simulace)

```c
typedef struct {
    void* src;
    void* dst;
    int count;
    int next;  // Index dalšího transferu (-1 = konec)
} DMA_Transfer_t;

DMA_Transfer_t transfer_list[10];
volatile int current_transfer = 0;

// Simulace linked list DMA
void DMA_LinkedTransfer(const DMA_Transfer_t* transfers, int start_index) {
    current_transfer = start_index;
    
    if (current_transfer >= 0) {
        const DMA_Transfer_t* t = &transfers[current_transfer];
        
        u32 config = DMA_CFG_MEMINC | DMA_CFG_PERINC | DMA_CFG_COMPINT |
                     DMA_CFG_MERM2MEM | DMA_CFG_MSIZE_8 | DMA_CFG_PSIZE_8;
        
        DMA_ChannelInit(1, t->src, t->dst, t->count, config);
        DMA_CompIntEnable(DMA1_Chan(1));
        DMA_Start(1);
    }
}

void DMA1_Channel1_Handler(void) {
    if (DMA1_Comp(1)) {
        DMA1_CompClr(1);
        DMA_Stop(1);
        
        // Přejít na další transfer
        int next = transfer_list[current_transfer].next;
        if (next >= 0) {
            const DMA_Transfer_t* t = &transfer_list[next];
            current_transfer = next;
            
            DMA_ChannelInit(1, t->src, t->dst, t->count, 
                           DMA_CFG_MEMINC | DMA_CFG_PERINC | DMA_CFG_COMPINT |
                           DMA_CFG_MERM2MEM | DMA_CFG_MSIZE_8 | DMA_CFG_PSIZE_8);
            DMA_Start(1);
        }
    }
}
```

## Praktické příklady

### Audio streaming

```c
#define AUDIO_BUFFER_SIZE 512
s16 audio_buffer_a[AUDIO_BUFFER_SIZE];
s16 audio_buffer_b[AUDIO_BUFFER_SIZE];
volatile Bool buffer_a_ready = False;
volatile Bool buffer_b_ready = False;

// Audio streaming s double buffering
void Audio_Stream_Init(void) {
    // Timer pro audio sampling (44.1kHz)
    TIM1_Presc(0);
    TIM1_Load(48000000 / 44100 - 1);    // 44.1kHz
    TIM1_CC1Mode(TIM_COMP_PWM1);
    TIM1_AutoReloadEnable();
    TIM1_Enable();
    
    // DMA pro timer update
    TIM1_DMAEnable();
    
    u32 config = DMA_CFG_MEMINC | DMA_CFG_CIRC | DMA_CFG_HALFINT | 
                 DMA_CFG_COMPINT | DMA_CFG_MSIZE_16 | DMA_CFG_PSIZE_16;
    
    DMA_ChannelInit(5, &TIM1->CH1CVR, audio_buffer_a, AUDIO_BUFFER_SIZE * 2, config);
    DMA_HalfIntEnable(DMA1_Chan(5));
    DMA_CompIntEnable(DMA1_Chan(5));
    DMA_Start(5);
}

void DMA1_Channel5_Handler(void) {
    if (DMA1_Half(5)) {
        // Buffer A je hotový
        buffer_a_ready = True;
        DMA1_HalfClr(5);
    }
    
    if (DMA1_Comp(5)) {
        // Buffer B je hotový
        buffer_b_ready = True;
        DMA1_CompClr(5);
    }
}

// Hlavní audio smyčka
void Audio_Process(void) {
    if (buffer_a_ready) {
        ProcessAudioBuffer(audio_buffer_a, AUDIO_BUFFER_SIZE);
        buffer_a_ready = False;
    }
    
    if (buffer_b_ready) {
        ProcessAudioBuffer(audio_buffer_b, AUDIO_BUFFER_SIZE);
        buffer_b_ready = False;
    }
}
```

### Data logger s SD kartou

```c
#define LOG_BUFFER_SIZE 1024
u8 log_buffer[LOG_BUFFER_SIZE];
volatile int log_index = 0;
volatile Bool buffer_full = False;

// High-speed data logging
void DataLogger_Init(void) {
    // SPI pro SD kartu
    SPI1_Master();
    SPI1_Baud(SPI_BAUD_DIV4);           // Rychlá komunikace
    SPI1_TxDMAEnable();
    SPI1_Enable();
    
    // ADC pro high-speed sampling
    ADC1_InitSingle();
    ADC1_SampTime(0, ADC_SAMP_15);      // Nejrychlejší
    ADC1_DMAEnable();
    
    // Timer pro vzorkování (100kHz)
    TIM1_Presc(0);
    TIM1_Load(480 - 1);                 // 100kHz
    ADC1_ExtSel(ADC_EXT_T1_CC1);
    ADC1_ExtTrigEnable();
    
    // DMA pro ADC
    u32 config = DMA_CFG_MEMINC | DMA_CFG_COMPINT |
                 DMA_CFG_MSIZE_8 | DMA_CFG_PSIZE_16;
    
    DMA_ChannelInit(1, &ADC1->RDATAR, log_buffer, LOG_BUFFER_SIZE, config);
    DMA_CompIntEnable(DMA1_Chan(1));
    
    TIM1_Enable();
    DMA_Start(1);
}

void DMA1_Channel1_Handler(void) {
    if (DMA1_Comp(1)) {
        buffer_full = True;
        DMA1_CompClr(1);
        
        // Restart DMA pro další batch
        DMA_Stop(1);
        DMA_Cnt(DMA1_Chan(1), LOG_BUFFER_SIZE);
        DMA_Start(1);
    }
}

// Zápis na SD kartu
void DataLogger_Process(void) {
    if (buffer_full) {
        // DMA transfer na SPI TX
        DMAchan_t* spi_ch = DMA1_Chan(3);
        DMA_ChanDisable(spi_ch);
        DMA_MemAddr(spi_ch, log_buffer);
        DMA_Cnt(spi_ch, LOG_BUFFER_SIZE);
        DMA_ChanEnable(spi_ch);
        
        buffer_full = False;
    }
}
```

### Memory stress test

```c
// Memory bandwidth test pomocí DMA
void Memory_BandwidthTest(void) {
    static u32 src_buffer[1024];
    static u32 dst_buffer[1024];
    
    // Inicializace test dat
    for (int i = 0; i < 1024; i++) {
        src_buffer[i] = i;
    }
    
    u32 start_time = Time();
    
    // 32-bit DMA transfer
    u32 config = DMA_CFG_MEMINC | DMA_CFG_PERINC | DMA_CFG_COMPINT |
                 DMA_CFG_MERM2MEM | DMA_CFG_MSIZE_32 | DMA_CFG_PSIZE_32 |
                 DMA_CFG_PRIOR_VERYHIGH;
    
    for (int test = 0; test < 100; test++) {
        DMA_ChannelInit(1, src_buffer, dst_buffer, 1024, config);
        DMA_Start(1);
        
        while (!DMA1_Comp(1)) {}
        DMA1_CompClr(1);
        DMA_Stop(1);
    }
    
    u32 end_time = Time();
    u32 duration = end_time - start_time;
    
    // Výpočet bandwidth (100 transfers × 1024 × 4 bytes)
    u32 bytes_transferred = 100 * 1024 * 4;
    u32 bandwidth = (bytes_transferred * HCLK_PER_SEC) / duration;
    
    printf("Memory bandwidth: %lu MB/s\n", bandwidth / (1024 * 1024));
}
```

## Kontrola stavu a chyb

### Status flagy

```c
// Kontrola stavu DMA kanálu
Bool DMA1_Int(int ch);              // Globální interrupt flag
Bool DMA1_Comp(int ch);             // Transfer complete
Bool DMA1_Half(int ch);             // Half transfer
Bool DMA1_TransErr(int ch);         // Transfer error

// Vymazání flagů
void DMA1_IntClr(int ch);           // Vymazat global flag
void DMA1_CompClr(int ch);          // Vymazat complete flag
void DMA1_HalfClr(int ch);          // Vymazat half flag
void DMA1_TransErrClr(int ch);      // Vymazat error flag
```

### Error handling

```c
// Obsluha DMA chyb
void DMA_ErrorHandler(int channel) {
    if (DMA1_TransErr(channel)) {
        printf("DMA%d transfer error!\n", channel);
        
        // Zastavit a resetovat kanál
        DMA_Stop(channel);
        DMA1_TransErrClr(channel);
        
        // Reinicializace nebo recovery akce
        // ...
    }
}
```

## Tipy a triky

1. **Priorita**: Vyšší priorita kanálů má přednost při současných požadavcích
2. **Alignment**: 16/32-bit transfery vyžadují správné zarovnání adres
3. **Cache**: U vyšších MCU pozor na cache coherency
4. **Interrupts**: Používejte Half Transfer pro double buffering
5. **Memory**: DMA přistupuje k paměti nezávisle na CPU

## Výpočty

### Throughput
```
Max_throughput = DMA_clock × Data_width
```

### Latency
```
Transfer_time = Transfer_count / DMA_clock
```

## Poznámky pro různé MCU

- **CH32V003**: DMA1 s 7 kanály, základní funkcionalita (2KB SRAM)
- **CH32V002**: Vylepšená varianta s 4KB SRAM, 12-bit ADC, touch sensing
- **CH32V006**: Pokročilá varianta s vyšším výkonem a více možnostmi
- **CH32X035**: Nejpokročilejší s rozšířenými DMA funkcemi

## DMA mapping

### Kanály a jejich typické použití:
1. **Kanál 1**: ADC continuous sampling
2. **Kanál 2**: SPI RX, vysokorychlostní čtení
3. **Kanál 3**: SPI TX, vysokorychlostní zápis  
4. **Kanál 4**: USART TX, sériový výstup
5. **Kanál 5**: USART RX, sériový vstup
6. **Kanál 6**: I2C TX, slow speed komunikace
7. **Kanál 7**: I2C RX, slow speed komunikace