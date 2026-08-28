# College One-Pagers — Project README

A single self-contained HTML file that renders one printable page per college
from an embedded JSON dataset, plus a screen-only table of contents, a mobile
layout, and a plain-text export.

**Deliverable:** `college-onepagers.html` (~178 KB, no build step, no
dependencies except Google Fonts). Open it directly in a browser; it works
offline apart from the webfonts and the campus photos.

---

## 1. Purpose and audience

Built for a student in **Cherry Hill, NJ** applying for **Fall 2027 entry**
(Class of 2031). The list is shaped around a specific set of priorities that
recur throughout the data:

- pre-law strength (the most developed dimension in the schema)
- theater programs
- small classes / close faculty contact
- ADHD support
- drive time from Cherry Hill
- campus setting (contained-urban vs. rural vs. large public)

Anything added to this project should respect those axes — they are why the
schema has the fields it has.

---

## 2. File anatomy

One file, three inlined parts:

| Part | Location | Notes |
|---|---|---|
| `<style>` | top of file | all CSS, layered screen → mobile → print |
| `const RAW = {...};` | between them, on **one single line** | the entire dataset, minified JSON |
| `<script>` | rest of file | rendering, verdict math, exports |

`RAW` is a **one-line** minified JSON blob. That matters: every edit script in
this project finds it by `line.startswith('const RAW = ')`, parses it, mutates
it, and writes it back as one line. Do not reformat it across multiple lines or
those scripts break.

### CSS section order (order is load-bearing)

```
TOKENS              CSS custom properties, fonts, colors
TOOLBAR (screen)    the control bar; all .no-print
SHEET               the fixed 8.5x11in page — the print layout
TABLE OF CONTENTS   screen-only, .no-print
MOBILE (screen)     @media screen and (max-width:820px) — overrides SHEET
PRINT               @media print — overrides everything
```

The mobile block **must** stay before the print block. Mobile is
`@media screen` so it can never affect paper output; this separation is what
makes it safe to restyle the screen view without touching print.

---

## 3. JSON schema

`RAW` is `{ "colleges": [ ... ] }`. **40 records**, all with an identical key
set. Adding a record with a different shape will break rendering — validate
against `colleges[0]` (see §7).

```jsonc
{
  "id": "muhlenberg-college",          // slug, unique; used for #sheet-<id> anchors
  "name": "Muhlenberg College",
  "location": {
    "city": "Allentown", "state": "PA", "region": "Mid-Atlantic",
    "campus_setting": "small-city",    // ENUM — see below
    "distance_to_nearest_major_city": "~1 hr to Philadelphia",
    "walkability_safety_notes": "..."  // prose, renders under "Campus on foot"
  },
  "undergraduate_population": 2100,    // int; drives Small/Mid-size/Large
  "student_faculty_ratio": "9:1",      // string as-is
  "personal_attention": {
    "known_for_small_classes": true,   // bool|null
    "notes": "..."
  },
  "admissions": {
    "sat_25th_percentile": 1250,       // int|null — null = no band published
    "sat_75th_percentile": 1400,
    "sat_data_year": null,             // string|null; values matching /^approx/i are SUPPRESSED in the UI
    "test_optional": true,             // bool|null
    "acceptance_rate": 55.0            // float|null — null triggers estimation
  },
  "academics": { "offers_pre_law_track": true, "pre_law_notes": "..." },
  "extracurriculars": { "theater_program": true, "theater_notes": "..." },
  "housing": {
    "first_year_singles": "limited",   // ENUM: "yes" | "limited" | "rare" | null
    "housing_notes": "..."
  },
  "media": { "image_urls": [], "source_urls": [] },   // image_urls[0] is the only one used
  "tags": ["small-college-feel"],      // NOT RENDERED anywhere — kept for filtering/future use
  "user_notes": "",                    // empty => sheet prints ruled lines to write on
  "overview": "...",                   // renders under "Summary"
  "distance_from_cherry_hill_nj": {
    "driving": "~1 hr 15 min (~70 mi)",
    "flight_from_phl": null            // null renders as "N/A"
  },
  "known_for": "...",
  "adhd_services": {
    "standard_accommodations": true,
    "structured_program": false,       // true => "Named program", else "Standard only"/"None found"
    "notes": "..."
  },
  "admissions_calendar": {
    "offers_early_decision": true, "ed1_deadline": "November 1, 2026", "ed1_notification": "mid-December",
    "offers_ed2": true,  "ed2_deadline": "...", "ed2_notification": "...",
    "offers_early_action": false, "ea_deadline": null, "ea_notification": null,
    "regular_decision_deadline": "...", "regular_decision_notification": "...",
    "cycle_year_reference": "Class of 2031 / Fall 2027 entry",  // NOT RENDERED (removed from UI)
    "verify_note": "..."                                         // NOT RENDERED (removed from UI)
  },
  "pre_law_activities": {              // each bool|null; only `true` values are displayed
    "mock_trial": true, "moot_court": null, "law_journal": null,
    "gov_internship_pipeline": true, "undergrad_law_courses": false, "debate_team": true
  },
  "dining": { "late_night": null }     // string|null; null => "Not available or unknown"
}
```

