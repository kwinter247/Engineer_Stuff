# Fire Engineer Hydraulics Trainer

A single-page app for practicing pump discharge pressure (PDP) calculations.

    PDP = NP + FL + EL + AP

| Term | Meaning |
| ---- | ------- |
| NP | Nozzle pressure (psi) |
| FL | Friction loss in the hose (psi) |
| EL | Elevation pressure (psi, may be negative below grade) |
| AP | Appliance loss (psi) |

The user enters all four values. The app totals the PDP live. Pressing **Enter** (or "Charge the line") checks the values against the answer key for the evolution. A correct answer charges the line and the nozzle flows water in the scene. A wrong answer shows an **Incorrect** banner.

## Current evolution

Engine 150 at a hydrant, 150′ of 1¾″ to a smooth bore nozzle.

| NP | FL | EL | AP | PDP |
| -- | -- | -- | -- | --- |
| 50 | 57 | 0 | 0 | 107 |

## Running it

Open `index.html` in a browser. There is no build step and no dependencies beyond Google Fonts.

## Adding evolutions

The answer key lives in the `scenario` object near the bottom of `index.html`:

```js
const scenario = {
  id: 'glendale-e150-150ft-175-smoothbore',
  hose: '150′ of 1¾″',
  nozzle: 'Smooth bore',
  answers: { np: 50, fl: 57, el: 0, ap: 0 }
};
```

Replace it, or swap in a loader for an uploaded file, to grade other layouts.
