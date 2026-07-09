# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single-page prototype: **`index.html`** — a Traditional Chinese (zh-TW) directory app that helps families find senior-care services and connects them to real referral businesses (BNI members). It's a home screen with a static list of category cards, plus a dedicated detail screen per category that features one or more of that business's products/contact info.

There is no build system, package manager, linter, or test suite in this repo — just:

- `index.html` — the app itself (markup + view logic). Deployed as-is via GitHub Pages (repo has a `.nojekyll` file and a `main`-branch/root Pages source; the file must stay named `index.html` for Pages to serve it as the default document).
- `support.js` — a vendored runtime, **generated code, do not hand-edit** (see header comment: "GENERATED from dc-runtime/src/*.ts — do not edit. Rebuild with `cd dc-runtime && bun run build`"). The `dc-runtime` source project is not part of this repo, so there is no local way to rebuild it — treat `support.js` as read-only.
- `assets/` — ad images/flyers from referral businesses (e.g. `bank.jpg`, `teeth.jpg`), each cropped into the "精選服務" card on that category's detail screen (see Navigation model below).

## Running it

The runtime `fetch()`es its own document (`location.href`) on boot to pick up the raw template, and `x-import`/`dc-import` sibling components are loaded via `fetch('./<name>.dc.html')`. Opening the file directly as `file://` can break these fetches in some browsers, so serve it over a local static server instead, e.g.:

```bash
python3 -m http.server 8000
# then open http://localhost:8000/index.html
```

There is nothing to build or install first — `support.js` pulls React 18 / ReactDOM UMD (and Babel standalone, only if an `x-import` needs JSX) from `unpkg.com` at runtime.

## Architecture: the "DC" (Design Component) format

This is a JSX-free, build-free way of writing a small React app in one `.dc.html` file. `support.js` is a self-booting runtime that:

1. Loads the React/ReactDOM UMD builds from CDN.
2. Parses the `<x-dc>...</x-dc>` block in the document into a template + a logic class.
3. Compiles the template into `React.createElement` calls and mounts it into a `<div id="dc-root">` that replaces `<x-dc>`.

### Template syntax (inside `<x-dc>`)

- `{{ expr }}` — interpolation/attribute binding. Supports dotted/indexed paths (`a.b`, `a[0]`), literals, `!x`, and `==`/`===`/`!=`/`!==` comparisons — **not** general JS expressions.
- `<sc-if value="{{ expr }}">...</sc-if>` — conditional block. `hint-placeholder-val="{{ ... }}"` is a hint used only while content is still streaming in (from an external authoring tool), not runtime logic you need to maintain by hand.
- `<sc-for list="{{ expr }}" as="item">...</sc-for>` — list rendering; `item` (or the chosen `as` name) and `$index` become available in the loop body.
- `ref="{{ someFn }}"` — calls `someFn(domNode)`, mirroring a React ref callback.
- Event attributes (`onClick`, `onInput`, etc.) bind directly to methods returned from the logic class's `renderVals()`.
- `<helmet>...</helmet>` — head content (styles/links/meta) to be hoisted into `<head>`; used once at the top of the file for global CSS and Google Fonts.
- `<x-import>` / `<dc-import>` — pull in another component (from a global, an ES module URL, or a sibling `<name>.dc.html` file). Not used in this app currently, but the runtime supports it if the directory grows into multiple `.dc.html` files.

### Logic (inside `<script type="text/x-dc" data-dc-script>`)

```js
class Component extends DCLogic {
  state = { view: 'home' };          // like React class component state
  someHandler = () => { this.setState({ view: 'x' }); };
  renderVals() {                     // flat object the template binds against
    return { isHome: this.state.view === 'home', someHandler: this.someHandler, ... };
  }
}
```

- `DCLogic` (aka `StreamableLogic`) gives you `this.state`, `this.setState(update, cb)`, `this.forceUpdate()`, and lifecycle hooks (`componentDidMount`, `componentDidUpdate`, `componentWillUnmount`).
- `renderVals()` is called on every render; its return value is the single flat "vals" object the whole template resolves `{{ }}` expressions against — anything the template needs (booleans for `sc-if`, data for text, handlers for events, ref callbacks) must be returned from here.
- There's exactly one `Component` class per `.dc.html` file/root.

### This app's navigation model

Everything lives in one `Component` with `state.view` acting as a simple router. There is no generic/shared detail template — each category has its own named view (`'home' | 'lingowave' | 'eplushome' | 'insurance' | 'beauty' | 'bank' | 'dentist'`) and its own `open*` handler (e.g. `openBank`) that just calls `setState({ view: '...' })`. Screens are toggled via `sc-if` blocks in the template rather than any real router. The home screen is a static list of category cards — there is no search box and no provider sign-up CTA.

Every detail screen follows the same layout: a big icon + `<h1>` title + one-paragraph intro, then a white "精選服務" card (a cropped banner from that business's ad in `assets/`, business name, one-line pitch, and a `tel:` call button), then a generic "聯絡我們" contact block. A category can feature more than one product from the same business — `insurance` stacks two 南山人壽 policy blocks (each with its own image/name/pitch) inside one card and shares a single `tel:` button at the bottom, since both share the same company phone number. When adding a new category, follow this pattern: add a home-screen card and a new `open*` handler/state name, then add a matching `sc-if` detail block with its own `assets/*.jpg` banner and `tel:` number.
