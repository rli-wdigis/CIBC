# DSA Export — Compose Chunks

## Power Automate Assembly

Final `Compose_FullHtml` expression:

```
concat(
  outputs('Compose_HtmlOpen'),
  outputs('Compose_HtmlHeader'),
  '<div class="doc-body">',
  outputs('Compose_Section1_Overview'),
  outputs('Compose_Section2_LinkedApplication'),
  outputs('Compose_Section3_DataConsumerDetails'),
  outputs('Compose_Section4_DataStewardReview'),
  outputs('Compose_Section5_StakeholderDecisions'),
  outputs('Compose_Section6_Notes'),
  outputs('Compose_Section7_Attachments'),
  outputs('Compose_Section8_RecordMetadata'),
  '</div>',
  outputs('Compose_HtmlFooter')
)
```

## Record Variable

All fields are referenced via `variables('REC')`. The flow must initialise this variable with the merged DSA + DRR record object before the Compose actions run.

## Notes Array

The notes section requires one Power Automate **Select** action before composing HTML:
- **`Select_NoteRows`** — Select action with **Input** = `variables('Notes')` that maps each note to an HTML `<tr>` row using:

```
concat(
  '<tr>',
    '<td class="note-created">',
      '<div class="note-col-label">Created</div>',
      '<div class="note-date">', formatDateTime(item()?['createdOn'], 'MMMM d, yyyy'), '</div>',
      '<div class="note-by">', item()?['createdBy'], '</div>',
      '<div class="note-email">', item()?['createdByEmail'], '</div>',
    '</td>',
    '<td class="note-modified-col">',
      '<div class="note-col-label">Modified</div>',
      '<div class="note-date">', formatDateTime(item()?['modifiedOn'], 'MMMM d, yyyy'), '</div>',
      '<div class="note-by">', item()?['modifiedBy'], '</div>',
      '<div class="note-email">', item()?['modifiedByEmail'], '</div>',
    '</td>',
    '<td class="note-text">', item()?['noteText'], '</td>',
  '</tr>'
)
```

## Attachments Array

The attachments section requires one Power Automate **Select** action before composing HTML:
- **`Select_AttachmentRows`** — Select action named **"Select Attachment Rows"** with **Input** = `variables('Attachments')` that maps each attachment to an HTML `<a>` chip using:

```
concat(
  '<a href="', item()?['url'], '" target="_blank" class="attach-chip attach-chip--', item()?['type'], '">',
    '<span class="attach-chip-icon">', if(equals(item()?['type'], 'image'), '🖼️', '📎'), '</span>',
    if(
      equals(item()?['type'], 'image'),
      concat(
        '<span class="attach-chip-body">',
          '<span class="attach-chip-name">', item()?['fileName'], '</span>',
          '<span class="attach-chip-hint">Add a file extension after downloading</span>',
        '</span>'
      ),
      concat('<span class="attach-chip-name">', item()?['fileName'], '</span>')
    ),
  '</a>'
)
```

> **Note on image type:** Due to a system limitation, image attachments may not have a proper file extension in the stored filename. The chip displays a hint prompting the user to add the correct extension (e.g. `.png`, `.jpg`) after opening the file.

## Field Mapping (placeholder → source JSON field)

### DSA Overview (Section 1)
| Placeholder | Source JSON field | Notes |
|---|---|---|
| `isStructuredDataRequest` | `edm_isstructureddatarequest` | Convert option-set → "Yes" / "No" / "Unsure" in flow |
| `isThisAuthorizationForOtherPurpose` | `edm_authorizationforotherpurpose` | Convert option-set → "Yes" / "No" / "Unsure" in flow |

