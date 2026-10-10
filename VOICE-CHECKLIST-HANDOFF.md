# Voice Checklist Drill: handoff for Alpha Medic

This document describes a working, field-tested pattern for a training app where the user looks at a picture, says out loud what they would do or check, and the page listens, grades them against a checklist, and shows what they missed. It was built for a fire engineer's apparatus operational check (Glendale Fire Department, Engine 151) and is live at https://kwinter247.github.io/Engineer_Stuff/ under **Ops Check**. The same idea is to be reused in Alpha Medic for static cardiology: show a rhythm strip, the student verbalizes the treatment, the app grades it.

Everything here was learned from three real runs by a firefighter on an iPhone. The lessons in "What went wrong and how it was fixed" are the valuable part. The matcher code at the end is drop-in reusable.

---

## 1. What the app does

- A single HTML file, no framework, no build, no server. Speech recognition comes from the browser's Web Speech API (`webkitSpeechRecognition` on iPhone Safari, `SpeechRecognition` on Chrome). No API key, nothing leaves the phone except the audio Apple or Google already process for dictation.
- The user taps **Enable microphone** once. The phone asks to allow the mic. From then on the mic stays on for the whole drill.
- The drill is a sequence of **photos** (20 for the ops check). Under each photo is a running panel: a mic status pill, a counter of items called out so far, small chips showing progress on the sections that photo covers, and the last item heard.
- **One master checklist** (139 items in 20 sections) is scored for the entire run. The user can move between photos in any order with Next, Previous, or a Jump-to picker, and can go back to any photo to call out something missed. Nothing resets on navigation.
- Items are hidden by default so it works as a memory test. **Show the list** peeks at the current photo's sections with live ticks.
- A **Finish** slide after the last photo holds the **Complete** button. (Lesson: when Next itself triggered the report, users hit it expecting another photo.)
- **Complete** stops the mic and shows the **answer sheet**: an overall score strip, then every section with each item in green (called out) or red (missed), plus a Missed-only filter, Back to the photos (resumes with nothing lost), and Start over.
- On the answer sheet the user can **tap any item to flag it**: a red item they believe they said gets "I said this", a green one they didn't say gets "Didn't say this". **Send report** opens the phone share sheet; **Copy report** copies a plain-text report containing the flags, the misses, the full transcript of everything the phone heard, and a debug line (browser, mic restarts, last error). This report is what the user sends back to the developer, and it is the entire tuning loop.
- **Pause / Resume** and **Restart mic** buttons. A **What I heard** drawer shows the raw transcript live, with in-progress guesses in italics.

### Data shape

```js
// Sections of the checklist. One entry per section of the source document.
const SECTIONS = [
  {
    id: 'tires', title: 'Cold Lap · Tires', sub: 'Each tire and wheel assembly',
    items: [
      { label: 'Tread depth: at least 4/32″ front, 2/32″ rear',
        say: ['tread depth', 'tread gauge', '4 32', '2 32', 'tread'] },
      { label: 'Lug nuts tight, checked by hand',
        say: ['lug nuts', 'lug nut', 'lugnuts', 'nuts tight', 'by hand', 'tight'] },
      // ...
    ],
  },
  // ...
];

// Photos, in order. `sections` names the checklist sections this photo is about.
const SLIDES = [
  { image: 'ops-13.jpg', title: 'Tire and wheel', sub: 'Every tire', sections: ['tires'] },
  { image: 'ops-20.jpg', title: 'Park brake', sub: 'Air system and brake test',
    sections: ['air-setup', 'air-popout', 'air-recovery', 'brake-test', 'complete'] },
  { finish: true, title: 'Finish', sub: 'All photos done', sections: [] },
];
```

The item `label` is what the answer sheet prints and always counts as accepted wording. `say` is the list of extra phrasings that also count. Most of the tuning work is editing `say`.

---

## 2. How matching works

Plain JavaScript, no language model. Deterministic, instant, works offline once loaded. This is the right choice for a drill: the user needs the tick to land the moment they finish a sentence, and the developer needs to be able to reason about exactly why something did or did not tick.

### Normalization (`words()`)

