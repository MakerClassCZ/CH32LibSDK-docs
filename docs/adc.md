# ADC - Analogově-digitální převodník

## Přehled

ADC (Analog-to-Digital Converter) umožňuje převod analogových signálů na digitální hodnoty. CH32 obsahuje 12-bitový SAR ADC s maximální frekvencí 24 MHz a podporou až 10 kanálů včetně interních referenčních napětí.

## Základní koncepty

### Dostupné ADC jednotky

CH32 mikrokontroléry mají typicky:
- **ADC1** - 12-bitový ADC (adresa 0x40012400)

### Parametry ADC

- **Rozlišení**: 12 bitů (0-4095)
- **Referenční napětí**: AVDD (typicky 3.3V)
- **Maximální frekvence**: 24 MHz (doporučeno ≤8 MHz)
- **Konverzní čas**: Sampling_time + 11 × Tadcclk
- **Kanály**: 0-7 (externí), 8 (1.2V reference), 9 (kalibrace)

### Režimy konverze

- **Jednorázová konverze**: Jeden vzorek na požádání
- **Kontinuální konverze**: Opakovaná konverze
- **Scan režim**: Postupná konverze více kanálů
- **Injection režim**: Konverze s vysokou prioritou

### Spouštění konverze

```c
// Software spouštění
#define ADC_EXT_SWSTART     7   // Software trigger

// Hardware spouštění
#define ADC_EXT_T1_TRGO     0   // Timer1 TRGO
#define ADC_EXT_T1_CC1      1   // Timer1 CC1
#define ADC_EXT_T1_CC2      2   // Timer1 CC2
#define ADC_EXT_T2_TRGO     3   // Timer2 TRGO
#define ADC_EXT_T2_CC1      4   // Timer2 CC1
#define ADC_EXT_T2_CC2      5   // Timer2 CC2
#define ADC_EXT_PD3_PC2     6   // Externí pin
```

## Základní inicializace

### GPIO nastavení

```c
// Nastavení analogových vstupů
void ADC_GPIO_Init(void) {
    // Kanály 0-7 na pinech PA0-PA7
    GPIO_Mode(PA0, GPIO_MODE_AIN);      // ADC kanál 0
    GPIO_Mode(PA1, GPIO_MODE_AIN);      // ADC kanál 1
    GPIO_Mode(PA2, GPIO_MODE_AIN);      // ADC kanál 2
    GPIO_Mode(PA3, GPIO_MODE_AIN);      // ADC kanál 3
    GPIO_Mode(PA4, GPIO_MODE_AIN);      // ADC kanál 4
    GPIO_Mode(PA5, GPIO_MODE_AIN);      // ADC kanál 5
    GPIO_Mode(PA6, GPIO_MODE_AIN);      // ADC kanál 6
    GPIO_Mode(PA7, GPIO_MODE_AIN);      // ADC kanál 7
}
```

### Jednoduchá inicializace

```c
// Inicializace pro jednoduché měření
void ADC_SimpleInit(void) {
    // GPIO nastavení
    ADC_GPIO_Init();
    
    // Inicializace ADC pro jednorázové konverze
    ADC1_InitSingle();
}
```

### Pokročilá inicializace

```c
// Kompletní ADC inicializace
void ADC_AdvancedInit(void) {
    // GPIO nastavení
    ADC_GPIO_Init();
    
    // Povolit hodiny ADC
    RCC_ADC1ClkEnable();
    
    // Reset ADC
    RCC_ADC1Reset();
    
    // Nastavení hodin ADC (max 8 MHz)
    RCC_ADCClkDiv(8);                   // PCLK/8
    
    // Základní konfigurace
    ADC1_Enable();                      // Povolit ADC
    DelayMs(1);                         // Stabilizace
    
    // Kalibrace
    ADC1_CalVol(ADC_CAL_24);            // 2/4 AVDD kalibrace
    ADC1_RstCalibStart();               // Reset kalibrace
    while (ADC1_RstCalibBusy()) {}
    
    ADC1_CalibStart();                  // Start kalibrace
    while (ADC1_CalibBusy()) {}
    
    // Nastavení triggeru
    ADC1_ExtSel(ADC_EXT_SWSTART);       // Software trigger (spouštění)
    ADC1_ExtTrigEnable();               // Povolit trigger (spouštění)
    
    // Scan režim (pro více kanálů)
    ADC1_ScanDisable();                 // Jeden kanál
    ADC1_ContEnable();                  // Kontinuální konverze
}
```