### Linked Application — DRR (Section 2)
| Placeholder | Source JSON field | Notes |
|---|---|---|
| `applicationName` | `applicationName` | From linked DRR record |
| `drrNumber` | `drrNumber` | From linked DRR record |
| `applicationType` | `applicationType` | From linked DRR record |
| `applicationinScope` | `applicationinScope` | From linked DRR record |
| `lob` | `lob` | From linked DRR record; Nullable |
| `mapId` | `mapId` | From linked DRR record; Nullable |
| `organizationalSpoke` | `organizationalSpoke` | From linked DRR record |
| `spokeSegment` | `spokeSegment` | From linked DRR record; Nullable |
| `dataAssetName` | `dataAssetName` | From linked DRR record; Nullable |
| `dataDomain` | `dataDomain` | From linked DRR record |
| `exclusionReason` | `exclusionReason` | From linked DRR record; Nullable |

### Data Consumer Details (Section 3)
| Placeholder | Source JSON field | Notes |
|---|---|---|
| `dataConsumer` | `dataConsumer` | |
| `pointOfContactInformation` | `pointOfContactInformation` | |
| `dataSourceName` | `dataSourceName` | |
| `dataStorageLocation` | `dataStorageLocation` | |
| `dataAccessMethod` | `dataAccessMethod` | |
| `associatedAccessControls` | `associatedAccessControls` | |
| `ciraId` | `ciraId` | |
| `usageAndIntent` | `usageAndIntent` | |
| `legalEntityName` | `legalEntityName` | |
| `techTriageId` | `techTriageId` | |
| `teamsoradgroupstosharewith` | `teamsoradgroupstosharewith` | |
| `physicalDataElementsRequested` | `physicalDataElementsRequested` | |
| `dataDisposalMethod` | `dataDisposalMethod` | |
| `agreementEndDate` | `agreementEndDate` | Date — formatted `MMMM d, yyyy` by template |

### Data Steward Review (Section 4)
| Placeholder | Source JSON field | Notes |
|---|---|---|
| `areThereAnyUsageRestrictions` | `areThereAnyUsageRestrictions` | |
| `doesDataIncludePI` | `doesDataIncludePI` | Convert option-set → "Yes" / "No" in flow |
| `doesDataIncludeConfidentialOrRestrictedInformation` | `doesDataIncludeConfidentialOrRestrictedInformation` | Convert option-set → "Yes" / "No" in flow |
| `doesDataIncludeVendorOrThirdPartyData` | `doesDataIncludeVendorOrThirdPartyData` | Convert option-set → "Yes" / "No" in flow |
| `willDataBeDeidentifiedBeforeSharing` | `willDataBeDeidentifiedBeforeSharing` | Convert option-set → "Yes" / "No" in flow |
| `mailboxToBeNotified` | `mailboxToBeNotified` | |

### Stakeholder's Decision (Section 5)
| Placeholder | Source JSON field | Notes |
|---|---|---|
| `dataConsumerDecision` | `dataConsumerDecision` | Nullable |
| `dataStewardDecision` | `dataStewardDecision` | Nullable |
| `dataOwnerDecision` | `dataOwnerDecision` | Nullable |
| `reviewerComments` | `reviewerComments` | Nullable |
| `additionalComments` | `additionalComments` | Nullable |

### DSA Notes (Section 6)
Populated via `body('Select_NoteRows')` — see Notes Array section above.

| Note field | Source JSON field |
|---|---|
| `createdOn` | `createdOn` |
| `createdBy` | `createdBy` |
| `createdByEmail` | `createdByEmail` |
| `modifiedOn` | `modifiedOn` |
| `modifiedBy` | `modifiedBy` |
| `noteText` | `noteText` |

### Record Metadata (Section 7)
| Placeholder | Source JSON field | Notes |
|---|---|---|
| `dsaNumber` | `dsaNumber` | Never empty — shown in header and Section 7 |
| `dsaStatus` | `dsaStatus` | |
| `createdOn` | `createdOn` | Datetime — formatted `MMMM d, yyyy h:mm tt` by template |
| `modifiedOn` | `modifiedOn` | Datetime — formatted `MMMM d, yyyy h:mm tt` by template |
| `createdBy` | `createdBy` | |
| `createdByEmail` | `createdByEmail` | |
| `modifiedBy` | `modifiedBy` | |
| `modifiedByEmail` | `modifiedByEmail` | |
