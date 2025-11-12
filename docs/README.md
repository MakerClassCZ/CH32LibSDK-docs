# CH32LibSDK - Dokumentace

> **⚠️ Upozornění:** Tato dokumentace byla vygenerována pomocí umělé inteligence ze zdrojových souborů, příkladů a analýzy kódu. Informace v ní obsažené nemusí být 100% přesné. V případě pochybností se prosím vždy obraťte na oficiální dokumentaci od WCH nebo ověřte informace přímo ve zdrojovém kódu.
> Dokumentaci připravil a vygeneroval [Vláďa Smitka](https://x.com/MakerClassCZ).

## Úvod

CH32LibSDK je vývojová sada (SDK) pro mikrokontroléry řady CH32 RISC-V od společnosti WCH. SDK je navrženo především pro vývoj her a aplikací pro retro herní konzole, ale lze jej použít pro jakýkoliv vývoj embedded aplikací na těchto mikrokontrolérech.

## Podporované mikrokontroléry

SDK podporuje následující řady mikrokontrolérů CH32:

- **CH32V002/003/006** - Základní RISC-V MCU s různou kapacitou RAM/Flash
- **CH32X035** - Pokročilejší varianta s více funkcemi
- **CH32V103/L103** - Výkonnější varianty s větší pamětí

## Podporovaná zařízení

SDK obsahuje podporu pro následující herní konzole a vývojová zařízení:

### BabyBoy
- 128x64 OLED displej (I2C, SSD1306)
- 6 tlačítek
- Vestavěný reproduktor
- Napájení z CR2032 baterie

### BabyPad
- VGA výstup (mono)
- 7 tlačítek
- Audio výstup
- Napájení 5V

### BabyPC (WCH80)
- Mini počítač založený na ZX80
- Dva procesory CH32V002
- BASIC80 programovací jazyk
- VGA výstup

### PidiBoy
- 128x64 OLED displej
- 7 tlačítek
- MicroSD karta
- Vestavěný reproduktor
- Boot loader pro načítání programů

### PidiPad
- VGA výstup
- 8 tlačítek
- MicroSD karta
- Audio výstup

### TweetyBoy
- 160x80 barevný LCD displej (SPI, ST7735S)
- 8 tlačítek
- MicroSD karta
- Vestavěný reproduktor

### BeatleBoyPad
- Duální konzole s vyměnitelnými cartridge
- Podpora CH32V002/003/006 v cartridge

## Struktura projektu

```
CH32LibSDK/
├── _sdk/           # SDK knihovny pro jednotlivé MCU
├── _devices/       # Implementace pro jednotlivá zařízení
├── _lib/           # Obecné knihovny (CRC, FAT, SD karta, atd.)
├── _font/          # Fonty pro zobrazení textu
├── _loader/        # Boot loadery pro SD karty
├── _tools/         # Nástroje pro build
├── docs/           # Dokumentace
└── [Device]/Games/ # Ukázkové hry pro jednotlivá zařízení
```

## Rychlý start

### Požadavky

1. **Kompilátor**: MounRiver Studio v1.92 nebo novější
2. **Programátor**: WCH-LinkE
3. **OS**: Windows (primárně), Linux přes WSL

### Instalace

1. Stáhněte a nainstalujte MounRiver Studio
2. Extrahujte kompilátor RISC-V Embedded GCC12 do samostatné složky
3. Upravte cestu v souboru `_c1.bat`:
   ```batch
   set GCC_CH32_PATH=C:\GCC_CH32\bin
   ```

### Kompilace projektu

1. Přejděte do složky s projektem (např. `Babyboy/Games/Atoms`)
2. Spusťte kompilaci:
   ```batch
   c.bat
   ```
3. Pro vyčištění:
   ```batch
   d.bat
   ```
4. Pro nahrání do MCU:
   ```batch
   e.bat
   ```

## SDK moduly

SDK obsahuje následující hlavní moduly:

### Základní periferie
- [GPIO](gpio.md) - Práce s digitálními piny
- [SPI](spi.md) - SPI komunikace
- [I2C](i2c.md) - I2C komunikace
- [TIM](timer.md) - Časovače
- [DMA](dma.md) - Přímý přístup do paměti
- [ADC](adc.md) - Analogově-digitální převodník
- [USART](usart.md) - Sériová komunikace
- [FLASH](flash.md) - Práce s flash pamětí
- [IRQ](irq.md) - Správa přerušení
- [PWR](pwr.md) - Řízení napájení
- [RCC](rcc.md) - Řízení hodin
- [SYSTICK](systick.md) - Systémový časovač

### Grafika a zařízení
- [Devices](devices.md) - Definice periferních zařízení (display, tlačítka, zvuk)
- [Display](display.md) - Práce s displayem, kreslení, fonty

### Herní vývoj
- [Game Patterns](game_patterns.md) - Obecné herní programovací vzory
- [Game Examples](game_examples.md) - Konkrétní příklady technik z her
- [Quick Start](quickstart.md) - Rychlý start s CH32LibSDK

## Vytvoření nového projektu

1. Zkopírujte existující projekt jako šablonu
2. Upravte `Makefile`:
   - `TARGET`: Název projektu
   - `DEVCLASS`: Cílové zařízení
   - `CSRC`: Zdrojové soubory
3. Zachovejte strukturu include souborů (`config.h`, `include.h`)

## Příklady

SDK obsahuje přes 25 ukázkových her demonstrujících použití různých funkcí:

- Atoms - Strategická hra
- Tetris - Klasická padající bloky
- Invaders - Space Invaders klon
- Maze - Bludiště
- Sokoban - Logická hra
- A mnoho dalších...

## Odkazy

- [GitHub repository](https://github.com/Panda381/CH32LibSDK)
- [Webová stránka projektu](https://www.breatharian.eu/hw/ch32libsdk/index.html)
- [WCH - výrobce čipů](http://www.wch-ic.com/)
- [CH32LibSDK Docker Build](https://github.com/MakerClassCZ/CH32LibSDK-docker)
- [Bastlířské kurzy MakerClass](https://makerclass.cz/)


## Licence

Copyright (c) 2025 Miroslav Nemecek
