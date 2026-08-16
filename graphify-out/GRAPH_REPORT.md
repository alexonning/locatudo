# Graph Report - locatudo  (2026-08-16)

## Corpus Check
- cluster-only mode — file stats not available

## Summary
- 141 nodes · 327 edges · 9 communities
- Extraction: 94% EXTRACTED · 6% INFERRED · 0% AMBIGUOUS · INFERRED: 21 edges (avg confidence: 0.56)
- Token cost: 575 input · 86 output

## Graph Freshness
- Built from commit: `857371b7`
- Run `git rev-parse HEAD` and compare to check if the graph is stale.
- Run `graphify update .` after code changes (no API cost).

## Community Hubs (Navigation)
- Image Slot Web Component
- Runtime Boot and Setup
- LocaTudo Project Assets
- Template Compilation Support
- Component Tree Walker
- Script Loading and Resolution
- React Attribute Compiler
- Path Resolution Logic
- CSS Utility Functions

## God Nodes (most connected - your core abstractions)
1. `ImageSlot` - 27 edges
2. `get()` - 23 edges
3. `createRuntime()` - 22 edges
4. `boot()` - 12 edges
5. `LocaTudo DC HTML Prototype` - 10 edges
6. `updateHtml()` - 9 edges
7. `walk()` - 9 edges
8. `walkElement()` - 9 edges
9. `walkXImport()` - 9 edges
10. `getReact()` - 9 edges

## Surprising Connections (you probably didn't know these)
- `LocaTudo DC HTML Prototype` --references--> `LocaTudo Constru&CIA Logo`  [EXTRACTED]
  LocaTudo.dc.html → assets/logo-locatudo.png
- `LocaTudo Equipment Rental Marketplace` --references--> `LocaTudo Constru&CIA Brand`  [INFERRED]
  LocaTudo.dc.html → assets/logo-locatudo.png
- `GitHub Sync Notes` --references--> `Instagram Post – Pistola Elétrica (Electric Spray Gun Rental)`  [EXTRACTED]
  github.md → uploads/WhatsApp Image 2026-08-16 at 10.35.09 (1).jpeg
- `LocaTudo Equipment Rental Marketplace` --references--> `Service Area: Sudoeste do Paraná (Capanema & Realeza)`  [EXTRACTED]
  LocaTudo.dc.html → uploads/WhatsApp Image 2026-08-16 at 10.35.10 (1).jpeg
- `GitHub Sync Notes` --references--> `LocaTudo DC HTML Prototype`  [EXTRACTED]
  github.md → LocaTudo.dc.html

## Import Cycles
- None detected.

## Hyperedges (group relationships)
- **LocaTudo Customer Contact Points (Capanema, Realeza, WhatsApp)** — locatudo_contact_capanema, locatudo_contact_realeza, locatudo_service_area [EXTRACTED 0.90]
- **Brand Reference Instagram Posts Used as Prototype Source** — uploads_wimg_pistola_eletrica, uploads_wimg_andaimes, uploads_wimg_instagram_profile, uploads_wimg_logo_splash, uploads_wimg_mais_obra, assets_logo_locatudo_png [EXTRACTED 0.95]
- **LocaTudo Marketplace Screen Flow (Home → Results → Detail)** — locatudo_screen_home, locatudo_screen_results, locatudo_screen_detail [EXTRACTED 1.00]

## Communities (9 total, 0 thin omitted)

### Community 0 - "Image Slot Web Component"
Cohesion: 0.13
Nodes (7): flushNow(), getSlot(), ImageSlot, load(), save(), setSlot(), toDataUrl()

### Community 1 - "Runtime Boot and Setup"
Cohesion: 0.18
Nodes (28): boot(), bundledBlob(), createComponentFactory(), getDC(), Dispatcher(), getError(), createHelmetManager(), applyCanvasBg() (+20 more)

### Community 2 - "LocaTudo Project Assets"
Cohesion: 0.14
Nodes (20): LocaTudo Constru&CIA Logo, GitHub Sync Notes, LocaTudo Constru&CIA Brand, image-slot Web Component, Contact: Capanema (46) 99105-4226, Contact: Realeza (46) 99917-0071, LocaTudo DC HTML Prototype, Google Fonts (Poppins + Plus Jakarta Sans) (+12 more)

### Community 3 - "Template Compilation Support"
Cohesion: 0.13
Nodes (8): compileTemplate(), dcNameFromPath(), encodeCamelAttrs(), encodeCase(), getReactDOM(), Placeholder(), rootNameForDocument(), safeDecode()

### Community 4 - "Component Tree Walker"
Cohesion: 0.31
Nodes (13): collectProps(), contentKey(), cssToObj(), hostPositionStyle(), isDeckMountTag(), kebabToCamel(), renderDeckKids(), walk() (+5 more)

### Community 5 - "Script Loading and Resolution"
Cohesion: 0.24
Nodes (12): cdnScriptFor(), createExternalModules(), ensureBabel(), load(), resolve2(), resolveGlobal(), waitForGlobal(), isElementClass() (+4 more)

### Community 6 - "React Attribute Compiler"
Cohesion: 0.38
Nodes (7): compileAttr(), evalDcLogic(), getReact(), walkFor(), walkIf(), walkText(), warnUnresolved()

### Community 7 - "Path Resolution Logic"
Cohesion: 0.50
Nodes (4): findTopLevelEquality(), parensWrapWhole(), resolve(), resolvePath()

### Community 8 - "CSS Utility Functions"
Cohesion: 1.00
Nodes (3): importantify(), scanUnquotedUrl(), stripComments()

## Knowledge Gaps
- **6 isolated node(s):** `LocaTudo Logo Splash Screen`, `support.js Script`, `Google Fonts (Poppins + Plus Jakarta Sans)`, `Screen: Equipment Detail`, `Screen: Home / Landing` (+1 more)
  These have ≤1 connection - possible missing edges or undocumented components.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `createRuntime()` connect `Runtime Boot and Setup` to `Template Compilation Support`, `Script Loading and Resolution`, `Path Resolution Logic`?**
  _High betweenness centrality (0.041) - this node is a cross-community bridge._
- **Why does `get()` connect `Runtime Boot and Setup` to `Component Tree Walker`, `Script Loading and Resolution`?**
  _High betweenness centrality (0.029) - this node is a cross-community bridge._
- **Are the 7 inferred relationships involving `createRuntime()` (e.g. with `adoptParsed()` and `dcUpdate()`) actually correct?**
  _`createRuntime()` has 7 INFERRED edges - model-reasoned connections that need verification._
- **What connects `LocaTudo Logo Splash Screen`, `support.js Script`, `Google Fonts (Poppins + Plus Jakarta Sans)` to the rest of the system?**
  _6 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Image Slot Web Component` be split into smaller, more focused modules?**
  _Cohesion score 0.1319073083778966 - nodes in this community are weakly interconnected._
- **Should `LocaTudo Project Assets` be split into smaller, more focused modules?**
  _Cohesion score 0.1368421052631579 - nodes in this community are weakly interconnected._
- **Should `Template Compilation Support` be split into smaller, more focused modules?**
  _Cohesion score 0.1286549707602339 - nodes in this community are weakly interconnected._