## Jednoduché měření

### Jednorázová konverze

```c
// Čtení hodnoty z kanálu
u16 ADC_ReadChannel(int channel) {
    // Předpokládá inicializaci pomocí ADC1_InitSingle()
    return ADC1_GetSingle(channel);
}

// Převod na napětí
float ADC_ToVoltage(u16 adc_value, float vref) {
    return ((float)adc_value * vref) / 4095.0f;
}

// Příklad použití
void ADC_Example_Simple(void) {
    ADC_SimpleInit();
    
    while (1) {
        u16 value = ADC_ReadChannel(0);             // Kanál 0
        float voltage = ADC_ToVoltage(value, 3.3f); // 3.3V reference
        
        printf("ADC: %d, Voltage: %.3fV\n", value, voltage);
        DelayMs(100);
    }
}
```

## Pokročilé funkce

### Sampling time

```c
// Nastavení sampling time pro kanály 0-9
// Čas = cycles × (1/ADC_clock)
#define ADC_SAMP_15     0   // 1.5 cyklů
#define ADC_SAMP_75     1   // 7.5 cyklů
#define ADC_SAMP_135    2   // 13.5 cyklů
#define ADC_SAMP_285    3   // 28.5 cyklů
#define ADC_SAMP_415    4   // 41.5 cyklů
#define ADC_SAMP_555    5   // 55.5 cyklů
#define ADC_SAMP_715    6   // 71.5 cyklů
#define ADC_SAMP_2395   7   // 239.5 cyklů

// Nastavení pro konkrétní kanál
void ADC1_SampTime(int channel, int cycles);

// Příklad
void ADC_SetSamplingTime(void) {
    ADC1_SampTime(0, ADC_SAMP_285);     // Kanál 0: 28.5 cyklů
    ADC1_SampTime(1, ADC_SAMP_135);     // Kanál 1: 13.5 cyklů
}
```

### Scan režim (více kanálů)

```c
// Inicializace scan režimu
void ADC_ScanInit(const int* channels, int count) {
    ADC_AdvancedInit();
    
    // Povolit scan režim
    ADC1_ScanEnable();
    ADC1_ContDisable();                 // Jednorázová sekvence
    
    // Nastavit počet kanálů
    ADC1_RNum(count);
    
    // Nastavit sekvenci kanálů
    for (int i = 0; i < count; i++) {
        ADC1_RSeq(i + 1, channels[i]);   // Sekvence 1-16
    }
}

// Příklad scan režimu
void ADC_Example_Scan(void) {
    int channels[] = {0, 1, 2, 3};      // Kanály 0-3
    u16 results[4];
    
    ADC_ScanInit(channels, 4);
    
    while (1) {
        // Spustit konverzi sekvence
        ADC1_SwStart();
        
        // Čekat na dokončení každé konverze
        for (int i = 0; i < 4; i++) {
            while (!ADC1_EndGet()) {}   // Čekat na EOC
            results[i] = ADC1_GetData();
            ADC1_EndClr();              // Vymazat flag
        }
        
        // Zobrazit výsledky
        for (int i = 0; i < 4; i++) {
            float voltage = ADC_ToVoltage(results[i], 3.3f);
            printf("CH%d: %.3fV ", i, voltage);
        }
        printf("\n");
        
        DelayMs(1000);
    }
}
```

