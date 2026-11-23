# Local Telemetry Ingestion Pipeline

## Project Overview
This project bridges hardware microcontrollers and PC-side data analysis. An **ESP32** generates environmental telemetry (a simulated temperature reading) and streams it over USB serial. A standard Python 3 script on the PC, **`logger.py`**, listens on the serial port, stamps each reading with the PC's local time, and appends it to a persistent **CSV file** for long-term analysis.

### Why this architecture?
The ESP32's internal flash memory has a limited number of write cycles (**flash wear**) and only a few megabytes of storage. Logging directly to it every few seconds would physically wear out the chip within months. In this pipeline, the microcontroller acts only as a **collector**, and a more powerful machine handles **storage**.

```
┌──────────────┐   USB Serial (115200 baud)   ┌────────────────┐   append    ┌──────────────────┐
│    ESP32     │ ───────────────────────────> │   logger.py    │ ─────────>  │ sensor_data.csv  │
│ (MicroPython)│    "70.42\n" every 2 sec     │  (PC Python 3) │  + timestamp│ (Excel / SQL / ML)│
└──────────────┘                              └────────────────┘             └──────────────────┘
   Collector                                      Ingestion                        Storage
```

## Prerequisites
- Python 3.x installed on the host PC
- The `pyserial` library:
  ```bash
  pip install pyserial
  ```
- An ESP32 connected over USB, running a MicroPython script that `print()`s one numeric temperature value per line (for example, `70.42`)

## Execution
1. Make sure the ESP32 is plugged in and running its emitter script.
2. **Close Thonny, mpremote, or any other serial monitor.** Only one program can use a COM port at a time.
3. Open `logger.py` and set `COM_PORT` to match your system:
   - Windows: `COM3`, `COM16`, etc. (check with `Get-PnpDevice -Class Ports -PresentOnly` in PowerShell, or Device Manager)
   - Linux/Mac: `/dev/ttyUSB0`, `/dev/ttyACM0`, etc.
4. Run the logger:
   ```bash
   python logger.py
   ```
5. Press **Ctrl+C** to stop. The script closes the CSV file and serial port cleanly before exiting.

### Example terminal output
```
Connecting to ESP32 on COM16...
Serial connection established!
Waiting for temperature data...
Press Ctrl+C to stop.

Ignored: ESP32 Temperature Sensor Started
[2026-11-22 18:10:32] ESP32_Node_1: 70.42 F
[2026-11-22 18:10:34] ESP32_Node_1: 70.65 F
[2026-11-22 18:10:36] ESP32_Node_1: 70.86 F
```
Lines marked `Ignored:` are the ESP32's boot messages. Opening the serial port reboots the board, and anything that isn't a valid number is filtered out instead of being logged.

## CSV Output Format
Data is appended to `sensor_data.csv`. The header is written only when the file is new or empty, so data from previous runs is never overwritten.

| Column | Example | Description |
|--------|---------|-------------|
| `Timestamp` | `2026-11-22 18:10:32` | PC local time when the reading arrived |
| `Device_ID` | `ESP32_Node_1` | Which sensor node sent it |
| `Temperature_F` | `70.42` | The temperature reading in °F |

```
Timestamp,Device_ID,Temperature_F
2026-11-22 18:10:32,ESP32_Node_1,70.42
2026-11-22 18:10:34,ESP32_Node_1,70.65
```

## Error Handling & Graceful Termination
`logger.py` is built to run indefinitely and shut down safely:
- **`with` blocks** automatically close both the serial port and the CSV file however the script exits, which prevents a locked COM port or a corrupted file.
- **`file.flush()`** after every row writes data to disk immediately, so nothing is lost if the program stops unexpectedly.
- **`KeyboardInterrupt`** (Ctrl+C) is caught and reported cleanly instead of printing a crash traceback.
- **`serial.SerialException`** is caught when the port is missing, wrong, or already in use by another program.
- **`OSError`** is caught for file problems (for example, if the CSV is open and locked in Excel).
- **`finally`** always prints a shutdown confirmation.

## Configuration
These values are set at the top of `logger.py`:

| Variable | Default | Purpose |
|----------|---------|---------|
| `COM_PORT` | `"COM16"` | Serial port the ESP32 is connected to |
| `BAUD_RATE` | `115200` | Must match the ESP32 (MicroPython default) |
| `CSV_FILENAME` | `"sensor_data.csv"` | Output file |
| `DEVICE_ID` | `"ESP32_Node_1"` | Label written in each row |

## Repository Notes
- `sensor_data.csv` is excluded by `.gitignore`, since data files can grow very large and are generated output, not source code.
- The MicroPython emitter script runs on the ESP32 itself and is intentionally not included in this repository.

## Author
Myles ([TheHardwareKnight](https://github.com/TheHardwareKnight)) for the LCCC IoT Scripting & Automation course