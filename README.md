# Engine 150 Trainer

Two phone-friendly drills behind one link. `index.html` is the menu.

| Page | Drill |
| --- | --- |
| `hydraulics.html` | Pump discharge pressure evolutions (below). |
| `opscheck.html` | Ops Check: walk the truck photo by photo, call out the operational check, the page listens and grades the whole thing. |

## Ops Check

Twenty photos of Engine 151 walk through the Glendale Fire Department Engineer Operational Check (Rev 10/2024). Tapping **Enable microphone** starts the browser's speech recognition (the phone asks to allow the microphone once) and it stays on for the whole run. One running checklist of every item on the check is scored the entire time: the user moves through the photos in any order with Next, Previous or the **Jump to** picker, calls out what they would check, and can go back to any photo to pick up something they missed. The panel shows how many items have been called out overall and for the sections the current photo covers, and names the last item heard. **Show the list** peeks at the current photo's sections with live ticks. **Complete** stops the microphone and shows the answer sheet: every section with its items in green (called out) or red (missed), a **Missed only** filter, **Back to the photos** to keep going, and **Start over**. **What I heard** shows the raw transcript, which is useful for tuning the accepted wording.

The checklist is the `SECTIONS` array in `opscheck.html`, one entry per section of the document, and the photos are the `SLIDES` array:

```js
{ image: 'ops-13.jpg', title: 'Tire and wheel', sub: 'Every tire', sections: ['tires'] }
```

`sections` names the checklist sections a photo is about. While that photo is up, its sections get first claim on whatever is said, and if an utterance fits one of them it counts only there. Anything else on the checklist can still be called out from any photo. Where the document repeats a step (the service brake is pumped in both the low PSI test and the pop-out test), the first mention ticks the first occurrence and the next mention ticks the next.

Each item's `say` lists extra wording that counts; the label itself always counts. Matching drops filler words (the, my, is, check…) and word endings, and accepts the phrase's words in any order within a short window, so "chock the wheels", "wheels are chocked" and "wheel chocks" all match `chock wheels`. Number words become digits ("four inch" matches "4 inch"). When an utterance fits more than one item, the longest matching phrase claims its words first, and a second item whose wording is fully covered by one just ticked is treated as the same thing said once.

Speech recognition works in Chrome on Android and Safari on iPhone, and needs the page served over HTTPS (GitHub Pages is fine). Firefox does not support it. The `ops-*.jpg` photos are resized to 1600 px on the long side.

# Hydraulics Trainer

A single-page app for practicing pump discharge pressure (PDP) calculations, using Glendale Fire Department hydraulics values.

    PDP = NP + FL + AP ± EL

| Term | Meaning |
| ---- | ------- |
| NP | Nozzle pressure (psi) |
| FL | Friction loss in the hose (psi) |
| AP | Appliance loss (psi) |
| EL | Elevation pressure: 0.5 psi per foot of elevation (negative below grade), or 5 psi per floor above ground in a building |

The user enters every value for each line. The app totals the PDP live. Pressing **Enter** (or "Charge the line") grades each line against the answer worked from the Glendale sheet. A correct line charges: water fills the hose in the photo and the nozzle flows. A wrong line stays dry and the app shows **Incorrect**. After a correct answer the worked math is shown under the panel. A **Teach me** button shows the same worked math on demand, for review before or instead of attempting the evolution. A correct answer plays a rushing-water sound; a wrong one plays an alarm and a flashing alert. Sounds are synthesized in the page and played through audio elements, so they work with a phone's mute switch on; the speaker button in the header silences them.

## Evolutions

| Evolution | Layout | NP | FL | AP | EL | PDP |
| --- | --- | -- | -- | -- | -- | --- |
| 1 | 150′ 1¾″ smooth bore | 50 | 57 | 0 | 0 | **107** |
| 2 | 200′ 1¾″ smooth bore | 50 | 76 | 0 | 0 | **126** |
| 3 · Line 1 | 150′ 1¾″ smooth bore | 50 | 57 | 0 | 0 | **107** |
| 3 · Line 2 | 200′ 1¾″ smooth bore | 50 | 76 | 0 | 0 | **126** |
| 4 | E159 supplying Ladder 152 through 450′ of 4″, 1⅜″ stack tip 100′ up | 80 | 22.5 | 40 | 50 | **192.5** |
| 5 | E151 relaying 450′ + 50′ of 4″ to E152, which feeds a ladder (1½″ tip, 600 gpm) | 0 | 37.5 | 20 | 0 | **57.5** |
| 6 · Deck gun | 2″ stack tip on E151, no hose, Stang gun | 80 | 0 | 25 | 0 | **105** |
| 6 · Handline | 200′ 1¾″ smooth bore | 50 | 76 | 0 | 0 | **126** |
| 7 · Deck gun | Fog master stream on E151, no hose, Stang gun | 100 | 0 | 25 | 0 | **125** |
| 7 · Line 1 | 150′ 1¾″ smooth bore | 50 | 57 | 0 | 0 | **107** |
| 7 · Line 2 | 200′ 2½″ into a Blitz, then 150′ 1¾″ smooth bore | 50 | 69 | 0 | 0 | **119** |
| 8 | Standpipe: 100′ 2½″ to the FDC, 150′ 1¾″ smooth bore on the 3rd floor | 50 | 63 | 0 | 10 | **123** |
| 9 | E150, 300′ 2½″ to a high pressure fog Blitz (500 gpm) | 100 | 120 | 0 | 0 | **220** |
| 10 | E156 supplying a ladder through two 100′ 4″ lines, 1¾″ stack tip (800 gpm) 100′ up | 80 | 3 | 40 | 50 | **173** |
| 11 · Line 1 | 350′ 1¾″ to a penetrating nozzle (100 psi, 95 gpm) | 100 | 64.75 | 0 | 0 | **164.75** |
| 11 · Wye line | 200′ 2½″ to a gated wye, two 150′ 1¾″ smooth bores (320 gpm through the wye) | 50 | 101 | 0 | 0 | **151** |
| 12 | E154: 400′ 4″ up 30′ to a Blitz with low pressure fog (55 psi, 500 gpm) | 55 | 20 | 0 | 15 | **90** |
| 13 | E151 supplying a ladder through two 100′ 4″ lines, fog master stream (1000 gpm) 60′ up | 100 | 5 | 40 | 30 | **175** |