### Kontinuální konverze

```c
// Kontinuální konverze jednoho kanálu
void ADC_ContinuousInit(int channel) {
    ADC_AdvancedInit();
    
    ADC1_ScanDisable();                 // Jeden kanál
    ADC1_ContEnable();                  // Kontinuální režim
    ADC1_RNum(1);                       // Jeden kanál
    ADC1_RSeq(1, channel);              // Nastavit kanál
    
    // Spustit kontinuální konverzi
    ADC1_SwStart();
}

// Čtení z kontinuální konverze
u16 ADC_ReadContinuous(void) {
    while (!ADC1_EndGet()) {}           // Čekat na nová data
    u16 value = ADC1_GetData();
    ADC1_EndClr();
    return value;
}
```

## Přerušení

### Povolení přerušení

```c
// ADC přerušení
void ADC1_IntEnable(void);              // End of conversion
void ADC1_AWDIntEnable(void);           // Analog watchdog
void ADC1_JIntEnable(void);             // Injection end

void ADC1_IntDisable(void);
void ADC1_AWDIntDisable(void);
void ADC1_JIntDisable(void);
```

### Obsluha přerušení

```c
volatile u16 adc_buffer[ADC_BUFFER_SIZE];
volatile int adc_index = 0;

// ADC přerušovací obsluha
void ADC1_Handler(void) {
    if (ADC1_EndGet()) {
        // Konverze dokončena
        u16 value = ADC1_GetData();
        
        if (adc_index < ADC_BUFFER_SIZE) {
            adc_buffer[adc_index++] = value;
        }
        
        ADC1_EndClr();
    }
    
    if (ADC1_AWDGet()) {
        // Analog watchdog událost
        printf("ADC Watchdog alert!\n");
        ADC1_AWDClr();
    }
}
```

## Analog Watchdog

### Nastavení watchdog

```c
// Nastavení analog watchdog
void ADC_WatchdogInit(int channel, u16 low, u16 high) {
    // Nastavit prahové hodnoty
    ADC1_WDLThr(low);                   // Dolní práh
    ADC1_WDHThr(high);                  // Horní práh
    
    // Nastavit kanál
    ADC1_AWDChan(channel);              // Monitorovaný kanál
    ADC1_AWDSingle();                   // Jeden kanál
    
    // Povolit watchdog
    ADC1_AWDEnable();                   // Pro regular kanály
    ADC1_JAWDEnable();                  // Pro injection kanály
    
    // Povolit přerušení
    ADC1_AWDIntEnable();
}

// Příklad watchdog pro monitoring napětí
void ADC_Example_Watchdog(void) {
    ADC_AdvancedInit();
    
    // Watchdog pro kanál 0: 1V - 2V
    u16 low_1v = (u16)(1.0f * 4095 / 3.3f);    // 1V
    u16 high_2v = (u16)(2.0f * 4095 / 3.3f);   // 2V
    
    ADC_WatchdogInit(0, low_1v, high_2v);
    
    // Spustit kontinuální konverzi
    ADC1_RSeq(1, 0);
    ADC1_ContEnable();
    ADC1_SwStart();
}
```

## DMA podpora

```c
// DMA pro ADC
void ADC1_DMAEnable(void);              // Povolit DMA požadavky
void ADC1_DMADisable(void);             // Zakázat DMA

// DMA inicializace pro ADC
void ADC_DMA_Init(u16* buffer, int size) {
    // DMA nastavení (specifické pro kanál DMA)
    // ... DMA konfigurace ...
    
    ADC1_DMAEnable();                   // Povolit ADC DMA
}
```

## Injection kanály

### Injection konverze

