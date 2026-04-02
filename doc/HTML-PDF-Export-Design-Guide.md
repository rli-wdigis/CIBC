# HTML-to-PDF Export — Design Guide

> **Purpose:** This is a reusable reference for designing and building branded HTML-to-PDF export templates integrated with Power Automate. Use this as the baseline schema for any new export (e.g., DRR Export, DSA Export, EDI Export).

---

## 1. Overview & Pattern

Each export consists of:

| Component | Description |
|---|---|
| **Master file** | A single self-contained `*-main.html` — the full rendered document. Used for standalone preview and as the source of truth. |
| **Compose chunks** | The master split into 8 modular HTML fragments, one per Compose action in Power Automate. |
| **Assembly expression** | A final `Compose_FullHtml` action that `concat()`s all chunks into the complete HTML string for the PDF action. |

---

## 2. File Naming Convention

| File | Pattern | Example |
|---|---|---|
| Master file | `{export-id}-main.html` | `drr-export-main.html` |
| HTML open (CSS + head) | `{export-id}-HtmlOpen.html` | `drr-export-HtmlOpen.html` |
| Header chunk | `{export-id}-HtmlHeader.html` | `drr-export-HtmlHeader.html` |
| Section chunks | `{export-id}-Section{N}_{Name}.html` | `drr-export-Section1_AppDetails.html` |
| Footer chunk | `{export-id}-HtmlFooter.html` | `drr-export-HtmlFooter.html` |

All files live in the export's own folder (e.g., `DRR Export/`, `DSA Export/`).

---

## 3. CSS Architecture

All CSS is self-contained in the `HtmlOpen` chunk / master `<head>`. No external stylesheets or fonts.

### 3.1 Color Tokens (CSS Variables)

```css
:root {
  --brand:          rgb(169, 29, 65);   /* Primary brand color */
  --brand-dark:     rgb(118, 18, 44);
  --brand-light:    rgb(208, 55, 93);
  --brand-subtle:   rgba(169, 29, 65, 0.06);
  --brand-border:   rgba(169, 29, 65, 0.16);
  --text-primary:   #111827;
  --text-secondary: #374151;
  --text-muted:     #9ca3af;
  --border:         #e5e7eb;
  --surface:        #f9fafb;            /* Shaded row background */
  --white:          #ffffff;
}
```

> **To rebrand:** update only the `--brand*` variables in `HtmlOpen`.

### 3.2 Page Layout

- Fixed max-width `820px`, centred with drop shadow (screen preview).
- `@page` rule: A4, `10mm` margins on all sides (print/PDF).
- `@media print` overrides reduce padding and font size for tighter PDF output.

### 3.3 Field Grid System

All data fields use a 2-column CSS Grid layout:

```
.field-grid              → default 2 columns (1fr 1fr)
.field-grid.cols-1       → single column
.field-grid.cols-3       → 3 columns
.field.shaded            → alternate row shading (apply to both cells of the same row)
.field.span-full         → field spans both columns (grid-column: 1 / -1)
.field.field-last        → removes bottom border on the last row
```

**Row shading rule:** Apply `.shaded` to **both** cells of an even row to achieve horizontal banding. Never shade individual cells (creates checkerboard).

### 3.4 Section Structure

```html
<div class="section [no-break]">
  <div class="section-header">
    <div class="section-icon"><!-- emoji or SVG --></div>
    <span class="section-title">Section Name</span>
  </div>
  <div class="field-grid">
    <!-- .field divs -->
  </div>
</div>
```

- Add `no-break` class to prevent a section splitting across PDF pages.
- Add `break-before` class to force a section onto a new page.

### 3.5 Repeating Tables

Used for one-to-many related records (stakeholders, audit events, etc.).

```html
<div class="table-wrap">
  <table class="sh-table">
    <thead>...</thead>
    <tbody>
      @{join(body('Select_RowsAction'), '')}
    </tbody>
  </table>
</div>
```

---

## 4. Power Automate Integration

### 4.1 Variable Convention

All scalar field values come from a single record variable:

```
@{variables('RECORD')?['fieldName']}
```

Use an `if(empty(...))` guard for optional fields:

```
@{if(empty(variables('RECORD')?['fieldName']), '&#8212;', variables('RECORD')?['fieldName'])}
```

### 4.2 Repeating Table Pattern (Two-Step Select)

Power Automate cannot output HTML directly from a Filter/array body. Use this pattern:

| Step | Action | Purpose |
|---|---|---|
| 1 | `Filter_array_{Entity}` | Filter source array to the related records |
| 2 | `Select_{Entity}Rows` | Map each object → `<tr>...</tr>` HTML string using `concat()` |
| 3 | `join(body('Select_{Entity}Rows'), '')` | Inline in the template to produce the `<tbody>` rows |

**Empty-state guard:**
```
@{if(equals(length(body('Filter_array_{Entity}')), 0),
  '<tr><td colspan="N" style="text-align:center;color:#6b7280;font-style:italic;">No records available.</td></tr>',
  join(body('Select_{Entity}Rows'), '')
)}
```

### 4.3 Conditional Field Visibility

Hide fields based on a record property value:

```
style="@{if(equals(toLower(coalesce(variables('RECORD')?['sourceField'], '')), 'expectedvalue'), 'display:none;', '')}"
```

### 4.4 Compose Chunk Architecture

Split the full HTML into 8 Compose actions to avoid Power Automate expression length limits and to allow isolated edits:

| # | Compose Action | Content |
|---|---|---|
| 1 | `Compose_HtmlOpen` | DOCTYPE, `<head>`, all CSS, opens `<body>` and `.page` |
| 2 | `Compose_HtmlHeader` | Branded gradient header + retention/notice band |
| 3–7 | `Compose_Section{N}` | One Compose per section |
| 8 | `Compose_HtmlFooter` | Footer div + `</div></body></html>` |

**Final assembly expression:**
```
concat(
  outputs('Compose_HtmlOpen'),
  outputs('Compose_HtmlHeader'),
  '<div class="doc-body">',
  outputs('Compose_Section1_...'),
  outputs('Compose_Section2_...'),
  ...
  '</div>',
  outputs('Compose_HtmlFooter')
)
```

---

## 5. Document Structure (Header / Footer)

### Header
- Left: Logo + org label + document type title + record name (from variable)
- Right: Retention badge (years) + generated date (`formatDateTime(utcNow(), 'MMMM d, yyyy')`)
- Below header: Optional retention/notice band

### Footer
- Left: Organisation name + platform + document type
- Right: Classification label (e.g., `Internal / Confidential`)

---

## 6. Checklist for a New Export

- [ ] Copy `HtmlOpen` — update `--brand*` CSS variables and `<title>`
- [ ] Copy `HtmlHeader` — update org label, document type title, retention period
- [ ] Create one `Section{N}` chunk per logical group of fields
- [ ] Add repeating table chunks for any one-to-many relationships
- [ ] Copy `HtmlFooter` — update footer text
- [ ] Create master `*-main.html` by combining all chunks
- [ ] Set up Power Automate Compose actions (one per chunk + `Compose_FullHtml`)
- [ ] Wire upstream Filter/Select actions for any repeating tables
- [ ] Test: generate a sample PDF and verify shading, conditional fields, and repeating rows

---

## 7. Existing Exports

| Export | Folder | Master File | Record Variable |
|---|---|---|---|
| Data Role Repository | `DRR Export/` | `drr-export-main.html` | `variables('DRR')` |
| Data Steward Agreement | `DSA Export/` | *(TBD)* | *(TBD)* |
| EDI | `EDI Export/` | *(TBD)* | *(TBD)* |
