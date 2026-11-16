# fw-nrf52840-ble-beacon-scanner

iBeacon transmitter and BLE scanner for the nRF52840 DK, switchable with a button. Zephyr RTOS, nRF Connect SDK.

## What it does

| Mode | Function |
|---|---|
| Beacon | Advertises Apple-format iBeacon packets (UUID, Major, Minor, TX power). The name `nRF52840_Beacon` is in the scan response. |
| Scanner | Passive scan of nearby BLE devices, printed over UART. iBeacons are flagged with their data. |

Controls:

| Button | Action |
|---|---|
| BTN1 | Switch between Beacon and Scanner |
| BTN2 | Scanner: start or stop. Beacon: increment Minor |
| BTN3 | Reset the counter and start a new scan |
| BTN4 | Print the current status |

| LED | Meaning |
|---|---|
| LED1 | blinks while running |
| LED2 | Beacon mode |
| LED3 | Scanner mode |
| LED4 | blinks when a device is found |

Scan settings: passive, interval 100 ms, window 50 ms, controller duplicate filter plus a software table of 64 unique addresses (cleared on each new scan).

Design: the scan callback runs on the BLE thread and only enqueues a message into a `K_MSGQ`. A dedicated print thread (priority 8) formats it and sends it over async UART with DMA, serialized by a `K_SEM` that signals TX done. Sending from the callback directly overwrote the shared buffer and garbled the output.

iBeacon payload: flags `0x06`, then Apple manufacturer data (`0x004C`), type `0x02`, length `0x15`, 16-byte UUID, Major, Minor, TX power.

## Hardware

| Item | Detail |
|---|---|
| Board | nRF52840 DK (PCA10056) |
| MCU | nRF52840, Cortex-M4F, 64 MHz, 1 MB flash, 256 KB RAM |
| UART | J-Link VCOM over USB, 115200 8N1, no flow control |
| Optional | phone with nRF Connect, or a second DK |

## Build

Requires nRF Connect SDK v3.x (tested with v3.0.2). Use a project path without spaces.

```bash
west build -b nrf52840dk/nrf52840 --no-sysbuild --pristine
```

Relevant `prj.conf` options: `CONFIG_BT_OBSERVER`, `CONFIG_BT_BROADCASTER`, `CONFIG_UART_ASYNC_API`, `CONFIG_DK_LIBRARY`. Logging goes to RTT and the UART is reserved for scanner output.

## Flash

```bash
west flash --runner jlink
```

## Test

Beacon mode:

1. The firmware starts in Beacon mode (LED2 on).
2. In nRF Connect on a phone, scan for `nRF52840_Beacon`. It appears as an iBeacon with UUID, Major, Minor and RSSI at 1 m.
3. Press BTN2 to increment Minor and watch it change in the app.

Scanner mode:

1. Press BTN1 (LED3 on) and open a serial terminal (115200 baud).
2. Nearby devices are listed as they are found, for example `[  3] 63:B0:6A:7D:82:B5 (rnd) RSSI:-37 dBm`.
3. Press BTN2 to stop or restart the scan and BTN4 for the status.

With two DKs, run one in Beacon mode and the other in Scanner mode: the second lists the first as an iBeacon.

## License

MIT, see [LICENSE](LICENSE).
