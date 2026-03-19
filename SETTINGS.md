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

### Padding γύρω από το export (PNG / PDF)
**Αρχείο:** `index.html`
**Μεταβλητή:** `exportPaddingPx` μέσα στη `renderDirectionalViewOutput`
**Τρέχουσα τιμή:** `targetResolution * 0.05` (5% του μεγέθους του export — 200px για 4000px)
**Αναζήτηση:** `exportPaddingPx = Math.round(targetRes *`

Προσθέτει λευκό περιθώριο γύρω από το σχέδιο στο εξαγόμενο PNG/PDF. Αλλάζοντας το `0.05` αλλάζει το ποσοστό margin ως fraction του target resolution.