```c
// Nastavení injection sekvence
void ADC_InjectionInit(const int* channels, int count) {
    // Nastavit počet injection kanálů
    ADC1_JNum(count);
    
    // Nastavit sekvenci (1-4 kanály)
    for (int i = 0; i < count; i++) {
        ADC1_JSeq(i + 1, channels[i]);
    }
    
    // Nastavit trigger
    ADC1_JExtSel(ADC_JEXT_JSWSTART);    // Software trigger
    ADC1_JExtTrigEnable();
    
    // Auto injection po regular konverzi
    ADC1_JAutoEnable();
}

// Čtení injection výsledků
void ADC_ReadInjection(u16* results, int count) {
    for (int i = 0; i < count; i++) {
        results[i] = ADC1_GetJData(i + 1);  // Injection data 1-4
    }
}
```

## Praktické příklady

### Měření teploty pomocí termistoru

```c
// NTC termistor (10kΩ při 25°C)
float ADC_ReadTemperature(int channel) {
    u16 adc_value = ADC1_GetSingle(channel);
    
    // Převod na odpor (voltage divider s 10kΩ)
    float voltage = ADC_ToVoltage(adc_value, 3.3f);
    float resistance = (voltage * 10000.0f) / (3.3f - voltage);
    
    // Steinhart-Hart rovnice (zjednodušená)
    float ln_r = logf(resistance / 10000.0f);
    float temp_k = 1.0f / (0.001129148f + 0.000234125f * ln_r + 
                          0.0000000876741f * ln_r * ln_r * ln_r);
    
    return temp_k - 273.15f;  // Celsius
}

// Použití
void Temperature_Monitor(void) {
    ADC_SimpleInit();
    
    while (1) {
        float temperature = ADC_ReadTemperature(0);
        printf("Temperature: %.1f°C\n", temperature);
        DelayMs(1000);
    }
}
```

### Baterie monitoring s watchdog

```c
// Monitoring napětí baterie
void Battery_Monitor_Init(void) {
    ADC_AdvancedInit();
    
    // Watchdog pro nízké napětí (2.8V pro 3V baterii)
    u16 low_voltage = (u16)(2.8f * 4095 / 3.3f);
    ADC_WatchdogInit(0, low_voltage, 4095);
    
    // Kontinuální monitoring
    ADC1_RSeq(1, 0);
    ADC1_ContEnable();
    ADC1_SwStart();
}

float Battery_GetVoltage(void) {
    u16 adc_value = ADC_ReadContinuous();
    return ADC_ToVoltage(adc_value, 3.3f);
}

// Obsluha nízké baterie
void ADC1_Handler(void) {
    if (ADC1_AWDGet()) {
        printf("WARNING: Low battery!\n");
        // Zavřít neesenciální systémy
        ADC1_AWDClr();
    }
}
```

### Audio spektrální analyzér

```c
#define FFT_SIZE 256
u16 audio_buffer[FFT_SIZE];
volatile int sample_index = 0;

// Rychlé vzorkování audia
void Audio_ADC_Init(void) {
    ADC_AdvancedInit();
    
    // Rychlé sampling time
    ADC1_SampTime(0, ADC_SAMP_15);      // Nejrychlejší
    
    // Timer trigger pro vzorkování (např. 8kHz)
    ADC1_ExtSel(ADC_EXT_T1_CC1);        // Timer1 CC1
    ADC1_ExtTrigEnable();
    
    // Přerušení pro každý vzorek
    ADC1_IntEnable();
    
    ADC1_RSeq(1, 0);                    // Audio vstup
}

void ADC1_Handler(void) {
    if (ADC1_EndGet()) {
        u16 sample = ADC1_GetData();
        
        if (sample_index < FFT_SIZE) {
            audio_buffer[sample_index++] = sample;
        } else {
            // Buffer plný - spustit FFT analýzu
            ProcessAudioFFT(audio_buffer, FFT_SIZE);
            sample_index = 0;
        }
        
        ADC1_EndClr();
    }
}
```

### Multi-kanálové datalogging

