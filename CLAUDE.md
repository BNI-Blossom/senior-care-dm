# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single-page prototype: **`index.html`** — a Traditional Chinese (zh-TW) directory app that helps families find senior-care services and connects them to real partner businesses. The home screen lists services in three groups; tapping one opens a detail sheet featuring that business's products and a fixed bottom contact bar.

There is no build system, package manager, linter, or test suite in this repo — just:

- `index.html` — the app itself (markup + view logic). Deployed as-is via GitHub Pages (repo has a `.nojekyll` file and a `main`-branch/root Pages source; the file must stay named `index.html` for Pages to serve it as the default document).
- `support.js` — a vendored runtime, **generated code, do not hand-edit** (see header comment: "GENERATED from dc-runtime/src/*.ts — do not edit. Rebuild with `cd dc-runtime && bun run build`"). The `dc-runtime` source project is not part of this repo, so there is no local way to rebuild it — treat `support.js` as read-only.
- `assets/` — ad images/flyers from referral businesses (e.g. `bank.jpg`, `teeth.jpg`), shown uncropped (natural aspect ratio) at the top of the "精選服務" card on that category's detail screen (see Navigation model below). Not every category has one — a business without supplied art gets a text-only card (see `inheritance`). Source flyers arrive at full marketing resolution (1000px+ wide, hundreds of KB) — downscale to ~900px width and recompress (e.g. `sips --resampleWidth 900 -s formatOptions 40 in.jpg --out out.jpg`) before committing, and update the `<img>`'s `width`/`height` attributes to the new pixel dimensions so the browser can reserve layout space before the image loads.

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

### This app's structure

The home screen is data-driven. Everything lives in one `Component`: `services` (an object keyed by id) holds each category's title, home blurb, intro, `providers` (name, pitch, `imgs`) and `actions` (`tel` / `line` / `apple` / `android`); `groupDefs` groups the ids into the three home sections (照護與生活 / 健康 / 財務與保障). `renderVals()` flattens these into the `groups` / `detail` objects the template loops over with nested `sc-for`.

There are no per-category templates and no `navigate()` router. `state.openId` is `null` on the home screen; tapping a `.tile` calls `openService(id)`, which `history.pushState`s and renders one shared full-screen `.sheet` (back button, icon, `<h1>`, intro, provider cards with their `assets/` images shown uncropped, a "有其他問題？" block linking to the app's own LINE `https://lin.ee/95a4T6s`, and a fixed bottom `.actionbar`). `popstate`, Escape and the 返回 button all close it; `document.body.style.overflow` is locked while it is open.

**Bottom action bar:** every service's contact buttons (phone and/or LINE, or app-store links for LingoWave) live in `.actionbar`, fixed at the bottom of the sheet so they are visible without scrolling. The app's own LINE stays in the body block, styled differently, so the two don't read as contacting the same party. A category can feature several providers/images (`insurance` and `bank` stack the rep's business card above the company ad) with one shared `tel:` action that dials the individual rep's line; for an extension use a comma in the `href` (`tel:0355963251,210`).

**Icons:** the template can't inject raw SVG, so icon markup lives in the `icons` map and `fillIcons()` writes it into every `<svg data-icon="key">` after each render. Add the key there when adding a service.

**Copy:** the site doesn't mention BNI anywhere in the UI — keep it that way.

**Adding a category:** add an entry to `services`, its id to a `groupDefs` group, an icon in `icons`, and (if it has art) a downscaled `assets/*.jpg`.

Responsive layout: tiles are a single-column list on phones; at ≥900px a sticky group nav appears on the left with a two-column tile grid on the right.
