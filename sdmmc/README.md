| Supported Targets | ESP32 | ESP32-P4 | ESP32-S3 |
| ----------------- | ----- | -------- | -------- |

# SD Card Test Application (SDMMC)

__WARNING:__ This application will potentially delete all data from your SD card (when formatting is enabled via menuconfig). Back up your data first before proceeding.

## Overview

This test application validates SDMMC peripheral functionality with SD cards using a visual status indicator on GPIO0. The application performs comprehensive SD card operations and signals success/failure through GPIO0 behavior.

### Application Flow

1. **Initialize GPIO0** as open-drain output (status indicator)
2. **Mount SD Card** using SDMMC peripheral with FAT filesystem
3. **Perform File Operations:**
   - Create and write `hello.txt`
   - Rename to `foo.txt`
   - Read back and verify content
4. **Optional Format** (if `CONFIG_EXAMPLE_FORMAT_SD_CARD` enabled)
5. **Create Additional File** (`nihao.txt`) for verification
6. **Signal Success** via GPIO0 toggle pattern

### GPIO0 Status Indicator

The application uses **GPIO0** as a visual status indicator with the following behavior:

| GPIO0 State | Meaning |
|-------------|---------|
| **LOW (0V)** | Initializing, mounting, formatting, or operation failed |
| **Toggling at 1Hz** | All operations completed successfully |

- During SD card operations (mount, read, write, format), GPIO0 remains **LOW**
- Only after **all operations succeed** does GPIO0 begin toggling
- If any operation fails, GPIO0 is set **LOW** and remains there

This allows visual monitoring of test progress with a simple LED or logic analyzer connected to GPIO0.

## Hardware Requirements

- ESP32, ESP32-S3, or ESP32-P4 development board with SDMMC peripheral
- SD card (SDSC, SDHC, or SDXC)
- **GPIO0** connected to LED or logic analyzer for status monitoring
- Proper pullup resistors on SD card lines (10kΩ recommended)

### Recommended Setup

For visual feedback, connect an LED to GPIO0:
```
ESP32 GPIO0 ----[LED]----[Resistor]---- GND
```
Note: GPIO0 is open-drain, so the LED cathode connects to GPIO0.

This test application does not use card detect (CD) or write protect (WP) signals.

### Pin assignments for ESP32

On ESP32, SDMMC peripheral is connected to specific GPIO pins using the IO MUX. GPIO pins cannot be customized. Please see the table below for the pin connections.

When using an ESP-WROVER-KIT board, this example runs without any extra modifications required. Only an SD card needs to be inserted into the slot.

ESP32 pin     | SD card pin | Notes
--------------|-------------|------------
GPIO14 (MTMS) | CLK         | 10k pullup in SD mode
GPIO15 (MTDO) | CMD         | 10k pullup in SD mode
GPIO2         | D0          | 10k pullup in SD mode, pull low to go into download mode (see Note about GPIO2 below!)
GPIO4         | D1          | not used in 1-line SD mode; 10k pullup in 4-line SD mode
GPIO12 (MTDI) | D2          | not used in 1-line SD mode; 10k pullup in 4-line SD mode (see Note about GPIO12 below!)
GPIO13 (MTCK) | D3          | not used in 1-line SD mode, but card's D3 pin must have a 10k pullup


### Pin assignments for ESP32-S3

On ESP32-S3, SDMMC peripheral is connected to GPIO pins using GPIO matrix. This allows arbitrary GPIOs to be used to connect an SD card. In this example, GPIOs can be configured in two ways:

1. Using menuconfig: Run `idf.py menuconfig` in the project directory and open "SD/MMC Example Configuration" menu.
2. In the source code: See the initialization of `sdmmc_slot_config_t slot_config` structure in the example code.

The table below lists the default pin assignments.

When using an ESP32-S3-USB-OTG board, this example runs without any extra modifications required. Only an SD card needs to be inserted into the slot.

