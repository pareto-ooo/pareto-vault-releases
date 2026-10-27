---
id: prototype
scope: scoped
severity: warning
audience: all
schemaVersion: 1
---

# Prototype build — Pareto discovery mockups

A discovery prototype shows the client the screens and flows agreed in Sessions 1–4 so they can be validated before the architecture is chosen. Every prototype is built on the Pareto design base, so each project shows the Pareto look and feel from the first mockup. It is thrown away after discovery. DO NOT use this document for developing real software. Prototypes only.

## UI kit

- Package: [`@pareto4/ui-vue`](https://www.npmjs.com/package/@pareto4/ui-vue) on public npm — always the latest version (`npm view @pareto4/ui-vue dist-tags.latest`). No registry setup or credential is needed.
- Scaffold: `npx --package=@pareto4/ui-vue@latest pareto-ui prototype <dir>`
- Docs in the package: `AGENTS.md` § "HTML prototypes", `COMPONENTS.md`, `demo/pages/`

## Output

- The scaffold writes `index.html` and `ui-kit.js`. `ui-kit.js` is self-contained — Vue, every `Ui*` component, the Lucide icons, the Pareto logos and the Tailwind runtime with the Pareto theme — so `index.html` opens from disk without a build step or a network request.
- One page template per screen: `<script type="text/x-template" id="scr-<nn>">` for `SCR-<nn>`, reachable from the sidebar and linkable on its own.
- Never inline `ui-kit.js` into the HTML, and never hand-write HTML or CSS for something a `Ui*` component already does.

## Language

Everything is delivered in English: screen titles, labels, buttons, messages, empty and error states, dialogs, sample-data text and the README. Only proper nouns in the sample data — company, person and place names — may be Dutch.

## Styling

- Layout with `UiAppShell` and `UiSidebar`; pages with `UiPage` or `UiEditPageHeader`; inputs, buttons, `UiDataTable` and `useDialog()` from the kit. Look up props, v-models, events and slots in `COMPONENTS.md`; do not guess them.
- Colours only through the kit's semantic tokens (`bg-card`, `bg-surface`, `text-text`, `text-muted`, `border-border`, `text-accent`, `bg-btn-primary`), never raw Tailwind colours such as `bg-white`. Dark mode then works by itself.
- Icons from the kit's `icons` (Lucide, by PascalCase name). Logos from `assets.logo` and `assets.logoIcon`.
- Write templates exactly as they would be in a `.vue` file, so a prototype page can later move into the real app almost unchanged.

## Layout and navigation

- The sidebar follows Session 4's **Navigation Model**: one `NavItem` per main section.
- Start from the closest page in the kit's `demo/pages/` — list page, editor page, forms page.

## Interaction level

- The flows in Session 3's **Prototype Validation Plan** are clickable end to end, including the error paths it names.
- Loading, empty and error states are visible states you can switch to. Confirmations and destructive actions go through `useDialog()`.
- Everything outside those flows may stay static.

## Sample data

Fictional business data that fits the client's domain in a plain `reactive` store, realistic in volume (a table shows 20–50 rows, not 3). No real customers, employees, addresses or EAN codes, and nothing copied from client documents. A reset action restores the sample data.

## Out of scope

No API calls, no authentication, no storage beyond the page, no analytics. The prototype never decides architecture, security or performance questions — those belong to Sessions 5 and later.