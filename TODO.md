# Milimetre — TODO

---

## Views

- **Per-layout pane direction memory:** κάθε preset (1 Side / 2 Sides / 4 Sides) να θυμάται ανεξάρτητα τα `viewPaneDirections` — σήμερα μοιράζονται το ίδιο array και αλληλοεπικαλύπτονται
  - Implementation: store `viewPaneDirections` per layout key (`"1"`, `"2"`, `"4"`) in state, apply on layout switch

---

## Export / Import

- **Export Styles:** ξεχωριστή action από full project export
  - Εξάγει μόνο style/settings payload (όχι geometry/content)
  - Περιλαμβάνει global visual settings + view visual settings

- **Import Styles:** merge operation (όχι full project replace)
  - Εφαρμόζει imported styles στο τρέχον project, κρατά geometry/content
  - Layer-default mapping rules: generic rules που εφαρμόζονται ανεξάρτητα από layer id (π.χ. `all layers → outline width = 1`)
  - Pre-import snapshot: one-time backup πριν το πρώτο Import Styles της session
  - **Revert To Pre-Import Styles:** επαναφορά στο snapshot — δεν αγγίζει geometry

---

## Renderer

- ~~**Cell Compute → Vector Render migration για Views:**~~
  - ~~Κρατάμε compute pipeline (visibility / occlusion / cut detection / depth ordering)~~
  - ~~Αντικαθιστούμε paint step με vector primitives (paths/polygons/segments) από computed regions~~
  - ~~Διατηρούμε draw-order/depth rules και export consistency (PNG/PDF/DXF)~~
  - ~~Target: cleaner edges, λιγότερα aliasing/striping artifacts, consistency across pane sizes~~
