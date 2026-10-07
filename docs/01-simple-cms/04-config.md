# 04 — Config catalogs

| Catalog | Owner | Seed | Read by | Written by |
|---|---|---|---|---|
| `theme` (prefix `thm`) | site | `config/base/site/theme/default.json` | site (validates `site.theme_name`); delivery, if included | humans, via git |

`ThemeConfiguration` embeds `basable.config.v1.ConfigHeader`; fields `layout_html`, `css`. Loaded at boot by `basable-config` in one transaction. The system never creates themes; a site *selects* one by name.
