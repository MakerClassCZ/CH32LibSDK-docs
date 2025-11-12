# TIMER - Časovače a PWM generátory

## Přehled

Timer moduly poskytují širokou škálu funkcí pro generování přesných časových intervalů, PWM signálů, čítání událostí, enkodérové vstupy a capture/compare operace (zachycování/porovnávání). CH32 mikrokontroléry obsahují pokročilé 16-bitové timery s až 4 kanály.

## Základní koncepty

### Dostupné timery

CH32 mikrokontroléry mají typicky tyto timery:
- **TIM1** - Pokročilý timer (adresa 0x40012C00) s brake a dead-time funkcemi
- **TIM2** - Obecný timer (adresa 0x40000000) se 4 kanály

### Základní architektura

```
Zdroj hodin → Předděl → Čítač → Auto-reload → Overflow/Update
                ↓
              4x kanály (Capture/Compare/PWM)
```

### Režimy zarovnání

```c
#define TIM_ALIGN_EDGE   0  // Hraniční zarovnání (počítá nahoru/dolů)
#define TIM_ALIGN_DOWN   1  // Centrální zarovnání 1 (flag při počítání dolů)
#define TIM_ALIGN_UP     2  // Centrální zarovnání 2 (flag při počítání nahoru)
#define TIM_ALIGN_BOTH   3  // Centrální zarovnání 3 (flag v obou směrech)
```

### Vstupní režimy

```c
#define TIM_INMODE_INT     0  // Interní hodiny
#define TIM_INMODE_ENC1    1  // Enkodér režim 1
#define TIM_INMODE_ENC2    2  // Enkodér režim 2  
#define TIM_INMODE_ENC3    3  // Enkodér režim 3
#define TIM_INMODE_RESET   4  // Reset režim
#define TIM_INMODE_GATE    5  // Hradlový režim
#define TIM_INMODE_TRIG    6  // Spouštěcí režim
#define TIM_INMODE_EDGE    7  // Hranový režim
```

## Základní ovládání

### Zapnutí/vypnutí timeru

```c
void TIM1_Enable(void);        // Spustit timer
void TIM1_Disable(void);       // Zastavit timer
void TIM2_Enable(void);        
void TIM2_Disable(void);
```

### Nastavení směru počítání

```c
void TIM1_DirUp(void);         // Počítat nahoru (default)
void TIM1_DirDown(void);       // Počítat dolů
void TIM2_DirUp(void);
void TIM2_DirDown(void);
```

### Režim běhu

```c
void TIM1_RepeatMode(void);    // Kontinuální běh (default)
void TIM1_PulseMode(void);     // Jednorázový běh
void TIM2_RepeatMode(void);
void TIM2_PulseMode(void);
```

## Nastavení základních parametrů

### Hodnoty čítače a auto-reload

```c
// Nastavení registrů
void TIM1_Presc(int prescaler);    // Předděl (0-65535)
void TIM1_Load(int value);         // Auto-reload hodnota
void TIM1_Cnt(int value);          // Aktuální hodnota čítače
void TIM1_Repeat(int count);       // Počet opakování

// Čtení registrů
int TIM1_GetPresc(void);           // Aktuální předděl
int TIM1_GetLoad(void);            // Auto-reload hodnota
int TIM1_GetCnt(void);             // Aktuální stav čítače
int TIM1_GetRepeat(void);          // Zbývající opakování
```

### Auto-reload a update události

```c
void TIM1_AutoReloadEnable(void);   // Povolit auto-reload (default)
void TIM1_AutoReloadDisable(void);  // Zakázat auto-reload

void TIM1_UEVEnable(void);          // Povolit update události (default)
void TIM1_UEVDisable(void);         // Zakázat update události
```

## PWM generování

### Základní PWM nastavení

```c
// Inicializace PWM na kanálu 1
void PWM_Init_Example(void) {
    // GPIO nastavení
    GPIO_Mode(PA8, GPIO_MODE_AF);       // TIM1_CH1
    
    // Timer základní nastavení
    TIM1_Presc(47);                     // 48MHz/48 = 1MHz
    TIM1_Load(999);                     // 1MHz/1000 = 1kHz PWM
    TIM1_AutoReloadEnable();
    
    // Kanál 1 jako PWM výstup
    TIM1_CC1Sel(TIM_CCSEL_OUT);         // Výstupní režim
    TIM1_OC1Mode(TIM_COMP_PWM1);        // PWM režim 1
    TIM1_OC1PreEnable();                // Povolit preload
    TIM1_CC1Enable();                   // Povolit kanál
    
    // Nastavení duty cycle (50%)
    TIM1_Comp1(500);                    // 50% duty cycle
    
    // Spustit timer
    TIM1_Update();                      // Vygenerovat update
    TIM1_Enable();                      // Spustit
}
```

