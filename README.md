<!-- SPDX-License-Identifier: MIT -->

# nrf-ble-stack

A small, readable Bluetooth Low Energy stack for nRF52 SoCs, written in C and
driving the radio directly, without a SoftDevice.

It is an educational stack, small enough to read end to end. The public API,
controller, host, and radio access live in separate layers, so you can follow a
packet through the code from the radio interrupt up to your application
callback.

It runs as either a peripheral or a central and covers advertising, scanning,
connections, and ATT/GATT. It does not implement security, so don't use it where
a link needs encryption or pairing. See [Limitations](#limitations).

## Features

**Advertising and scanning**

- Legacy advertising with separately configured advertising and scan-response
  data: name, flags, TX power, service UUID lists, service data, and
  manufacturer-specific data
- Passive and active legacy scanning with scan report callbacks
- Scan filter that connects automatically on a matching address, name, or
  service UUID

**Connections**

- Central connection initiation and peripheral connection handling
- Connection parameter updates from either role, plus an optional delayed
  request from the peripheral when preferred parameters are configured
- Automatic feature exchange, data length update, and LE 1M/2M PHY update after
  a central connects
- Connection timing re-anchored to the central's packet address, for better
  interoperability with active centrals

**ATT and GATT**

- GATT server with services and characteristics registered by the application
- Notifications and indications
- GATT client: MTU exchange, service, characteristic, and descriptor discovery,
  reads, writes, and CCCD updates
- ATT MTU up to 247 bytes
- 16-bit SIG UUIDs, and vendor UUIDs expanded from one registered 128-bit base

## Getting started

### Requirements

- An nRF52840 Dongle (PCA10059), which all three examples are built for
- `make`
- An Arm GCC toolchain that includes newlib. The SDK's default is
  `gcc-arm-none-eabi-10.3-2021.10`. Homebrew's `arm-none-eabi-gcc` formula has
  no C library, and builds with it stop at a missing `stdint.h`.
- `nrfjprog` and an SWD debug probe to use `make flash`. The dongle has no
  on-board probe.

### Build

Fetch the nRF5 SDK submodule, then build an example:

```sh
git submodule update --init --recursive
make -C examples/peripheral_demo -j4
```

The output is written to `_build/` inside the example's directory.

The Makefiles find the compiler on your `PATH`. To use one that isn't on your
`PATH`, pass its `bin` directory, including the trailing slash:

```sh
make -C examples/peripheral_demo -j4 GNU_INSTALL_ROOT=/path/to/toolchain/bin/
```

### Flash

```sh
make -C examples/peripheral_demo flash
```

> [!WARNING]
> `make erase` runs `nrfjprog --eraseall`, which also removes the dongle's
> factory USB bootloader.

Each example also has a `generate_dfu_packet` target, which expects `nrfutil`
and your own signing key at `_build/private.pem`. The key is not part of this
repository.

### Examples

| Example | What it does |
| --- | --- |
| [`peripheral_demo`](examples/peripheral_demo) | Connectable GATT peripheral with a custom service. Sends periodic notifications or indications from a counter characteristic and also exposes a text characteristic. |
| [`central_demo`](examples/central_demo) | Scans for `peripheral_demo`, connects, discovers its notify characteristic, and logs the counter updates. |
| [`thermometer_adv_demo`](examples/thermometer_adv_demo) | Nonconnectable advertiser that broadcasts a temperature byte as Health Thermometer service data. |

To see a complete link, flash `peripheral_demo` and `central_demo` onto two
dongles. Every example logs over the dongle's USB serial port (CDC ACM).

## Using the stack

Include `nrf_ble.h`, then pick a role. This minimal advertiser is adapted from
`thermometer_adv_demo`:

```c
#include "app_error.h"
#include "nrf.h"
#include "nrf_ble.h"

static const ble_gap_adv_name_config_t m_adv_name = {
    .name_type = BLE_GAP_ADV_NAME_FULL,
};
static const ble_gap_adv_config_t m_adv_config = {
    .flags = (uint8_t)(BLE_GAP_ADV_FLAG_LE_GENERAL_DISC_MODE |
                       BLE_GAP_ADV_FLAG_BR_EDR_NOT_SUPPORTED),
    .interval_ms = 100U,
    .adv_type = BLE_GAP_ADV_TYPE_NONCONNECTABLE_NONSCANNABLE_UNDIRECTED,
    .adv_data = {
        .p_name = &m_adv_name,
    },
};

int main(void)
{
  clock_init(); // Start LFCLK and HFCLK; see the examples

  ble_stack_init(BLE_GAP_ROLE_PERIPHERAL);
  ble_gap_set_device_name("BLE-Demo");
  APP_ERROR_CHECK_BOOL(ble_gap_adv_init(&m_adv_config));
  ble_gap_start_advertising();

  while (1)
  {
    __WFE();
  }
}
```

The application starts the clocks itself. The examples use the SDK clock driver
for this.

### Public headers

| Header | Contents |
| --- | --- |
| [`nrf_ble.h`](stack/include/nrf_ble.h) | Umbrella header and `ble_stack_init()` |
| [`ble_gap.h`](stack/include/ble_gap.h) | Advertising, scanning, connections, and GAP events |
| [`ble_gatt_server.h`](stack/include/ble_gatt_server.h) | Service registration, notifications, and indications |
| [`ble_gatt_client.h`](stack/include/ble_gatt_client.h) | Discovery, reads, writes, and CCCD updates |
| [`ble_uuid.h`](stack/include/ble_uuid.h) | SIG and vendor UUIDs |
| [`ble_att.h`](stack/include/ble_att.h) | ATT MTU and value size limits |

### Events

Each kind of event has its own callback:

| Events | Register with |
| --- | --- |
| GAP (connected, disconnected, parameter and PHY updates, and so on) | `ble_gap_register_evt_handler()` |
| Scan reports | `ble_gap_register_scan_report_handler()` |
| GATT server (MTU exchange) | `ble_gatt_server_register_evt_handler()` |
| GATT client procedures | `ble_gatt_client_register_evt_handler()` |
| Characteristic writes and notification or indication state | The characteristic's `evt_handler` field |

Stack-level and characteristic callbacks are deferred to the low-priority
`SWI1_EGU1` interrupt, so they don't run in the radio interrupt path. To read a
written value, use `p_evt->p_characteristic->p_value` and `value_len`.

### UUIDs

SIG UUIDs are plain 16-bit values. Custom UUIDs are 16-bit values combined with
one stack-wide 128-bit base set by `ble_uuid_set_vendor_base()`. Application
tables stay compact, and the stack sends the full 128-bit UUID over the air.

## How it works

| Path | Layer | Responsibility |
| --- | --- | --- |
| `stack/core/` | Core | Entry points, runtime state, UUID helpers, and deferred event delivery |
| `stack/host/gap/` | Host | GAP-facing APIs |
| `stack/host/gatt/` | Host | ATT database, GATT server, and GATT client |
| `stack/host/l2cap/` | Host | Routes connection data to ATT or signaling handling |
| `stack/controller/` | Link layer | Advertising, scanning, connection timing, LL control, and retransmission |
| `stack/radio/` | Driver | The only layer that accesses `NRF_RADIO` directly |

A few design choices explain most of the code:

- **One exchange per interval.** Each connection interval handles one RX and one
  TX. A response generated from a received packet goes out in the next
  connection event, which keeps the code simple and limits throughput.
- **Bounded TX queue.** ATT responses, notifications, and signaling PDUs share a
  four-entry L2CAP queue. LL control traffic has dedicated slots and goes first.
- **Timing from the radio.** `TIMER0` schedules connection events, and the fixed
  PPI path from `RADIO ADDRESS` to `TIMER0 CAPTURE[1]` re-anchors them to actual
  on-air receive timing.
- **Best-effort advertising data.** Fields that don't fit in a packet are
  omitted. A complete service UUID list that doesn't fit is truncated and
  advertised as incomplete.

For role-specific flows and state machines, see
[`BLE_STACK_FLOWCHARTS.md`](BLE_STACK_FLOWCHARTS.md).

## Limitations

- No security, pairing, or bonding
- One role at a time, with no simultaneous peripheral and central operation
- One RX and one TX per connection interval
- No L2CAP fragmentation or reassembly
- No long writes or prepare/execute writes
- Legacy advertising only
- nRF52 only, with no nRF51 support. The examples build only for the nRF52840,
  so other nRF52 parts need their own build setup.
- `TIMER0` is reserved by the controller, along with `RADIO` and `SWI1_EGU1`

## License

MIT. See [LICENSE](LICENSE).