1. Lowercase, strip apostrophes, keep letters, digits and the fraction characters used in the source document.
2. **Number words to digits** (`twelve` → `12`) so "twelve volts" and "12 volts" are the same.
3. **Alias table** for what the phone hears instead of the word that was said (`break` → `brake`, `stain` → `stang`, `ceiling` → `sealing`, `labrador` → `ladder`, `admission` → `ignition`, `ears` → `air`, `wheelchair` → `wheel chock`). An alias may expand to several words.
4. **Stop words** removed: the, a, is, and, my, check, make, sure, okay, good, and so on. So "check the chocks" and "chocks" compare equal.
5. **Crude stemming**: strip `ing`, `ed`, `es`, `s`, and `ies` → `y`, on words longer than three letters. "chocked", "chocks", "chock" all become "chock".

### Word comparison (`same()`)

Two stems match if equal, or if one is a prefix of the other and they differ by at most two characters. This absorbs stemming artifacts ("engag" vs "engage") without letting "high" match "highbeams" (the 2-character cap was added after exactly that false positive).

### Phrase matching (`phraseIn()`)

A phrase matches when all of its words appear, in any order, inside a sliding window of the transcript that is the phrase length plus two. So "chock wheels" matches "chock the wheels", "wheels are chocked", and "wheel chocks". It returns the transcript positions it used.

### Deciding which items tick (`match()`)

Each utterance is scored once, against the whole checklist. Every phrase of every unticked item that matches becomes a candidate. Then:

1. **Most specific phrase wins.** Candidates are sorted by phrase length, longest first. "Stow the wheel chocks" beats a bare "chocks" phrase on another item.
2. **The current photo breaks ties.** Items in the sections the current photo covers come first among equal lengths. The source document repeats some steps (the service brake is pumped in both the low-PSI test and the pop-out test); the first mention ticks the first occurrence, the next mention ticks the next, which follows the order of the procedure.
3. **Ambiguous single words stay on their own photo.** A one-word phrase whose word appears in more than one section (leaks, noises, damage, idle, lights) counts only when the current photo covers that item's section. Unique single words (ladder, antennas, dipstick) can be said from any photo. The set of ambiguous words is computed from the data at load time, so adding wording keeps it correct.
4. **Word consumption.** Once a candidate ticks, the transcript positions it used are consumed; later candidates that need those positions are skipped. This is what stops "air leaks" from also ticking a fluid-leaks item.
5. **Same thing said once.** A later candidate whose words are all already in the vocabulary of an item ticked by this utterance is skipped. This stops "system info 12 volts" from ticking the pre-start voltage check and the running voltage check at the same time.

### Only finished sentences are scored

The Web Speech API streams partial guesses while the user talks. Those partials are shown in the transcript drawer in italics but are never scored. A partial like "any leaks" can arrive before "air" is resolved and would tick the wrong item. Final results arrive at natural pauses, so ticks land about a second after the end of a sentence. Users accept this immediately.

### Required qualifiers for shared concepts

Where the checklist asks for the same thing in more than one place, each place needs its own distinguishing words, and the bare generic word is removed from `say`. Leaks is the worked example:

| Item | Must include |
| --- | --- |
| Leaks beneath the vehicle (walk-around) | under, beneath, underneath, or a fluid name (oil, coolant, hydraulic, foam, water), or puddles/drips |
| Listen for air leaks (walk-around) | air, hissing, or abnormal noise |
| Leaks under the pump (pump check) | under the pump, pump leaks, vibrations |
| New leaks on the hot-lap walk | new leaks, leaks or noises, leaks as you walk |

Rule of thumb: **a single generic word is only acceptable in `say` if that word belongs to exactly one item in the whole checklist.** Run the audit script (section 5) after every edit.

---

## 3. Hosting and the microphone

These cost real time and are not obvious.