ESP32-S3 pin  | SD card pin | Notes
--------------|-------------|------------
GPIO36        | CLK         | 10k pullup
GPIO35        | CMD         | 10k pullup
GPIO37        | D0          | 10k pullup
GPIO38        | D1          | not used in 1-line SD mode; 10k pullup in 4-line mode
GPIO33        | D2          | not used in 1-line SD mode; 10k pullup in 4-line mode
GPIO34        | D3          | not used in 1-line SD mode, but card's D3 pin must have a 10k pullup

### Pin assignments for ESP32-P4

On ESP32-P4, Slot 1 of the SDMMC peripheral is connected to GPIO pins using GPIO matrix. This allows arbitrary GPIOs to be used to connect an SD card. In this example, GPIOs can be configured in two ways:

1. Using menuconfig: Run `idf.py menuconfig` in the project directory and open `SD/MMC Example Configuration` menu.
2. In the source code: See the initialization of `sdmmc_slot_config_t slot_config` structure in the example code.

The table below lists the default pin assignments.

ESP32-P4 pin  | SD card pin | Notes
--------------|-------------|------------
GPIO43        | CLK         | 10k pullup
GPIO44        | CMD         | 10k pullup
GPIO39        | D0          | 10k pullup
GPIO40        | D1          | not used in 1-line SD mode; 10k pullup in 4-line mode
GPIO41        | D2          | not used in 1-line SD mode; 10k pullup in 4-line mode
GPIO42        | D3          | not used in 1-line SD mode, but card's D3 pin must have a 10k pullup

Default dedicated pins on ESP32-P4 are able to connect to an ultra high-speed SD card (UHS-I) which requires 1.8V switching (instead of the regular 3.3V). This means the user has to provide an external LDO power supply to use them, or to enable and configure an internal LDO via `idf.py menuconfig` -> `SD/MMC Example Configuration` -> `SD power supply comes from internal LDO IO`.

When using different GPIO pins this is not required and `SD power supply comes from internal LDO IO` setting can be disabled.

### 4-line and 1-line SD modes

By default, this example uses 4 line SD mode, utilizing 6 pins: CLK, CMD, D0 - D3. It is possible to use 1-line mode (CLK, CMD, D0) by changing "SD/MMC bus width" in the example configuration menu (see `CONFIG_EXAMPLE_SDMMC_BUS_WIDTH_1`).

Note that even if card's D3 line is not connected to the ESP chip, it still has to be pulled up, otherwise the card will go into SPI protocol mode.

### Note about GPIO2 (ESP32 only)

GPIO2 pin is used as a bootstrapping pin, and should be low to enter UART download mode. One way to do this is to connect GPIO0 and GPIO2 using a jumper, and then the auto-reset circuit on most development boards will pull GPIO2 low along with GPIO0, when entering download mode.

- Some boards have pulldown and/or LED on GPIO2. LED is usually ok, but pulldown will interfere with D0 signals and must be removed. Check the schematic of your development board for anything connected to GPIO2.

### Note about GPIO12 (ESP32 only)

GPIO12 is used as a bootstrapping pin to select output voltage of an internal regulator which powers the flash chip (VDD_SDIO). This pin has an internal pulldown so if left unconnected it will read low at reset (selecting default 3.3V operation). When adding a pullup to this pin for SD card operation, consider the following:

- For boards which don't use the internal regulator (VDD_SDIO) to power the flash, GPIO12 can be pulled high.
- For boards which use 1.8V flash chip, GPIO12 needs to be pulled high at reset. This is fully compatible with SD card operation.
- On boards which use the internal regulator and a 3.3V flash chip, GPIO12 must be low at reset. This is incompatible with SD card operation.
    * In most cases, external pullup can be omitted and an internal pullup can be enabled using a `gpio_pullup_en(GPIO_NUM_12);` call. Most SD cards work fine when an internal pullup on GPIO12 line is enabled. Note that if ESP32 experiences a power-on reset while the SD card is sending data, high level on GPIO12 can be latched into the bootstrapping register, and ESP32 will enter a boot loop until external reset with correct GPIO12 level is applied.
    * Another option is to burn the flash voltage selection efuses. This will permanently select 3.3V output voltage for the internal regulator, and GPIO12 will not be used as a bootstrapping pin. Then it is safe to connect a pullup resistor to GPIO12. This option is suggested for production use.

