# Corne (crkbd) Firmware Build — Vial

## Overview

This markdown documents the steps to build a custom Vial-enabled QMK firmware
for the **Corne / crkbd** keyboard (rev4_1, standard layout) using the
[kbd_firmware](https://github.com/foostan/kbd_firmware) repository by foostan.

The custom keymap lives in `keymap.c` at the repo root of `kbd_firmware`
(under `keyboards/crkbd/vial-kb/vial-qmk/keymaps/vial/` once initialized).

---

## Prerequisites

1. Set up the QMK environment — follow steps 1–3 of the
   [QMK Getting Started guide](https://docs.qmk.fm/#/newbs_getting_started).
   This installs the `qmk` CLI tool.
2. Clone the kbd_firmware repo:

```sh
git clone https://github.com/foostan/kbd_firmware.git
cd kbd_firmware
```

3. Drop your custom `keymap.c` into:

```
keyboards/crkbd/qmk/qmk_firmware/keymaps/vial/
```

(or the equivalent path used by kbd_firmware after `vial-qmk-init`).

---

## Build Commands (from the kbd_firmware repo root)

The Makefile in `kbd_firmware` exposes these targets:

| Target               | Purpose                                              |
|----------------------|------------------------------------------------------|
| `git-submodule`      | Fetch QMK + Vial-QMK git submodules                  |
| `vial-qmk-clean`     | Clean previous Vial build artifacts                  |
| `vial-qmk-init`      | Symlink `keyboards/<kb>` into `src/.../keyboards/tmp`|
| `vial-qmk-compile`   | Compile the Vial firmware                            |
| `vial-qmk-flash`     | Compile + flash to a connected bootloader            |

### Full build pipeline

```sh
make git-submodule
make vial-qmk-clean
kb=crkbd make vial-qmk-init
kb=crkbd kr=rev4_1/standard km=vial make vial-qmk-compile
```

### One-shot (the command you ran)

```sh
cd kbd_firmware \
  && kb=crkbd kr=rev4_1/standard km=vial make vial-qmk-compile 2>&1 | tail -10
```

Output artifacts land at:

```
keyboards/crkbd/vial-kb/vial-qmk/.build/crkbd_rev4_1_standard_vial.hex
keyboards/crkbd/vial-kb/vial-qmk/.build/crkbd_rev4_1_standard_vial.uf2
```

---

## Variables

| Var   | Meaning                                  | Example             |
|-------|------------------------------------------|---------------------|
| `kb`  | Keyboard directory under `keyboards/`    | `crkbd`             |
| `kr`  | Revision / variant                       | `rev4_1/standard`   |
| `km`  | Keymap name                              | `vial`              |

---

## References

- Upstream: <https://github.com/foostan/kbd_firmware>
- QMK docs:  <https://docs.qmk.fm/>
- Vial:      <https://get.vial.today/>
