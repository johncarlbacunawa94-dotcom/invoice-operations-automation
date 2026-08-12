# Data Dictionary

The Invoice Operations Register is the cumulative operational source of truth for invoice processing, human review, and consolidated reporting.

The public documentation below describes the fields used by the verified workflows. It intentionally documents field purpose rather than exposing live identifiers, local paths, or environment-specific values.

## Field groups

### Record identity and source

| Field | Purpose |
|---|---|
| `record_key` | Stable operational key used to identify the invoice record across processing, review, and reporting. |
| `received_at` | Timestamp associated with receipt of the invoice into the operational flow. |
| `source_channel` | Channel through which the invoice entered the system, such as Gmail. |
| `source_filename` | Original or normalized filename associated with the invoice source document. |
| `source_file_path` | Internal path to the source/processed document used by the local processing environment. Public evidence does not expose real local path values. |
| `batch_execution_id` | Identifier linking the record to the batch-processing execution that produced or updated it. |

### Invoice data

| Field | Purpose |
|---|---|
| `supplier` | Supplier or vendor name extracted from the invoice. |
| `invoice_number` | Current invoice number. This field can be corrected during controlled human review. |
| `invoice_date` | Canonical invoice date printed on the supplier invoice. Current records use a date-only `YYYY-MM-DD` representation. |
| `invoice_date_legacy` | Preserved legacy date value for older records created before the date-only migration. |
| `currency` | Invoice currency code, such as `USD`. |
| `amount` | Invoice total amount used by processing, review, and reporting. |
| `confidence` | Extraction confidence associated with the processed invoice data. |

`invoice_date` and `processed_at` serve different purposes. `invoice_date` is the business date printed on the invoice; `processed_at` is the operational timestamp used to determine when the automation processed and registered the record.

### Processing and risk state

| Field | Purpose |
|---|---|
| `processing_status` | Current operational processing state of the invoice. |
| `risk_level` | Risk classification used to provide additional context for exception handling. |
| `duplicate_type` | Duplicate classification assigned by the invoice-processing controls. |
| `processed_at` | Timestamp recording when the invoice was processed and registered. |
| `last_updated_at` | Application-level timestamp for the latest controlled update to the operational record. |

Typical processing states demonstrated by the system include:

- `Processed`
- `Needs Review`
- `Needs Correction`
- `Approved`
- `Rejected`
- `Quarantined`

The list above reflects implemented/demonstrated states and should not be treated as a universal accounting-status vocabulary.

### Human review

| Field | Purpose |
|---|---|
| `review_required` | Boolean control indicating whether human review is still required. |
| `review_reason` | Reason the invoice was routed for review rather than treated as clear. |
| `review_status` | Current state of the human-review process. |
| `assigned_reviewer` | Reviewer identity recorded when the review is assigned or a decision is submitted. |
| `sla_due_at` | Review deadline/SLA timestamp when a review action requires a due time. |
| `review_decision` | Human-readable audit text describing the accepted reviewer decision and any submitted notes. |
| `decision_at` | Timestamp when the accepted review decision was recorded. |

Typical review states demonstrated by the system include:

- `Not Required`
- `Pending`
- `Correction Requested`
- `Approved`
- `Rejected`
- `Duplicate Confirmed`

Review decisions supported by the verified portal logic include approval, correction requests, rejection, approval as unique, and confirmation of a duplicate. The portal reloads the current record before accepting a write so stale or invalid submissions do not overwrite newer state.

### Platform-managed audit fields

| Field | Purpose |
|---|---|
| `createdAt` | Data-table creation timestamp managed by the platform. |
| `updatedAt` | Data-table update timestamp managed by the platform. |

These fields are distinct from `processed_at`, `decision_at`, and `last_updated_at`, which carry application-specific operational meaning.

## State examples

### Clean invoice

A normal processed invoice is typically persisted with:

```text
processing_status = Processed
review_required = false
review_status = Not Required
```

This indicates that the invoice passed the automated controls without requiring a reviewer.

### Review-required invoice

An exception is typically persisted with:

```text
processing_status = Needs Review
review_required = true
review_status = Pending
```

The record remains in the cumulative register and is surfaced to the review workflow.

### Correction requested

A reviewer can return an exception for correction. The verified portal logic uses:

```text
processing_status = Needs Correction
review_required = true
review_status = Correction Requested
```

The reviewer, decision text, SLA, decision timestamp, and update timestamp are retained with the record.

### Closed review

Approved, rejected, or confirmed-duplicate decisions clear the active review requirement and preserve the final decision in the register. Completed review records become read-only in the review interface.

## Reporting use

The consolidated reporting workflow reads a controlled subset of the Invoice Operations Register rather than scanning the processed-invoice folder.

For period filtering, operational reporting uses `processed_at` as the normal time basis because it answers when the automation processed the invoice. `invoice_date` remains the supplier-document date and is not interchangeable with the processing timestamp.

The reporting workflow reads invoice state but does not change invoice records.

## Data ownership and update boundaries

Different workflows own different parts of the record lifecycle:

- **Batch Invoice Processor** produces normalized invoice results and exception classifications.
- **Invoice Register Sync** upserts clean and review-required records into the cumulative register.
- **Invoice Review Portal API** can update the controlled review fields and permitted corrected invoice values after validating the latest record state.
- **Invoice Operations Consolidated Reporting** has read-only access to invoice records for reporting.
- **Invoice Automation Failure Handler** handles technical execution failures separately from invoice business/review state.

This separation keeps transient processing output, human decisions, reporting, and technical failure handling from being conflated into one state path.

## Public repository boundary

The public portfolio does not expose actual values for:

- local source file paths;
- credentials or API secrets;
- environment-specific workflow or Data Table identifiers;
- local webhook URLs;
- private account identifiers.

The field names and state model are retained because they are necessary to explain the architecture and operational controls.