### Enums

`campus_setting` — exactly one of:
`urban`, `urban-contained`, `suburban`, `small-city`, `small-town`, `rural`.
Anything else falls through to a raw-string label with no descriptive note.
Defined in the `SETTING` map in JS.

`first_year_singles` — `yes` | `limited` | `rare` | `null`.
Mapped by `SINGLES` / `singlesInfo()` to the labels
"Available in dorms" / "Limited, lottery or preference" / "Rare for first-years"
/ "Unconfirmed".

### Fields present but deliberately not rendered

`tags`, `cycle_year_reference`, `verify_note`, `media.source_urls`,
`sat_data_year` when it starts with "approx". These were removed from the UI on
request but kept in the data. **`verify_note` is important context** — it flags
which records were never verified against the school's own site (see §8).

---

## 4. The verdict algorithm (Safety / Target / Reach / High reach)

This is the core logic. There is **no** reach/safety field in the data — the
verdict is computed live from the user's SAT (toolbar input, default 1400).

`selectivity(c)` — if `acceptance_rate` is null, estimate from the 75th
percentile: `>=1540 → 12%`, `>=1510 → 18%`, `>=1480 → 25%`, `>=1450 → 32%`,
`>=1420 → 42%`, else `55%`. Estimated rates display with `(est.)`.

`verdictFor(c, you)`:

```
mid   = (p25 + p75) / 2
half  = max((p75 - p25) / 2, 1)
z     = (you - mid) / half          // +1 = at the 75th percentile

shift = +0.4  if rate >= 50
         0.0  if 40 <= rate < 50
        -0.5  if 25 <= rate < 40
        -1.0  if 15 <= rate < 25
        -1.75 if rate < 15

eff   = z + shift

eff >= 1.2   → safety
eff >= -0.2  → target
eff >= -1.5  → reach
else         → high
```

**Missing-band fallback.** If `p25`/`p75` are null (test-blind schools — UCLA is
the only current case), the score path is skipped entirely and the verdict comes
from admit rate alone: `<15% → high`, `<30% → reach`, `<55% → target`, else
`safety`. The sheet then prints "No score band published for this school"
instead of the ruler. Do not invent a score band to fill this gap.

Both threshold tables are hand-tuned judgment calls, not derived from anything.
They are the right thing to adjust if verdicts look wrong.

---

## 5. Rendering pipeline

```
render()
  ├─ reads SAT + sort + active verdict filters
  ├─ verdictFor() on every college
  ├─ filters, sorts (odds | list | name | size-asc | size-desc | deadline)
  ├─ stack.innerHTML = tocHTML(list, you) + list.map(sheetHTML).join('')
  ├─ autofit()
  └─ wirePhotos()
```

`render()` runs on every SAT keystroke/slider move, sort change, and filter
toggle. It replaces all innerHTML, so **all `<img>` elements are destroyed and
recreated**, restarting every download. Keep that in mind before adding
anything stateful to a sheet.

### Key functions

| Function | Role |
|---|---|
| `sheetHTML(c, i, total, you, domain)` | builds one `<article class="sheet" id="sheet-<id>">` |
| `tocHTML(list, you)` | screen-only contents page, grouped by verdict, `id="toc"` |
| `tri(state, title, sub, icon, extra, subAlways)` | one Attributes cell; `extra='wide'` spans both grid columns; `subAlways` forces the sub-label regardless of state |
| `preLawYes(c)` | returns only the `true` pre-law activities, as title-case labels |
| `plansOf(c)` | ED I / ED II / EA / RD objects with parsed dates and timeline lane |
| `timelineDomain(list)` | **shared** date range so every sheet's timeline uses one scale |
| `autofit()` | two-step density cascade (see below) |
| `collegeText(c, you)` / `buildText(list, you)` | plain-text export |

