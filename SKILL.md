---
name: bem
description: Audit and auto-fix CSS/SCSS and markup to a strict, search-friendly BEM convention. Use when requests mention BEM, БЭМ, block element modifier, class naming, SCSS nesting cleanup, "fix my styles to BEM", style linting, or reviewing .scss/.css/.ejs/.jsx/.tsx markup and selectors. Supports an optional, opt-in project-overrides layer for a codebase's own conventions.
---

# BEM Methodology (strict, search-first flavor)

Audit code against the rules below and apply minimal-diff fixes, or hold the rules in context while writing new code. The rules are tuned for hand-written static pages — landings, marketing pages, anything built as markup plus a stylesheet — where the whole point is a small, structured, searchable codebase; they hold just as well inside a component framework, with the extra conventions a component stack needs kept in `references/component-stacks.md`. This is a practical BEM dialect based on [bem.info](https://bem.info/) with two opinionated additions: zero abbreviations (so any class is grep-able) and zero style nesting beyond pseudo-classes and at-rules (so a selector is found exactly where its name lives).

## When invoked

1. Decide the mode from the request:
   - **audit-fix** (default) — scan target files, report violations, apply minimal-diff fixes.
   - **audit-only** — report violations with `file:line` and a suggested fix, change nothing.
   - **guide** — no scan; keep these rules in context while authoring new code.
   Honor an explicit argument (`audit`, `fix`, `guide`) if the caller passes one.
2. Detect the stack per file so fixes match the surrounding style instead of imposing one:
   - markup: `.ejs`/`.html` vs `.jsx`/`.tsx` — conform to the file's existing tag/class conventions (a project-overrides layer, if opted in, may prescribe a discipline per file type).
   - styles: `.scss` vs `.css`. Rules 15-16 assume a tokens/variables file; a plain `.css` project without custom properties gets those findings flagged, not auto-fixed. Pseudo-classes/elements and breakpoints always nest under their block with `&` (core rule 5); a legacy flat `.block:hover { }` rule found standing alone should be folded back into its block.
3. A project may add its own overrides layer (theme mixin, shared-block list, grandfathered tokens, per-file-type markup discipline) — see `references/project-overrides.example.md` for the shape. Such a layer is **off by default**: load it only on explicit opt-in — an argument like `--project-rules`, or the user pointing at the file. Never auto-detect it from repo path, git remote, or file fingerprints. Without opt-in, audit with the universal core only.
4. Audit each in-scope file **in full**, never only the lines a diff/PR touched. Once a file is in scope, every rule applies to the whole file: a stray blank line between sibling rules, a comment, a flat pseudo, or a combinator selector is a finding even when the current change did not introduce it. Auditing "just the diff" is the most common way violations are missed — read the file top to bottom.
5. For a component-framework codebase (`.jsx`/`.tsx`/`.vue`/`.svelte`), also load `references/component-stacks.md` — block folders, declared modifier sets, and how the hook rule applies there. Skip it for a static-page stack.
6. Work through `references/audit-checklist.md` rule by rule. Report each finding as `file:line — rule — before → after`.

## Core rules (universal)

### 1. Style classes only — never tags or ids
No element selectors (`div`, `a`, `h2`, `ul`) and no id selectors (`#hero`) in stylesheets. Every style hangs off a class. Tag selectors leak across unrelated markup; id selectors force specificity fights (id beats any class) and are not reusable. The only place a bare tag is acceptable is a global reset file, clearly scoped as such.

An `id` is fine in markup — for analytics, anchors (`#pricing`), JS targets, and `aria` references — and several per page is fine. It is simply never a styling hook. The element that carries the `id` also carries a class, and the class does the styling; if such an element needs styles, add a class beside the `id` and style the class.

```scss
/* bad */
section {
    padding: 80px;
}
#hero {
    padding: 80px;
}
.card h3 {
    font-size: 30px;
}

/* good — markup: <section id="hero" class="hero"> */
.hero {
    padding: 80px;
}
.card__title {
    font-size: 30px;
}
```

### 2. Naming: block, element, modifier
- **block** — independent component, kebab-case noun: `.hero`, `.sliding-tooltip`, `.comparison-table`. The top-level block of a page or component is named after its file (`pricing.ejs` / `pricing.scss` → `.pricing`; `SlidingTooltip.jsx` → `.sliding-tooltip`). Sections then derive from it as elements: `.pricing__price`, `.pricing__hero`, `.pricing__comparison`.
- **element** — part of a block, `block__element` with a single `__`: `.hero__title`, `.sliding-tooltip__content`.
- **modifier** — a flag/variant, `block--modifier` or `block__element--modifier`, kebab-case: `.btn--red`, `.card--featured`, `.comparison-row--title`.

**Shared blocks.** A reusable element has its own block name and stays that way everywhere — it is never rewritten into a page's element. `.btn`, `.gradient-text`, `.cta-row`, `.nowrap` are blocks, not `.pricing__button`. They are customized **only through their own modifiers** (`.btn--red`, `.gradient-text--orange`), never by renaming and never by adding a page element class onto them. To place or vary a shared block inside a page you have exactly two tools: add a modifier to the block, or wrap it in a page wrapper that handles layout.

A class counts as a **shared block** (and so is exempt from the "block = page name" and abbreviation rules) when any of these hold:
- it is used inside two or more different page/component blocks, or
- it lives in a shared/common styles location rather than a single page's stylesheet, or
- its name carries no page/component name (it reads as a generic reusable thing: `btn`, `gradient-text`, `cta-row`, `nowrap`, `inner`).

A class that names a page (`pricing__cta`) and appears only on that page is a page element, not a shared block — spell it out and treat it as a normal element. When in doubt, flag it as ambiguous rather than silently assuming either way. (A project-overrides layer is the right place to pin an explicit closed list of shared blocks.)

```html
<!-- bad — page element class mixed onto the shared block -->
<a class="pricing__hero-cta btn btn--red">Talk to Sales</a>

<!-- good — block styled by its own modifiers, positioned by a wrapper -->
<div class="cta-row">
    <a class="btn btn--red btn--cta">Talk to Sales</a>
</div>
<span class="gradient-text gradient-text--orange">Offload the Macs.</span>
```

The audit must not flag a shared block as a naming/abbreviation violation or try to fold it into the page block — and must flag a page element class added directly onto a shared block. A shared block's modifier is always its own top-level rule (`.btn--red { }`), never nested inside a page block.

Never chain elements. A deeper DOM nesting does **not** become `block__a__b`. Flatten the element name with hyphens:

```text
bad:  .hero__cards__card__title
good: .hero__card-title
```

The block name is not repeated inside element names beyond the single `block__` prefix.

### 3. No abbreviations
Full words only, so any class is findable by a plain text search across markup, styles, and JS. `.nav__button`, not `.nav__btn`; `.card__description`, not `.card__desc`. (An established project-wide token like `btn` may be grandfathered by a project-overrides layer, but new names spell it out.)

### 4. Flat selectors — every rule lives at its own name
Each block, element, and modifier is its own top-level rule. Do not nest one class inside another. A reader searching for `.hero__title` must land on the rule that styles it, not discover it buried under `.hero`.

```scss
/* bad */
.hero {
    .hero__title { font-size: 60px; }
    .hero__cta { margin-top: 50px; }
}

/* good */
.hero__title {
    font-size: 60px;
}
.hero__cta {
    margin-top: 50px;
}
```

**Combinator selectors are the same violation.** Reaching across classes with a descendant (`.a .b`), child (`.a > .b`), or `:has()` selector — or synthesising a new state by chaining two modifiers (`.block--a.block--b`) — is nesting by another syntax: the style no longer lives at its own single name. Express a style that depends on another element's presence or state as one flat modifier on the element being styled, and toggle that modifier in markup. A parent that must react to a child (`.box:has(.placeholder)`) becomes a modifier on the parent (`.box--with-placeholder`); a child styled only inside a parent state (`.box--disabled .label`) becomes a modifier on the child (`.label--disabled`). One state, one modifier, one flat rule — exactly how a theme class is applied. This is a rename-class fix (markup + style together), so flag it unless the caller opts into applying it.

### 5. The only thing that may nest is what is *not itself a class*
A class is never nested inside another class — that is the whole point of rule 4. The things that may nest inside a rule are exactly the things that are **not** selectors of their own; they are states, sub-parts, and conditions *of the current class*, so they belong inside it:

- pseudo-classes and pseudo-elements — anything written with `:` or `::`: `:hover`, `:focus`, `:active`, `:first-child`, `:not()`, `:where()`, `::before`, `::after`. These are not separate elements, they are states/parts of the same class.
- at-rules — `@media`, `@container`, `@supports`. These are conditions on the same class, not new selectors.
- `@at-root` written *by the theme mixin* to emit the light-theme override of a property. This too is not a new class — it is the same class's value under a different theme. A theme mixin of this shape (see the project-overrides example) is therefore allowed and must **not** be flagged as illegal nesting, even though syntactically it produces an `@at-root .light-theme &` block. Treat the mixin call as one declaration of the class.

All of these nest **inside** their block with `&`. This is the one and only place nesting is used: one level deep, and only for things that are not classes. If what you are about to nest is a class selector, it is a violation — unnest it to its own top-level rule.

```scss
.hero__cta {
    margin-top: 50px;

    &:active {
        scale: 0.95;
    }
    &::after {
        content: '';
    }
    @media (max-width: $width-tablet-landscape) {
        margin-top: 24px;
    }
}
```

No nesting deeper than one level. No class-inside-class. A media query or pseudo may not contain another nested block. Do **not** write a pseudo or breakpoint as its own flat top-level rule (`.hero__cta:active { }` standing alone) — nest it under its block.

### 6. Order inside a rule
Within a single rule, declarations and nested groups appear in a fixed order so every rule reads the same way:

1. plain property declarations (`margin`, `color`, `display`, …);
2. then the theme `@include` (the mixin call), if any;
3. then a blank line, followed by pseudo-classes and pseudo-elements (`&:hover`, `&::before`, `&::after`, and any other `&:`/`&::` group);
4. then at-rules — `@media`, `@container`, `@supports`.

```scss
.hero__cta {
    margin-top: 50px;
    @include themeProperty(color, $text-dark, $text-light);

    &:hover {
        opacity: 0.8;
    }
    &::after {
        content: '';
    }
    @media (max-width: $width-tablet-landscape) {
        margin-top: 24px;
    }
}
```

### 7. Never collapse a rule onto one line
Every declaration sits on its own line; the opening brace stays on the selector line and the closing brace on its own line. Do not write `.x { color: red; }` inline, even for a single-declaration rule. This keeps diffs one-property-per-line and every property independently grep-able.

```scss
/* bad */
.hero__title { font-size: 60px; }

/* good */
.hero__title {
    font-size: 60px;
}
```

### 8. No blank lines between top-level rules
Sibling rules sit flush against each other — compact and scannable. A single blank line is allowed **inside** a rule to separate the declaration list (and its `@include`) from its nested pseudo/at-rule groups (see rule 6).

```scss
.hero__title {
    font-size: 60px;
}
.hero__subtitle {
    font-size: 21px;
}
.hero__cta {
    margin-top: 50px;
}
```

### 9. Every value for a class lives inside that class
All of a class's theme and responsive variation stays in its own block — via the theme mixin and nested media queries — never scattered into separate theme blocks or media-query buckets elsewhere in the file. One class, one place to read its full behavior.

```scss
.comparison-title {
    text-align: center;
    @include themeProperty(color, $text-dark, $text-light);

    @media (max-width: $width-phone) {
        font-size: 40px;
    }
}
```

### 10. No comments in stylesheets
Stylesheets carry no comments — neither `//` line comments nor `/* */` blocks. Because every rule is flat, fully spelled out, and one property per line (rules 2–7), the selector and its values already say what they are; section banners (`// Sidebar`, `// Footer`) and explanatory notes add nothing and drift out of sync with the code they sit above. Strip every comment during an audit (safe to auto-apply). The one exception is a license/SPDX header at the very top of a vendor or reset file, which stays verbatim.

```scss
/* bad */
// Sidebar
.hero__sidebar {
    width: 300px; // fixed rail
}

/* good */
.hero__sidebar {
    width: 300px;
}
```

### 11. Whole-pixel values
Pixel values for layout — sizes, spacing, positions, offsets (`width`, `height`, `padding`, `margin`, `top`/`right`/`bottom`/`left`, `gap`, `border-width`) — are whole integers. A fractional `px` is almost always an artifact of a design tool exporting a centered or auto-laid-out element (e.g. `bottom: -344.5px` from a Figma blob centered on a half-pixel grid); it renders blurry and reads as noise. Round to the nearest integer. Safe to auto-apply. This rule is about `px` length values only — it does **not** touch deliberate sub-pixel design tokens in other units or properties (`letter-spacing: 1.2px` tracking, percentage gradient stops like `3.677%`, `em`/`rem`/`%` values), which are intentional and come straight from the design (core rule: read values from the design).

```scss
/* bad */
.hero__glow {
    bottom: -344.5px;
    width: 87.4285px;
}

/* good */
.hero__glow {
    bottom: -345px;
    width: 87px;
}
```

### 12. Fully-rounded corners use percentages, never a magic pixel
For a fully-rounded shape, never fake it with a huge pixel radius (`border-radius: 9999px`, `999px`, `100px` "big enough"). Use a percentage: `50%` for a circle/dot (an element with equal width and height), `100%` for a pill/capsule (a rectangular element whose short ends should be fully round). The percentage scales with the element, so it stays correct if the size changes and reads as intent rather than a magic number. A genuine fixed corner radius (`border-radius: 12px` on a card) is not affected — this rule is only about the "make it fully round" case.

```scss
/* bad */
.avatar {
    width: 24px;
    height: 24px;
    border-radius: 9999px;
}

/* good */
.avatar {
    width: 24px;
    height: 24px;
    border-radius: 50%;
}
```

### 13. No dead or redundant declarations
Every declaration must actually change something. A property set to the value it already has by default — with nothing in the file or a parent to override — is dead weight that reads as intent it does not carry. The common cases:

- `padding: 0` / `margin: 0` on an element whose default is already `0` (a bare `<div>`, a grid/flex container you did not give padding elsewhere) and where no rule, mixin, or reset sets a non-zero value to reset. If nothing sets it, there is nothing to zero out — delete the line.
- a property duplicating the browser default (`display: block` on a `<div>`, `position: static`, `background: transparent`) with no higher-specificity rule to override.
- a property repeated with the same value the block already declares.

Before deleting, confirm there is genuinely nothing to override: grep the class and its element/modifier siblings, the mixins it `@include`s, and any parent rule. A `padding: 0` that cancels a `modalBody()` mixin's `20px 50px` is **not** dead — it does real work. Only flag/remove the ones that override nothing. Safe to auto-apply once confirmed; when the override is ambiguous, flag rather than delete.

```scss
/* bad — nothing sets padding on this grid container, so padding:0 is dead */
.form__confirmation {
    padding: 0;
    display: grid;
    gap: 20px;
}

/* good */
.form__confirmation {
    display: grid;
    gap: 20px;
}
```

### 14. JS never hangs off a style class
A class is a styling name and nothing else. Behavior attaches to a dedicated data attribute — `data-js-<name>` — so the two concerns can change independently: a class can be renamed without breaking a script, and a handler can be moved without touching a stylesheet. A `querySelector('.header')` silently turns every class name into a public API that nobody dares touch, and the coupling is invisible from the stylesheet.

Name the hook after what the script does with the element, in the same kebab-case as a class, and never style it: `data-js-*` appears in no selector in any stylesheet.

```html
<!-- bad — the script owns a styling name -->
<div class="cookie-notice">
<script>document.querySelector('.cookie-notice')</script>

<!-- good -->
<div class="cookie-notice" data-js-cookie-notice>
<script>document.querySelector('[data-js-cookie-notice]')</script>
```

Inside a component framework the component already owns its own DOM, so a ref covers its internal wiring; `data-js-*` is for everything reaching in from outside — analytics, e2e selectors, a third-party script. The rule that matters either way is the second half: a styling class is never the selector.

→ flag a stylesheet rule whose selector contains `[data-js-...]`, and flag JS/test code selecting an element by a styling class where a hook attribute exists or should be added.

### 15. Values come from tokens, not literals
Before writing a literal color, gradient, font, spacing, breakpoint, or page width, search the project's variables/tokens file for one that already holds that value. A literal repeated across files is the same design decision copied by hand — it cannot be changed in one place and cannot be found by searching for its name.

Use the variable when an exact or near-exact match exists, even if it differs trivially (a gradient stop a few percent off, a shade that reads identically). When the closest variable differs *materially* — a visibly different hue, a substantially different size — do not silently substitute it and do not silently hardcode either: surface both options and ask. When there is genuinely no matching token, take the exact value from the design. Defining a new variable is worth it only when the value is reused or is clearly a shared token; a true one-off stays inline.

```scss
/* bad */
.hero__title {
    color: #0957ff;
    font-family: 'Metropolis', sans-serif;
}

/* good */
.hero__title {
    color: $blue;
    font-family: $font;
}
```

### 16. Breakpoints are named variables
Every `@media` width is a named token (`$width-tablet`, `$width-phone`), never a literal `768px` typed per file. Breakpoints are the single most copied value in a stylesheet: typed by hand they drift by a pixel, cannot be audited for consistency, and cannot be changed at all. The name also states *which* device class the rule is for, which a bare number does not.

```scss
/* bad */
@media (max-width: 768px) {
}

/* good */
@media (max-width: $width-tablet) {
}
```

A project that has no breakpoint tokens yet gets the finding flagged, not auto-fixed — inventing the scale is the user's call.

### 17. Mixin or shared block — pick deliberately
Both reuse a chunk of styling, and they are not interchangeable:

- **Shared block** — the default, whenever you control the markup. The styling gets a name that appears in the DOM, so it is greppable and inspectable: `.btn`, `.cta-row`.
- **Mixin** — when you *cannot* put a class on the element: markup owned by a third-party library (a chart tooltip, a date picker, an embedded widget), or a generated structure you do not author. Also fine for a styling fragment that would be meaningless as a DOM name — `ellipsis`, `center-content`, a `placeholder` rule — and as a shared base that several named blocks build on, where each block keeps its own class and the mixin only holds what they genuinely share.

The violation is the middle case: a mixin standing in for a block you *could* have named, so the same visual thing appears under five unrelated class names and nothing in the DOM says they are related. → flag that; leave third-party, fragment, and shared-base mixins alone. Never auto-apply — converting a mixin to a block is a markup change.

### 18. On a static page, the tag carries as much meaning as the class
A hand-written page is structure first: the class says which block a node belongs to, the tag says what the node *is*. Getting the tag right removes code — a `<nav>` needs no `role`, a `<button>` needs no key handler, an `<h2>` needs no font rule repeated from the last section — and it is what makes the page readable in a screen reader, in search results, and in the outline view of an editor.

- Real tags for real things: `header`, `nav`, `main`, `section`, `footer`, `h1`-`h3`, `p`, `ul`/`li`, `a`, `button`, `img`. A `div` is for grouping and layout, not for content that already has an element.
- One `h1` per page, and heading levels that descend without skipping. The outline is the page's table of contents; a heading chosen for its size is a styling decision leaking into structure — size it with a class.
- `a` navigates, `button` acts. Never a `div` with a click handler: it loses focus, keyboard activation, and the browser's own semantics, then needs three attributes to fake them back.
- Accessibility attributes are not an extra pass: `alt` on every `img` (empty `alt=""` only for decorative images, and say so), `rel="noopener noreferrer"` on every `target="_blank"`, `lang` on `html`, a visible focus state, and `aria-*`/`role` only where a native element cannot express the state.

```html
<!-- bad -->
<div class="pricing__section">
    <div class="pricing__title">Plans</div>
    <div class="pricing__link" onclick="go()">Compare</div>
    <img src="plans.png">
</div>

<!-- good -->
<section class="pricing__section">
    <h2 class="pricing__title">Plans</h2>
    <button class="pricing__link" type="button">Compare</button>
    <img class="pricing__image" src="plans.png" alt="Plan comparison table">
</section>
```

→ flag a `div` standing in for a semantic element, a skipped or duplicated heading level, a click handler on a non-interactive tag, and every missing accessibility attribute. Adding an attribute is safe to auto-apply; swapping a tag is a structural change — propose it.

## Fix discipline

- Minimal diff. Preserve the author's formatting, indentation width, and ordering. When unsure, mirror an existing analogous rule in the same file.
- **Renaming a class is never silent.** A rename touches markup and styles together — and any JS still selecting by that class, which is exactly the coupling core rule 14 removes. Flag abbreviation/naming violations and propose the rename, but apply it only when the caller explicitly opts in (`fix --rename`). All other fixes (unnesting, removing tag selectors, collapsing blank lines, reordering within a rule, moving stray values into the class) are safe to apply directly.
- Do not "correct" a project's established spelling or token even if it looks like a typo — match what the repo already uses (a project-overrides layer is the place to pin it).
- **Read values from the design, not from generated code.** When translating a design (Figma etc.) into styles, take every numeric value — sizes, spacing, colors, blur radius, font metrics — from the design's own inspect panel or its variables/tokens, never from the React/Tailwind reference snippet the design tool auto-generates. That generated code routinely mis-converts values (e.g. a Figma Layer Blur of `100` is emitted as `blur-[50px]`). Treat the generated snippet only as a structural hint; the design panel is the source of truth.
- **Don't add comments unless asked, and strip generated ones the change introduced.** This extends core rule 10 (no comments in stylesheets) to *all* code — JS/JSX/TS included. Don't write explanatory comments, section banners, or "what/why" notes in any file unless the user explicitly requests them. Self-describing names and flat structure carry the intent; comments drift out of sync and add noise. A section banner (`// copy`, `// search`) or "what/why" narration (`// re-run after images load so the offset is correct`) sitting in code the current change *adds or edits* is generated noise no one requested — remove it. Unlike the rule 10 stylesheet sweep, this is **scoped to the diff**: strip such comments only from the lines the change touches; leave pre-existing comments in untouched code alone, and never strip a comment a human deliberately wrote as an instruction (`TODO`/`FIXME`/`MIGRATE`/`DEPRECATED` — see the next bullet), a tool directive (`eslint-disable`, `@ts-expect-error`), or a license/SPDX header.
- **A code comment marking something for replacement is an instruction, not decoration.** If a style or component carries a `TODO`/`MIGRATE`/`DEPRECATED`/"use X instead" note on the thing you're about to use (e.g. a legacy `BtnSet layout=` prop annotated `// MIGRATE → props: justify + gap`, or a class marked for removal), do not reach for the old approach — use the new one the note points to. Touching that code is the moment to honor the migration, not extend the legacy path.
- **Open the source of every primitive you use — do not audit from the call site alone.** When markup uses a shared uikit component, prop, or class (`<BtnSet layout=…>`, `<Spinner>`, `.btn`), open that primitive's own source (`.jsx`/`.scss`) and read it before judging the usage. The call site never shows a `MIGRATE`/`DEPRECATED` note living on the prop's class, the primitive's current prop contract (a new `justify`/`gap`/`stackOnMobile` API replacing an old `layout`), or an idiom the primitive forces on its callers (e.g. an inline SVG with no color prop, recolored via `svg path`). Auditing the call site in isolation is how a legacy prop or a stale idiom slips through — the finding, and its correct fix, only exist in the primitive's source.
- After fixing, restate the report so the user sees what changed.

## Reference files

- `references/audit-checklist.md` — the rule-by-rule pass/fail list to run during an audit.
- `references/component-stacks.md` — extra conventions for component frameworks (block folders with one public entry, closed modifier sets). Load only for a component codebase.
- `references/project-overrides.example.md` — template for an optional, opt-in project layer (theme mixin, shared-block list, grandfathered tokens, markup discipline per file type). Copy it into your own project and edit; the core skill stands on its own without it.
