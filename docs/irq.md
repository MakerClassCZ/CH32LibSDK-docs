# IRQ - Správa přerušení

## Úvod

CH32LibSDK poskytuje kompletní API pro správu přerušení (IRQ - Interrupt Request) prostřednictvím PFIC (Programmable Fast Interrupt Controller), který je kompatibilní s NVIC. Tento dokument pokrývá nastavení přerušení, priorit, handlerů a external interrupts (EXTI).

## PFIC (Programmable Fast Interrupt Controller)

PFIC je interrupt controller na adrese `0xE000E000`.

### Definované IRQ č

ísla

**CPU Internal Interrupts:**
```c
#define IRQ_RESET         0    // Reset
#define IRQ_NMI           2    // Non Maskable Interrupt
#define IRQ_HARDFAULT     3    // Hard Fault Exception
#define IRQ_SYSTICK      12    // System Timer
#define IRQ_SW           14    // Software interrupt
```

**External Interrupts (příklad CH32V2):**
```c
#define IRQ_WWDG         16    // Window WatchDog
#define IRQ_FLASH        20    // Flash Controller
#define IRQ_RCC          21    // Reset & Clock Control
#define IRQ_EXTI0        22    // External Line 0
#define IRQ_DMA1_CH1     27    // DMA1 Channel 1
#define IRQ_TIM2         44    // Timer 2
#define IRQ_USART1       53    // USART 1
```

## Registrace handlerů

### Metoda 1: Slabé handlery (Weak Handlers)

```c
// Handler označení
#define HANDLER __attribute__((interrupt("WCH-Interrupt-fast")))

// Vaše implementace přepíše slabý handler
HANDLER void TIM2_IRQHandler(void)
{
    // Obsluha přerušení
    TIM2_UpdateFlagClr();  // MUSÍTE vymazat flag!
}
```

### Metoda 2: Dynamická registrace

```c
typedef void (*irq_handler_t)(void);

HANDLER void my_custom_handler(void)
{
    // Custom obsluha
}

int main(void)
{
    SetHandler(IRQ_TIM2, (irq_handler_t)my_custom_handler);
}
```

### Metoda 3: VTF (Vector Table Free) - 4 hardware kanály

```c
// Registrace VTF kanálu (kanál 0-3)
void NVIC_VTFEnable(int ch, int irq, irq_handler_t handler);

// Příklad
HANDLER void my_fast_handler(void)
{
    // Velmi rychlá obsluha
}

int main(void)
{
    NVIC_VTFEnable(0, IRQ_TIM2, (irq_handler_t)my_fast_handler);
}
```

## Prioritní systém

```c
// Předvolené priority (nižší číslo = vyšší priorita)
#define IRQ_PRIO_VERYHIGH   0x00    // Nejvyšší
#define IRQ_PRIO_HIGH       0x40    // Vysoká
#define IRQ_PRIO_NORMAL     0x80    // Normální (výchozí)
#define IRQ_PRIO_LOW        0xc0    // Nejnižší

// Nastavit prioritu
INLINE void NVIC_IRQPriority(int irq, int prio);

// Příklady
NVIC_IRQPriority(IRQ_EXTI0, IRQ_PRIO_VERYHIGH);
NVIC_IRQPriority(IRQ_SYSTICK, IRQ_PRIO_HIGH);
NVIC_IRQPriority(IRQ_USART1, IRQ_PRIO_NORMAL);
```

## Enable/Disable přerušení

### Jednotlivé IRQ

```c
// Povolit/zakázat IRQ
INLINE void NVIC_IRQEnable(int irq);
INLINE void NVIC_IRQDisable(int irq);
INLINE Bool NVIC_IRQIsEnabled(int irq);

// Příklad
NVIC_IRQEnable(IRQ_TIM2);
```

### Globální enable/disable

```c
// Globálně vypnout/zapnout přerušení
INLINE void di(void);    // Disable interrupts
INLINE void ei(void);    // Enable interrupts
INLINE Bool geti(void);  // Get interrupt state

// Kritické sekce
INLINE Bool LockIRQ(void);      // Zakázat a vrátit původní stav
INLINE void UnlockIRQ(Bool state);

// Makra pro kritické sekce
#define IRQ_LOCK        Bool irq_state = LockIRQ()
#define IRQ_UNLOCK      UnlockIRQ(irq_state)
```

### Příklad kritické sekce

```c
volatile int counter = 0;

void increment_safe(void)
{
    IRQ_LOCK;               // Zakázat přerušení
    counter++;              // Kritická operace
    IRQ_UNLOCK;             // Obnovit přerušení
}
```

## Pending a aktivní stavy