### PWM režimy

```c
// Compare/PWM režimy
#define TIM_COMP_FREEZE      0  // Zmrazit výstup
#define TIM_COMP_VALIDEQ     1  // High při shodě
#define TIM_COMP_INVALIDEQ   2  // Low při shodě
#define TIM_COMP_FLIP        3  // Invertovat při shodě
#define TIM_COMP_INVALID     4  // Vždy Low
#define TIM_COMP_VALID       5  // Vždy High
#define TIM_COMP_PWM1        6  // PWM režim 1
#define TIM_COMP_PWM2        7  // PWM režim 2

// Nastavení PWM režimu pro kanály
void TIM1_OC1Mode(int mode);    // Kanál 1
void TIM1_OC2Mode(int mode);    // Kanál 2
void TIM1_OC3Mode(int mode);    // Kanál 3
void TIM1_OC4Mode(int mode);    // Kanál 4
```

### Duty cycle a compare hodnoty

```c
// Nastavení compare hodnot (určuje duty cycle)
void TIM1_Comp1(int value);     // Kanál 1 compare
void TIM1_Comp2(int value);     // Kanál 2 compare
void TIM1_Comp3(int value);     // Kanál 3 compare
void TIM1_Comp4(int value);     // Kanál 4 compare

// Čtení compare hodnot
int TIM1_GetComp1(void);
int TIM1_GetComp2(void);
int TIM1_GetComp3(void);
int TIM1_GetComp4(void);
```

## Input Capture

### Capture nastavení

```c
// Capture na kanálu 1
void Capture_Init_Example(void) {
    // GPIO nastavení
    GPIO_Mode(PA8, GPIO_MODE_IN);       // TIM1_CH1 vstup
    
    // Timer základní nastavení  
    TIM1_Presc(47);                     // 1MHz časová základna
    TIM1_Load(65535);                   // Maximum rozsah
    
    // Kanál 1 jako capture vstup (zachycování)
    TIM1_CC1Sel(TIM_CCSEL_TI1);         // TI1 vstup
    TIM1_IC1Filter(TIM_FILTER_0);       // Bez filtrace
    TIM1_IC1Rise();                     // Capture na vzestupné hraně
    TIM1_CC1Enable();                   // Povolit kanál
    
    // Povolit přerušení
    TIM1_CC1IntEnable();
    
    TIM1_Enable();
}
```

### Filtrace vstupů

```c
// Nastavení filtrů pro input capture (vstupní zachycování)
#define TIM_FILTER_0      0   // Bez filtru
#define TIM_FILTER_1_2    1   // CK_INT, N=2
#define TIM_FILTER_1_4    2   // CK_INT, N=4
#define TIM_FILTER_1_8    3   // CK_INT, N=8
// ... až do TIM_FILTER_32_8

void TIM1_IC1Filter(int filter);   // Filtr kanálu 1
void TIM1_IC2Filter(int filter);   // Filtr kanálu 2
void TIM1_IC3Filter(int filter);   // Filtr kanálu 3
void TIM1_IC4Filter(int filter);   // Filtr kanálu 4
```

### Polarita capture (zachycování)

```c
// Nastavení polarity capture (na kterou hranu reagovat)
void TIM1_IC1Rise(void);        // Vzestupná hrana
void TIM1_IC1Fall(void);        // Sestupná hrana
void TIM1_IC1Both(void);        // Obě hrany

void TIM1_IC2Rise(void);
void TIM1_IC2Fall(void);
void TIM1_IC2Both(void);
```

## Enkodér

### Enkodérový režim

