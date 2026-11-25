# Angular UI Libraries for FamilyLedger: Five Newer Candidates

Requirements: open source, classical and elegant, minimalistic, small download size, and a full set of components including rich dropdowns, tables and charts. Researched 2026-11-25.

## Verdict

None of these five replaces spartan/ui for the "everything in one library" requirement, because none documents a chart component. The two closest are **Guild of Gleks UI** and **ng-oat**, and both are very young. Most size and quality claims below are the authors' own; none were measured independently.

## Comparison

|  | Table | Rich select | Chart | Maturity |
| --- | --- | --- | --- | --- |
| spartan/ui (baseline) | Yes | Yes | Yes | 1.0, Thoughtworks Radar "Assess" |
| Guild of Gleks UI | Unverified | Unverified | Unverified | Very young |
| ng-oat | Yes | Unclear | No | 0.x |
| Frakton NG | Yes | Yes | No | Beta |
| eagami/ui | No | Basic dropdown | No | 0.2.0 |
| Revo UI | Via RevoGrid | Web component dropdown | Unclear | Unclear |

## spartan/ui (baseline)

Links: [website](https://www.spartan.ng) · [documentation](https://www.spartan.ng/documentation) · [repo](https://github.com/spartan-ng/spartan) · [changelog](https://www.spartan.ng/documentation/changelog) · [Thoughtworks Radar](https://www.thoughtworks.com/radar/languages-and-frameworks/spartan-ui)

spartan/ui is the Angular port of shadcn/ui, MIT-licensed. You install accessible, unstyled primitives from npm (`@spartan-ng/brain`, built on the Angular CDK) and copy shadcn-style Tailwind styles (`helm`) into your own codebase, so the look is yours to edit. Version 1.0 shipped in June 2026 with more than 55 components.

**Pros**

- Quiet, neutral shadcn look that suits a classical, low-noise interface.
- Small by construction: you copy in only the components you use, so the bundle grows with what you actually use.
- Covers the stated needs: Select, Combobox, Autocomplete, Command, Table, Data Table, Date Picker, Calendar, Chart, Dialog and Sidebar, plus ready-made blocks for authentication, sidebars and steppers.
- Built on signals and standalone components; zoneless-ready and SSR-compatible. Accessibility (ARIA, keyboard navigation, focus management) is maintained in the brain layer.
- Thoughtworks puts it at "Assess" on its November 2026 Radar and calls it a credible Angular Material alternative. One of its teams used it as the basis for a branded component library across several apps.

**Contra**

- Requires Tailwind CSS, so you adopt Tailwind if you do not use it already.
- Because the styled layer is copied into your repo, you maintain it; upstream fixes do not arrive automatically.
- Not verified: what the Chart component uses underneath, and how the Data Table handles large datasets. Check both before committing.
- A young library compared with Angular Material or PrimeNG; 1.0 is only a few months old.

## Guild of Gleks UI (@guildofgleks/ui)

Links: [docs](https://ui.guildofgleks.com/general/overview) · [repo](https://github.com/GuildOfGleks/gleks_web_ui) · [size comparison](https://ui.guildofgleks.com/general/compare)

**Pros**

- 30 standalone, signal-based, OnPush components themed with plain CSS variables. No CDK, no Material, no Sass build step.
- Size is its main selling point. The author measures the whole 30-component library at 103.8 KB gzipped, versus 153.5 KB for just Angular Material's Button, Select, Dialog and Table (author's own measurement).
- Zero packages beyond Angular, versus 3 for Material and 13 for PrimeNG.
- Theming through CSS variables only, with Slate, One Dark and One Light presets.

**Contra**

- Could not confirm a data table, a rich select or a chart. The readable pages did not list the 30 components. Check the component list directly.
- Supports Angular 21 and newer only (peer dependencies ^21.2.0). The comparison page (about 100 days old when read) shows it peering on Angular 21 while Material and PrimeNG had moved to 22. This may be stale; verify before choosing an Angular version.
- Single entry point: unused code is removed by the bundler rather than never imported.
- Very small project; no adoption signals found.

## ng-oat (@letsprogram/ng-oat)

Links: [package page](https://www.pkgstats.com/pkg:@letsprogram/ng-oat) · [repo](https://github.com/yshashi/ng-oat-workspace) · [Oat CSS base](https://github.com/knadh/oat)

**Pros**

- Angular wrapper over the Oat CSS framework: signals-first, lightweight, accessible.
- Oat itself is about 10 KB of CSS and JS with zero dependencies, the smallest and most minimal base in this comparison.
- Covers Table, Pagination, a Select among its signal-form controls, Dialog, Tabs, Sidebar and alerts. MIT-licensed.

**Contra**

- No chart component listed.
- Oat is sub-v1 and likely to have breaking changes until v1; ng-oat is also 0.x, so you are exposed to both.
- Version 0.6.x targets Angular ^22.0.0.
- Unclear whether its Select is a rich, searchable dropdown or a styled native select.
- Started as a personal project.

## Frakton NG (frakton-ng)

Links: [docs](https://frakton-ng.pages.dev) · [package page](https://www.pkgstats.com/pkg:frakton-ng) · [repo mirror](https://github-users.yhajoua.workers.dev/Samukaii/frakton-ng)

**Pros**

- Select, Autocomplete, Date Picker, Table, Calendar, Paginator, Dialog, Popover, Drawer and Tabs. More complete than ng-oat on forms and navigation.
- Imposes no global styles or CSS resets, so it can sit alongside Angular Material for gradual migration or mixing.
- Claims enforced accessibility, modern architecture and complete visual freedom.
- Claims a package under 1 MB unpacked versus about 8.5 MB for Material and 11.4 MB for PrimeNG. Unpacked size on disk says little about your bundle.

**Contra**

- No chart component.
- Labelled Beta (seen version 21.0.0-beta.4); docs do not confirm supported Angular versions.
- License not stated on the docs page. Check the repo, since open source is a requirement.
- Package page showed 46 downloads, i.e. almost no usage.
- The README compares itself to Material and PrimeNG in a clearly promotional way.

## eagami/ui (@eagami/ui)

Links: [repo](https://github.com/mwiraszka/eagami-design-system) · [package page](https://www.pkgstats.com/pkg:@eagami/ui)

**Pros**

- Lightweight, accessible, built on CSS custom properties, with zero runtime dependencies beyond Angular.
- README claims about 2 KB for a button and about 4 KB for an input, versus 12 KB and 25 KB for Angular Material (author's approximate figures).
- Standalone components; import only what you use. MIT-licensed.

**Contra**

- Too thin for this project. Documented components are Button, Input, Checkbox, Radio, Card, Dropdown and Dialog. No table, chart or date picker.
- Version 0.2.0; requires Angular 21 or later.
- Looks like a one-person project; the package page showed 229 downloads.

## Revo UI (revolist)

Links: [RevoGrid page](https://rv-grid.com/angular-data-grid) · [catalog entry](https://skillsllm.com/skill/revo-ui)

**Pros**

- MIT-licensed companion component library to RevoGrid, a data grid. If a heavy table is the main need, the RevoGrid side is the relevant part; it has an Angular package with standalone-component setup.
- Described as a low-weight, framework-agnostic component library for analytics dashboards and business apps.
- The pages mention search bars, editing forms, settings panels and charts, the only chart-related hint among the five.

**Contra**

- Could not confirm what is actually in the library, or whether the components are Angular-native. The related revo-dropdown is a Stencil web component, so Revo UI probably is too. That matters for Angular forms integration and theming.
- Built around a data-grid product, a different focus from general application UI.

## Recommendation

Money, dates and tables are the core of a family finance app, and a library that rewrites itself between 0.x versions is a real cost over several years. Start with spartan/ui, then Taiga UI, and treat the five above as things to watch. Guild of Gleks is the one to look at first: if its component list includes a proper table and select, its size claim is the most interesting of the five.

**Next step:** check Guild of Gleks' component list and run a bundle-size spike of it against spartan (table, select, chart, then `ng build` and inspect the stats).