```c
#define SENSOR_COUNT 8
typedef struct {
    u16 values[SENSOR_COUNT];
    u32 timestamp;
} SensorReading_t;

SensorReading_t readings[100];
int reading_index = 0;

// Periodické čtení všech senzorů
void Datalogger_Init(void) {
    ADC_AdvancedInit();
    
    // Nastavit sampling times
    for (int i = 0; i < SENSOR_COUNT; i++) {
        ADC1_SampTime(i, ADC_SAMP_285);  // Přesné měření
    }
    
    // Timer pro periodické spouštění (1Hz)
    TIM2_Init_1Hz();
    TIM2_IntEnable();
}

void TIM2_Handler(void) {
    // Čtení všech kanálů
    SensorReading_t* reading = &readings[reading_index];
    reading->timestamp = Time();
    
    for (int i = 0; i < SENSOR_COUNT; i++) {
        reading->values[i] = ADC1_GetSingle(i);
    }
    
    reading_index = (reading_index + 1) % 100;
    
    // Log data
    LogReading(reading);
}

void LogReading(const SensorReading_t* reading) {
    printf("%lu:", reading->timestamp);
    for (int i = 0; i < SENSOR_COUNT; i++) {
        float voltage = ADC_ToVoltage(reading->values[i], 3.3f);
        printf(" %.3f", voltage);
    }
    printf("\n");
}
```

### Precision reference měření

```c
// Použití interních referenčních napětí
float ADC_GetPreciseVoltage(int channel) {
    // Změřit externí kanál
    u16 ext_value = ADC1_GetSingle(channel);
    
    // Změřit interní 1.2V referenci
    u16 ref_value = ADC1_GetSingle(8);  // Kanál 8 = 1.2V
    
    // Vypočítat skutečné AVDD
    float actual_vdd = (1.2f * 4095.0f) / (float)ref_value;
    
    // Přesný výpočet napětí
    return ((float)ext_value * actual_vdd) / 4095.0f;
}

// Kalibrace pomocí známého napětí
void ADC_Calibrate(void) {
    // Změřit kalibrační napětí (3/4 AVDD)
    ADC1_CalVol(ADC_CAL_34);            // 3/4 AVDD
    u16 cal_value = ADC1_GetSingle(9);   // Kanál 9 = kalibrace
    
    float expected = 3.3f * 0.75f;      // Očekávané napětí
    float measured = ADC_ToVoltage(cal_value, 3.3f);
    
    float correction = expected / measured;
    printf("ADC correction factor: %.4f\n", correction);
}
```

## Tipy a triky

1. **Stabilizace**: Počkejte alespoň 1ms po zapnutí ADC před kalibrací
2. **Kalibrace**: Vždy proveďte kalibraci po resetu nebo změně napájení
3. **Sampling time**: Delší čas = přesnější výsledky pro vysoké impedance
4. **Filtrování**: Použijte software filtrování pro potlačení šumu
5. **Reference**: Použijte interní 1.2V referenci pro přesná měření

## Výpočty

### Konverzní čas
```
Tconv = Sampling_time + 11 × (1/ADC_clock)
```

### Rozlišení napětí
```
Resolution = VREF / 4095
```

### Maximální sampling rate
```
Max_rate = ADC_clock / (Sampling_time + 11)
```

## Poznámky pro různé MCU

- **CH32V003**: 10-bitový ADC, 8+2 kanálů, DMA podpora (2KB SRAM)
- **CH32V002**: 12-bitový ADC s vylepšeními, 4KB SRAM, touch sensing
- **CH32V006**: Pokročilá varianta s vyšší rychlostí a více kanály
- **CH32X035**: Nejpokročilejší s rozšířenými ADC funkcemi

## Mapování pinů

### ADC1 kanály:
- **CH0**: PA0
- **CH1**: PA1  
- **CH2**: PA2
- **CH3**: PA3
- **CH4**: PA4
- **CH5**: PA5
- **CH6**: PA6
- **CH7**: PA7
- **CH8**: Interní 1.2V reference
- **CH9**: Kalibrační napětí