### `autofit()` — the print density cascade

Sheets are a fixed `279.2mm` tall with `overflow:hidden`. `autofit()` measures
`.body` and, if it overflows, applies `.tight`, then `.tighter` — progressively
smaller type and padding. It **skips entirely on mobile** (`max-width:820px`)
because sheets there grow to fit; running it would wrongly shrink everything.
It also runs on `document.fonts.ready` and `beforeprint`, since font loading
changes metrics.

### Icons

There are none. All SVG icons were removed on request. Attribute state is
conveyed by background/border only: filled grey = yes, plain outline = no,
dashed outline = unconfirmed.

---

## 6. Photos

`media.image_urls[0]` renders as a `<div class="photo"><img ...></div>` at the
top of the sheet's left column. Absent → the block isn't emitted at all and the
layout closes up. 31mm tall, `object-fit:cover` (so non-2:1 images crop);
mobile switches to `aspect-ratio:16/10`.

**History worth knowing:** an earlier version had failure-detection CSS
(`.photo.failed img{display:none}`) that could hide images. It was removed —
the `<img>` is now plain, and a broken URL shows a broken-image icon rather
than silently vanishing. **Do not reintroduce any rule that hides images on
error.** A visible failure is the desired behaviour.

Diagnostics that remain: a live "photos: N loaded · N failed · N pending"
counter in the toolbar, a **Retry photos** button, and console helpers
`photoReport()` (prints a table) and `failedPhotos()` (returns failing URLs).

Roughly 32 of 40 have photos. Some are hotlinked from college CMSs and may rot;
some are hosted by the user on GitHub Pages
(`https://berrym-web.github.io/college-onepagers/<Name>.jpg`), which is the
more reliable pattern. Query strings (`?itok=`) have been stripped from all
URLs at the user's request.

---

## 7. How to edit safely

The established workflow, which has caught several errors:

```bash
# 1. Data edits — parse the RAW line, mutate, write back as ONE line
python3 - <<'PY'
import json
p='college-onepagers.html'
lines=open(p).read().split('\n')
i=[n for n,l in enumerate(lines) if l.startswith('const RAW = ')][0]
d=json.loads(lines[i][len('const RAW = '):].rstrip(';'))

# ... mutate d['colleges'] ...

# validate every record against the reference shape before writing
ref=d['colleges'][0]
SUBS=['location','personal_attention','admissions','academics','extracurriculars',
      'housing','media','distance_from_cherry_hill_nj','adhd_services',
      'admissions_calendar','pre_law_activities','dining']
for c in d['colleges']:
    assert set(c.keys())==set(ref.keys()), c['id']
    for s in SUBS: assert set(c[s].keys())==set(ref[s].keys()), (c['id'],s)

lines[i]='const RAW = '+json.dumps(d,ensure_ascii=False,separators=(',',':')).replace('</script>','<\\/script>')+';'
open(p,'w').write('\n'.join(lines))
PY

# 2. Template edits — assert the anchor is UNIQUE before replacing
#    (a non-unique anchor once caused a duplicated code block)
#    def rep(old,new): assert h.count(old)==1; h=h.replace(old,new,1)

# 3. ALWAYS validate after any edit
python3 -c "
import re;h=open('college-onepagers.html').read()
js=re.search(r'<script>(.*?)</script>',h,re.S).group(1)
open('app.js','w').write(\"require('./shim.js');\n\"+js+'\nmodule.exports={colleges,sheetHTML,tocHTML,collegeText,timelineDomain,verdictFor,VERDICT,plansOf};')
css=re.search(r'<style>(.*?)</style>',h,re.S).group(1)
print('css balanced:',css.count('{')==css.count('}'))"
node --check app.js
```

### The headless test shim

If a browser isn't available, the render functions can still be exercised with a
small DOM stub. Create `shim.js`:

```js
const mk = (id) => ({
  id, value: id==='sat'||id==='satRange' ? '1400' : (id==='sort' ? 'odds' : ''),
  innerHTML:'', textContent:'', dataset:{},
  classList:{toggle(){},add(){},remove(){},contains(){return false}},
  addEventListener(){}, click(){}, setAttribute(){}, getAttribute(){return 'true'},
  querySelectorAll(){return []}, querySelector(){return null}, files:[], title:'',
});
const store={};
global.document={
  getElementById:(id)=> store[id] || (store[id]=mk(id)),
  querySelectorAll:()=>[], addEventListener(){}, createElement:()=>mk('x'),
  body:{classList:{toggle(){},contains(){return false}}, appendChild(){}},
  fonts:null,
};
global.window={print(){},addEventListener(){},matchMedia:()=>({matches:false})};
global.alert=console.error; global.FileReader=class{};
global.navigator={clipboard:{writeText(){}}};
```

