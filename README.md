# OPT Premium Processing Tracker

Two pages covering the two halves of the wait for an EAD card.

| Page | Covers | State |
|---|---|---|
| [`index.html`](index.html) | I-765 filing → premium processing decision | **Complete**, frozen |
| [`card.html`](card.html) | Approval → physical card in hand | **In progress**, 4 of 5 stages |

**Live:** https://suhasadidela.github.io/premium-processing-tracker/ ·
[card tracker](https://suhasadidela.github.io/premium-processing-tracker/card.html)

> **Approved Fri Aug 14, 2026 — business day 19 of 30, eleven days inside the
> guarantee.** `index.html` is frozen at that moment: the clock is stopped, the
> status reads APPROVED, and nothing re-renders. Setting `approved` back to
> `null` in its `CONFIG` returns it to a live countdown.

The two pages look identical and behave oppositely, and that is the point. The
first half of the wait has a guarantee; the second half does not. See
[Two halves, two different promises](#two-halves-two-different-promises).

## The problem

Premium processing for an I-765 carries a 30 **business day** guarantee. That
sounds like a countdown, but you cannot build one with a subtraction: business
days are not calendar days, and the conversion is not a fixed ratio. Thirty
business days from a July filing and thirty from a November filing land
different distances out, because Thanksgiving and Christmas sit in the second
window and nothing sits in the first.

So the deadline is not something you can eyeball off a calendar. It has to be
counted, and the counting has three rules that are each easy to get wrong.

## Counting the days

### 1. The filing date is day 1, not day 0

The window is inclusive of the day you filed. This is the single most
consequential line in the project — treating the filing date as day 0 shifts
every subsequent date by one and puts the deadline on the wrong day.

For this filing it is what makes the arithmetic land exactly:

```
filed Tue Jul 21, 2026  →  business day 1
      Mon Aug 31, 2026  →  business day 30
```

Off by one, and Aug 31 becomes day 29 and the real deadline silently moves to
Sep 1.

### 2. Weekends don't count

Straightforward, and the only one of the three that most people remember.

### 3. Federal holidays don't count — on the day they're *observed*

There are eleven US federal holidays, and they are not all fixed dates. They
come in three shapes, so the code computes rather than hardcodes them:

| Shape | Examples | How it's found |
|---|---|---|
| Nth weekday of a month | MLK Day (3rd Mon of Jan), Thanksgiving (4th Thu of Nov) | `nthWeekday()` |
| Last weekday of a month | Memorial Day (last Mon of May) | `lastWeekday()` |
| Fixed date, shifted if it lands on a weekend | New Year's, Juneteenth, July 4, Veterans Day, Christmas | `observed()` |

That last shape is the subtle one. A fixed-date holiday falling on a weekend is
observed on the nearest weekday — **Saturday shifts back to Friday, Sunday
shifts forward to Monday** — and the *observed* day is the federal holiday, so
that is the day the count skips.

```js
function observed(date) {
  const day = date.getDay();
  const d = new Date(date);
  if (day === 6) d.setDate(d.getDate() - 1); // Sat → observed Friday
  if (day === 0) d.setDate(d.getDate() + 1); // Sun → observed Monday
  return d;
}
```

This is not hypothetical for 2026: **July 4 falls on a Saturday, so the holiday
is observed Friday July 3.** A tracker that skipped Jul 4 itself would skip a
day that was never a business day to begin with, and count Jul 3 — an actual
federal holiday — as a working day. It would be wrong twice in one weekend.

As it happens, **no federal holiday falls inside this particular filing
window**, so the holiday logic changes nothing for these dates. It is there
because the window is a config value: change `filedDate` to a November filing
and the holiday handling starts doing real work immediately.

## Two halves, two different promises

This is the single most important thing to keep straight when editing either
page, because getting it wrong would make the site lie.

**`index.html` counts against a guarantee.** Premium processing carries a
30-business-day commitment from USCIS. A deadline exists, so a countdown is the
honest shape: time remaining is a real quantity, running out means something,
and OVERDUE is a real failure state worth showing in red.

**`card.html` counts against nothing.** There is no service commitment on card
production and delivery. The ~12 days is what people typically report, not a
promise anyone made. So that page has no countdown, no deadline, and no failure
state, by construction:

- The hero counts **elapsed**, never remaining. There is nothing to remain
  against.
- The caption keeps the tilde — `DAY 5 OF ~12 TYPICAL`. The tilde is doing real
  work; dropping it silently converts a typical range into a deadline.
- Passing day 12 is **not** an error. The grid appends amber cells rather than
  clipping or turning red, and the pill reads LONGER THAN TYPICAL. Amber means
  unusual, not wrong. Red is not in this page's vocabulary at all.
- The day grid lights **every elapsed day**, the same way `index.html`'s segment
  bar does. Elapsed is lit, today pulses, future is dark — that is the whole
  vocabulary, with no per-cell milestone markers. Past day 12 it appends amber
  cells rather than clipping.
- The footer's TYPICAL date keeps its tilde and goes amber, never red, once
  passed. It sits among the other dates rather than as a headline figure,
  precisely so it does not read as a target.

If you later find yourself adding "days left until the card arrives" to
`card.html`, stop. That number does not exist.

## Design decisions

### The segments are discrete on purpose

The progress bar is thirty separate blocks, not a continuous fill.

A smooth bar would imply the quantity is continuous — that at noon on a
Wednesday you are "half a day" further along. You are not. Business days are
discrete units that tick over at a boundary and sit still in between, and they
sit *completely* still across a weekend. One block per day says that; a smooth
bar misrepresents it.

The same reasoning drives the weekday in the header. On a Saturday the counter
reads identically to Friday and does not move for two days. Without the day
name on screen, a correct tracker looks like a frozen one, so non-business days
label themselves as such.

### Zero build

One `index.html` — inline `<style>`, vanilla JS, no dependencies, no build
step. GitHub Pages serves the file as-is, so `git push` is the deploy.

The whole thing is a countdown and some date math. A toolchain would add more
moving parts than the problem has, and would need to still work in a year when
the only thing that should have changed is a date in a config object.

## Layout invariants

These are load-bearing and each has broken at least once. If you edit the CSS,
re-check all three at a wide desktop window *and* a phone width:

1. **The clock stays on one line.** Digit size is keyed to the *card* via
   container query units (`cqw`), not the viewport. Sizing it in `vw` breaks it:
   the card stops growing at `max-width: 900px` while `vw` keeps going, so past
   ~1100px the four groups outgrow the card and the seconds wrap.
2. **The segment bar is one row of 30.** It's a
   `grid-template-columns: repeat(30, 1fr)`, so the count holds structurally
   rather than relying on fixed widths that happen to fit. If
   `totalBusinessDays` changes, update the `30` to match.
3. **Everything fits the viewport** — the card *and* the links row beneath it.
   Vertical dimensions are capped with `min(…vw, …vh)`. Keying a vertical size
   to viewport *width* alone makes the layout outgrow a wide-but-short window
   and pushes content below the fold. Anything added outside the card counts
   against this budget: the links row overflowed a 1280×640 window by 3px until
   its padding was capped by `vh` too.

Both pages are checked against all three at 1920×1080, 1366×768, 1280×640,
1024×600, 375×812 and 320×568 before every push. `card.html` needs its own
short-landscape values rather than a copy of `index.html`'s — it carries a
five-row pipeline and a notes log on top of everything the first page has, and
reusing the first page's numbers made its hero *larger* on a short screen, not
smaller.

**Check `card.html` with notes present, not just as it currently stands.** With
`notes: []` the block is hidden and the page fits easily; a populated one has
repeatedly overflowed where the empty one passed. An empty optional block
passing is not evidence the populated one does.

Because the notes block costs real height that is usually unused, the page
carries a `has-notes` class on `<body>`: the roomy proportions stand while
notes are absent, and the layout tightens only when they exist. Those rules
outrank the media queries on specificity, so each breakpoint restates its own
`has-notes` values — otherwise the desktop-scale versions would *enlarge*
elements on a phone.

`card.html` has **two** phone breakpoints, and the reason matters. Sizing the
phone layout in fixed pixels means tuning it for the smallest screen it must
survive — a 568px-tall SE — and every taller phone then inherits that cramped
layout and leaves a third of the screen empty. So `max-width: 420px` carries
height-aware clamps sized for a modern phone, and a nested
`(max-width: 420px) and (max-height: 680px)` restores fixed tight values only
where the height genuinely demands them. At 375×812 the card now fills 67% of
the screen, against `index.html`'s 63%.

## Configuration — `index.html`

Everything configurable is in the `CONFIG` object at the top of the `<script>`
block:

```js
const CONFIG = {
  filedDate:   new Date(2026, 6, 21),   // months are 0-indexed: 6 = July
  estimated:   new Date(2026, 7, 20),
  lastDay:     new Date(2026, 7, 31),
  deadline:    new Date(2026, 7, 31, 23, 59, 59),
  totalBusinessDays: 30,
  approved:    new Date(2026, 7, 14, 14, 0, 0),  // null while still pending
  receiptNumber: "",     // keep empty — see note below
  showReceipt: false,
  uscisUrl: "https://egov.uscis.gov/"
};
```

Months are zero-indexed — January is `0`, July is `6`.

`lastDay` and `deadline` are the counted deadline. `estimated` is displayed for
reference and is not derived from the business-day math.

`approved` is the single switch between the two states. When it holds a date,
`render()` treats that moment as "now" instead of the wall clock, the 1-second
interval is never started, and the labels change to past tense. When it is
`null`, everything counts live again.

## Configuration — `card.html`

```js
const CONFIG = {
  approvalDate: new Date(2026, 7, 14),   // Aug 14, 2026 — months are 0-indexed
  typicalDays: 12,                       // TYPICAL, not promised

  stages: [
    { name: "Case Approved",              date: new Date(2026, 7, 14) },
    { name: "Case Approved In Portal",    date: new Date(2026, 7, 16), own: true },
    { name: "Card Was Produced",          date: new Date(2026, 7, 20) },
    { name: "Tracking Number Received",   date: new Date(2026, 7, 20), own: true },
    { name: "Card Delivered",             date: null }
  ],

  notes: [],            // empty hides the whole block

  trackingNumber: "9205…",   // empty = generic USPS link; set = deep link to parcel
  uscisUrl: "https://egov.uscis.gov/"
};
```

**Updating it is one line.** "Card produced today" means putting today's date on
`stages[2]`; everything else — the pipeline dot, the stage count — derives from
that. Nothing is entered twice.

### Five stages: three official, two personal

Three names are **USCIS's verbatim portal wording**. Do not reword them — the
page's value is that it matches what the portal literally says, so a paraphrase
makes it harder to reconcile the two, not easier.

| Stage | Kind |
|---|---|
| Case Approved | USCIS |
| Case Approved In Portal | personal |
| Card Was Produced | USCIS |
| Tracking Number Received | personal |
| Card Delivered | USCIS |

The two personal ones carry `own: true`, but **all completed stages render
identically** — filled cyan dot, same weight. The flag is documentation rather
than a style hook: it records which names came from the portal and which are
personal, which matters when reconciling the list against USCIS, but the
distinction is not drawn on screen.

`Card Delivered` is shortened from USCIS's "Card Was Delivered To Me".

### Closing footer

The card ends with a full-width three-date row — APPROVED / IN PORTAL / TYPICAL
— matching `index.html`'s FILED / APPROVED / LAST DAY. It closes the
composition rather than letting the card stop after the stat boxes.

The typical date lives here rather than in a stat box: as a headline figure it
read as a target, and it duplicated what the stat cards already said. As a
footer date beside the two real ones it reads as what it is — a reference
point. It keeps its tilde and goes amber, never red, once passed.

That freed a stat slot for **DAYS SINCE LAST STAGE**, which is the number that
actually signals a stalled case. Days elapsed can be large and fine; a long gap
with no stage change is the thing worth acting on, so that stat goes amber at a
week while the day count never does.

`Tracking Number Received` is the event USCIS itself calls **"Card Was Mailed To
Me"** — that is the wording the portal will show. It is tracked here under the
personal name because what is actually being recorded is the moment the tracking
number arrived, which is not the same instant USCIS writes its status row.

Anything that is not a milestone at all — "no movement today", a phone call —
belongs in `notes`, which render below the stat cards and are visibly quieter.

`notes` is capped in height and scrolls internally. Notes accumulate without
bound over a long wait, and without the cap invariant 3 holds today and breaks
silently on whichever note eventually pushes the card past the fold.

`trackingNumber` deep-links to the parcel and displays the number when set. When
empty the link stays, pointing at USPS's general tracking page — a control that
still goes somewhere useful beats a hidden or dead one.

### Elapsed in the hero, inclusive day in the caption

These are two different numbers and the page shows both, because showing only
one is ambiguous.

`index.html` counts **inclusively** — the filing day is day 1 — because the
30-business-day guarantee is defined that way. Card delivery has no such
definition, so inclusive counting here would be inherited rather than required,
and a bare "5" would be genuinely unclear: four days had actually passed.

So the hero shows **elapsed days and hours**, and the caption directly beneath
carries the **inclusive** day number against the typical range. On Aug 18 that
reads `04D 04H` over `DAY 5 OF ~12 TYPICAL` — both true, neither guessed at.

This is why `approvalDate` carries a time (2:00 PM). With hours on screen, a
midnight approval time would overstate elapsed by most of a day.

### The expected-date card

`EXPECTED AROUND / Aug 25, 2026` is derived from `approvalDate + typicalDays`,
never hardcoded. **"AROUND" is load-bearing** — ~12 days is a community average,
not a USCIS commitment, and the page must not imply Aug 26 is a failure.

Past that date the card goes **amber at most** and the label changes to
`LONGER THAN TYPICAL`. It never turns red and never reads as overdue. There is
no red anywhere on this page; that is asserted in testing, not just intended.

> Note: this card replaced **DAYS SINCE LAST MOVEMENT**. That stat was the one
> number that would actually signal a stalled case — a case can sit at day 20
> and be fine, or sit eight days with no stage change and be worth a phone call.
> The expected date does not carry that signal. If the wait starts feeling
> stuck, the pipeline dates are where to look.

When the final stage gets a date the page switches to a result rather than a
wait — the accent turns green, the hero relabels to DAYS FROM APPROVAL TO
DELIVERY and holds at the delivery day instead of following the wall clock, and
days-since-last-movement becomes `—` because nothing is pending.

`card.html` re-renders once an hour. Hours are the finest unit on screen; a
seconds display would imply a precision this process does not have.

One caveat worth knowing: USCIS writes case status in a nightly batch, so a
portal timestamp records when the row was written rather than when the event
happened. The hours figure is honest about elapsed wall-clock time since
approval — it is not evidence about when anything was actually decided.

## Interaction

- Status: **ON TRACK** (cyan) → **DUE SOON** (amber, last 3 business days) →
  **OVERDUE** (red), and **APPROVED** (green) once a decision lands. The clock
  digits and segment bar take the status color.
- While pending, clicking the clock (or Enter/Space when focused) toggles
  between time remaining and time since filing; the hint fades permanently
  after first use. Once approved the toggle is removed rather than left inert —
  there is only one duration left to show, so the hero stops being a control
  and drops its `role`, `tabindex`, and pointer cursor.
- On `card.html`: **IN PROGRESS** (cyan) → **LONGER THAN TYPICAL** (amber, past
  day ~12) → **CARD DELIVERED** (green). There is deliberately no red state.
- The two pages link to each other. They are one story told in two halves.

## Deploying

Pages serves the repo root from `main`, so both `index.html` and `card.html`
deploy together. There's no build step, so pushing is the deploy:

```bash
git add index.html && git commit -m "..." && git push
```

Give it 30–60 seconds, then hard-refresh (Cmd+Shift+R) past the browser cache.

## A note on the receipt number

`receiptNumber` is empty and `showReceipt` is `false`. Leave them that way.

The flag alone is not enough to keep the number private. `index.html` is served
verbatim to every visitor, so a value in `CONFIG` is readable from View Source
whether or not it is ever rendered — turning `showReceipt` off hides the line,
not the data. The only way the number stays off the site is to keep it out of
the file.

The page also carries a `noindex` meta tag, which keeps it out of search
results but is not privacy either: anyone with the URL can open it.
