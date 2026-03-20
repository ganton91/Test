# Milimetre — Tunable Settings

Σημειώσεις για settings που μπορεί να χρειαστεί review και αλλαγή στο μέλλον.

---

## View Panels

### Padding γύρω από το AutoFit
**Αρχείο:** `index.html`
**Μεταβλητή:** `padding` μέσα στη `renderDirectionalViewOutput`
**Τρέχουσα τιμή:** `64` (px από κάθε πλευρά)
**Αναζήτηση:** `const padding = exportOverride ? 0 :`

Ελέγχει πόσο κενό αφήνει γύρω-γύρω το AutoFit μέσα στα side/plan panels. Μεγαλύτερη τιμή = πιο μικρό σχέδιο, περισσότερος αέρας.

### Max Zoom των panels (offscreen render scale)
**Αρχείο:** `index.html`
**Μεταβλητή:** `MAX_OFFSCREEN_PX` και το `Math.min(4, ...)` μέσα στη `renderDirectionalViewOutput`
**Τρέχουσα τιμή:** max 8× zoom (canvas έως 8192px)
**Αναζήτηση:** `const MAX_OFFSCREEN_PX =`

Τα panels ζωγραφίζονται μία φορά σε `offscreenScale × CSS size` pixels (όχι περισσότερο από 2048px). Το zoom/pan είναι pure CSS transform χωρίς re-render. Το max zoom είναι πάντα **4×** (sharp ποιότητα μέχρι `offscreenScale ×`, ελαφρά blur μετά). Αύξηση `MAX_OFFSCREEN_PX` → καλύτερη ποιότητα σε μεγαλύτερα panels, βαρύτερος render. Αλλαγή `maxZoom: 4` → αλλαγή max zoom.

### Target Resolution του export (PNG / PDF)
**Αρχείο:** `index.html`
**Μεταβλητή:** `targetResolution` μέσα στη `renderDirectionalViewOutput`
**Τρέχουσα τιμή:** `4000` (px στη μεγαλύτερη διάσταση)
**Αναζήτηση:** `const targetRes =`

Ορίζει τη βασική ανάλυση του εξαγόμενου PNG/PDF. Το export scaling προσαρμόζεται ώστε το content να χωράει σε αυτή την ανάλυση (με αναλογία aspect ratio). Αύξηση → μεγαλύτερο αρχείο, καλύτερη ποιότητα.

### Padding γύρω από το export (PNG / PDF)
**Αρχείο:** `index.html`
**Μεταβλητή:** `exportPaddingPx` μέσα στη `renderDirectionalViewOutput`
**Τρέχουσα τιμή:** `targetResolution * 0.05` (5% του μεγέθους του export — 200px για 4000px)
**Αναζήτηση:** `exportPaddingPx = Math.round(targetRes *`

Προσθέτει λευκό περιθώριο γύρω από το σχέδιο στο εξαγόμενο PNG/PDF. Αλλάζοντας το `0.05` αλλάζει το ποσοστό margin ως fraction του target resolution.