- **claude.ai artifacts cannot use the microphone.** The artifact viewer sandboxes the page in a frame that never grants the mic, so the browser denies it without a prompt. The page itself was fine. Do not ship a mic feature on an artifact link.
- **The Claude app's in-app browser also blocks it.** Links opened inside the Claude app go to a web view with no mic. The user must open the link in Safari or Chrome proper.
- **Plain HTTPS static hosting works.** GitHub Pages was the fix: repo made public, Pages set to deploy from the branch, link is `https://<user>.github.io/<repo>/`. Every push redeploys the same link in a minute or two; nobody needs a new link. Free GitHub accounts only serve Pages from public repos.
- **The page needs its own document head.** The artifact host had been silently adding `<!DOCTYPE html>`, a viewport meta tag, and a rule making the `hidden` attribute work. On a raw static host those are missing, so the phone rendered at desktop width and hidden elements showed. Always write a complete document and include `[hidden] { display: none !important; }` because any `display: flex` rule otherwise overrides the attribute.
- **iPhone Safari specifics.** Speech recognition goes through Apple's dictation service, so Settings → General → Keyboard → Enable Dictation must be on, and Settings → Safari → Microphone must not be Deny. Safari stops recognizing after a pause and fires `onend`; the page restarts it automatically (250 ms). A muted source fires `audio-capture`; retry every 3 seconds rather than giving up. `rec.start()` must be called synchronously inside the tap handler the first time, or the permission prompt never appears.
- **Show the error code on screen.** A debug line with the recognizer name, number of restarts, the last error code and message, and the user agent turned "it's not working" into a one-screenshot diagnosis.

---

## 4. What went wrong and how it was fixed, run by run

**Run 1: 137/139.** Both misses were the phone mishearing jargon: "Stang gun" came through as "stain gun" every time, "stow the wheel chocks" as "stole the wheel trucks". Fix: aliases and `say` entries. Also harvested from the transcript the mishearings that only ticked by luck: "Q sir" for Q-siren, "hum" for Humat, "Dooley" for dually, "death levels" for DEF, "surface brake" for service brake, "optical" for Opticom, "Kevin" for captain. Found that several `say` phrases collapsed to one common word after stop-word removal ("brake in" → "brake", "has to activate" → "activate") and would have ticked on any mention of that word. Removed them.

**Run 2: 135/139, and a design complaint.** The user reported that checking for air leaks was also crediting the fluid-leaks-under-the-truck item. Two causes: partial speech guesses were being scored, and one-word phrases could reach items on any photo. Fixes: score final results only; confine single words to their photo; require qualifiers for leaks. The four misses were wording ("bumper is aligned", "10 as" for antennas, "brake does" for brake dust) plus one genuine miss.

**Run 3: 133/139, four flagged.** The single-word confinement was too blunt: "ladders are functional" said one photo early and "seatbelt latches" said from the cab door photo were blocked. Refined to confine only ambiguous single words (those used by more than one section), tagged the cab-door photos as interior, and let the most specific phrase win before photo priority so a mixed sentence ("AC vents clear, light bar undamaged") credits both items. "Ceiling to the rim" was the phone's version of "sealing to the rim", three times. Fixed with an alias.

**Interaction fixes along the way.** Finish slide so Next never surprises with the report. Pause button. Answer-sheet flagging and a sendable report, because screenshots of the sheet were not enough; the transcript is what makes tuning possible. Previous/Next moved directly under the photo.

### The tuning loop that works

1. User runs the drill and sends the report (flags + misses + transcript + debug line).
2. For each flagged miss, find the sentence in the transcript. It is almost always a mishearing of a domain word or a phrasing not in `say`.
3. Add the mishearing as an alias if it is a one-to-one word swap, or add the phrasing to `say`.
4. Run the audit script: no phrase may tick two items on its photo, every `say` phrase must tick its own item, and list every single-word phrase that other sections also use.
5. Test the exact sentences from the transcript against the matcher before pushing.

Expect three or four runs to converge. Jargon-heavy domains converge slower.

---

## 5. Reusable code

The complete matcher, as deployed. Depends only on `SECTIONS` (for `SHARED`) and the item list.

