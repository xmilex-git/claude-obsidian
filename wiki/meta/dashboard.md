---
type: meta
title: "Dashboard"
created: 2026-04-08
updated: 2026-04-24
tags:
  - meta
  - dashboard
status: evergreen
related:
  - "[[index]]"
  - "[[overview]]"
  - "[[log]]"
  - "[[concepts/_index]]"
---

# Wiki Dashboard

Navigation: [[index]] | [[overview]] | [[log]] | [[hot]]

The dashboard uses **Obsidian Bases**. A core Obsidian feature shipped in v1.9.10 (August 2025). No plugin install required.

> [!tip] Embedded Bases view
> The interactive dashboard lives in [[dashboard.base]]. Open that file directly, or use the embed below.

![[dashboard.base]]

---

## Legacy Dataview Dashboard (Optional)

If you are on Obsidian < 1.9.10 or prefer Dataview, the queries below still work. Just install the Dataview community plugin.

### Recent Activity

```dataview
TABLE type, status, updated FROM "wiki" WHERE !contains(file.folder, "_legacy") SORT updated DESC LIMIT 15
```

### Seed / Developing Pages

```dataview
LIST FROM "wiki" WHERE (status = "seed" OR status = "developing") AND !contains(file.folder, "_legacy") SORT updated ASC
```

### CUBRID Sources

```dataview
TABLE date_ingested, updated FROM "wiki/sources" WHERE type = "source" SORT updated DESC LIMIT 10
```

### Recent Components

```dataview
TABLE parent_module, status, updated FROM "wiki/components" WHERE type = "component" SORT updated DESC LIMIT 15
```

### Flows

```dataview
LIST FROM "wiki/flows" WHERE type = "flow"
```

### Lint reports

```dataview
LIST FROM "wiki/meta" WHERE contains(file.name, "lint-report") SORT file.name DESC LIMIT 5
```
