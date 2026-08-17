# Corne v4.1 Vial — Master (left half) freezes / USB host resets

## Final conclusion: **EMI / PCB design flaw**, NOT firmware

### Root cause identified via upstream issue tracker

- `foostan/crkbd` **issue #291** ("Random Disconnection with TRS Cable") is a duplicate of **issue #265** ("usb connection issues", label `fix/pcb`, **open**, 167 comments).
- Owner `foostan` confirmed in #291 comment 2826524082 that the issue is **electromagnetic interference (EMI)** and pointed to #265 as the canonical thread.
- Owner also confirmed in #265 comment 2408884204 that the PCB design has problems and is being reviewed for a v4.2 revision; in comment 3577231778 he engaged with `colinski8189` who produced a re-routed PCB (`github.com/colinski8189/crkbd-emi-fix`) that foostan is now adopting.

### Why this is not fixable in firmware

1. Multiple users reproduced the issue with QMK, Vial, KMK, and stock foostan firmware — same crashes. So it's not keymap- or firmware-specific.
2. `dahmwern` swapped in a different v4.1 PCB and reproduced the same behavior → eliminates firmware corruption as a cause.
3. `xkonni` reproduced it on multiple machines and operating systems (Ubuntu 22.04, Arch 6.11.x) — so it's not OS-specific.
4. `veikolippand` (issue #291) reports phones in proximity trigger it; `joostwestra` (issue #265) reports the closest half crashes when phone is within 20 cm. This is classic EMI behavior.
5. Owner `foostan` (issue #265, comment 2408884204) explicitly attributes it to a PCB design defect: USB differential pair routing on the v4.1 board has inadequate signal-integrity, missing TVS diodes, and problematic USB_VBUS detection (the GP13 pin on the v4.1 PCB is NOT actually wired to VBUS, despite QMK assuming it is).
6. `michaelfranzl` (issue #265, latest comment 3769690490 from 2026-01-19) confirmed: connecting both halves via separate USB cables (no TRRS) fixed it — proving the PCB is the source, not the user's environment.

### Three hardware-only mitigations (community-verified)

**1. Faraday shield mod (cheapest, fastest, highest reported success)**
- Cover the underside of each PCB around the RP2040 with **kapton (non-conductive) tape**, then **copper (conductive) tape** over it. Ground the copper to a switch pin's GND pad.
- Multiple users (dahmwern, joostwestra, diegogfcvv) report this **eliminates disconnects in normal use**.
- Procedure: https://github.com/foostan/crkbd/issues/265 — see comment 3331036121 (joostwestra) for an image of where to apply it.

**2. EMI-fixed PCB re-route**
- `colinski8189/crkbd-emi-fix` provides reworked gerbers + BOM + CPL.
- Main changes per the README and discussion:
  - USB DP/DM traces re-routed on a single layer for proper impedance matching.
  - TVS diodes (USBLC6-2P6) added near USB connector.
  - 27-ohm series resistors on USB_DP/USB_DM removed (controversial; datasheet says they should be there).
  - Proper VBUS sensing via 5.6k + 10k voltage divider instead of the diode-OR trick.
- foostan is reviewing this for v4.2.

**3. Run both halves via separate USB cables (no TRRS)**
- Last-resort workaround. Eliminates EMI coupling via the TRRS path.

### Why the firmware changes I made earlier don't fully solve this

The applied changes were:
- `#undef OLED_ENABLE`
- `#undef USB_VBUS_PIN`
- `#define ENCODER_MAP_KEY_DELAY 0`
- Removed `QK_BOOTLOADER` from layer 3.

These address **main-thread stalls and split-detection noise** (causes A–E in the previous version of this doc). They reduce the surface area for EMI-induced glitches to manifest as a full wedge, but they cannot stop the EMI itself. EMI on USB_DP/USB_DM causes the host to see a malformed packet; if the firmware is mid-task when it happens, the host's reset storm looks exactly like the dmesg evidence here. Even a fully idle, perfectly running firmware will produce the same dmesg if the EMI event is strong enough to corrupt the USB PHY at the electrical layer.

## What the user must do (not code)

1. **Apply the Faraday shield mod** (kapton + copper tape on the underside of each PCB, grounded to a switch pin). This is the recommended path.
2. **Move phones, BLE peripherals, wifi routers away** from the keyboard.
3. **Use a different USB port and cable** to rule out the host side.
4. If still broken after the shield mod, source an EMI-fixed PCB (`colinski8189/crkbd-emi-fix` or wait for foostan v4.2).

## Firmware changes that DID help (kept, do not revert)

These changes address different root causes and are still valuable even with EMI:

- `#undef OLED_ENABLE` — removes I2C init that can stall on a board without OLED wired.
- `#undef USB_VBUS_PIN` — disables `SPLIT_USB_DETECT` which toggles on the miswired GP13.
- `#define ENCODER_MAP_KEY_DELAY 0` — removes `wait_ms` inside encoder callback chain.
- Removed `QK_BOOTLOADER` from layer 3 — prevents accidental bootloader entry.

## Updated context

- Same hardware, same firmware base.
- User now reports: **left half (master) stops working, right half OK**.
- New dmesg evidence (current boot, fresh): USB host repeatedly resets the device, sometimes giving up and re-enumerating.
- Master is the left half on crkbd (USB only goes there).

## Raw dmesg (current boot)

```
[  274.899043] usb 1-4.3: reset full-speed USB device number 9
[  332.828435] usb 1-4.3: USB disconnect, device number 9
[  333.313607] usb 1-4.3: new full-speed USB device number 10 ... 4653:0004 Corne v4
[  334.742165] usb 1-4.3: reset full-speed USB device number 10
[  334.951180] usb 1-4.3: reset full-speed USB device number 10
[  336.668431] usb 1-4.3: USB disconnect, device number 10
[  337.145956] usb 1-4.3: new full-speed USB device number 11 ... 4653:0004 Corne v4
[  892.081615] usb 1-4.3: reset full-speed USB device number 11
[  892.293608] usb 1-4.3: reset full-speed USB device number 11
[ 2441.185279] usb 1-4.3: reset full-speed USB device number 11
[ 2441.396353] usb 1-4.3: reset full-speed USB device number 11
[ 2441.618413] usb 1-4.3: reset full-speed USB device number 11
[ 2553.970924] usb 1-4.3: reset full-speed USB device number 11
[ 2554.180977] usb 1-4.3: reset full-speed USB device number 11
[ 2554.404872] usb 1-4.3: reset full-speed USB device number 11
[ 2808.986287] usb 1-4.3: reset full-speed USB device number 11
[ 2809.194294] usb 1-4.3: reset full-speed USB device number 11
[ 2809.416319] usb 1-4.3: reset full-speed USB device number 11
[ 2968.379456] usb 1-4.3: reset full-speed USB device number 11
[ 2968.469251] usb 1-4.3: Device not responding to setup address.
[ 2968.672174] usb 1-4.3: Device not responding to setup address.
[ 2968.880045] usb 1-4.3: device not accepting address 11, error -71
[ 2968.880299] usb 1-4.3: WARN: invalid context state for evaluate context command.
[ 2968.968074] usb 1-4.3: reset full-speed USB device number 11
[ 2969.187455] usb 1-4.3: reset full-speed USB device number 11
[ 2972.957977] usb 1-4.3: USB disconnect, device number 11
[ 2973.392094] usb 1-4.3: new full-speed USB device number 12 ... 4653:0004 Corne v4
[ 3271.008993] usb 1-4.3: reset full-speed USB device number 12
[ 3272.256992] usb 1-4.3: reset full-speed USB device number 12
[ 3272.467125] usb 1-4.3: reset full-speed USB device number 12
[ 3272.687999] usb 1-4.3: reset full-speed USB device number 12
[ 3490.831772] usb 1-4.3: reset full-speed USB device number 12
[ 3491.043847] usb 1-4.3: reset full-speed USB device number 12
[ 3491.263778] usb 1-4.3: reset full-speed USB device number 12
```

Pattern: long healthy stretches (e.g. 337s–892s ≈ 9 min, 892s–2441s ≈ 26 min) punctuated by **catastrophic wedges** where the host resets the device many times in rapid succession, sometimes declaring it dead with `-71 / Device not responding to setup address`. Then re-enumeration succeeds and it works again.

`Device not responding to setup address` is the smoking gun: the firmware **never responded** to a USB control transfer (SETUP packet). USB polls are IRQ-driven on RP2040, so even a fully blocked main thread should still service setup packets. So this is **not** just a long busy-wait.

## Possible causes

### E. Catastrophic firmware hang (interrupt deadlock or hard fault)

The only way to wedge the USB IRQ on ChibiOS RP2040 is:
- A higher-priority interrupt that never returns.
- A NVIC hard fault (e.g., NULL deref, bad stack).
- `chSysHalt()` reached during ISR.

On RP2040, NVIC priorities are inverted relative to ARM: **lower numeric = higher priority**. Default ChibiOS priorities: USB at priority low (e.g. 3), system tick at 2, higher-priority peripherals at 1 or 0.

Suspect: **DMA completion ISR for the ws2812 vendor driver** (`RP_DMA_PRIORITY_WS2812 = 3` in `src/vial-kb/vial-qmk/platforms/chibios/drivers/vendor/RP/RP2040/ws2812_vendor.c:29`). If for some reason the DMA fires while the system tick handler is processing, and `chSemSignalI` blocks, the main loop would stall. The kernel tick and USB IRQ are at lower priority, but DMA completion has nothing to do with USB polling, so it shouldn't hang USB.

More likely: **USB ISR runs and tries to call into a Vial/VIA raw_hid path that hits a fault**. The user's `layer_state_set_user` is called from `layer_state_set`, which is called from `process_record` via action handlers — only from main thread. Not from IRQ.

### F. I2C OLED init hang

`config.h` defines `I2C_DRIVER I2CD1` with `I2C1_SDA_PIN GP6`, `I2C1_SCL_PIN GP7`. The keyboard json enables OLED. **No OLED module is initialized in the keymap**, but the `oled_task_user`/init paths may still attempt to talk to a non-existent OLED. If the I2C bus hangs (no pullups, no device) with default ChibiOS timeouts of several seconds, the main thread will block for ~that long every matrix scan iteration. Not a full USB wedge.

Less likely to be the cause of `-71 error` unless I2C has a `chSysHalt` on timeout. Check the ChibiOS I2C driver for that.

### G. Encoder + RGB blocking cascade

`src/vial-kb/vial-qmk/quantum/encoder.c:38–46`:

```c
action_exec(clockwise ? MAKE_ENCODER_CW_EVENT(index, true) : MAKE_ENCODER_CCW_EVENT(index, true));
wait_ms(ENCODER_MAP_KEY_DELAY);    // busy wait
action_exec(clockwise ? MAKE_ENCODER_CW_EVENT(index, false) : MAKE_ENCODER_CCW_EVENT(index, false));
wait_ms(ENCODER_MAP_KEY_DELAY);
```

User has `ENCODER_MAP_ENABLE = yes` and 4 encoders mapped to RGB_MOD / RGB_HUI / RGB_VAI / RGB_SAI etc. Each `action_exec(RGB_MOD)` runs `process_record` → `rgb_matrix_step()` → `eeconfig_flag_rgb_matrix()` and sets `rgb_task_state = STARTING`. **The next matrix scan** runs `rgb_task_render()` which fills the buffer and calls `ws2812_flush()` → `sync_ws2812_transfer()` which does a **busy-wait up to 23 ms** (`WS2812_LED_COUNT` ms timeout).

So one encoder detent = ~10–30 ms main-thread blocking for the WS2812 PIO DMA. **Rotating encoder fast = sustained main thread stall** = long enough for USB to miss polls.

BUT — USB IRQ is higher priority than main thread. Even with main thread stalled, USB polls should be answered. So this doesn't explain the full wedge. **Unless** the WS2812 PIO DMA ISR is at the same priority as USB and runs to completion during USB IRQ entry, blocking USB. That'd require priority inversion.

### H. SPLIT_USB_DETECT noise from miswired USB_VBUS_PIN GP13

Same as before. v4.1 board's GP13 isn't actually wired to VBUS, but QMK thinks it is, so `SPLIT_USB_DETECT` toggles based on noise. Could the master crash as a side effect? Probably not directly, but combined with main-thread stalls could cause `usb_vbus_present()` to flap, leading to repeated transport re-init. Not a `-71` cause.

### I. Hard fault from a peripheral conflict (most likely on v4.1)

v4.1 has these pin assignments on the left half:

```jsonc
"matrix_pins": {
    "direct": [
        ["GP22", "GP20", "GP23", "GP26", "GP29", "GP0",  "GP4"],
        ["GP19", "GP18", "GP24", "GP27", "GP1",  "GP2",  "GP8"],
        ["GP17", "GP16", "GP25", "GP28", "GP3",  "GP9",   null],
        [  null,   null,   null, "GP14", "GP15", "GP11",   null]
    ]
},
"encoder": {
    "rotary": [
        {"pin_a": "GP5", "pin_b": "GP7"},
        {"pin_a": "GP6", "pin_b": "GP7"}
    ]
}
```

Plus `I2C1_SDA_PIN GP6`, `I2C1_SCL_PIN GP7`. **GP6 and GP7 are simultaneously the encoder pin A2 (and B) on left, AND the I2C OLED bus.**

RP2040 GPIO has a 5-or-so function multiplexer, but you can only have one function per pin at a time. If QMK tries to:
- init I2C1 → claims GP6/GP7 as I2C pads.
- encoder_driver_init() → reads GP6/GP7 as GPIO inputs via `palSetLineMode(input)` which would conflict with the I2C alt function.

If i2c pins are mid-transaction and the encoder driver sets them as GPIO, **the I2C peripheral can hang waiting for the line to release**. Conversely, if I2C starts and the encoder is reading raw GPIO without re-init, the encoder might read garbage (no effect on master).

Most importantly: the OLED task is **disabled in the keymap but the OLED feature flag is enabled**. The OLED driver code may try to talk to a non-existent SSD1306. The ChibiOS I2C driver default timeout = `TIME_INFINITE` or very long. This is plausible.

To verify: read the OLED enabled code path; see if it ever blocks indefinitely.

## Recommendations (NOT applied)

The fix that gives most ROI without hardware changes:

1. **Disable OLED in this keymap** — it isn't used in the keymap code, but the feature flag is on. Add to `keymap/config.h`:
   ```c
   #undef OLED_ENABLE
   ```
   This removes I2C1 init and the per-task OLED read attempts. (May not actually undefine the feature flag if the json sets it; verify after rebuild.)

2. **Lower encoder blocking** — set in `keymap/config.h`:
   ```c
   #define ENCODER_MAP_KEY_DELAY 0
   ```
   to remove the busy-wait inside the encoder queue handler.

4. **Remove `QK_BOOTLOADER` from layer 3** — accidental activation is rare but possible. Replace with a 5-tap combo or a hidden layer.

3. **Disable split USB detection** in `keymap/config.h`:
   ```c
   #undef USB_VBUS_PIN
   #undef SPLIT_USB_DETECT
   ```
   This makes the master not check VBUS at all and assume the slave is always present.

4. **Reduce matrix scan frequency / RGB flush limit** — in keymap/config.h:
   ```c
   #define RGB_MATRIX_LED_FLUSH_LIMIT 16
   ```
   to spread ws2812 transfers across more scan iterations (default 16 — already low). Or reduce `WS2812_TRST_US` if set explicitly.

If after all that it still wedges, the cause is likely **hardware**: flaky solder on the USB connector, or a flaky reset/supercap on the RP2040, or actual power issues. Test:
- Plug the keyboard into a different port (not through a hub).
- Try a different cable.
- Add `#define FIRMWARE_RESET_WITHOUT_BOOTLOADER` if needed.

## Files involved

- `keyboards/crkbd/vial-kb/vial-qmk/keymaps/vial/keymap.c` — has `QK_BOOTLOADER` at [3][0] and `layer_state_set_user` calling `rgb_matrix_sethsv_noeeprom` + `rgb_matrix_mode_noeeprom`.
- `keyboards/crkbd/vial-kb/vial-qmk/keymaps/vial/rules.mk` — `VIA_ENABLE, VIAL_ENABLE, VIALRGB_ENABLE, ENCODER_MAP_ENABLE = yes`.
- `keyboards/crkbd/vial-kb/vial-qmk/keymaps/vial/config.h` — UID, unlock combo, has `RP2040_BOOTLOADER_DOUBLE_TAP_RESET` undef from previous round.
- `keyboards/crkbd/qmk/qmk_firmware/rev4_1/config.h` — `USB_VBUS_PIN GP13` (miswired on v4.1 hardware), `RP2040_BOOTLOADER_DOUBLE_TAP_RESET`, `PICO_XOSC_STARTUP_DELAY_MULTIPLIER 64`, `SERIAL_USART_TX_PIN GP12`, `I2C1_SDA_PIN GP6`, `I2C1_SCL_PIN GP7`.
- `keyboards/crkbd/qmk/qmk_firmware/rev4_1/standard/keyboard.json` — OLED enabled, RGB matrix enabled, encoder enabled. GP6/GP7 used for both encoder and I2C.

## Build command

```
kb=crkbd kr=rev4_1/standard km=vial make vial-qmk-compile
```

Output: `keyboards/crkbd/vial-kb/vial-qmk/.build/crkbd_rev4_1_standard_vial.uf2`