```js
const NUMBERS = { zero:'0', one:'1', two:'2', three:'3', four:'4', five:'5', six:'6', seven:'7', eight:'8', nine:'9', ten:'10',
  eleven:'11', twelve:'12', thirteen:'13', fourteen:'14', fifteen:'15', sixteen:'16', seventeen:'17', eighteen:'18',
  nineteen:'19', twenty:'20', thirty:'30', forty:'40', fifty:'50', hundred:'100' };
// what the phone tends to hear instead of the word that was said; a value may be several words
const ALIASES = { break:'brake', stain:'stang', ceiling:'sealing', labrador:'ladder', admission:'ignition', ears:'air', wheelchair:'wheel chock' };
const STOP = new Set(['the','a','an','is','are','and','to','of','my','our','its','it','on','in','at','with','for','that','this',
  'i','im','ive','we','have','has','be','been','got','get','make','sure','check','checked','okay','ok','good']);

function stem(w) { return w.length <= 3 ? w : w.replace(/ies$/, 'y').replace(/(ing|ed|es|s)$/, ''); }
function words(text) {
  return text.toLowerCase().replace(/['’]/g, '').replace(/[^a-z0-9¾½¼⅜⅛ ]+/g, ' ').split(/\s+/).filter(Boolean)
    .flatMap(w => (NUMBERS[w] || ALIASES[w] || w).split(' ')).filter(w => !STOP.has(w)).map(stem);
}
function same(a, b) {
  if (a === b) return true;
  if (a.length < 3 || b.length < 3 || Math.abs(a.length - b.length) > 2) return false;
  return a.startsWith(b) || b.startsWith(a);
}
// every phrase word inside a window of the transcript, any order; returns positions used or null
function phraseIn(phrase, spoken) {
  if (!phrase.length || spoken.length < phrase.length) return null;
  const span = phrase.length + 2;
  for (let i = 0; i + phrase.length <= spoken.length; i++) {
    const end = Math.min(spoken.length, i + span), pos = [];
    for (const p of phrase) {
      let k = -1;
      for (let j = i; j < end; j++) if (!pos.includes(j) && same(p, spoken[j])) { k = j; break; }
      if (k < 0) break;
      pos.push(k);
    }
    if (pos.length === phrase.length) return pos;
  }
  return null;
}
function compile(item) { return [item.label, ...(item.say || [])].map(words).filter(p => p.length); }

// ITEMS: flat list of every item with its section index; COMPILED: ITEMS.map(compile)
// SHARED: stems that appear in the wording of more than one section
const SHARED = (() => {
  const bySec = new Map();
  ITEMS.forEach((it, i) => COMPILED[i].flat().forEach(w => { if (!bySec.has(w)) bySec.set(w, new Set()); bySec.get(w).add(it.sec); }));
  return new Set([...bySec].filter(([w, secs]) => secs.size > 1).map(([w]) => w));
})();

// tick the items one finished utterance calls out. `already` = hits so far; `priority` = item indexes
// belonging to the current photo's sections
function match(items, compiled, spoken, already, priority) {
  const cands = [];
  items.forEach((_, i) => {
    if (already[i]) return;
    const pri = priority.has(i) ? 1 : 0;
    compiled[i].forEach(p => { const pos = phraseIn(p, spoken); if (pos) cands.push({ i, len: p.length, pos, pri }); });
  });
  cands.sort((a, b) => b.len - a.len || b.pri - a.pri || a.i - b.i);
  const scoped = cands.filter(c => c.pri || c.len >= 2 || !SHARED.has(spoken[c.pos[0]]));
  const used = new Set(), ticked = [], vocab = new Set();
  for (const c of scoped) {
    if (ticked.includes(c.i) || c.pos.some(x => used.has(x))) continue;
    if (ticked.length && c.pos.every(x => vocab.has(spoken[x]))) continue;
    c.pos.forEach(x => used.add(x));
    compiled[c.i].forEach(p => p.forEach(w => vocab.add(w)));
    ticked.push(c.i);
  }
  return ticked;
}
```

Recognizer setup that works on iPhone Safari and Android Chrome:

```js
const SR = window.SpeechRecognition || window.webkitSpeechRecognition;
const r = new SR();
r.lang = 'en-US'; r.continuous = true; r.interimResults = true; r.maxAlternatives = 3;
r.onresult = e => {
  for (let i = e.resultIndex; i < e.results.length; i++) {
    const res = e.results[i];
    if (res.isFinal) { transcript += ' ' + res[0].transcript; for (let k = 0; k < res.length; k++) score(res[k].transcript); }
    else interim = res[0].transcript;   // show it, never score it
  }
};
r.onend = () => { if (wantListening) setTimeout(() => r.start(), retryDelay); };   // phones stop at pauses
r.onerror = e => { if (e.error === 'audio-capture') retryDelay = 3000; /* not-allowed, service-not-allowed: tell the user */ };
// first r.start() must run synchronously inside the user's tap
```

Audit script to run after any data edit (Node, reads the page and evals the data and matcher):