```c
// Pending (čekající) přerušení
INLINE void NVIC_IRQForce(int irq);     // Vnutit přerušení
INLINE void NVIC_IRQClear(int irq);     // Vymazat pending
INLINE Bool NVIC_IRQIsPending(int irq); // Je pending?

// Aktivní přerušení
INLINE Bool NVIC_IRQIsActive(int irq);  // Je aktivní?
INLINE u8 NVIC_GetNest(void);           // Získat nesting level
```

## External Interrupts (EXTI)

EXTI umožňuje generovat přerušení z GPIO pinů a interních periférií.

### EXTI řádky

```c
#define EXTI_LINE0      0       // GPIO PA0, PB0, PC0...
#define EXTI_LINE1      1       // GPIO PA1, PB1, PC1...
// ... až EXTI_LINE15
#define EXTI_LINE16     16      // PVD Output
#define EXTI_LINE17     17      // RTC Alarm
```

### Ovládání EXTI

```c
// Enable/Disable interrupt
INLINE void EXTI_Enable(int line);
INLINE void EXTI_Disable(int line);

// Rising/Falling edge trigger
INLINE void EXTI_RiseEnable(int line);   // Vzestupná hrana
INLINE void EXTI_FallEnable(int line);   // Sestupná hrana

// Check/Clear pending
INLINE Bool EXTI_IsPending(int line);
INLINE void EXTI_Clear(int line);
```

## Příklady použití

### Příklad 1: Timer přerušení

```c
// Handler
HANDLER void SysTick_Handler(void)
{
    SysTick_ClrCmp();       // Vymazat flag
    // Váš kód...
}

// Inicializace
void SysTick_Init(void)
{
    SysTick_Disable();
    SysTick_Set(0);
    SysTick_Enable();
    
    NVIC_IRQEnable(IRQ_SYSTICK);    // Povolit v NVIC
    SysTick_IntEnable();             // Povolit v periferii
}
```

### Příklad 2: External interrupt (tlačítko)

```c
// GPIO + EXTI konfigurace
void Button_Init(void)
{
    // GPIO jako vstup
    GPIO_Mode(PA0, GPIO_MODE_IN_PU);
    
    // EXTI na PA0 = EXTI_LINE0
    GPIO_EXTILine(GPIO_PORT_A, 0);
    
    // EXTI konfigurace
    EXTI_RiseEnable(EXTI_LINE0);    // Vzestupná hrana
    EXTI_Enable(EXTI_LINE0);
    
    // NVIC konfigurace
    NVIC_IRQEnable(IRQ_EXTI0);
    NVIC_IRQPriority(IRQ_EXTI0, IRQ_PRIO_HIGH);
}

// Handler
HANDLER void EXTI0_IRQHandler(void)
{
    if (EXTI_IsPending(EXTI_LINE0)) {
        // Button pressed
        HandleButtonPress();
        
        EXTI_Clear(EXTI_LINE0);     // MUSÍTE vymazat flag!
    }
}
```

### Příklad 3: Timer s měřením frekvence

```c
volatile uint32_t frequency = 0;

HANDLER void TIM1_CC_Handler(void)
{
    if (TIM1_CC1Flag()) {
        uint16_t current_capture = TIM1_GetComp1();
        // Výpočet frekvence...
        
        TIM1_CC1FlagClr();          // Vymazat flag!
    }
}

// V main():
NVIC_IRQEnable(IRQ_TIM1_CC);
```

## Memory Barriers

Pro synchronizaci paměti a instrukcí:

```c
INLINE void cb(void);    // Compiler barrier
INLINE void isb(void);   // Instruction sync barrier
INLINE void dmb(void);   // Data memory barrier
INLINE void dsb(void);   // Data sync barrier
```

**Kdy použít:**
- `cb()` - Po změně vektoru
- `dmb()` - Po zápisu do registrů
- `isb()` - Po kritických zápisech (vektory, CSR)

## Kontrolní seznam

1. ✅ **Handler označení** - Vždy použít `HANDLER` makro
2. ✅ **Enable/Disable** - Povolit jak v periferii, tak v NVIC
3. ✅ **Flag clearing** - VŽDY vymazat flag na konci handleru
4. ✅ **Memory barriers** - Použít po kritických zápisech
5. ✅ **Priorita** - Nastavit vhodnou prioritu
6. ✅ **Kritické sekce** - Chránit se `IRQ_LOCK`/`IRQ_UNLOCK`

## Související dokumentace

- [SysTick](systick.md) - Systémový časovač s přerušením
- [Timer](timer.md) - Timer přerušení
- [GPIO](gpio.md) - EXTI konfigurace
