# BEM Methodology — an Agent Skill

A skill that teaches an AI agent a strict, search-first BEM dialect and makes it audit and auto-fix CSS/SCSS and markup against it.

Written for hand-built static pages — landings, marketing pages, anything that is markup plus a stylesheet — where the whole job is a small, structured, searchable codebase. The rules hold inside a component framework too; the extra conventions a component stack needs live in a separate file you load only for that.

Works with Claude Code, OpenAI Codex, Cursor, Windsurf, and any agent that supports the [Agent Skills spec](https://agentskills.io/).

## Why

The goal is less code — fewer classes, fewer selectors, fewer lines to read — so that a bug is found by searching for one name instead of by reading a whole file.

Utility-first and component frameworks pull in the opposite direction. A Tailwind element carries a dozen single-purpose classes inline, so markup becomes unreadable and the same visual decision is repeated at every call site with nothing to grep for; Bootstrap ships a large stylesheet of generic names (`col-md-6`, `d-flex`, `text-center`) that describe presentation rather than the thing being styled, and overriding it means fighting its specificity with extra code of your own. Both leave you with markup you cannot scan and styles you cannot search.

This dialect makes the opposite trade:

- **one name, one place.** Every block, element and modifier is a flat top-level rule — searching `.card__title` lands you on the rule that styles it, always.
- **every name is greppable.** No abbreviations, so a class found in markup is the same string in the stylesheet and in JS. One search shows every use.
- **markup stays readable.** An element carries one or two meaningful class names that say what it is, not a stack of utilities that say how it looks.
- **nothing dead stays.** Comments, redundant declarations, duplicated values, blank-line noise and magic numbers are stripped during an audit, so what remains is only code that does something.

The result is a stylesheet where the number of lines tracks the number of real design decisions, and where reading a class name tells you what it belongs to.

### The same button, both ways

```html
<!-- utility-first: the styling is the markup -->
<button class="inline-flex items-center justify-center gap-2 rounded-full
               bg-blue-600 px-6 py-3 text-sm font-semibold text-white
               shadow-sm transition hover:bg-blue-700 focus-visible:outline
               focus-visible:outline-2 focus-visible:outline-offset-2
               disabled:cursor-not-allowed disabled:opacity-50">
    Start build
</button>
```

Nothing here is searchable. To find every primary button you grep for a color class that also matches a dozen unrelated elements; to change the brand blue you edit every call site or build a `@apply` layer that quietly reintroduces a stylesheet. The variant is not named anywhere, so nobody can tell whether this button is the same component as the one three files over.

```html
<!-- named: the markup says what it is -->
<button class="button button--primary" data-js-build-start>Start build</button>
```

```scss
.button {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    padding: 12px 24px;
    border-radius: 100%;
    font-size: 14px;
    font-weight: 600;
    transition: background-color 0.2s;

    &:disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }
}
.button--primary {
    background-color: $blue;
    color: $white;

    &:hover {
        background-color: $blue-hover;
    }
}
```

Three searchable names. `button--primary` finds every primary button in the project; the hover colour is defined once; the markup line is short enough to read at a glance; and the `data-js-` attribute means a script can grab the element without the styling class becoming an API that nobody dares rename.

### What about CSS Modules and styled-components?

They are closer in spirit — both keep a style attached to the component it belongs to, which is the same instinct as a block. The difference is what they solve. They solve **collisions** through isolation: the tool generates a unique name (`Button_root__a3f91`), so two components can both call something `.root` and never clash.

That is a real problem, solved well. But isolation is not the same as findability, and it is the second one this skill is after. A generated name cannot be grepped — the string in the DOM is not the string in your source, so debugging in devtools means going through a sourcemap instead of a text search. Scoping also removes the pressure to name things precisely: once a collision is impossible, `.wrapper`, `.container` and `.inner` reappear in every file, and a name that means nothing outside its own 40 lines tells you nothing when you meet it in a stack trace or a screenshot.

BEM solves collisions with discipline rather than tooling — a globally unique name you wrote yourself — and gets findability for free, at the cost of typing a longer class. If your project already runs on CSS Modules, these rules still apply inside it: the block/element/modifier naming, the flat selectors and the no-nesting rule all work unchanged, and you keep the scoping as a safety net on top.

## What it does

Point the agent at `.scss`, `.css`, `.ejs`, `.html`, `.jsx` or `.tsx` files and it reports every violation as `file:line — rule — before → after`, applying the safe fixes and flagging the ones that need a rename across markup, styles and JS at once.

Three modes:

- `audit` — report only, change nothing
- `fix` — report and apply minimal-diff fixes (default)
- `guide` — no scan, just keep the rules in context while writing new code

## The dialect

Standard BEM from [bem.info](https://bem.info/) — `block`, `block__element`, `block--modifier` — plus two opinionated additions:

- **zero abbreviations**, so every class is findable with a plain text search (`.nav__button`, never `.nav__btn`)
- **zero nesting of classes**, so a selector lives exactly at its own name — the only things that may nest are pseudo-classes, pseudo-elements and at-rules

On top of that the skill enforces 18 rules: classes only (no tag or id selectors), no element chaining, flat selectors with no descendant/child/`:has()` combinators, a fixed declaration order inside a rule, no single-line rules, no blank lines between sibling rules, no comments in stylesheets, whole-pixel layout values, percentage radii for fully-rounded shapes, and no dead declarations.

Four of them keep a class name a *styling* name and nothing else, and keep the page itself small:

- **behavior lives on `data-js-*` attributes**, never on a class — so a class can be renamed without breaking a script, and a handler can move without touching a stylesheet
- **values come from tokens**, not literals — including every `@media` width, which is a named breakpoint variable rather than a `768px` retyped per file
- **the tag carries as much meaning as the class** — real `section`/`h2`/`button`/`nav`, a heading outline that descends, and the accessibility attributes that come with them; a correct tag deletes the code needed to fake it
- **a mixin is the exception, a named block is the rule** — reuse something under a name you can see in the DOM, unless the markup is not yours to name

## Install

Claude Code — clone into your skills directory:

```bash
git clone https://github.com/AstDaria/bem-skill.git ~/.claude/skills/bem
```

Or drop the folder into a project at `.claude/skills/bem/`. Then invoke it with `/bem`, or just ask the agent to "fix these styles to BEM".

## Project overrides

Every codebase has a few conventions BEM itself does not cover — a theme mixin, a list of shared blocks, a grandfathered token, a markup discipline per file type. `references/project-overrides.example.md` is a template for that layer. Copy it, fill it in, and point the agent at it — it stays **off by default** and is never auto-detected, so the core rules work the same in any repo.

## Files

| File | What it is |
|---|---|
| `SKILL.md` | the skill itself — modes, the 18 rules, fix discipline |
| `references/audit-checklist.md` | the rule-by-rule pass/fail list run during an audit |
| `references/component-stacks.md` | extra conventions for React/Vue/Svelte codebases — load only for those |
| `references/project-overrides.example.md` | template for an optional, opt-in project layer |

## License

[MIT](LICENSE) — use it however you want.
