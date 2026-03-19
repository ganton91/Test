# Milimetre — Tunable Settings

Σημειώσεις για settings που μπορεί να χρειαστεί review και αλλαγή στο μέλλον.

---

## View Panels

### Padding γύρω από το AutoFit
**Αρχείο:** `index.html`
**Μεταβλητή:** `padding` μέσα στη `renderDirectionalViewOutput`
**Τρέχουσα τιμή:** `64` (px από κάθε πλευρά)
**Αναζήτηση:** `const padding = exportOverride ? 0 :`

Ελέγχει πόσο κενό αφήνει γύρω-γύρω το AutoFit μέσα στα side/plan panels. Μεγαλύτερη τιμή = πιο μικρό σχέδιο, περισσότερος αέρας. Το export πάντα χρησιμοποιεί `0`.
