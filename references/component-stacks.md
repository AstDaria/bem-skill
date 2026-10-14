# Component stacks (optional)

Load this file only for a component-framework codebase — `.jsx`, `.tsx`, `.vue`, `.svelte`. A static-page stack (`.html`, `.ejs`, and a stylesheet) ignores it: the core rules in `SKILL.md` are complete on their own.

Nothing here replaces a core rule. These are the two conventions a component stack needs on top, plus how the hook rule lands there.

## A block is a folder with one public entry

In a static page a block is a name; in a component stack it is a directory holding everything the block owns, exposing one public entry point. Outside code imports the folder, never a file inside it — that is what lets the internals be renamed or split without touching a call site.

```text
Button/
    Button.jsx            markup + behavior
    Button.scss           the .button block, its elements and modifiers
    Button.const.js       the modifier sets (below)
    Button.stories.jsx    one story per visible state
    index.js              the only public surface: re-exports the component + its const
```

→ flag a deep import that bypasses the entry point (`from '…/Button/Button'` instead of `from '…/Button'`), a stylesheet that is not co-located with the component it styles, and a block whose styles sit in a shared file for no reason.

## Modifiers are a closed, declared set

A component's modifiers are enumerated in one place and exported, not passed as free-form strings at the call site. A closed set makes the variants discoverable from the component's own source, lets a linter catch a typo, and turns a modifier rename into one edit instead of a project-wide grep.

```js
export const BUTTON_THEME = {
    DEFAULT: 'default',
    PRIMARY: 'primary',
    GHOST: 'ghost',
}
```

The component maps the value to its class (`button--primary`) with the project's class-name helper; the call site passes `BUTTON_THEME.PRIMARY`, never `"primary"`.

→ flag a modifier passed as a raw string **where the component already declares a set**, and a variant class assembled by string concatenation. A component that has never had a declared set is a design suggestion, not a violation — raise it once, do not flag every call site.

Also worth reporting, but as information rather than a finding: a modifier present in the stylesheet but absent from the declared set, and a declared variant with no matching rule (dead).

## How the hook rule applies here

Core rule 14 says a styling class is never a JS selector. Inside a component that owns its own DOM, a ref is the natural mechanism and no attribute is needed. `data-js-*` is for anything reaching in from **outside** the component — analytics, e2e selectors, a third-party script — where a styling class would otherwise become a public API.

→ flag a `querySelector` or test selector built on a styling class; do not demand a `data-js-*` attribute where a ref already does the job.

## Markup discipline

Core rule 18 (semantic tags, accessibility) is written for hand-authored pages. In a component stack the usual trade is div-first: non-functional wrappers are `<div>` so that no tag-based style can reach them, and semantic tags appear where function requires them (`input`, `label`, `select`, `textarea`, `button`, `a`, `form`) or where the component renders real page content. Accessibility attributes stay mandatory either way.

Pin which way your project goes in `project-overrides.md`; without an explicit project rule, follow the file's existing convention rather than imposing one.

## Checklist additions
1. **Deep import past the block entry point** — importing a file inside a block folder (`…/Button/Button`) instead of the folder itself (`…/Button`), or a block whose stylesheet is not co-located with the component it styles. → import the entry point; move the stylesheet next to its component. (see above.)
2. **Free-form modifier string** — a variant passed at the call site as a raw string (`theme="primary"`) where the component declares a modifier set, or a variant class assembled by string concatenation. → pass the declared constant and map it to the class with the project's class-name helper. Also flag a modifier that exists in the stylesheet but not in the declared set, and a declared variant with no matching rule (dead). (see above.)
3. **Div-first vs semantic wrapper** — a semantic tag used as a non-functional wrapper in a div-first project, or a `div` where the component renders real page content in a semantic project. → follow the project's stated discipline; where none is stated, follow the file.
4. **Conditional modifier built by string concatenation** — a toggled modifier class assembled inline where the project uses a class-name helper. → use the helper.
5. **No stray ids** — the application UI barely uses `id`. Unlike a static page, a component has no page-level analytics id. → flag any `id` that is not a genuine anchor/`aria`/JS target; default to a class.
6. **Story covers the style change** (only where the project uses stories) — a style change that adds or alters a visible state (new modifier, variant, responsive/theme behavior) with no matching story in the co-located `Component.stories.jsx`, or a component changed that has no `.stories.jsx` at all. → add/update a story that renders the new state, toggled via the component's own prop, not a hardcoded modifier class; flag if a story file must be created.
