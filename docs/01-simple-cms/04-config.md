# 04 \u2014 Config catalogs

| Catalog | Owner | Seed | Read by | Written by |
|---|---|---|---|---|
| `theme` (prefix `thm`) | site | `config/base/site/theme/default.json`, `config/base/site/theme/minimal.json` | `site` (validates `site.theme_name`; returns the effective layout/CSS in `ResolvePublicSite`) | humans, via git |

`ThemeConfiguration` embeds `basable.config.v1.ConfigHeader`; fields `layout_html` (slots `{{site_name}}`, `{{title}}`, `{{nav}}`, `{{body}}`), `css`. Loaded at boot by `basable-config` in one transaction. The system never creates themes; a site *selects* one by name. `delivery` never reads the catalog itself \u2014 it receives the effective values from `site` (Directive \u00a72, \u00a77).
