# ESPHome firmware

Firmware to drive two AiP33628 drivers, plus a config that puts
the panel in Home Assistant as a single light.

Mapping, scan timing and the current budget are in
`../../docs/display-map.md`. Every entity and action this exposes is in
`../../docs/home-assistant.md`.

## Start here

I recommend you build and flash the firmware onto the ESP32 before you wire anything to the clock board.

An ESP32 connected via a USB cable with nothing attached to it will still boot, join
WiFi and turn up in Home Assistant. Then when you connect the six wires the panel just
lights up.

1. Install ESPHome and fill in `secrets.yaml`.
2. `esphome run clock.yaml` over USB, from this directory.
3. Adopt the device in Home Assistant.
4. Build the clock board, `../../docs/wiring.md`.
5. Wire the six lines to the ESP32 and power it up.

Use the firmware in `../esp32/panel-test/` as it drives the panel with no WiFi and no Home Assistant, so if the display misbehaves it tells you whether the hardware is right before you start suspecting this
component.

## Requirements

ESPHome 2026.8.0 or newer, on Python 3.11 or newer.

Install it into a virtual environment. Recent Linux distributions and Homebrew
refuse a plain `pip install` into the system Python, and a user install puts
the `esphome` command somewhere that is often not on PATH.

```
python3 -m venv venv
source venv/bin/activate
pip install --upgrade esphome
```

On Windows, create it with `py -m venv venv` and activate with
`venv\Scripts\activate`. Every `esphome` command below assumes the environment
is active, and you activate it again in each new terminal.

## Which ESP32

I used a plain **ESP32-WROOM-32** devkit, the common 30 or 38 pin board that
sells for a few dollars. Anything in that family works unchanged: WROOM-32,
WROVER, DevKitC, NodeMCU-32S. If the listing says ESP32 with no letter after
it, that is the one. You will need a board that can be powered directly from 5V on the USB port on the clock, and the WROOM-32 board has a voltage regulator on board for this.

The four display pins have to sit below GPIO32. A frame goes out as a single
store to the low GPIO output register, and that register only reaches GPIO0
through GPIO31. `__init__.py` checks this and refuses to build if you pick a
higher pin.

### ESP32-C6 Super Mini

This folder targets the ESP32-C6 Super Mini and ships the chip fix already
applied. The C6 declares the GPIO output registers as unions, so `send_pair_()`
writes through `.val`:

```cpp
GPIO.out_w1ts = set;      // classic ESP32
GPIO.out_w1ts.val = set;  // C3, C6, S2, S3
```

The C6 Super Mini also has a different pin set: no GPIO10/11 exist at all, and
GPIO4 through GPIO9 are strapping or flash pins, so the wiring table below uses
safe pins (14, 18, 19, 20 for the display, 2 and 3 for the buttons). All of
them are below GPIO25, so the single-store frame send in `send_pair_()` still
works unchanged.

Config validation will not warn you if you pick a strapping pin, but the first
sign of trouble is a compile error inside `aip33628.cpp` or a board that
misbehaves at boot.

### ESP8266

Nope, this requires an ESP32. The scan leans on `gptimer` and on that single-store frame send, so porting it is a real project rather than a config change.

## Setup

```
cp secrets.yaml.example secrets.yaml
```

On Windows that is `copy` instead of `cp`.

Fill in the WiFi credentials, timezone and API key the way the comments in
that file describe.

The timezone in `secrets.yaml` is a fallback for a boot with no Home Assistant.
Change it to yours.

Home Assistant pushes its own timezone on every time sync and that path wins,
but only while the `homeassistant` time platform has no timezone of its own.
Do not add one there.

## Build and flash

```
esphome run clock.yaml
```

USB flashing on a WROOM-32 devkit may need to hold down BOOT while trying to program. Auto-reset into the bootloader does not work on every board.

Once it is on the network, updates go over the air, and the API carries the log stream:

```
esphome run clock.yaml --device wifi-clock.local
esphome logs clock.yaml --device wifi-clock.local
```

Two things to know before pushing an OTA build:

- **A bad build is not recoverable remotely.** If it fails to bring up WiFi
  the only way back is a USB cable, which means opening the case. Compile
  before uploading. The fallback access point in `clock.yaml` is the safety
  net, so look for its setup SSID before assuming a flash is dead.
- **Do not open the serial port for about a minute after an OTA.** Opening it
  asserts DTR, which resets the board before ESPHome marks the new partition
  valid, and the device rolls back to the previous image with no error
  anywhere.

## Adopt in Home Assistant

Home Assistant finds the device on its own. Look under Settings > Devices and
Services for a discovered ESPHome node.

It asks for an encryption key. That is `api_key` out of your `secrets.yaml`.

