# 04 — Config catalogs

| Catalog | Owner | Seed | Read by | Written by |
|---|---|---|---|---|
| `theme` (prefix `thm`) | site | `config/base/site/theme/default.json`, `config/base/site/theme/minimal.json` | `site` (validates `site.theme_name`; returns the effective layout/CSS in `ResolvePublicSite`) | humans, via git |

`ThemeConfiguration` embeds `basable.config.v1.ConfigHeader`; fields `layout_html` (slots `{{head}}`, `{{site_name}}`, `{{title}}`, `{{nav}}`, `{{body}}`), `css`. Both seeds style the blog index, tag pages and the contact form block. Loaded at boot by `basable-config` in one transaction. The system never creates themes; a site *selects* one by name. `delivery` never reads the catalog — it receives the effective values from `site` (Directive §2, §7).

No other catalogs: rate limits, reserved slugs and the domain re-check policy are code constants, not things an operator edits.