```c
// Inicializace enkodéru
void Encoder_Init_Example(void) {
    // GPIO nastavení
    GPIO_Mode(PA8, GPIO_MODE_IN_PU);    // TIM1_CH1 (A)
    GPIO_Mode(PA9, GPIO_MODE_IN_PU);    // TIM1_CH2 (B)
    
    // Enkodérový režim
    TIM1_InMode(TIM_INMODE_ENC3);       // Enkodér režim 3 (obě hrany)
    TIM1_CC1Sel(TIM_CCSEL_TI1);         // CH1 = TI1
    TIM1_CC2Sel(TIM_CCSEL_TI2);         // CH2 = TI2
    
    // Filtrace
    TIM1_IC1Filter(TIM_FILTER_1_4);
    TIM1_IC2Filter(TIM_FILTER_1_4);
    
    // Nastavení rozsahu
    TIM1_Load(65535);                   // Plný 16-bit rozsah
    
    TIM1_Enable();
}

// Čtení enkodéru
int Encoder_GetPosition(void) {
    return TIM1_GetCnt();
}

void Encoder_Reset(void) {
    TIM1_Cnt(0);
}
```

## Měření frekvence a periody

### PWM Input režim

```c
// Měření duty cycle a frekvence pomocí PWM Input
void PWM_Input_Init_Example(void) {
    // GPIO nastavení
    GPIO_Mode(PA8, GPIO_MODE_IN);       // TIM1_CH1
    
    // PWM Input nastavení
    TIM1_CC1Sel(TIM_CCSEL_TI1);         // IC1 = TI1
    TIM1_IC1Rise();                     // IC1 vzestupná hrana
    
    TIM1_CC2Sel(TIM_CCSEL_TI1);         // IC2 = TI1 (stejný vstup)
    TIM1_IC2Fall();                     // IC2 sestupná hrana
    
    // Trigger a reset
    TIM1_Trig(TIM_TRIG_TI1FP1);         // Trigger z TI1FP1
    TIM1_InMode(TIM_INMODE_RESET);      // Reset při triggeru
    
    TIM1_CC1Enable();
    TIM1_CC2Enable();
    TIM1_Enable();
}

// Čtení výsledků PWM Input
void PWM_Input_Read(uint16_t* period, uint16_t* duty) {
    *period = TIM1_GetComp1();          // Celková perioda
    *duty = TIM1_GetComp2();            // Doba high stavu
}
```

## Přerušení

### Povolení přerušení

```c
// Update přerušení
void TIM1_IntEnable(void);          // Povolit update přerušení
void TIM1_IntDisable(void);         // Zakázat update přerušení

// Compare/Capture přerušení
void TIM1_CC1IntEnable(void);       // Kanál 1
void TIM1_CC2IntEnable(void);       // Kanál 2
void TIM1_CC3IntEnable(void);       // Kanál 3
void TIM1_CC4IntEnable(void);       // Kanál 4

// Trigger přerušení
void TIM1_TrigIntEnable(void);      // Trigger událost
void TIM1_BreakIntEnable(void);     // Break událost (pouze TIM1)
```

### Kontrola flagů

```c
// Kontrola přerušovacích flagů
Bool TIM1_UpdateFlag(void);         // Update flag
Bool TIM1_CC1Flag(void);            // Compare/Capture 1
Bool TIM1_CC2Flag(void);            // Compare/Capture 2
Bool TIM1_CC3Flag(void);            // Compare/Capture 3
Bool TIM1_CC4Flag(void);            // Compare/Capture 4
Bool TIM1_TrigFlag(void);           // Trigger flag
Bool TIM1_BreakFlag(void);          // Break flag
```

### Vymazání flagů

```c
// Vymazání flagů
void TIM1_UpdateFlagClr(void);      // Vymazat update flag
void TIM1_CC1FlagClr(void);         // Vymazat CC1 flag
void TIM1_CC2FlagClr(void);         // Vymazat CC2 flag
void TIM1_CC3FlagClr(void);         // Vymazat CC3 flag
void TIM1_CC4FlagClr(void);         // Vymazat CC4 flag
```

## Pokročilé funkce (TIM1)

### Break a Dead-time

```c
// Break funkce (emergency stop)
void TIM1_BreakEnable(void);        // Povolit break
void TIM1_BreakDisable(void);       // Zakázat break
void TIM1_BreakHigh(void);          // Break aktivní high
void TIM1_BreakLow(void);           // Break aktivní low

// Dead-time pro komplementární výstupy
void TIM1_DeadTime(int dt);         // Nastavit dead-time

// Lock konfigurace
void TIM1_Lock(int level);          // Lock level 0-3
```

### Komplementární výstupy