Evolution 11: the wye trunk carries both branches, 2 × 160 = 320 gpm, so the 2½″ loses (320 × 1) ÷ 10 − 10 = 22 per 100′, 44 for 200′; one 150′ branch adds 57. The wye is under 350 gpm, so no appliance loss. Engine PDP is the penetrating nozzle line, 164.75 (165 grades correct).

Evolution 10: two supply lines split the 800 gpm to 400 each, so FL per 100′ is 4 × 3 ÷ 4 = 3 psi, added once.

Evolution 7, line 2: the same 160 gpm flows through both sections, so FL is 12 on the 2½″ ((160 × 1) ÷ 10 − 10 = 6 per 100′) plus 57 on the 1¾″. The Blitz is used as a gated wye, and a gated wye costs nothing under 350 gpm (10 psi over), so AP is 0.

On a multi-line evolution the app also asks for the **Engine PDP**, which must be the highest line's pressure (evolutions 3 and 6: 126; evolution 7: 125, the deck gun; evolution 11: 164.75). The lower lines are gated down at their discharges.

Friction loss for a 1¾″ smooth bore handline at 160 gpm: (160 × 3) ÷ 10 − 10 = 38 psi per 100′.

Friction loss for 4″ supply at 500 gpm: Q = 5, so 5 × 4 ÷ 4 = 5 psi per 100′, and 22.5 psi for 450′. Entered values within half a psi of the exact answer are accepted, so 22 or 23 for FL and 192 or 193 for PDP all grade as correct.

## Reference values in the app

Encoded in the `REF` object in `hydraulics.html`, from the Glendale hydraulics sheet:

- Nozzle pressures and flows (1¾″ and 2½″ smooth bore handlines, master streams, penetrating nozzle, blitz nozzles).
- Rule of Eights stack tips at 80 psi: 1¼″ 400 gpm through 2″ 1000 gpm.
- Handline friction loss: FL/100′ = (GPM × HS) ÷ 10 − 10, with HS = 3 for 1¾″ and 1 for 2½″.
- Master stream / supply line friction loss: FL/100′ = Q × (Q − 1) ÷ HS, Q = gpm ÷ 100, HS = 4 for 4″. A line with `size: '4'` uses this formula automatically.
- Elevation by height: EL = feet of elevation × 0.5 psi, added above the pump and subtracted below.
- Elevation in a building (`floor: 3` on a line): 5 psi per floor above the ground floor, so the 3rd floor is 10 psi.
- Two supply lines to the same appliance (`supplyLines: 2` on a line) split the flow: FL is figured for one line at half the gpm and added once.
- More than one line flowing: the engine's PDP is the highest line's PDP; the rest gate down.
- Series hose (`hose: [{length, size}, …]` on a line): FL is figured per section at the line's gpm and summed.
- Gated wye (`branches: 2` with the trunk section marked `trunk: true`): the trunk carries branches × the nozzle flow, FL is added for one branch, and the wye's 350 gpm threshold is checked against the trunk flow.
- Deck gun (`noHose: true` on a line): no friction loss; NP from the tip plus the Stang gun appliance.
- Relay pumping (`relay: 'E152'` on a line): the next engine is in service, so the pump only feeds its intake. NP 0, FL on the line between the engines at the downstream flow, AP 20 pump to pump, EL 0.
- Appliance losses: gated wye 0 under 350 gpm and 10 over (a Blitz used as a wye follows the same rule), ladder 40, Stang gun 25, pump to pump 20; foam eductor operates at 200 psi.

The answers are computed from these tables, not hard-coded, so adding a line with a different length, hose size, nozzle or appliance only needs a new entry in `SCENARIOS`.

## Running it

Open `index.html` in a browser and pick a drill. There is no build step and no dependencies beyond Google Fonts. Keep the `scene-*.jpg` files next to it; each is a reference photo with the attack line recolored yellow and any spray removed, and the page draws the water and spray as SVG over it.

## Adding an evolution

Add an entry to `SCENARIOS` in `hydraulics.html`:

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
