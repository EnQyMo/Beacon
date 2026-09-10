# ESP32 BLE Beacon

## Description

This project is firmware for an ESP32 that works as a Bluetooth Low Energy (BLE) beacon.
The firmware sends a BLE advertising packet for 1 second, then enters deep sleep for 60 seconds.
After the sleep, the chip resets and repeats the cycle.
The short pulse and long sleep keep the average power low.
A small capacitor and a small organic photovoltaic (OPV) panel can then supply the beacon.

The beacon carries a 128-bit UUID that identifies a room.
A scanner reads the UUID and maps it to a known location.

## Code Information

Source files:

- `main/beacon_airqy.c` - main firmware. It configures BLE, starts advertising, waits 1 second, stops advertising, and enters deep sleep.
- `main/CMakeLists.txt` - ESP-IDF component build file.
- `CMakeLists.txt` - top-level ESP-IDF project build file.
- `sdkconfig` - saved ESP-IDF configuration. Bluetooth is enabled in this file.
- `uuid.md` - the four UUID values for beacon 1 to beacon 4.
- `circuito.md` - notes on power components (battery, charger module, voltage regulator, capacitor). Portuguese.
- `relatorio.md` - energy measurements and derived capacitor and panel sizes. Portuguese.

Build system: ESP-IDF (Espressif IoT Development Framework).
BLE stack: Bluedroid.
Target chip: ESP32.

### Advertising packet

The firmware sends the raw advertising data in `raw_adv_data[]`. The packet holds three fields:

- Flags (type `0x01`), value `0x06` - general discoverable mode, BR/EDR not supported.
- Complete list of 128-bit service UUIDs (type `0x07`) - one UUID, for example `4a215260-da59-42f6-b273-38fcde738f98`.
- TX power level (type `0x0a`), value `0x09` - +9 dBm.

The firmware stores the UUID bytes in little-endian order inside `raw_adv_data[]`.
One advertising packet holds at most 31 bytes.
To change the transmitted content, edit `raw_adv_data[]` and keep the 31-byte limit.

### Advertising parameters

`adv_params` sets the advertising behavior:

- Minimum and maximum interval: `0x20` (20 ms).
- Type: `ADV_TYPE_NONCONN_IND` - non-connectable, undirected.
- Address type: public.
- Channels: all three advertising channels.

### Program flow

1. `app_main()` starts NVS flash. If NVS has no free pages or a new version, it erases NVS and starts it again.
2. `app_main()` releases the memory of the classic Bluetooth mode.
3. `app_main()` starts the BT controller in BLE mode, then starts Bluedroid.
4. `app_main()` registers `gap_event_handler` as the GAP callback.
5. `app_main()` creates the FreeRTOS task `beacon_task`.
6. `beacon_task` calls `esp_ble_gap_config_adv_data_raw()` with `raw_adv_data[]`.
7. On event `ESP_GAP_BLE_ADV_DATA_RAW_SET_COMPLETE_EVT`, `gap_event_handler` calls `esp_ble_gap_start_advertising()`.
8. `beacon_task` waits 1 second with `vTaskDelay()`, then calls `esp_ble_gap_stop_advertising()` and deletes itself.
9. On event `ESP_GAP_BLE_ADV_STOP_COMPLETE_EVT`, `gap_event_handler` calls `esp_deep_sleep()` for 60 seconds.
10. The deep sleep ends with a chip reset, so the firmware runs from step 1 again.

The pulse length is `PULSO` (1 second). The sleep length is `SEGUNDOS` (60 seconds). Both are `#define` values at the top of `beacon_airqy.c`.

## Requirements

- ESP-IDF (latest stable release). The install guide is at <https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/>.
- An ESP32 board and a USB cable.
- A host with Linux, macOS, or Windows and a USB serial driver for the board.

ESP-IDF supplies the toolchain, FreeRTOS, the Bluedroid stack, and the `idf.py` build tool. The project needs no extra libraries.

## Usage Instructions

### Build

1. Install ESP-IDF and open a terminal with the ESP-IDF environment active.
2. Clone this repository and change into its directory.
3. Set the target: `idf.py set-target esp32`.
4. Open the configuration menu: `idf.py menuconfig`.
5. In the menu, open `Component config` > `Bluetooth` and enable `Bluetooth`.
6. Press `S` to save, then exit the menu.
7. Build the firmware: `idf.py build`.

Step 4 to step 6 are optional if you keep the `sdkconfig` file in this repository, because Bluetooth is already enabled there.

### Find the serial port on Linux

1. Before you connect the board, list the serial devices: `ls /dev/tty*`.
2. Connect the board and run `ls /dev/tty*` again.
3. The new entry is the board port, for example `/dev/ttyUSB0`.

### Flash and monitor

1. Run `idf.py -p /dev/ttyUSB0 flash monitor`. Replace `/dev/ttyUSB0` with your port.
2. `idf.py` builds the firmware, writes it to the ESP32, and opens the serial monitor.
3. The monitor shows the log lines from the beacon, such as `Advertising iniciado.` and `Advertising parado.`.
4. To leave the monitor, press `Ctrl` and `]`.

### Change the beacon identity

1. Open `main/beacon_airqy.c`.
2. Pick a UUID from `uuid.md`, or generate a new 128-bit UUID.
3. Write the 16 UUID bytes into `raw_adv_data[]` in little-endian order, after the `0x11, 0x07` header.
4. Rebuild and flash.

## Dataset Information

This repository holds firmware, not a dataset.

The file `relatorio.md` reports the energy study behind the 1-second pulse and 60-second sleep design.
The study covers three pulse and sleep pairs: 1 s / 60 s, 10 s / 10 s, and 60 s / 60 s.
It tests each pair at 5 V and at 3.3 V.
For each case it lists:

- energy per cycle in joules,
- the capacitor size in farads,
- the power the source must supply,
- the OPV panel area in cm².

The file `circuito.md` lists candidate power parts and shop links.

## Methodology

1. Run the ESP32 as a BLE beacon with a set pulse and sleep pair.
2. Measure the current and voltage on the bench for one full cycle.
3. Compute the energy per cycle from the measured current, the voltage, and the cycle time.
4. Compute the minimum capacitor size from the energy per cycle and the allowed voltage drop.
5. Compute the OPV panel area from the average power and an ambient-light irradiance estimate.
6. Repeat for each pulse and sleep pair and for both supply voltages.
7. Compare the results. The 1-second pulse with the 60-second sleep needs the least energy. It also needs only a sub-1 F capacitor and a panel of about 5 cm².

The measurements and the derived values are in `relatorio.md`.

## Reproduction script

There is no reproduction script.
The energy values come from bench measurements.
The capacitor and panel sizes come from hand calculations.
The Methodology section lists the formulas. The dataset is small enough to read and check by hand in `relatorio.md`.

## Citations

Add the paper reference here after publication.

## License and Contribution Guidelines

No license file is present. Contact the author before you reuse this code.

Author: Lucas Manoel Martins de Souza.

To contribute, open an issue or a pull request on the repository at <https://github.com/EnQyMo/Beacon>.
