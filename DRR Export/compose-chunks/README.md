# Power Automate — Compose Action Breakdown

## Overview

The full HTML is split into **8 independent Compose actions**, then assembled by
one final **Compose_FullHtml** action using `concat()`.

Each chunk is a self-contained HTML fragment — you can update any section
without touching the others.

---

## Compose Actions (in order)

| # | Compose Action Name               | File                                        | Description                                      |
|---|-----------------------------------|---------------------------------------------|--------------------------------------------------|
| 1 | `Compose_HtmlOpen`                | `drr-export-HtmlOpen.html`                  | DOCTYPE, `<head>`, all CSS, opens `<body><div class="page">` |
| 2 | `Compose_HtmlHeader`              | `drr-export-HtmlHeader.html`                | Branded header gradient + retention notice band  |
| 3 | `Compose_Section1_AppDetails`     | `drr-export-Section1_AppDetails.html`       | Section 1 — Application Details                  |
| 4 | `Compose_Section2_OrgClassification` | `drr-export-Section2_OrgClassification.html` | Section 2 — Organizational Classification      |
| 5 | `Compose_Section3_RecordMetadata` | `drr-export-Section3_RecordMetadata.html`   | Section 3 — Record Metadata                      |
| 6 | `Compose_Section4_Stakeholders`   | `drr-export-Section4_Stakeholders.html`     | Section 4 — Stakeholders (repeating table)       |
| 7 | `Compose_Section5_AuditInfo`      | `drr-export-Section5_AuditInfo.html`        | Section 5 — Audit Information (repeating table)  |
| 8 | `Compose_HtmlFooter`              | `drr-export-HtmlFooter.html`                | Footer + closes `</div></body></html>`           |

---

## Final Assembly — Compose_FullHtml

Create one more Compose action called **`Compose_FullHtml`** with this expression:

```
concat(
  outputs('Compose_HtmlOpen'),
  outputs('Compose_HtmlHeader'),
  '<div class="doc-body">',
  outputs('Compose_Section1_AppDetails'),
  outputs('Compose_Section2_OrgClassification'),
  outputs('Compose_Section3_RecordMetadata'),
  outputs('Compose_Section4_Stakeholders'),
  outputs('Compose_Section5_AuditInfo'),
  '</div>',
  outputs('Compose_HtmlFooter')
)
```

Then use `outputs('Compose_FullHtml')` as the HTML input for your
"Convert HTML to PDF" action (or wherever you need the final HTML).

---

## How to Set Up Each Compose Action

1. In Power Automate, add a **Compose** action
2. Rename it to the exact name in the table above (e.g., `Compose_HtmlOpen`)
3. Copy the **entire contents** of the corresponding `.html` file into the
   Compose action's **Inputs** field
4. Repeat for all 8 chunks
5. Add the final `Compose_FullHtml` Compose action with the `concat()` expression above

---

## Dependencies (unchanged)

These upstream actions must still exist before the Compose chunks:

- **`Filter_array_Stakeholders`** → source array for stakeholders
- **`Select_StakeholderRows`** → converts stakeholder objects to `<tr>` HTML strings
- **`Select_EventTrackers`** → source array for audit events
- **`Select_EventTrackerRows`** → converts event objects to `<tr>` HTML strings
- **`variables('DRR')`** → the DRR record variable with all scalar fields

---

## Benefits

- **Isolate changes** — update one section without risk of breaking the full HTML
- **Easier debugging** — test each Compose output independently
- **Readable flow** — each Compose is small and focused
- **No expression limits** — avoids hitting Power Automate's expression length cap