- For every `say` phrase, simulate saying it on a photo that covers its section: it must tick its own item and nothing else.
- For every single-word `say` phrase, list the other sections whose vocabulary contains that word. Each one is a decision: keep (the word genuinely belongs to one place), qualify (make it two words), or remove.
- Feed every sentence from the latest real transcript through `match()` and confirm the expected item.

---

## 6. Blueprint for Alpha Medic static cardiology

Same engine, different data. Each rhythm strip is a slide; the checklist is the treatment for that rhythm.

### What changes

- **Slides are rhythm strips.** `SLIDES` entries carry the strip image and point at the treatment section for that rhythm. The student sees the strip and says the interpretation and treatment. Images can be swapped for better ones later; the data does not care.
- **One section per rhythm, items are the treatment steps.** Decide up front whether identification is an item ("this is ventricular fibrillation") and whether order matters. The engine does not enforce order; if order matters for a rhythm, that is a new feature (an item could carry `after: 'cpr'` and only tick once its predecessor has).
- **Per-strip scoring, not one running list.** The ops check is one continuous walk around one truck, so one master list fits. Cardiology is a set of independent cases, so score each strip on its own: when the student moves to the next strip, grade the one they left, and the answer sheet lists each rhythm with its hits and misses. Keep the mic on across strips; keep the Finish slide and the flag-and-send report exactly as they are.
- **Priority is per strip.** The current strip's treatment section is the priority set. Because drugs and actions repeat across rhythms (CPR, epinephrine, shock, oxygen), this matters more than in the ops check. With the current rules, a generic single word like "shock" would count only on the strip whose section is current, which is what you want.

### Expected mishearings to plan for

The phone will mangle drug names and doses the way it mangled "Stang gun". Seed aliases and `say` lists before the first run, then harvest the rest from transcripts:

- Amiodarone: "ammo during", "amio", "amiodarone 300", "300 of amio"
- Epinephrine: "epi", "epinephrine 1 milligram", "1 of epi", "epi every 3 to 5"
- Adenosine: "adenosine 6", "6 milligrams rapid push", "adenosine 12"
- Atropine: "atropine 1 milligram", "atropine point five" (write doses both ways: `1 mg`, `1 milligram`, `point 5`, `0.5`)
- Joules: "200 joules", "200 jewels", "biphasic 120 to 200"
- Synchronized cardioversion: "sync cardiovert", "synchronized cardioversion", "sync shock"
- Defibrillate: "defib", "shock", "unsynchronized shock"
- Transcutaneous pacing: "pace", "pacing", "TCP"
- "Milligrams" often arrives as "mg", "milligram", or just the number; "micrograms" as "mics" or "mcg". Put `mg`, `milligram`, `milligrams` in `say` for every dose item, and add `mcg`/`mics`/`micrograms` the same way.
- Numbers: the NUMBERS table only covers 0 to 20, 30, 40, 50, 100. Add the dose values and energy levels used (`120`, `150`, `200`, `300`, `360`, `0.5`). Decimal doses arrive as "point five" or "0.5"; the current normalizer turns "0.5" into "0 5", so write `say` entries as `0 5` and `point 5` or extend `words()` to keep decimals.

### Required qualifiers, cardiology version

Anything in more than one treatment needs its own words, same as leaks. Examples: "epi 1 milligram every 3 to 5 minutes" (arrest rhythms) versus "epi infusion 2 to 10 mics per minute" (bradycardia); "amiodarone 300 bolus" (VF/pulseless VT) versus "amiodarone 150 over 10 minutes" (stable wide-complex tachycardia); "shock" (VF) versus "synchronized cardioversion" (unstable tachycardia). Write the `say` lists so the bare drug name alone does not satisfy an item that has a specific dose or route.

### Data to provide

1. The rhythm images, one per strip, in the order they should appear.
2. For each rhythm: its name, and the treatment as a list of discrete items in the wording the student is expected to use. Mark any item where order matters.
3. Any abbreviations or slang the students actually say ("epi", "amio", "sync it"), so they go into `say` from day one.
4. After the first real run, the report from the answer sheet, which includes the transcript.

### Suggested first slice

Six rhythms is enough to prove it: normal sinus (no treatment, identification only), VF, pulseless VT, asystole, PEA, and symptomatic bradycardia. Each with 4 to 8 items. Run it on a phone, send the report, tune, then scale to the full set.