The following command can be used to program flash voltage selection efuses **to 3.3V**:

```sh
    components/esptool_py/esptool/espefuse.py set_flash_voltage 3.3V
```

This command will burn the `XPD_SDIO_TIEH`, `XPD_SDIO_FORCE`, and `XPD_SDIO_REG` efuses. With all three burned to value 1, the internal VDD_SDIO flash voltage regulator is permanently enabled at 3.3V. See the technical reference manual for more details.

`espefuse.py` has a `--do-not-confirm` option if running from an automated flashing script.

See [the document about pullup requirements](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/peripherals/sd_pullup_requirements.html) for more details about pullup support and compatibility of modules and development boards.

## Configuration Options

Use `idf.py menuconfig` to configure the application:

### Key Configuration Options

1. **SD/MMC Example Configuration**
   - `CONFIG_EXAMPLE_FORMAT_IF_MOUNT_FAILED` - Auto-format if mount fails
   - `CONFIG_EXAMPLE_FORMAT_SD_CARD` - Explicitly format card after initial operations
   - `CONFIG_EXAMPLE_SDMMC_BUS_WIDTH_4` - Use 4-line mode (recommended) or 1-line mode
   - Pin assignments (for GPIO matrix chips like ESP32-S3, ESP32-P4)
   - Speed settings (Default/HS/UHS-I SDR50/DDR50)

2. **Debug Options**
   - `CONFIG_EXAMPLE_DEBUG_PIN_CONNECTIONS` - Enable pin diagnostics and ADC voltage monitoring

### Format Behavior

- **Format on Mount Failure**: Enable `CONFIG_EXAMPLE_FORMAT_IF_MOUNT_FAILED` to automatically partition and format unformatted cards
- **Explicit Format**: Enable `CONFIG_EXAMPLE_FORMAT_SD_CARD` to format the card after initial file operations (deletes all data)
- **Both Disabled** (default): Will not format the card; requires pre-formatted FAT32 card

## Build and Flash

```bash
# Configure the project
idf.py menuconfig

# Build the application
idf.py build

# Flash to device and monitor output
idf.py -p PORT flash monitor
```

Replace `PORT` with your serial port (e.g., `/dev/ttyUSB0` on Linux).

To exit the serial monitor, press `Ctrl-]`.

## Expected Output

### Successful Operation

When all operations complete successfully, you should see:

```
I (336) example: Initializing SD card
I (336) example: Using SDMMC peripheral
I (596) example: Mounting filesystem
I (596) example: Filesystem mounted
Name: XA0E5
Type: SDHC/SDXC
Speed: 20 MHz
Size: 61068MB
I (7386) example: Opening file /sdcard/hello.txt
I (7396) example: File written
I (7396) example: Renaming file /sdcard/hello.txt to /sdcard/foo.txt
I (7396) example: Reading file /sdcard/foo.txt
I (7396) example: Read from file: 'Hello XA0E5!'
I (7396) example: Opening file /sdcard/nihao.txt
I (7396) example: File written
I (7396) example: Reading file /sdcard/nihao.txt
I (7396) example: Read from file: 'Nihao XA0E5!'
I (7396) example: Card unmounted
I (7396) example: All operations successful, toggling GPIO0 at 1Hz
```

**GPIO0 will toggle at 1Hz** (500ms on, 500ms off) indicating success.

### With Formatting Enabled

If `CONFIG_EXAMPLE_FORMAT_SD_CARD` is enabled:

```
...
I (7396) example: Read from file: 'Hello XA0E5!'
W (7400) example: Formatting card...
W (8200) example: Formatting done
I (8200) example: file doesn't exist, formatting done
I (8200) example: Opening file /sdcard/nihao.txt
...
```

The card is formatted between the initial file operations and final verification.

## Troubleshooting

