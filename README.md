# Fire Engineer Hydraulics Trainer

A single-page app for practicing pump discharge pressure (PDP) calculations, using Glendale Fire Department hydraulics values.

    PDP = NP + FL + AP ± EL

| Term | Meaning |
| ---- | ------- |
| NP | Nozzle pressure (psi) |
| FL | Friction loss in the hose (psi) |
| AP | Appliance loss (psi) |
| EL | Elevation pressure: 0.5 psi per foot of elevation (negative below grade), or 5 psi per floor above ground in a building |

The user enters every value for each line. The app totals the PDP live. Pressing **Enter** (or "Charge the line") grades each line against the answer worked from the Glendale sheet. A correct line charges: water fills the hose in the photo and the nozzle flows. A wrong line stays dry and the app shows **Incorrect**. After a correct answer the worked math is shown under the panel. A **Teach me** button shows the same worked math on demand, for review before or instead of attempting the evolution.

## Evolutions

| Evolution | Layout | NP | FL | AP | EL | PDP |
| --- | --- | -- | -- | -- | -- | --- |
| 1 | 150′ 1¾″ smooth bore | 50 | 57 | 0 | 0 | **107** |
| 2 | 200′ 1¾″ smooth bore | 50 | 76 | 0 | 0 | **126** |
| 3 · Line 1 | 150′ 1¾″ smooth bore | 50 | 57 | 0 | 0 | **107** |
| 3 · Line 2 | 200′ 1¾″ smooth bore | 50 | 76 | 0 | 0 | **126** |
| 4 | E159 supplying Ladder 152 through 450′ of 4″, 1⅜″ stack tip 100′ up | 80 | 22.5 | 40 | 50 | **192.5** |
| 5 | E151 relaying 450′ of 4″ to E152, which feeds Ladder 152 (1½″ tip, 600 gpm) | 0 | 33.75 | 20 | 0 | **53.75** |
| 6 · Deck gun | 2″ stack tip on E151, no hose, Stang gun | 80 | 0 | 25 | 0 | **105** |
| 6 · Handline | 200′ 1¾″ smooth bore | 50 | 76 | 0 | 0 | **126** |
| 7 · Deck gun | Fog master stream on E151, no hose, Stang gun | 100 | 0 | 25 | 0 | **125** |
| 7 · Line 1 | 150′ 1¾″ smooth bore | 50 | 57 | 0 | 0 | **107** |
| 7 · Line 2 | 200′ 2½″ into a Blitz, then 150′ 1¾″ smooth bore | 50 | 69 | 0 | 0 | **119** |
| 8 | Standpipe: 100′ 2½″ to the FDC, 150′ 1¾″ smooth bore on the 3rd floor | 50 | 63 | 0 | 10 | **123** |
| 9 | E150, 300′ 2½″ to a high pressure fog Blitz (500 gpm) | 100 | 120 | 0 | 0 | **220** |
| 10 | E156 supplying a ladder through two 100′ 4″ lines, 1¾″ stack tip (800 gpm) 100′ up | 80 | 3 | 40 | 50 | **173** |
| 11 | E157 relaying 300′ of 4″ to E153, whose deck gun flows a 1¾″ tip (800 gpm) | 0 | 42 | 20 | 0 | **62** |

Evolution 10: two supply lines split the 800 gpm to 400 each, so FL per 100′ is 4 × 3 ÷ 4 = 3 psi, added once.

Evolution 7, line 2: the same 160 gpm flows through both sections, so FL is 12 on the 2½″ ((160 × 1) ÷ 10 − 10 = 6 per 100′) plus 57 on the 1¾″. The Blitz is used as a gated wye, and a gated wye costs nothing under 350 gpm (10 psi over), so AP is 0.

On a multi-line evolution the app also asks for the **Engine PDP**, which must be the highest line's pressure (evolutions 3 and 6: 126; evolution 7: 125, the deck gun). The lower lines are gated down at their discharges.

Friction loss for a 1¾″ smooth bore handline at 160 gpm: (160 × 3) ÷ 10 − 10 = 38 psi per 100′.

Friction loss for 4″ supply at 500 gpm: Q = 5, so 5 × 4 ÷ 4 = 5 psi per 100′, and 22.5 psi for 450′. Entered values within half a psi of the exact answer are accepted, so 22 or 23 for FL and 192 or 193 for PDP all grade as correct.

## Reference values in the app

Encoded in the `REF` object in `index.html`, from the Glendale hydraulics sheet:

- Nozzle pressures and flows (1¾″ and 2½″ smooth bore handlines, master streams, penetrating nozzle, blitz nozzles).
- Rule of Eights stack tips at 80 psi: 1¼″ 400 gpm through 2″ 1000 gpm.
- Handline friction loss: FL/100′ = (GPM × HS) ÷ 10 − 10, with HS = 3 for 1¾″ and 1 for 2½″.
- Master stream / supply line friction loss: FL/100′ = Q × (Q − 1) ÷ HS, Q = gpm ÷ 100, HS = 4 for 4″. A line with `size: '4'` uses this formula automatically.
- Elevation by height: EL = feet of elevation × 0.5 psi, added above the pump and subtracted below.
- Elevation in a building (`floor: 3` on a line): 5 psi per floor above the ground floor, so the 3rd floor is 10 psi.
- Two supply lines to the same appliance (`supplyLines: 2` on a line) split the flow: FL is figured for one line at half the gpm and added once.
- More than one line flowing: the engine's PDP is the highest line's PDP; the rest gate down.
- Series hose (`hose: [{length, size}, …]` on a line): FL is figured per section at the line's gpm and summed.
- Deck gun (`noHose: true` on a line): no friction loss; NP from the tip plus the Stang gun appliance.
- Relay pumping (`relay: 'E152'` on a line): the next engine is in service, so the pump only feeds its intake. NP 0, FL on the line between the engines at the downstream flow, AP 20 pump to pump, EL 0.
- Appliance losses: gated wye 0 under 350 gpm and 10 over (a Blitz used as a wye follows the same rule), ladder 40, Stang gun 25, pump to pump 20; foam eductor operates at 200 psi.

The answers are computed from these tables, not hard-coded, so adding a line with a different length, hose size, nozzle or appliance only needs a new entry in `SCENARIOS`.

Evolution 11's scene is a composite built from the earlier photos (the deck gun engine from evolution 6 and the hydrant-fed engine from evolution 4) with the 4″ line and callouts drawn in, so the hose path is known exactly.

## Running it

Open `index.html` in a browser. There is no build step and no dependencies beyond Google Fonts. Keep the `scene-*.jpg` files next to it; each is a reference photo with the attack line recolored yellow and any spray removed, and the page draws the water and spray as SVG over it.

## Adding an evolution

Add an entry to `SCENARIOS` in `index.html`:

```js
{
  id: 'e4', title: 'Evolution 4', sub: '200′ 2½″ smooth bore', image: 'scene-e4.jpg',
  lines: [{
    name: 'Line 1', length: 200, size: '2.5', nozzle: 'sb-250', appliance: 'none', elevationFt: 0,   // feet above (+) or below (−) the pump
    path: 'M … L …',        // hose centerline in image pixels
    tip: [x, y],            // nozzle tip, where the spray starts
    mask: 'x,y x,y …',      // polygon hiding the water where the hose passes behind the firefighter
  }],
}
```

The hose path and mask were traced from the photo with a small OpenCV/scikit-image script (skeletonize the red hose pixels, walk the skeleton from the pump panel, simplify).
