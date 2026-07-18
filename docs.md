# Dynamic Variables — Documentation

**Change one base variable — every dependent color, spacing and type value updates itself.**

Dynamic Variables is a Figma plugin that lets variables derive from other
variables: write a small JavaScript formula in a variable's **description**
field, and the plugin keeps its value in sync with everything it depends on.

> 🛝 **Interactive version:** every example below runs live (and is editable)
> at **[the interactive docs page](https://neelts.github.io/figma-dynamic-variables-docs/)** —
> this file is the plain-markdown mirror.

---

## Start here — your first dynamic variable

1. **Write a formula in a description.** Select a variable and, in its
   *description* field, write an expression between double curly braces,
   referencing another variable with `$`:

   ```js
   {{ $base-spacing * 2 }}
   ```

2. **Run the plugin.** *Plugins → Dynamic Variables → Update Once* (one shot)
   or *Start Monitoring Changes* (live updates; pause/resume and the interval
   are in the menu).

3. **That's it.** The value is now computed. Edit `base-spacing` and it
   follows. Rename a referenced variable — references fix themselves.

Prefer a real editor over description fields? *Plugins → Dynamic Variables →
Open Dynamic Variables Editor*: your collections as a tree, a full code editor
with syntax highlighting and inline errors — and the AI menu (below).

> The `{{ }}` wrapper lives in the description field — that's what you see
> when editing descriptions directly (the only way in 1.x). The editor hides
> the wrapper by default and shows just the expression.

---

## Referencing variables

| Write | Meaning |
|---|---|
| `$name` | Same-collection variable (shorthand; fine for simple names) |
| `$('name')` | Same-collection, call form — **recommended**, unambiguous for any name |
| `$('Collection/group/name')` | Cross-collection reference |
| `$('./sibling')` | Relative: same group (`./sub/name` reaches a subgroup) |
| `$('../name')` | Relative: one group up (repeat `../` to climb) |

Two rules that answer most "why isn't it resolving?" questions:

- **A variable's name is its full slash path.** For a variable shown as
  `spacing/Base`, the name is `spacing/Base`, not `Base`. Reference it as
  `$('spacing/Base')` from the same collection, or `$('./Base')` from a
  sibling. A bare `$('Base')` only resolves if a variable is literally named
  `Base`.
- **References resolve per-mode automatically.** In a Light/Dark collection,
  `$('Surface')` gives the Light value when evaluating Light and the Dark
  value when evaluating Dark — one expression covers all modes. (Don't reach
  for `$$('Surface').values[...]` for this — see metadata below.)

Legacy raw forms from 1.x (`$Collection/name`) still work; the quoted call
form is recommended because it can never be misread inside a larger
expression.

---

## Variable metadata — `$$`

`$$.prop` reads the *current* variable's own metadata; `$$('other.prop')`
reads another variable's.

Available props: `name`, `id`, `type`, `alias`, `collection`, `collectionId`,
`mode`, `modeIndex`, `modeId`, `modes` (array of mode names), `values`
(array of the variable's values, ordered by mode).

> ⚠️ **The one rule people trip on:** metadata substitutes as raw **text**
> before your code runs. String props must be quoted *by you*:
>
> - `'$$.name'.split('/')[1]` ✓
> - `'$$.mode' === 'Dark'` ✓
> - `$$.mode === 'Dark'` ✗ — after substitution this reads `Light === 'Dark'`, an error.

This is what makes **name-generic expressions** possible: a whole scale can
share one expression that reads its own step from its name:

```js
{{ const step = Number('$$.name'.split('-')[1]); Math.round($base * Math.pow($ratio, step)) }}
```

---

## Engine rules worth knowing

- **Multi-statement expressions**: the value is the **last statement** —
  `{{ const t = $a * 2; Math.round(t) }}`. A top-level `return` is a syntax
  error (wrap in an IIFE if you want one).
- **Computed reference paths** resolve *before* the expression body runs.
  Build them only from literals and `$$` text —
  `$('ramp/' + '$$.name'.split('-')[1])` ✓ — never from a `const` declared in
  the same expression.
- **String values splice as raw text** — use them inside string literals
  (`"$greeting, $user-name" + "!"`), punctuation outside the reference.
- **Async is fine**: if the value is a Promise (e.g. a `fetch` chain), the
  plugin awaits it.
- **Colors** may be returned as hex strings, `rgb()`/`hsl()` strings, colord
  instances, or Figma `{r,g,b,a}` objects — the plugin coerces. Bare CSS names
  need a parser: `rgba("green")` or `cd("green")`.
- **Disable an expression** without deleting it: comment the braces — `//{{ … }}`.

---

## Libraries in scope

| Shorthand | What | Example |
|---|---|---|
| `cd` | [colord](https://github.com/omgovich/colord) — chainable color ops | `cd($brand).darken(0.1).toHex()` |
| `cl` | [culori](https://culorijs.org/api/) — color science. *Functions, not methods*: `cl.oklch(c)`, `cl.formatHex(c)`, `cl.wcagLuminance(c)`; results are plain objects | `cl.formatHex(cl.rgb({mode:'oklch', l:0.7, c:0.15, h:250}))` |
| `cv` | `culori.converter()` — parse/convert any CSS color to RGB | `cl.clampRgb(cv("oklch(90% 0.4 135)"))` |
| `rgba` | Figma-style CSS color parser → `{r,g,b,a}` | `rgba("green")` |
| `fn` | Number/date formatting (Intl) | `fn($price, "en-US", {style:"currency", currency:"USD"})` |
| `cdRandom`, `cdGetFormat` | colord extras | `cdRandom().toHex()` |
| `alias` | Alias this variable to another **same-type** variable (a real Figma alias — and the only valid write for Timing/Easing) | `alias($dark ? "Theme/night" : "Theme/day")` |

---

## Global Functions

Repeated logic across many expressions? Create a **Global Functions**
collection in the editor and define plain JavaScript helpers once:

```js
function onColor(bg) {
  return cd('#fff').contrast(cd(bg)) >= cd('#000').contrast(cd(bg)) ? '#FFFFFF' : '#000000';
}
```

…then call from any expression: `{{ onColor($('Brand/Primary')) }}` — the ramp
recipe above does exactly this with `rampStep()`.

Remember: helper code is **plain JS** — no `$` references inside it; resolve
values in the expression and pass them as arguments. The `cd`/`cl` libraries
*are* available inside helpers.

---

## Recipes

Every recipe below is live-editable on the
[interactive page](https://neelts.github.io/figma-dynamic-variables-docs/),
and pinned as a test against the real engine in the plugin repository.

### The classic: concentric corner radius

The plugin's cover formula, in its everyday form: a card wraps a rounded
button with some padding, and only one radius is truly a choice — derive the
card's from the button's and the corners stay concentric:

```js
card-radius → {{ $button-radius + $card-padding }}
```

Change the button radius or the padding and the card follows. (For deep
nesting, clamp at zero: `{{ Math.max(0, $('../radius') - $('Primitives/Nesting Gap')) }}`.)

### Color ramp from one seed (OKLCH) — with a Global Function

The math lives in ONE Global Function; every step is a readable one-line call
that passes its own name. Adding `ramp-300` later means adding a variable, not
another copy of the formula.

Fully algorithmic — lightness is computed from the step number in the name, so
*any* `ramp-NNN` works, including steps invented long after the function was
written:

```js
// Global Functions collection
function rampStep(seed, name) {
  const n = parseInt(name.replace(/\D/g, ''), 10);  // digits from the name: ramp-500, ramp500, 500
  const s = cl.oklch(cv(seed));
  return cl.formatHex(cl.clampRgb(cl.rgb({mode:'oklch', l: 0.97 - 0.85 * n / 1000, c: s.c, h: s.h})));
}
```

```js
// every ramp step
{{ rampStep($seed, '$$.name') }}
```

### Spacing scale with density modes

`Base` carries per-mode values (Compact 12 / Default 16 / Comfortable 20);
steps live beside it and reference it relatively:

```js
spacing/xs → {{ Math.round($('./Base') * 0.5) }}
spacing/md → {{ Math.round($('./Base') * 1.5) }}
spacing/xl → {{ Math.round($('./Base') * 3) }}
```

### Type scale — build on the previous step

Reference the already-computed token; don't repeat its formula:

```js
heading-md → {{ Math.round($body * $ratio) }}
heading-lg → {{ Math.round($('heading-md') * $ratio) }}
```

### Accessible on-color

```js
text           → {{ cd('#FFFFFF').contrast(cd($background)) >= cd('#000000').contrast(cd($background)) ? '#FFFFFF' : '#000000' }}
contrast-ratio → {{ cd($text).contrast(cd($background)) }}
passes-aa      → {{ $('contrast-ratio') >= 4.5 }}
```

### Dark mode derived from Light

Author a single-mode `Source` palette; the two-mode `Theme` derives Dark by
inverting OKLCH lightness and damping chroma:

```js
{{
  const light = $('Source/surface');
  const src = cl.oklch(cv(light));
  const dark = { mode: 'oklch', l: 1 - src.l, c: src.c * 0.6, h: src.h };
  '$$.mode' === 'Dark' ? cl.formatHex(cl.clampRgb(cl.rgb(dark))) : light;
}}
```

### Localization — languages as modes

Make the modes **EN / DE / FR** and compose copy that translates itself.
String values splice as raw **text**, so build sentences *inside* a string
literal — and keep punctuation out of the reference (the name matcher is
greedy: `$user-name!` reads as a variable named `user-name!`):

```js
welcome     → {{ "$greeting, $user-name" + "!" }}     // Hello, Neil! / Hallo, Neil! / Bonjour, Neil!
likes-label → {{ "$count " + ($count === 1 ? ["like","Like","j'aime"][$$.modeIndex]
                                             : ["likes","Likes","j'aime"][$$.modeIndex]) }}
```

### Live data — weather, prices, time (throttled)

Expressions can `fetch`; the plugin awaits the Promise. Two hand-set
variables — a `city` STRING and a `use-fahrenheit` BOOLEAN — drive a
geocoding fetch that feeds a weather fetch. And because monitoring
re-evaluates every tick, the expression throttles itself: it reads its OWN
previous value (`$$.values[$$.modeIndex]` — the self metadata array indexed
by the numeric mode index) and only re-fetches when the stamp no longer
matches. The stamp bakes in the *inputs* plus a minute clock, so editing the
city or flipping the unit re-fetches immediately, while an unchanged setup
waits out the minute:

```js
updated → {{ new Date().toLocaleTimeString("en-GB", { hour: "2-digit", minute: "2-digit" }) }}
```

```js
feels-like → {{
  const suffix = $use-fahrenheit ? "°F" : "°C";
  const stamp = " · " + "$city" + " · " + "$updated";
  const cached = String($$.values[$$.modeIndex]);
  cached.endsWith(stamp) && cached.split(" · ")[0].endsWith(suffix)
    ? cached                     // same city, same unit, same minute — no API call
    : fetch("https://geocoding-api.open-meteo.com/v1/search?count=1&name=" + encodeURIComponent("$city"))
        .then(r => r.json())
        .then(g => fetch("https://api.open-meteo.com/v1/forecast?current=apparent_temperature"
          + "&latitude=" + g.results[0].latitude + "&longitude=" + g.results[0].longitude
          + "&temperature_unit=" + ($use-fahrenheit ? "fahrenheit" : "celsius")))
        .then(r => r.json())
        .then(d => Math.round(d.current.apparent_temperature) + suffix + stamp);
}}
```

The result reads like `92°F · New York · 14:03` — the stamp doubles as an
honest cache key, and a small derivation splits the display value back out:

```js
temperature → {{ String("$feels-like").split(" · ")[0] }}   // "92°F"
caption     → {{ "feels like · updated $updated" }}         // for the card's footer
```

Multi-statement expressions read top-to-bottom: set up locals with `const`,
and the **last statement** is the variable's value (a trailing semicolon is
fine). `return` works inside the arrow bodies — it's only a *top-level*
`return` that's a syntax error.

…or simply `{{ new Date().toLocaleTimeString("en-GB") }}`.

### Motion variables (Timing & Easing) — select, don't compute

Figma's new Timing and Easing variable types are **read-only for plugins**:
the Plugin API rejects every attempt to create one or write a value into
one. The single exception — which this plugin turns into a feature — is
that a motion variable may be **aliased to another variable of the same
type**. So an expression on a Timing or Easing variable doesn't return a
value; it returns the **name** of the variable it should follow:

```js
// on Motion/duration (a TIMING variable):
{{ $reduced-motion ? "slow" : "fast" }}          // same-collection names

// on Motion/active-curve (an EASING variable):
{{ $reduced-motion ? "Motion/Hold" : "Motion/Bouncy" }}

// the explicit spelling — alias() works on EVERY variable type:
{{ alias($use-brand-b ? "Theme/brand-b" : "Theme/brand-a") }}   // e.g. on a COLOR
```

Flip the `reduced-motion` boolean and every animation bound to
`active-curve` switches to Hold — an accessibility toggle for your whole
motion system, or brand-level motion personalities from one driver. And
since `alias()` writes a real Figma alias for any same-type pair, the
same trick switches colors, numbers or strings by reference instead of
copying values.

> ⚠️ **Use with care.** Because raw writes are blocked, the plugin can
> re-point a motion variable but can **never write its original value
> back** — undoing an alias means re-picking the curve in Figma's own
> panel (or Cmd/Ctrl+Z right after). Keep your source curves in dedicated
> preset variables (like `Hold` and `Bouncy` above) and put expressions
> only on *selector* variables that were born to be aliases. If Figma
> later opens raw writes, computed values start working here by
> themselves — the plugin probes the API on every write.

---

🧳 Coming from Tokens Studio or another token pipeline? The recipes above
are your parity bridge — and code `MIGRATING` at
[checkout](https://neelts.lemonsqueezy.com/buy/7753e037-3c90-404f-8cbb-508ba5bfc2eb?checkout[discount_code]=MIGRATING) acknowledges
that switching takes work.

## AI editing — describe it, don't code it *(2.0)*

In the editor, the ✨ AI menu gives you a two-step loop that works with
**whatever assistant you already use** — no API key, no extra subscription:

1. **Copy Brief for AI** — puts a complete brief on your clipboard: your
   collections, variables, current values, and the full expression syntax.
   Paste it into ChatGPT, Claude, Gemini — then ask in plain words:
   *"derive a 10-step ramp from brand/seed, and make button text pick black
   or white by contrast."*
2. **Paste AI Result** — paste the assistant's reply back. The plugin parses
   it and shows a **preview of every proposed change** — old value → new
   expression, per variable. Nothing touches your file until you apply.

**Models that do well** (from the plugin's own test harness — 20
design-system scenarios): Claude Haiku 4.5, Codex and GPT-5-mini solve nearly
everything; DeepSeek is a solid budget option. Small local models tend to
produce invalid expressions — the preview catches them, but you'll have a
better time with the tier above.

---

## Run modes, errors, housekeeping

- **Start Monitoring Changes** — re-evaluates on your interval
  (*Configure → Set Monitoring Interval*), pause/resume from the menu.
  **Update Once** — a single pass, full control.
- **Errors are per-variable, per-mode**: cyclic dependencies, unresolved
  references, type mismatches — reported with the variable name; inline in
  the editor.
- **Renames are safe**: the plugin tracks references and rewrites them when
  variables are renamed or moved.
- **Library variables**: bring team-library variables into a local collection
  as aliases, then reference them like any local variable.
- **Sandbox limits**: expressions run in Figma's plugin sandbox — standard
  JavaScript plus `fetch` and the bundled libraries; no `window`/`document`,
  no npm imports. Shared helpers belong in Global Functions.

---

## FAQ

**Why isn't my cross-collection reference resolving?**
A variable's name is its full slash path — see
[Referencing variables](#referencing-variables).

**My variable names have spaces or emoji.**
Use the call form: `$('🎨 Brand Primary')`.

**Can I do HSB / LCH / OKLCH?**
Yes — culori (`cl`, `cv`) covers every CSS color space, plus gamut clamping
(`cl.clampRgb`) so results stay displayable.

**Numbers show too many decimals.**
`Math.round(x)`, `x.toFixed(2)`, or locale-aware `fn(x, "en-US")`.

**Can an expression react to its mode?**
`{{ '$$.mode' === 'Dark' ? … : … }}` (mind the quotes) — though plain
references already resolve per-mode, which is usually all you need.

**Will the plugin change variables I didn't write expressions for?**
No. Only variables whose description contains `{{ }}` are touched — and
AI-proposed changes always go through the preview first.

---

## Free & Pro

Everything is free for up to **5** dynamic variables, forever. A
full-featured trial (7 or 14 days depending on variant) has no cap.
[Pro](https://neelts.lemonsqueezy.com/buy/7753e037-3c90-404f-8cbb-508ba5bfc2eb)
removes the limit and funds development — activate via *Dynamic Variables →
Pro License → Activate Key* (5 devices, 10 on Unlimited; write me for
activation resets).

---

🎓 **You read the whole thing.** That deserves something: use code
`IREADTHEDOCS` at [checkout](https://neelts.lemonsqueezy.com/buy/7753e037-3c90-404f-8cbb-508ba5bfc2eb?checkout[discount_code]=IREADTHEDOCS)
for a reader's discount on Pro. Thanks for caring about the details — you're
exactly who this plugin is for.

[Dynamic Variables on Figma Community](https://www.figma.com/community/plugin/1365339609215368041) ·
[Subscribe for updates](https://neelts.lemonsqueezy.com)