```c
// Povolení komplementárních výstupů (TIM1_CH1N, etc.)
void TIM1_CC1NEnable(void);         // Komplementární výstup 1
void TIM1_CC2NEnable(void);         // Komplementární výstup 2
void TIM1_CC3NEnable(void);         // Komplementární výstup 3

// Main Output Enable
void TIM1_MOEEnable(void);          // Povolit hlavní výstupy
void TIM1_MOEDisable(void);         // Zakázat hlavní výstupy
```

## Praktické příklady

### VGA Timing generování

```c
// Reálný příklad z BabyPad VGA driveru - HSYNC generování
void VGA_Timer_Init(void) {
    // Timer 2 remapování na PD4 (kanál 1)
    GPIO_Remap_TIM2(0);
    
    // PWM pro HSYNC signál:
    // Systémová frekvence 50 MHz
    // Celková perioda 31.77756 μs = 1600 hodinových cyklů
    // Sync pulz 3.81331 μs = 192 hodinových cyklů
    // Začátek obrazu na 5.71996 μs = 288 cyklů
    TIM2_InitPWM(1,             // Kanál 1
                 1,             // Dělič 1 (50 MHz)
                 1600-1,        // Reload (perioda)
                 192,           // Compare (šířka sync pulzu)
                 False);        // LOW->HIGH polarita
    
    // Timer 2 kanál 2 pro interrupt při začátku obrazu
    TIM2_Comp2(288 - 60);       // Compare hodnota pro start
    TIM2_OC2Mode(TIM_COMP_FREEZE); // Freeze režim pro interrupt
    TIM2_CC2IntEnable();        // Povolit přerušení CC2
    
    TIM2_Enable();              // Spustit timer
}

// Obsluha přerušení pro pixel output
void TIM2_Handler(void) {
    if (TIM2_CC2Flag()) {
        // Začátek aktivní oblasti řádku - pošli pixely přes SPI
        SendPixelLine();
        TIM2_CC2FlagClr();
    }
}
```

### Servo ovládání

```c
// Servo PWM (50Hz, 1-2ms pulz)
void Servo_Init(void) {
    // GPIO nastavení
    GPIO_Mode(PA8, GPIO_MODE_AF);       // TIM1_CH1
    
    // 50Hz PWM (20ms perioda)
    TIM1_Presc(47);                     // 48MHz/48 = 1MHz
    TIM1_Load(19999);                   // 1MHz/20000 = 50Hz
    
    // PWM nastavení
    TIM1_CC1Sel(TIM_CCSEL_OUT);
    TIM1_OC1Mode(TIM_COMP_PWM1);
    TIM1_OC1PreEnable();
    TIM1_CC1Enable();
    TIM1_AutoReloadEnable();
    
    // Střední pozice (1.5ms)
    TIM1_Comp1(1500);                   // 1.5ms pulz
    
    TIM1_Update();
    TIM1_Enable();
}

// Nastavení pozice serva (-90° až +90°)
void Servo_SetAngle(int angle) {
    // Omezit rozsah
    if (angle < -90) angle = -90;
    if (angle > 90) angle = 90;
    
    // Převod na pulzní šířku (1000-2000 μs)
    int pulse_width = 1500 + (angle * 500) / 90;
    TIM1_Comp1(pulse_width);
}
```

### Měření frekvence

```c
volatile uint32_t frequency = 0;
volatile uint16_t last_capture = 0;

// Inicializace měření frekvence
void Frequency_Init(void) {
    GPIO_Mode(PA8, GPIO_MODE_IN);
    
    TIM1_Presc(47);                     // 1MHz časová základna
    TIM1_Load(65535);
    
    TIM1_CC1Sel(TIM_CCSEL_TI1);
    TIM1_IC1Filter(TIM_FILTER_1_4);
    TIM1_IC1Rise();
    TIM1_CC1Enable();
    TIM1_CC1IntEnable();
    
    TIM1_Enable();
}

// Obsluha přerušení
void TIM1_CC_Handler(void) {
    if (TIM1_CC1Flag()) {
        uint16_t current_capture = TIM1_GetComp1();
        uint16_t period = current_capture - last_capture;
        last_capture = current_capture;
        
        // Frekvence = 1MHz / perioda
        if (period > 0) {
            frequency = 1000000UL / period;
        }
        
        TIM1_CC1FlagClr();
    }
}
```

### RGB LED s PWM

