# Power-savings and keymap sync

## Why

Battery life on the Sofle halves was poor. The shared config left the RGB
underglow on permanently, kept the MCU out of deep sleep, and ran the BLE
radio at +8 dBm. Studio was also enabled on both halves via the shared conf
rather than just the left half that needs it.

Separately, the source `sofle.keymap` had drifted from the live keymap on
the device because edits made in ZMK Studio persist to the device's NVS
partition but are never written back to source.

## Changes

### `config/sofle.conf`

- **`CONFIG_ZMK_SLEEP=y`** + **`CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=900000`** — MCU
  now deep-sleeps after 15 minutes of inactivity. Biggest single win;
  drops idle current from ~mA to ~µA range.
- **`CONFIG_ZMK_EXT_POWER=y`** — allows the EXT_POWER GPIO on the nice!nano
  to gate the LED/display rail during sleep.
- **`CONFIG_ZMK_RGB_UNDERGLOW_AUTO_OFF_IDLE=y`** + **`_AUTO_OFF_USB=y`** —
  underglow shuts off when idle, and doesn't auto-light on USB.
- **`CONFIG_ZMK_RGB_UNDERGLOW_ON_START=n`** — underglow no longer comes on
  at boot. Toggle it on with the existing `RGB_TOG` keybind on Adjust.
- **Removed `CONFIG_BT_CTLR_TX_PWR_PLUS_8=y`** — radio drops from +8 dBm
  back to the default 0 dBm. Roughly 6× lower radio TX current. Re-add
  only if range becomes an issue.
- **Removed `CONFIG_ZMK_STUDIO=y`** from the shared conf. Studio is still
  enabled on the left half via the `cmake-args` in `build.yaml`; this just
  stops it being compiled into the right half where it's unused.

### `config/sofle.keymap`

Conservative sync to match the Studio state seen in the screenshots:

- Layer `display-name` values renamed: `MAC → Mac`, `RAISE → Symbols`,
  `LOWER → Function`, `ADJUST → Adjust`. `OPT` unchanged.
- **Function** layer: top row extended from F1–F11 to F1–F12 (the trailing
  `&trans` slot is now `&kp F12`).
- **Symbols** layer: added `&kp KP_NUM` to the right-half row-1 leading
  position (was `&trans`).

### Not changed (and why)

The Symbols (formerly RAISE) layer in Studio shows a meaningfully
reorganised numpad cluster on the right half, plus a few label oddities
on the left half ("KeypadBang", "KeypadAt") that don't cleanly map back
to source bindings from screenshots alone. Rather than guess, the
existing source bindings are kept and **only the unambiguous additions
(NUMLOCK, F12) are applied**. Review and adjust manually if needed —
the device's NVS overrides will keep the Studio version active in the
meantime.

ADJUST media-key positions in Studio differ slightly from source but the
behaviour is functionally equivalent; left as-is.

## Flashing notes

ZMK preserves the NVS settings partition across firmware flashes, so the
existing Studio edits on the device **will continue to override** the
source keymap after flashing this build. To force the source keymap to
take effect, flash the `settings_reset` artifact on each half after
flashing the new firmware. That wipes NVS and the device will boot with
the keymap from this repo.

## Battery percentage curve (optional follow-up)

This board is a SuperMini nRF52840 clone. The on-board battery-divider
footprints are unpopulated, but the nRF52840 is wired in VDDH mode and
the chip's internal `VDDHDIV5` SAADC channel reads battery voltage
directly. The `nice_nano_v2` board definition we build against already
configures the battery driver to use that channel, so **voltage
measurement is working**.

The percentage reported by ZMK is wrong because ZMK's default
voltage-to-percentage mapping is a single linear segment, and LiPo
cells don't discharge linearly. The result is the classic "looks 80%
forever then dies in an hour" behaviour.

[`patches/0001-lipo-curve.patch`](patches/0001-lipo-curve.patch) replaces
the linear segment with an 11-point piecewise-linear LiPo discharge
curve. To enable it:

1. Fork `zmkfirmware/zmk` to your GitHub account (one-click on the
   ZMK repo page).
2. Apply the patch on a branch named `lipo-curve`:
   ```
   git clone git@github.com:keeper-of-memes/zmk.git
   cd zmk
   git checkout -b lipo-curve
   git am /path/to/zmk-config/patches/0001-lipo-curve.patch
   git push -u origin lipo-curve
   ```
3. In `config/west.yml`, change the `zmk` project's remote from
   `zmkfirmware` to `keeper-of-memes` and its revision from `main` to
   `lipo-curve`.

GitHub Actions will then build against the patched ZMK and the OLED
battery widget (plus the BLE battery service the host sees) will report
a realistic percentage.

The `west.yml` in this PR adds the `keeper-of-memes` remote ready to be
flipped, but leaves the `zmk` project pointing at upstream so the build
keeps working until the fork exists.

## Expected impact

Before: idle current dominated by always-on RGB and active MCU; battery
life roughly days.

After: with RGB off-at-idle, deep sleep enabled, ext-power gating, and
0 dBm radio, expected idle current is two-to-three orders of magnitude
lower. Active typing battery life should improve to weeks.
