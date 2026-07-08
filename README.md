# Display Detective

> **Knows nothing about your display — but will find its pins.  Perhaps**
> If you have an ESP32 board with a built-in display, and you can't find the correct pin config for TFT libraries, this may help.
> After wasting a lot of time trying to get the ESP board from hell, with built-in display, working, I semi vib-coded this.
> It cycles through scraped known pin configs - it uses the serial port.  It will try each one, and update you - if the thing hangs, just add 1
> to a line at the top of the code, upload, and it will carry on.  It treies hard not to break it.
>
> The program WILL crash your board as it runs.  It may crash it so that you have to do a hardware reset on the board.
> If you've come here, that probably doesn't amtter, as the next step is the bin.
> 
> Upload, watch the serial monitor, wait for a red screen.

---

## How to Use

### 1. Upload the sketch

Set mode to `MODE_DB` (it's the default):

```c
#define MODO MODE_DB
```

Compile and upload to your ESP32.

**ESP32-S3 needs USB CDC On Boot = Enabled** or you'll see no serial output:

| Method | Command / Setting |
|--------|------------------|
| arduino-cli | `--fqbn "esp32:esp32:esp32s3:CDCOnBoot=cdc"` |
| Arduino IDE | Tools → USB CDC On Boot → Enabled |

### 2. Open Serial Monitor at 115200 baud

You'll see the probe iterate through configs:

```
Config 0/202  ILI9341  MOSI:23  SCLK:18  CS:15  DC:2  RST:4  BL:-1  240x320
  SPI mode 0...done — see red fill?
  SPI mode 1...done — see red fill?
  ...
```

Each config is tested for ~5 seconds (SPI modes 0–3).

### 3. Watch the display

When the screen turns **solid red**, check the serial output for the config number.

That config's pin assignments work for your display. Use them in TFT_eSPI or any other library.

---

## How it Works

You have an ESP32 board with a display attached — maybe a CYD, maybe a T-Display, maybe an AliExpress special with no documentation. The display is soldered on; you can't probe the pins.

The probe has a database of ~200 known ESP32+LCD pin configurations scraped from GitHub, forums, and product pages. It tries each one:

1. Sets up SPI on the config's pins
2. Sends the display's init sequence
3. Fills the screen red
4. Moves to the next config

No wiring, no guessing, no User_Setup.h edits. Just upload and wait.

---

## Modes

| Mode | What it does | When to use |
|------|-------------|-------------|
| `MODE_DB` | Walk the built-in config database | Default — works for most boards |
| `MODE_BRUTE` | DB pins × one controller type | You know the LCD controller chip but not the pins |
| `MODE_HAIL` | DB pins × all controllers × all SPI modes | Nothing worked — cast a wide net |
| `MODE_MANUAL` | Test one hardcoded pinout | You already know the pins and just want to verify they work |

Filters:

- `START_FROM` — skip ahead in the list (useful when a known config is near the bottom)
- `#define FILTER_CTRL CTRL_XXX` — test only one controller type in DB mode

---

## Troubleshooting

### "No serial output"

**ESP32-S3:** USB CDC On Boot must be enabled. See step 1.

**ESP32 (original):** Verify you're on the correct UART USB port (usually the one labelled "UART" if the board has dual USB ports).

### "The probe runs but no red screen"

- **Wrong controller type** — the database may list your board's pins but with the wrong LCD controller. Try BRUTE mode pinned to the controller chip on your display.
- **Wrong SPI mode** — some displays need mode 1, 2, or 3. HAIL mode tries all four.
- **Your board uses parallel, not SPI** — the probe only tests SPI. If your display is an 8-bit I80 parallel type (many GC9A01 round modules *are* SPI despite I80 labels, but some aren't), you need a parallel library.

### "Solid colour but wrong / flickering / interlaced"

The init sequence may be incorrect for your controller revision. The minimal init (SLPOUT + COLMOD + DISPON) is tried as a fallback. If the red fill is still wrong, the display is working — the TFT driver library handles the fine-tuning.

### "GPIO3 crash in TFT_eSPI"

If the probe reports a config using GPIO3 as SCLK, note that GPIO3 is the JTAG TMS pin on ESP32-S3. Raw SPI works fine (the probe uses it), but TFT_eSPI will crash with a StoreProhibited exception. Choose a config that avoids GPIO3, or remap to a different SCLK pin before using TFT_eSPI.

### "Brownout resets"

The display draws significant current. A 100–470 µF capacitor across 3.3V/GND near the display connector often fixes it.

### "No CS pin"

Some displays tie CS to GND permanently. Configs with `CS=-1` in the database are handled automatically — the probe skips CS toggling.

---

## Project Structure

```
DisplayDetective/
├── firmware/
│   ├── esp32_lcd_detective/
│   │   ├── esp32_lcd_detective.ino   # Main probe
│   │   └── configs.h                 # 200 pin configs (auto-generated)
│   ├── generate_header.py            # configs.json → configs.h
│   ├── spi_test/
│   │   └── spi_test.ino              # Manual SPI verification
│   └── parallel_test/
│       └── parallel_test.ino         # Manual parallel test
├── scraper/
│   ├── scrape_configs.py             # Harvests configs from the web
│   └── requirements.txt
├── data/
│   └── configs.json                  # Master database with sources
└── esp32_lcd_configs.py              # Standalone database viewer
```

The database (`data/configs.json`) includes every pin, the controller type, resolution, and source URLs. The header generator (`firmware/generate_header.py`) packs it into a PROGMEM struct for the ESP32.

---

## Adding a New Config

1. Add an entry to `data/configs.json` (see existing entries for the schema).
2. Run `firmware/generate_header.py` to rebuild `configs.h`.
3. Recompile and upload.

Configs can also be contributed by adding seed URLs to `scraper/scrape_configs.py`.

---

## License

MIT