```c
// RGB LED ovládání
void RGB_Init(void) {
    // GPIO nastavení
    GPIO_Mode(PA8, GPIO_MODE_AF);       // Červená - TIM1_CH1
    GPIO_Mode(PA9, GPIO_MODE_AF);       // Zelená - TIM1_CH2  
    GPIO_Mode(PA10, GPIO_MODE_AF);      // Modrá - TIM1_CH3
    
    // PWM nastavení (1kHz)
    TIM1_Presc(47);                     // 1MHz
    TIM1_Load(999);                     // 1kHz PWM
    
    // Všechny 3 kanály jako PWM
    for (int ch = 1; ch <= 3; ch++) {
        switch(ch) {
            case 1:
                TIM1_CC1Sel(TIM_CCSEL_OUT);
                TIM1_OC1Mode(TIM_COMP_PWM1);
                TIM1_OC1PreEnable();
                TIM1_CC1Enable();
                TIM1_Comp1(0);          // Začít vypnuto
                break;
            case 2:
                TIM1_CC2Sel(TIM_CCSEL_OUT);
                TIM1_OC2Mode(TIM_COMP_PWM1);
                TIM1_OC2PreEnable();
                TIM1_CC2Enable();
                TIM1_Comp2(0);
                break;
            case 3:
                TIM1_CC3Sel(TIM_CCSEL_OUT);
                TIM1_OC3Mode(TIM_COMP_PWM1);
                TIM1_OC3PreEnable();
                TIM1_CC3Enable();
                TIM1_Comp3(0);
                break;
        }
    }
    
    TIM1_AutoReloadEnable();
    TIM1_Update();
    TIM1_Enable();
}

// Nastavení RGB hodnot (0-255)
void RGB_SetColor(uint8_t red, uint8_t green, uint8_t blue) {
    // Převod 8-bit na PWM hodnoty (0-999)
    TIM1_Comp1((red * 999) / 255);     // Červená
    TIM1_Comp2((green * 999) / 255);   // Zelená
    TIM1_Comp3((blue * 999) / 255);    // Modrá
}
```

### Stepperový motor

```c
// 4-fázový stepper motor
void Stepper_Init(void) {
    // GPIO nastavení pro 4 fáze
    GPIO_Mode(PA8, GPIO_MODE_AF);       // Fáze A
    GPIO_Mode(PA9, GPIO_MODE_AF);       // Fáze B
    GPIO_Mode(PA10, GPIO_MODE_AF);      // Fáze C
    GPIO_Mode(PA11, GPIO_MODE_AF);      // Fáze D
    
    // Timer pro krokování
    TIM2_Presc(4799);                   // 48MHz/4800 = 10kHz
    TIM2_Load(99);                      // 10kHz/100 = 100Hz (10ms kroky)
    TIM2_AutoReloadEnable();
    TIM2_IntEnable();
    TIM2_Enable();
}

volatile int stepper_phase = 0;
volatile int stepper_steps = 0;
volatile int stepper_target = 0;

// Tabulka fází pro plný krok
const uint8_t stepper_phases[4] = {
    0b0001,  // Fáze A
    0b0010,  // Fáze B  
    0b0100,  // Fáze C
    0b1000   // Fáze D
};

// Obsluha přerušení pro krokování
void TIM2_Handler(void) {
    if (TIM2_UpdateFlag()) {
        if (stepper_steps != stepper_target) {
            // Nastavit fáze podle tabulky
            uint8_t phase = stepper_phases[stepper_phase];
            
            GPIO_Out(PA8, (phase & 0x01) ? 1 : 0);
            GPIO_Out(PA9, (phase & 0x02) ? 1 : 0);
            GPIO_Out(PA10, (phase & 0x04) ? 1 : 0);
            GPIO_Out(PA11, (phase & 0x08) ? 1 : 0);
            
            // Posun fáze a kroky
            if (stepper_steps < stepper_target) {
                stepper_phase = (stepper_phase + 1) % 4;
                stepper_steps++;
            } else {
                stepper_phase = (stepper_phase - 1 + 4) % 4;
                stepper_steps--;
            }
        }
        
        TIM2_UpdateFlagClr();
    }
}

// Pohyb na target pozici
void Stepper_MoveTo(int target) {
    stepper_target = target;
}
```

### Herní timing loops - 60 FPS synchronizace

