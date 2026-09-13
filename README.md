# Fire Engineer Hydraulics Trainer

A single-page app for practicing pump discharge pressure (PDP) calculations, using Glendale Fire Department hydraulics values.

    PDP = NP + FL + AP ± EL

| Term | Meaning |
| ---- | ------- |
| NP | Nozzle pressure (psi) |
| FL | Friction loss in the hose (psi) |
| AP | Appliance loss (psi) |
| EL | Elevation pressure (psi, negative below grade) |

The user enters every value for each line. The app totals the PDP live. Pressing **Enter** (or "Charge the line") grades each line against the answer worked from the Glendale sheet. A correct line charges: water fills the hose in the photo and the nozzle flows. A wrong line stays dry and the app shows **Incorrect**. After a correct answer the worked math is shown under the panel.

## Evolutions

| Evolution | Layout | NP | FL | AP | EL | PDP |
| --- | --- | -- | -- | -- | -- | --- |
| 1 | 150′ 1¾″ smooth bore | 50 | 57 | 0 | 0 | **107** |
| 2 | 200′ 1¾″ smooth bore | 50 | 76 | 0 | 0 | **126** |
| 3 · Line 1 | 150′ 1¾″ smooth bore | 50 | 57 | 0 | 0 | **107** |
| 3 · Line 2 | 200′ 1¾″ smooth bore | 50 | 76 | 0 | 0 | **126** |

Friction loss for a 1¾″ smooth bore handline at 160 gpm: (160 × 3) ÷ 10 − 10 = 38 psi per 100′.

## Reference values in the app

Encoded in the `REF` object in `index.html`, from the Glendale hydraulics sheet:

- Nozzle pressures and flows (1¾″ and 2½″ smooth bore handlines, master streams, penetrating nozzle, blitz nozzles).
- Handline friction loss: FL/100′ = (GPM × HS) ÷ 10 − 10, with HS = 3 for 1¾″ and 1 for 2½″.
- Master stream / supply line friction loss: FL/100′ = Q × (Q − 1) ÷ HS, Q = gpm ÷ 100, HS = 4 for 4″.
- Appliance losses: gated wye 10, ladder 40, Stang gun 25, pump to pump 20; foam eductor operates at 200 psi.

The answers are computed from these tables, not hard-coded, so adding a line with a different length, hose size, nozzle or appliance only needs a new entry in `SCENARIOS`.

## Running it

Open `index.html` in a browser. There is no build step and no dependencies beyond Google Fonts. Keep the `scene-*.jpg` files next to it; each is a reference photo with the attack line recolored yellow and any spray removed, and the page draws the water and spray as SVG over it.

## Adding an evolution

Add an entry to `SCENARIOS` in `index.html`:

```js
{
  id: 'e4', title: 'Evolution 4', sub: '200′ 2½″ smooth bore', image: 'scene-e4.jpg',
  lines: [{
    name: 'Line 1', length: 200, size: '2.5', nozzle: 'sb-250', appliance: 'none', el: 0,
    path: 'M … L …',        // hose centerline in image pixels
    tip: [x, y],            // nozzle tip, where the spray starts
    mask: 'x,y x,y …',      // polygon hiding the water where the hose passes behind the firefighter
  }],
}
```

The hose path and mask were traced from the photo with a small OpenCV/scikit-image script (skeletonize the red hose pixels, walk the skeleton from the pump panel, simplify).
