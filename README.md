# zmk-trackpoint

ZMK module with the devicetree and Kconfig glue for a PS/2 TrackPoint
(e.g. IBM/Lenovo style stick) on an nRF52840 board. It wraps
[kb_zmk_ps2_mouse_trackpoint_driver](https://github.com/vrifmus/kb_zmk_ps2_mouse_trackpoint_driver),
which provides the actual drivers and bindings and **must be in your
`west.yml` as well**.

What the module contains:

| File | Purpose |
|------|---------|
| `dts/trackpoint/ps2.dtsi` | Overlay-time nodes: pinctrl, `&uart0`, `uart_ps2` / `gpio_ps2`, `mouse_ps2`, `trackpoint_listener` (all `disabled`) |
| `dts/trackpoint/irq_priorities_nrf52840.dtsi` | nRF52840 interrupt priority table (GPIOTE 0, everything else 3) |
| `dts/trackpoint/keymap.dtsi` | Keymap-time part: `U_MSS_TP_*` settings keys and `status = "okay"` for the nodes |
| `Kconfig` | `CONFIG_TRACKPOINT`, selects `INPUT` and `ZMK_POINTING` |

What stays in your own config: the wiring (pins), the layer numbers and the
static tuning values.

## Install

`config/west.yml`:

```yaml
manifest:
  remotes:
    - name: vrifmus
      url-base: https://github.com/vrifmus
  projects:
    - name: kb_zmk_ps2_mouse_trackpoint_driver
      remote: vrifmus
      revision: <commit>
    - name: zmk-trackpoint
      remote: vrifmus
      revision: <commit>
```

## Use

**1. Shield `.conf`**

```
CONFIG_TRACKPOINT=y
CONFIG_INPUT_INIT_PRIORITY=99
```

**2. Shield overlay** (e.g. `trackpoint.dtsi` included from `m13.overlay`):
define the wiring, then include the module.

```c
#define MOUSE_PS2_DRIVER_UART            // omit for the GPIO driver

#define MOUSE_PS2_PIN_SCL_PRO_MICRO <&pro_micro 1 GPIO_ACTIVE_HIGH>
#define MOUSE_PS2_PIN_SDA_PRO_MICRO <&pro_micro 0 GPIO_ACTIVE_HIGH>
#define MOUSE_PS2_PIN_SDA_PINCTRL   <NRF_PSEL(UART_RX, 0, 8)>
// #define MOUSE_PS2_PIN_RST_PRO_MICRO <&pro_micro 14 GPIO_ACTIVE_HIGH>
#define MOUSE_PS2_UART_BAUD_RATE    <14400>      // optional, default 14400
#define MOUSE_PS2_PIN_UNEXPOSED_TX  <NRF_PSEL(UART_TX, 0, 27)>
#define MOUSE_PS2_PIN_UNEXPOSED_RX  <NRF_PSEL(UART_RX, 0, 28)>

#define TRACKPOINT_SCROLL_LAYER 9        // optional, default 9

#include <trackpoint/ps2.dtsi>
```

Define `TRACKPOINT_NO_IRQ_PRIORITIES` before the include on a non-nRF52840 SoC.

**3. Keymap**

```c
#define HAS_MOUSE_TP
#ifdef HAS_MOUSE_TP
    #include <trackpoint/keymap.dtsi>
#endif
```

Without `HAS_MOUSE_TP` the nodes stay disabled and are not compiled in.
This gives you `U_MSS_TP_S_I/D`, `U_MSS_TP_V6_I/D`, `U_MSS_TP_NI_I/D`,
`U_MSS_TP_PT_I/D`, `U_MSS_RESET` and `U_MSS_LOG` to put on a layer.
Settings changed at runtime are saved to flash after
`CONFIG_ZMK_SETTINGS_SAVE_DEBOUNCE` (60s by default); wait that long before
resetting the board.

**4. Tuning** (optional, in your keymap, after the include):

```c
&mouse_ps2 {
    tp-sensitivity = <127>;        // default 128, max 255
    tp-val6-upper-speed = <97>;    // default 97, max 255
    // tp-neg-inertia = <6>;
    // tp-press-to-select;         // not every TP supports it
    // tp-press-to-select-threshold = <10>;
    sampling-rate = <200>;         // 10,20,40,60,80,100,200
    // disable-clicking;
    // scroll-mode;                // Intellimouse mice only
};
```

Axis swap/invert and the scroll-layer behaviour are `input-processors` on
`&trackpoint_listener` (defined in `ps2.dtsi`); override them from your keymap
if you need something different.

Values hardcoded on `&mouse_ps2` win over values saved in flash on every boot.