```c
// Reálný příklad z her - VGA frame synchronizace pro 60 FPS hry
volatile uint32_t frame_counter = 0;
volatile Bool frame_ready = False;

// Timer pro 60 FPS herní loop (16.67ms perioda)
void Game_Timer_Init(void) {
    // Timer pro 60 FPS (16667 μs perioda)
    TIM1_Presc(47);                     // 48MHz/48 = 1MHz
    TIM1_Load(16666);                   // 1MHz/16667 = ~60Hz
    TIM1_AutoReloadEnable();
    TIM1_IntEnable();
    TIM1_Enable();
}

// Obsluha přerušení pro herní timing
void TIM1_Handler(void) {
    if (TIM1_UpdateFlag()) {
        frame_counter++;
        frame_ready = True;
        TIM1_UpdateFlagClr();
    }
}

// Hlavní herní smyčka s fixním časováním
void Game_Main_Loop(void) {
    Game_Timer_Init();
    
    while (1) {
        // Čekání na další frame
        while (!frame_ready) {
            // Můžeme zde dělat jiné operace nebo uspat CPU
        }
        frame_ready = False;
        
        // Herní logika - přesně 60x za sekundu
        Game_Update_Logic();
        Game_Render_Graphics();
        Game_Process_Audio();
    }
}

// VGA frame-based synchronizace (reálný příklad z RVPC her)
void VGA_Game_Loop(void) {
    while (1) {
        // Čekání na konec VGA frame
        if (vga_is_frame_end()) {
            // Herní logika synchronizovaná s VGA
            run_app_state_machine();
            update_game_entities();
        }
        
        // Vstupní handling může běžet častěji
        run_keyboard_state_machine();
    }
}
```

### Herní timing s různými rychlostmi

```c
// Multi-speed timing pro různé herní elementy
volatile uint8_t fast_counter = 0;    // 60 FPS
volatile uint8_t medium_counter = 0;  // 30 FPS  
volatile uint8_t slow_counter = 0;    // 15 FPS

void Game_Multi_Speed_Loop(void) {
    while (1) {
        if (frame_ready) {
            frame_ready = False;
            fast_counter++;
            
            // Rychlé updaty (60 FPS) - hráč, vstup, fyzika
            Update_Player_Movement();
            Update_Physics();
            
            // Střední rychlost (30 FPS) - nepřátelé, projektily
            if (fast_counter % 2 == 0) {
                medium_counter++;
                Update_Enemies();
                Update_Bullets();
            }
            
            // Pomalé updaty (15 FPS) - animace, AI, efekty
            if (fast_counter % 4 == 0) {
                slow_counter++;
                Update_Animations();
                Update_AI();
                Update_Particle_Effects();
            }
            
            // Rendering vždy na konci
            Render_All();
        }
    }
}
```

### Timing pro Tetris-style hry

```c
// Reálný příklad z Tetris - postupné zrychlování hry
volatile uint16_t drop_timer = 0;
volatile uint16_t drop_speed = 1000;  // ms mezi pádem dílku

// Timing pro Tetris
void Tetris_Timer_Init(void) {
    // Timer každý 1ms pro herní timing
    TIM2_Presc(47);                     // 1MHz
    TIM2_Load(999);                     // 1kHz = 1ms
    TIM2_AutoReloadEnable();
    TIM2_IntEnable();
    TIM2_Enable();
}

void TIM2_Handler(void) {
    if (TIM2_UpdateFlag()) {
        drop_timer++;
        
        // Automatický pád dílku
        if (drop_timer >= drop_speed) {
            drop_timer = 0;
            Drop_Current_Piece();
        }
        
        TIM2_UpdateFlagClr();
    }
}

// Zrychlování hry podle levelu
void Update_Game_Speed(uint8_t level) {
    // Exponenciální zrychlování
    drop_speed = 1000 - (level * 80);
    if (drop_speed < 100) drop_speed = 100;  // Minimální rychlost
}

// Herní smyčka s WaitMs delays pro animace
void Tetris_Animation_Example(void) {
    // Animace mizení řádků (reálný kód z her)
    for (int y = full_row; y < 20; y++) {
        for (int j = 0; j < 10; j++) {
            Clear_Tile(8+j, y);
        }
        WaitMs(50);                     // Krátký delay pro vizuální efekt
    }
}
```

### Audio syntéza pomocí PWM