Then render everything and scan for damage:

```bash
node -e "
const A=require('./app.js');const d=A.timelineDomain(A.colleges);
let all='';A.colleges.forEach((c,i)=>all+=A.sheetHTML(c,i,A.colleges.length,1400,d));
console.log('sheets:',(all.match(/class=\"sheet\"/g)||[]).length);
['NaN','undefined','Infinity'].forEach(b=>console.log(b+':',(all.match(new RegExp(b,'g'))||[]).length));
const p=[...all.matchAll(/(?:left|width|right):(-?[\d.]+)%/g)].map(m=>parseFloat(m[1]));
console.log('css % range:',Math.min(...p),'-',Math.max(...p));
"
```

`NaN`/`undefined`/`Infinity` counts must be 0 and percentages must stay within
0–100. This catches most breakage immediately.

---

## 8. Data provenance — read this before trusting any number

**Most of this dataset was written from an LLM's general knowledge, not
retrieved from the schools' websites.** Records carry a `verify_note` saying so
(no longer displayed, but still in the JSON). Specifically:

- **Deadlines** are the least reliable field. Many read "(approx.)".
- **SAT bands** are approximations of real published figures — plausible, not
  verified.
- **`pre_law_activities`** was entirely inferred. Many values are `null`
  (unconfirmed) and only `true` values display, so an activity a school
  genuinely has may simply be missing.
- **`first_year_singles`** was assigned by school-type heuristic (small LACs
  get some singles, large publics rarely), not from housing sites.
- **`dining.late_night`** is populated for only 7 schools, from user-supplied
  facts. Everything else says "Not available or unknown".

A past failure worth not repeating: **URLs cannot be recalled, only retrieved.**
An earlier session generated ~26 plausible-looking campus photo URLs; every one
404'd, while the two actually fetched from live pages both worked. Do not
produce a URL you have not fetched.

---

## 9. Features, briefly

- **Toolbar:** SAT input + slider, sort (odds / list order / name / size asc /
  size desc / earliest deadline), verdict filter chips, photo status, Retry
  photos, Plain text, Print all, Load JSON, Hide UI.
- **Hide UI** — button top-right or the **H** key (suppressed while typing in a
  field). Collapses the toolbar for reading.
- **Load JSON** — swaps the dataset at runtime, in memory only; a reload
  restores the embedded copy. Accepts either a bare array or `{colleges:[...]}`.
- **Plain text** — `collegeText()`/`buildText()` produce label-and-value lines
  with blank-line paragraph breaks, **no ASCII art, no alignment padding, no
  rule characters**, so it survives pasting into Word or Google Docs. It carries
  *more* than the sheet does: the full `notes` prose for each attribute, which
  the printed page compresses into badges. Exports what is currently on screen
  (respects filter and sort).
- **Table of contents** — screen-only, rebuilt every render, grouped by verdict,
  anchors to `#sheet-<id>`. Each sheet has a matching "↑ Back to list" link.
  Both are `.no-print`.

---

## 10. Layout notes

**Print** — `@page{size:215.9mm 279.4mm;margin:0}`, one sheet per page,
`break-after:page`. The verdict color runs down a left rail so pages are
identifiable when stacked. Blank `user_notes` renders as ruled lines that
expand to fill remaining space.

**Score ruler** — the school's 25th–75th band drawn on a fixed 1100–1600 scale
with the user's score as a dashed marker. This is the one graphic that answers
"reach or safety" directly.

**Deadline timeline** — binding plans (ED I ◼, ED II ◆) above the axis,
non-binding (EA ○, RD ▲) below. All sheets share one date domain from
`timelineDomain()` so they're visually comparable. This is the densest element
and the most likely to need attention on narrow screens.

**Removed on request, do not reinstate without asking:** the size meter (a log
scale that misleadingly implied linear comparison), all SVG icons, the tag
pills, the deadline verify caveat, "Fit checks" (now "Attributes"), the
per-sheet "Odds estimated · not a prediction" disclaimer, the shortlist
counter, and the "At SAT ####" line above the admit rate (it implied the rate
was conditioned on the score, which it is not).
