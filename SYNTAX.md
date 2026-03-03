# tscircuit syntax primer

## 1) Root element

A circuit is typically a default export that returns a `<board />` or a form-factor board component from `@tscircuit/common`.

Example:

```tsx
import React from "react"

export default () => (
  <board width="10mm" height="10mm">
    <resistor name="R1" resistance="1k" footprint="0402" />
  </board>
)
```

## 2) Layout properties

You can place nearly any element with:
- `pcbX`, `pcbY` (PCB position)
- `pcbRotation`
- `layer` (e.g., `"bottom"`)

For schematics:
- `schX`, `schY`
- `schRotation`
- `schOrientation`

Units
- Numbers are interpreted as mm.
- Strings can include units (e.g., `"0.1in"`, `"2.54mm"`).

## 3) Pin labels with `pinLabels`

Use `pinLabels` to map physical pin numbers to meaningful names. This is essential for chips and ICs.

Basic syntax (single label per pin):

```tsx
<chip
  name="U1"
  footprint="soic8"
  pinLabels={{
    pin1: "VCC",
    pin2: "GND",
    pin3: "IN",
    pin4: "OUT",
  }}
/>
```

Multi-alias syntax (multiple names for a pin):

```tsx
<chip
  name="U1"
  footprint="qfp32"
  pinLabels={{
    pin1: ["GP0", "SPI0_RX", "I2C0_SDA", "UART0_TX"],
    pin2: ["GP1", "SPI0_CS", "I2C0_SCL", "UART0_RX"],
    pin3: "GND",
  }}
/>
```

With multi-alias, any of the names can be used in traces:

```tsx
<trace from="U1.GP0" to="U2.SDA" />
<trace from="U1.SPI0_RX" to="U3.MISO" />  // Same pin, different alias
```

## 4) Pin attributes with `pinAttributes`

Use `pinAttributes` to add semantic metadata to pins. This enables DRC checks, schematic arrows, and board-level pinout exposure.

Example using a 555 timer (NE555):

```tsx
<chip
  name="U1"
  footprint="dip8"
  pinLabels={{
    pin1: "GND",
    pin2: "TRIG",
    pin3: "OUT",
    pin4: "RESET",
    pin5: "CTRL",
    pin6: "THRES",
    pin7: "DISCH",
    pin8: "VCC",
  }}
  pinAttributes={{
    VCC: { requiresPower: true },
    RESET: { mustBeConnected: true },
  }}
/>
```

Available attributes:

| Attribute | Type | Description |
|-----------|------|-------------|
| `requiresPower` | boolean | Signal goes INTO the chip (shows input arrow on schematic) |
| `providesPower` | boolean | Signal comes OUT of the chip (shows output arrow on schematic) |
| `mustBeConnected` | boolean | DRC error if this pin is left floating |
| `includeInBoardPinout` | boolean | Expose this pin to the board-level pinout |

Bidirectional pins (like I2C data lines) can have both requiresPower and providesPower:

```tsx
pinAttributes={{
  SDA: { requiresPower: true, providesPower: true },
  SCL: { requiresPower: true, providesPower: true },
}}
```

## 5) Type-safe chip components

For reusable chip components, define `pinLabels` as a const and use `ChipProps` for type safety:

```tsx
import type { ChipProps } from "tscircuit"

const pinLabels = {
  pin1: "VCC",
  pin2: "GND",
  pin3: ["SDA", "I2C_DATA"],
  pin4: ["SCL", "I2C_CLK"],
} as const

export const MyChip = (props: ChipProps<typeof pinLabels>) => (
  <chip
    {...props}
    pinLabels={pinLabels}
    footprint="soic4"
  />
)
```

This provides autocomplete and type checking when using the component:

```tsx
<MyChip name="U1" />
<trace from="U1.SDA" to="U2.pin1" />
```

## 6) Connectivity with `<trace />`

Connect pins with port selectors:

```tsx
<trace from="R1.pin1" to="C1.pin1" />
```

Connect to named nets. Net names must start with a letter or underscore and can only contain letters, numbers and underscores. 

```tsx
<trace from="U1.pin1" to="net.GND" />
<trace from="U1.pin8" to="net.VCC" />
```

When using `pinLabels`, reference pins by their label:

```tsx
<trace from="U1.VCC" to="net.V3_3" />
<trace from="U1.GND" to="net.GND" />
<trace from="U1.SDA" to="U2.I2C_DATA" />
```

Pin labels (in `pinLabels`) can contain letters, numbers, and underscores. Unlike net names, pin labels **can** start with a number (e.g., `"3V3"` is valid).

Useful trace props (optional)
- `width` / `thickness`
- `minLength` / `maxLength`

## 7) Grouping for PCB layout

Use `<group />` like a container to move/layout parts together.

```tsx
<board width="20mm" height="20mm">
  <group pcbX={5} pcbY={5}>
    <resistor name="R1" resistance="1k" footprint="0402" pcbX={2.5} pcbY={2.5} />
    <resistor name="R2" resistance="1k" footprint="0402" pcbX={2.5} pcbY={0} />
    <resistor name="R3" resistance="1k" footprint="0402" pcbX={2.5} pcbY={-2.5} />
  </group>
</board>
```

## 8) Schematic pin arrangement

Control how pins appear on the schematic symbol with `schPinArrangement`:

```tsx
<chip
  name="U1"
  pinLabels={{
    pin1: "VIN",
    pin2: "GND",
    pin3: "VOUT",
    pin4: "EN",
  }}
  schPinArrangement={{
    leftSide: { pins: ["VIN", "GND"], direction: "top-to-bottom" },
    rightSide: { pins: ["VOUT", "EN"], direction: "top-to-bottom" },
  }}
/>
```

Available sides: `leftSide`, `rightSide`, `topSide`, `bottomSide`

## 9) Autorouter choices

Boards and subcircuits can set an `autorouter` preset (e.g., `"auto"`, `"sequential-trace"`, `"auto-local"`, `"auto-cloud"`). For complex routing, cloud autorouting is often the most capable.

## 10) Manufacturing helpers

For turnkey assembly you will often want:
- `supplierPartNumbers` (pin a specific supplier SKU/part number)
- `doNotPlace` (exclude from automated placement)

Example:

```tsx
<capacitor
  name="C1"
  capacitance="100nF"
  footprint="0402"
  supplierPartNumbers={{ jlcpcb: "C14663" }}
/>

<resistor name="R1" resistance="10k" footprint="0402" doNotPlace />
```