```c
// Reálné příklady z her - audio syntéza přes timer PWM
// Timer setup pro audio výstup (obvykle PA1 - TIM2_CH2)

// Audio frekvence a notace (Hz)
#define NOTE_C4     262
#define NOTE_D4     294
#define NOTE_E4     330
#define NOTE_F4     349
#define NOTE_G4     392
#define NOTE_A4     440
#define NOTE_B4     494
#define NOTE_C5     523
#define NOTE_PAUSE  0

// Audio timer inicializace
void Audio_Init(void) {
    // GPIO pro audio výstup
    GPIO_Mode(PA1, GPIO_MODE_AF);       // TIM2_CH2 audio out
    
    // Základní timer nastavení
    TIM2_Presc(47);                     // 48MHz/48 = 1MHz
    TIM2_AutoReloadEnable();
    
    // Kanál 2 jako PWM pro audio
    TIM2_CC2Sel(TIM_CCSEL_OUT);
    TIM2_OC2Mode(TIM_COMP_PWM1);
    TIM2_OC2PreEnable();
    TIM2_CC2Enable();
    
    TIM2_Enable();
}

// Přehrání tónu o dané frekvenci (reálný kód z her)
void PlayTone(uint16_t frequency_div) {
    if (frequency_div == 0) {
        // Ticho - vypnout PWM
        TIM2_CC2Disable();
        return;
    }
    
    // Nastavit frekvenci a 50% duty cycle
    TIM2_Load(frequency_div);
    TIM2_Comp2(frequency_div / 2);      // 50% duty cycle
    TIM2_CC2Enable();
}

// Zastavení zvuku
void StopSound(void) {
    TIM2_CC2Disable();
}

// Herní funkce pro přehrání tónu (z reálných her)
void Sound(uint8_t freq, uint8_t dur) {
    if (freq == 0) {
        WaitMs(dur);                    // Pauza
    } else {
        // Výpočet frekvence: tone period = 510 - 2*freq [us]
        // frequency in [Hz] = 1000000/(510-2*freq)
        int n = (510 - 2*freq);
        PlayTone(n - 1);
        
        // Délka tónu
        n *= dur;
        n += n/2;                       // Mírné prodloužení
        WaitUs(n);
        StopSound();
    }
}

// Moderní buzz funkce (z RVPC her)
void buzz(uint32_t hz, uint32_t timeMS) {
    if (hz == 0) {
        WaitMs(timeMS);
        return;
    }
    
    // Převod Hz na timer divider
    uint32_t divider = 1000000 / hz;   // Perioda v μs
    PlayTone(divider - 1);
    WaitMs(timeMS);
    StopSound();
}
```

### Herní zvukové efekty

```c
// Zvukové efekty z reálných her
void Game_Audio_Effects(void) {
    // Efekt střelby (krátký vysoký tón)
    buzz(1000, 50);
    
    // Exploze (sestupný sweep)
    for (int f = 500; f > 100; f -= 20) {
        buzz(f, 10);
    }
    
    // Power-up (vzestupný sweep)
    for (int f = 200; f < 800; f += 50) {
        buzz(f, 20);
    }
    
    // Chyba/smrt (nízký brzký tón)
    buzz(150, 200);
    
    // OK zvuk (dvojtón)
    buzz(880, 100);
    WaitMs(50);
    buzz(1100, 100);
}

// Rytmické beepy pro hry typu Tetris
void Game_Beat_Sound(uint8_t level) {
    uint16_t tempo = 500 - (level * 30);  // Zrychlování s levelem
    if (tempo < 100) tempo = 100;
    
    buzz(440, 50);                        // A4 nota
    WaitMs(tempo);
}

// Background engine sound pro racing hry
volatile Bool engine_running = False;
volatile uint16_t engine_freq = 100;

void Engine_Sound_Init(void) {
    // Timer pro modulaci engine zvuku
    TIM1_Presc(4799);                     // 10kHz
    TIM1_Load(99);                        // 100Hz modulace
    TIM1_IntEnable();
    TIM1_Enable();
}

void TIM1_Handler(void) {
    if (TIM1_UpdateFlag() && engine_running) {
        // Modulace frekvence pro realistický engine zvuk
        engine_freq += (RandByte() % 20) - 10;  // ±10 Hz random
        if (engine_freq < 80) engine_freq = 80;
        if (engine_freq > 200) engine_freq = 200;
        
        PlayTone(1000000 / engine_freq);
        TIM1_UpdateFlagClr();
    }
}

void Engine_Start(void) {
    engine_running = True;
    engine_freq = 120;
}

void Engine_Stop(void) {
    engine_running = False;
    StopSound();
}
```

