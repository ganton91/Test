# Milimetre — TODO

---

## Views

- **Per-layout pane direction memory:** κάθε preset (1 Side / 2 Sides / 4 Sides) να θυμάται ανεξάρτητα τα `viewPaneDirections` — σήμερα μοιράζονται το ίδιο array και αλληλοεπικαλύπτονται
  - Implementation: store `viewPaneDirections` per layout key (`"1"`, `"2"`, `"4"`) in state, apply on layout switch

- **Multi-section axes μέσα σε View Box**

  **Τι κάνει:** Στο View Properties ο χρήστης ορίζει επιπλέον section planes μέσα στο view box. Κάθε section γίνεται νέα επιλογή στον pane direction selector (μαζί με τα υπάρχοντα T-B / B-T / L-R / R-L / PL). Έτσι μπορεί να βλέπει πολλές τομές από διαφορετικές θέσεις μέσα στο ίδιο view.

  ---

  **1. Data model — τι αποθηκεύεται στο `view` object:**

  ```js
  view.sectionAxes = {
    z: [
      { id, elevation }  // meters, snapped — οριζόντια τομή σε συγκεκριμένο ύψος
    ],
    x: [
      { id, fromSide, distance, direction }
      // fromSide: "left" | "right"
      // distance: meters από αυτή την πλευρά του view box
      // direction: "leftToRight" | "rightToLeft"
    ],
    y: [
      { id, fromSide, distance, direction }
      // fromSide: "top" | "bottom"
      // distance: meters από αυτή την πλευρά του view box
      // direction: "topToBottom" | "bottomToTop"
    ]
  }
  ```

  Κάθε axis έχει μοναδικό `id` ώστε να μπορεί να αντιστοιχιστεί σε pane.

  ---

  **2. View Properties modal — UI αλλαγές:**

  Προσθήκη τριών collapsible groups κάτω από το υπάρχον `Plan Elevation`:
  - **Z Sections:** λίστα από elevation values — κουμπί `+` προσθέτει νέο, κάθε entry έχει elevation input + delete
  - **X Sections:** λίστα — κάθε entry έχει: `From` dropdown (Left / Right) + `Distance` input (meters) + `Direction` dropdown (L→R / R→L) + delete
  - **Y Sections:** λίστα — κάθε entry έχει: `From` dropdown (Top / Bottom) + `Distance` input (meters) + `Direction` dropdown (T→B / B→T) + delete

  ---

  **3. Pane direction selector — επέκταση:**

  Σήμερα ο selector δείχνει: `T-B | B-T | L-R | R-L | PL`

  Με τη νέα λογική δείχνει επίσης τα user-defined sections με label π.χ.:
  - `Z 2.40m` (Z section στα 2.40m)
  - `X 3m←` (X section 3m από αριστερά, κοιτά L→R)
  - `Y 5m↓` (Y section 5m από πάνω, κοιτά T→B)

  Το label παράγεται αυτόματα από τα στοιχεία κάθε axis.

  ---

  **4. Render — αρχιτεκτονική αλλαγή:**

  Σήμερα το `buildDirectionalOcclusionGrid(view, direction)` κόβει **πάντα στο front boundary του view box** (το άκρο). Αυτή η τιμή (`frontBoundaryValue`) είναι hardcoded στο `bounds[config.depthAxis === "y" ? "minY"/"maxY" : "minX"/"maxX"]`.

  Για extra X/Y sections: το `frontBoundaryValue` παραμετροποιείται. Αντί για το άκρο, δίνεται η θέση του section axis σε cells (μετατροπή από meters + view box origin). Η υπόλοιπη λογική (projection, `isCut`, `isCutGeometry`, occlusion, depth) παραμένει **ακριβώς η ίδια**.

  Για extra Z sections: χρησιμοποιείται το υπάρχον `buildPlanOcclusionGrid` με διαφορετικό `planElevation` — δεν χρειάζεται καμία αλλαγή στη συνάρτηση.

  Πρακτικά: η `renderDirectionalViewOutput` θα δέχεται ένα `sectionOverride` αντικείμενο που περιγράφει αν το pane είναι standard direction ή user-defined section, και τι `frontBoundaryValue` ή `planElevation` να χρησιμοποιήσει.

  ---

  **5. Σειρά υλοποίησης:**
  1. Data model στο `view` object + serialize/deserialize
  2. View Properties modal UI (τα τρία groups)
  3. Pane direction selector — επέκταση για να δέχεται section ids
  4. `renderDirectionalViewOutput` — `sectionOverride` parameter
  5. `buildDirectionalOcclusionGrid` — παραμετροποίηση `frontBoundaryValue`
  6. Labels + export filenames για τα section panes

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