Do this before you wire anything. Until the device is adopted the clock shows
the time and ignores everything else, because the light, the switches and
every effect entity live on the Home Assistant side.

## Wiring

Six wires into the empty DIP-16 MCU footprint. Four for the display, two for
the buttons, plus a ground.

Full build notes are in `../../docs/wiring.md`.

| ESP32-C6 Super Mini | Input | Net |
| --- | --- | --- |
| GPIO14 |  | CLK, driver 1 |
| GPIO18 |  | DATA, driver 1 |
| GPIO19 |  | CLK_1, driver 2 |
| GPIO20 |  | DATA_1, driver 2 |
| GPIO2  |  | S1, top button |
| GPIO3  |  | S2, bottom button |
| GND |  | GND |
## Configuration

```yaml
aip33628:
  id: panel
  clk_pin: GPIO14
  data_pin: GPIO18
  clk2_pin: GPIO19
  data2_pin: GPIO20
  time_id: ha_time
  twelve_hour: true
  colon_mode: Blink  # Off, On, or Blink
  max_current: 15
```

| Key | Default | What |
| --- | --- | --- |
| `clk_pin`, `data_pin` | required | Driver 1 bus, below GPIO32 |
| `clk2_pin`, `data2_pin` | required | Driver 2 bus, below GPIO32 |
| `time_id` | none | A time source. Without one the panel shows dashes. |
| `max_current` | 15 | Ceiling on `IS[3:0]`, 0 to 15 |
| `twelve_hour` | true | 12h or 24h. Power-on default; HA switch controls it |
| `colon_mode` | Blink | `Off`, `On`, or `Blink`; HA select controls it |

The light platform takes an `aip33628_id` and is otherwise a normal RGB light. Its first-boot default is on at 50%; saved light state is restored. The HA `Colon Brightness` and `AM Brightness` numbers adjust those indicators independently of the digits.

## What it does

- Shows the time
- Colon mode is selectable as off, steady on, or blinking; colon and AM
  indicator brightness are independently adjustable. The upper colon dot drops
  while the network is down.
- Four dashes while there is no valid time. The board has no RTC, so a cold
  boot with no network means no time at all until the network comes back.
- One RGB "light" entity for the whole display, plus per digit color, per LED
  color, gradients, a color cycle and a flash effect.
- Temporary number displays light the degree mark when their unit is C or F.
- A lamp test lights every populated LED white for three seconds, then restores
  the previous display settings.

## Color resolution

Each COM slot is split into four binary weighted sub-frames, which gives 16
duty cycle levels per channel and 4096 colors.

The AiP33628 has no grayscale engine, so every one of those levels costs a
sub-frame. More levels would mean more sub-frames per COM pair, a shorter tick
and a narrower window. Four is where the flicker margin still looks
comfortable at 416Hz.

## Current

`max_current` is a fixed ceiling on `IS[3:0]`, 0 to 15. The Home Assistant
brightness slider maps onto that range and nothing else touches it.

It deliberately does not depend on what is on screen. Counting lit sinks to
hold a milliamp budget gives you a panel that dims itself once a second as the
colon blinks, gets brighter when you pick a saturated color because fewer
sinks are lit, and stalls part way through a fade.

The stock firmware never did this either. It held `IS` at 0xD across colon on
and colon off, and ran 0xF with a white digit lit, drawing 404mA against a
250mA label.

If the drivers run hotter than you want, lower `max_current`. That trades
brightness for heat without making the brightness depend on the content.

## Brightness

Sixteen current steps, spread across the whole slider.

Gamma correction lives in the component, and `gamma_correct` on the light is
set to 1.0 so it is not applied twice.

It has to live there because a plain gamma curve assumes the output can reach
zero, and this panel bottoms out at `IS` 0, which is 2.5mA. With a stock curve
the bottom 29 percent of the slider sits clamped against that floor doing
nothing. The component interpolates perceived output between the floor and
full instead, which uses all sixteen steps and leaves only the natural width
of one step at the bottom.

Color components go through the same exponent before they are rounded onto
duty cycle levels, because duty is linear light and Home Assistant sends gamma
encoded values. Skipping that step rounds every pastel up toward a saturated
color, most visibly by turning pink into white.

Both arrive linear from the light platform. The magnitude comes from
`current_values_as_brightness()` rather than `current_values.get_brightness()`,
because only the helper carries the transition state, without which a fade
never moves.

## Scan timing

The scan runs from a `gptimer` interrupt, not from `esp_timer`.

The esp_timer task dispatch path runs at task priority on core 0 next to the
WiFi task, which preempts it and stretches whichever COM slot is lit at the
time. A full duty slot rides that out. A 40us sub-frame does not, and the same
jitter reads as uneven digits and a visible pulse on any color that is not
saturated.