### GPIO0 Stays LOW (Not Toggling)

If GPIO0 remains LOW and doesn't toggle, an operation has failed. Check the serial output for error messages:

- **Mount failure**: Check SD card connections and pullup resistors
- **File operation failure**: Verify the card is properly formatted (FAT32)
- **Format failure**: Card may be write-protected or damaged

### Failure to Download Firmware

```
Connecting........_____....._____....._____....._____
A fatal error occurred: Failed to connect to Espressif device
```

**Solution**: GPIO0 is a bootstrap pin. If connected to an LED with strong pullup/pulldown:
- Disconnect GPIO0 temporarily for flashing
- Use a high-value resistor (≥1kΩ) in series with LED
- Use open-drain configuration (already set in code)

### Card Initialization Failures

**Error: `sdmmc_init_sd_scr: send_scr (1) returned 0x107`**
- Check all SD card connections
- Verify pullup resistors are present (10kΩ on all data lines)
- Reset the ESP32 after connecting the card

**Error: `sdmmc_check_scr: send_scr returned 0xffffffff`**
- Connections are too long or have poor signal integrity
- Use shorter wires
- Reduce clock frequency in menuconfig
- Add stronger pullups if using breakout adapter

### Mount Failure

```
example: Failed to mount filesystem. If you want the card to be formatted, 
set the EXAMPLE_FORMAT_IF_MOUNT_FAILED menuconfig option.
```

**Solutions:**
- Enable `CONFIG_EXAMPLE_FORMAT_IF_MOUNT_FAILED` in menuconfig (will erase card)
- Format the card as FAT32 using a computer before use
- Verify the card is not corrupted

### Debugging Pin Connections

Enable `CONFIG_EXAMPLE_DEBUG_PIN_CONNECTIONS` in menuconfig to run diagnostics that measure:
- **Pin recovery time**: Verifies pullup presence (should be <300 cycles with pullups)
- **Pin voltage levels**: Should be 3.1-3.3V with 10kΩ pullups
- **Pin cross-talk**: Detects interference between pins

Example diagnostic output:
```
**** PIN recovery time ****
PIN 14 CLK  250 cycles    ← Good (pullup present)
PIN 15 CMD  10034 cycles  ← Bad (no pullup detected)

**** PIN voltage levels ****
PIN 14 CLK  3.2V  ← Good
PIN 15 CMD  0.3V  ← Bad (no pullup)
```

### GPIO2 and GPIO12 Notes (ESP32 Only)

- **GPIO2**: Bootstrap pin, may need disconnection during flashing
- **GPIO12**: Controls flash voltage regulator
  - Most ESP32 boards: Can use internal pullup (`gpio_pullup_en`)
  - Some boards: May need to burn efuses for 3.3V flash voltage

See [ESP-IDF pullup requirements documentation](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/peripherals/sd_pullup_requirements.html) for details.

## Test Verification

### Manual Testing

1. **Power on** - Observe GPIO0 is LOW
2. **Wait for operations** - GPIO0 remains LOW (3-10 seconds depending on card)
3. **Success** - GPIO0 begins toggling at 1Hz
4. **Failure** - GPIO0 stays LOW, check serial output

### Automated Testing

The application behavior makes it suitable for automated testing:
- Monitor GPIO0 with logic analyzer or test fixture
- Timeout after 30 seconds if no toggle detected = FAIL
- Detect 1Hz toggle pattern = PASS

## Project Structure

```
sdmmc/
├── main/
│   ├── sdmmc_test.c          # Main test application
│   ├── Kconfig.projbuild     # Configuration options
│   └── CMakeLists.txt        # Component build config
├── components/
│   └── sd_card/
│       ├── sd_test_io.c      # Pin diagnostic utilities
│       └── sd_test_io.h
├── CMakeLists.txt            # Project build config
├── sdkconfig                 # Current configuration
├── sdkconfig.defaults        # Default configuration
└── README.md                 # This file
```

## License

This test application code is in the Public Domain (or CC0 licensed, at your option).