### Hudební přehrávání

```c
// Struktura pro melodii
typedef struct {
    uint16_t frequency;     // Frekvence v Hz (0 = pauza)
    uint16_t duration;      // Délka v ms
} Note_t;

// Příklad melodie (úvodní motiv)
const Note_t Startup_Melody[] = {
    {NOTE_C4, 200},
    {NOTE_E4, 200}, 
    {NOTE_G4, 200},
    {NOTE_C5, 400},
    {NOTE_PAUSE, 100},
    {NOTE_G4, 200},
    {NOTE_C5, 600},
    {0, 0}              // Konec melodie
};

// Game Over melodie
const Note_t GameOver_Melody[] = {
    {NOTE_C5, 300},
    {NOTE_B4, 300},
    {NOTE_A4, 300},
    {NOTE_G4, 600},
    {NOTE_PAUSE, 200},
    {NOTE_F4, 400},
    {NOTE_E4, 800},
    {0, 0}
};

// Přehrání melodie
void PlayMelody(const Note_t* melody) {
    int i = 0;
    while (melody[i].frequency != 0 || melody[i].duration != 0) {
        if (melody[i].frequency == NOTE_PAUSE) {
            StopSound();
            WaitMs(melody[i].duration);
        } else {
            uint32_t divider = 1000000 / melody[i].frequency;
            PlayTone(divider - 1);
            WaitMs(melody[i].duration);
        }
        i++;
    }
    StopSound();
}

// Herní situační zvuky
void Game_Situational_Audio(void) {
    // Úspěšné dokončení levelu
    PlayMelody(Startup_Melody);
    
    // Game Over
    PlayMelody(GameOver_Melody);
    
    // Rychlý beep pro countdown
    for (int i = 3; i > 0; i--) {
        buzz(800, 200);
        WaitMs(300);
    }
    buzz(1200, 500);  // GO!
}
```

## DMA podpora

```c
// DMA pro timer
void TIM1_DMAEnable(void);          // Povolit DMA pro update
void TIM1_CC1DMAEnable(void);       // DMA pro CC1
void TIM1_CC2DMAEnable(void);       // DMA pro CC2
void TIM1_CC3DMAEnable(void);       // DMA pro CC3
void TIM1_CC4DMAEnable(void);       // DMA pro CC4
```

## Tipy a triky

1. **PWM frekvence**: Vyšší frekvence = menší rozlišení duty cycle
2. **Filtrace**: Použijte vstupní filtry pro rušivé signály
3. **Dead-time**: Povinné pro H-bridge aplikace
4. **Přerušení**: Preferujte DMA pro vysokofrekvenční aplikace
5. **Synchronizace**: Timery lze řetězit pomocí trigger výstupů (spouštěcí signály)

## Výpočty

### PWM frekvence
```
PWM_freq = Timer_freq / (Prescaler + 1) / (Auto_reload + 1)
```

### Duty cycle
```
Duty_cycle = Compare_value / (Auto_reload + 1) * 100%
```

### Timer frekvence
```
Timer_freq = System_clock / (Prescaler + 1)
```

## Poznámky pro různé MCU

- **CH32V003**: TIM1 a TIM2 s DMA podporou (2KB SRAM)
- **CH32V002**: Vylepšená varianta s 4KB SRAM, 12-bit ADC, touch sensing
- **CH32V006**: Pokročilá varianta s více možnostmi remapování
- **CH32X035**: Nejpokročilejší s rozšířenými timer funkcemi

## Mapování pinů

### TIM1 standardní mapování:
- **CH1**: PA8, **CH1N**: PB13
- **CH2**: PA9, **CH2N**: PB14  
- **CH3**: PA10, **CH3N**: PB15
- **CH4**: PA11
- **ETR**: PA12, **BKIN**: PB12

### TIM2 standardní mapování:
- **CH1**: PA0
- **CH2**: PA1
- **CH3**: PA2  
- **CH4**: PA3
- **ETR**: PA15

### Remapování:
```c
GPIO_Remap_TIM1(map);   // TIM1 remapování
GPIO_Remap_TIM2(map);   // TIM2 remapování
```