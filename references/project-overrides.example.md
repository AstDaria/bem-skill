# Project overrides layer (template)

Copy this file into your own project (or into this skill as `references/project-overrides.md`), fill it in, and delete what does not apply. The core skill works without it — this layer only pins the decisions that are specific to one codebase.

## Activation

**Off by default, explicit opt-in only.** The core skill never loads this layer on its own and never auto-detects it from a repo path, a git remote, or file fingerprints — that is brittle and breaks the moment repos are merged or renamed. Apply it only when:

- the skill is invoked with an explicit argument requesting it (`--project-rules`), or
- the user explicitly asks for the project rules.

Without opt-in, audit with the universal core only — never apply a project-specific name, grandfathered token, legacy spelling, or markup discipline to a codebase that did not ask for it.

## Markup discipline per file type

Pick the discipline by **file type**, not by repo — it is intrinsic to the file and survives repo reorganisation.

- `.ejs` / `.html` → semantic-HTML stack: real `section`/`header`/`nav`/`h1`–`h3`/`p`/`a`/`img`/lists, no div soup; accessibility attributes mandatory (`alt` on every `img`, `rel="noopener noreferrer"` on every `target="_blank"`, `lang` on `html`, `aria-*`/`role` where needed); a page root `id` named for the page, never styled.
- `.jsx` / `.tsx` → div-first stack: non-functional wrappers are `<div>`, semantic tags only where function requires them (`input`, `label`, `select`, `textarea`, `button`, `a`, `form`); `id` only for a genuine anchor/`aria`/JS target; conditional modifiers built with the project's class-name helper, not string concatenation.

Adjust or drop either stack to match your project.

## Theme mixin

If the project themes, pin the mixin here and route every themeable color through it — dark value first, light value second (or whatever order the project uses).

```scss
@mixin themeProperty($property, $darkThemeValue, $lightThemeValue) {
    #{$property}: $darkThemeValue;

    @at-root .light-theme & {
        #{$property}: $lightThemeValue;
    }
}
```

The `@at-root` block the mixin emits is part of the mixin, not a nested class — the audit never flags it (core rule 5, checklist rule 3).

**Every themeable color goes through the mixin — even when both theme values are identical.** A raw `color: $blue` carries no theme semantics, so the day the brand color forks per theme, or a `.light-theme` rule of higher specificity lands above it, it silently fails to override. Write `@include themeProperty(color, $blue, $blue)` even when dark and light match. This applies to states too (`&:hover`, `&:focus`, `&:active`). → flag any `color`/`fill`/`background-color`/`border-color` literal in a themed file that is not routed through the mixin. Exceptions: `transparent`/`currentColor` and mechanism values (e.g. `color: transparent` paired with `background-clip: text`), and brand gradients that are intentionally identical across themes.

**Never "fix" an established spelling.** If the project imports a misspelled mixin name, match the import — renaming it breaks the import. Pin the exact token here.

## Shared blocks and grandfathered tokens

List the project's shared blocks explicitly so the audit never tries to fold them into a page block or rename them as abbreviations:

```text
shared blocks: btn, gradient-text, cta-row, nowrap, inner
grandfathered abbreviations: btn (and btn--*)
```

New, non-shared classes still spell words out (core rule 3).

## Keyframes location

`@keyframes` live in one shared animations partial, never inside a component/page stylesheet; the component keeps only the `animation`/`animation-*` properties referencing them by name. Pin the path here, e.g. `styles/_animations.scss`. → flag a `@keyframes` block found anywhere else, move it, leave the usage in place.

## Inline SVG coloring

If icons are inlined `<svg>` with no color prop, they are recolored from the outside with a descendant tag selector:

```scss
.block__icon svg path {
    fill: $icon-color;
}
```

This is the one sanctioned exception to core rule 1 ("style classes only, never tags"): a bare `svg` / `svg path` / `svg circle` / `svg rect` reached off a component class **only to set paint** (`fill`/`stroke`/`color`) is not a violation. The exception is scoped to that — a bare `svg` selector setting layout (`width`, `margin`, `display`) still needs a class, and every other tag selector (`div`, `a`, `button`, …) is flagged as normal. Drop this section if your icons take a color prop.

## Stories / visual coverage

If the project uses Storybook or an equivalent: a style change that adds or alters a visible state (new modifier, variant, responsive/theme behavior) needs a matching story in the co-located `Component.stories.*`, rendering the new state through the component's own prop — not a hardcoded modifier class. → flag a missing or stale story.
