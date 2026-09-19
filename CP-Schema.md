# Custom Packly OS: all tables and their relationships

Design reference · 19 September 2026 · Based on the existing logical data model

**Use one PostgreSQL database and one application schema.** In this discussion, “schemas” means the definitions of tables such as `invoices` and `payments`. Each table has a Drizzle definition; related definitions can share a TypeScript module. Authentication-provider internal tables are managed separately and are not included in this count.

**The detailed plan contains 57 defined tables: 53 for the complete basic business scope and four later additions.** This is a count of this design, not a universal minimum. You still have six main user journeys. Supporting tables do not each need a screen.

| Build stage | New tables | Total at this stage |
| --- | ---: | ---: |
| F — Foundation | 18 | 18 |
| I — Invoice → payment → job pilot | 12 | 30 |
| O — Production, shipping, vendors and costs | 11 | 41 |
| B — Complete basic scope: partners, close and reports | 12 | 53 |
| L — Later automation | 4 | 57 |

Start with F + I. Add foreign keys and fields pointing at a later-stage table when that stage is built. For example, the pilot payment command needs customer fields before it needs partner and vendor fields. The full financial reports become available after their supporting costs, opening balances and recognition records are implemented.

## How to read the diagrams

Each box is a table. `PK` identifies a row, `FK` points to another table, and `UK` is a unique key. `||` means exactly one, `o|` means zero or one, and `o{` means zero or many. Read the cardinality at each end. Repeated table names in different diagrams refer to the same table, not copies.

The 14 panels cover every defined table. They show selected columns and relationships to remain readable; the complete field register and explicit foreign-key register follow them. Common entity ownership, currency links and actor fields are not drawn repeatedly. Zero-line drafts are allowed where relevant; issuing/posting imposes the stronger requirements described in the rules.

All diagrams are embedded below as Mermaid. A Mermaid-enabled Markdown viewer can show them together. Individual editable sources are in `diagrams/`; paste one `.mmd` file into Mermaid Live Editor when needed.

## Table inventory

| Area | Count | Tables |
| --- | ---: | --- |
| Business, markets and access | 6 | `currencies` (F), `legal_entities` (F), `markets` (F), `entity_markets` (F), `app_users` (F), `entity_memberships` (F) |
| Bookkeeping configuration | 5 | `document_sequences` (F), `accounting_periods` (F), `accounting_policies` (F), `gl_accounts` (F), `tax_codes` (F) |
| Customers and invoicing | 6 | `customers` (I), `invoices` (I), `invoice_lines` (I), `payment_links` (I), `customer_credit_notes` (I), `customer_credit_lines` (I) |
| Payments and cash controls | 8 | `money_accounts` (F), `payments` (I), `customer_payment_allocations` (I), `vendor_payment_allocations` (O), `account_transfers` (B), `payment_disputes` (B), `reconciliation_sessions` (B), `reconciliation_items` (B) |
| Jobs, production and shipping | 6 | `jobs` (I), `job_items` (I), `production_runs` (O), `production_run_items` (O), `shipments` (O), `shipment_items` (O) |
| Vendors, bills and expected costs | 5 | `vendors` (O), `vendor_roles` (O), `vendor_bills` (O), `vendor_bill_lines` (O), `job_cost_estimates` (O) |
| Ledger and recognition | 5 | `journal_entries` (F), `journal_lines` (F), `accounting_adjustments` (B), `recognition_events` (B), `recognition_lines` (B) |
| Partner accounts | 5 | `partners` (B), `partner_accounts` (B), `partner_events` (B), `profit_distributions` (B), `profit_distribution_lines` (B) |
| Documents and reliability | 7 | `files` (F), `generated_documents` (I), `document_deliveries` (I), `job_file_links` (O), `audit_events` (F), `idempotency_requests` (F), `outbox_tasks` (F) |
| Later automation | 4 | `recurring_expense_templates` (L), `profit_policies` (L), `profit_policy_shares` (L), `webhook_events` (L) |

## Relationship diagrams

### 1. Business, markets and access

A legal entity owns a set of books. US, UK and AU are markets, not automatically separate companies. Global is a filter. Memberships connect logins to the business books they may access. Currency is a lookup used throughout the model.

```mermaid
erDiagram
  currencies {
    string code PK
    int minor_unit_scale
  }
  legal_entities {
    uuid id PK
    string legal_name
    string functional_currency FK
  }
  markets {
    uuid id PK
    string code UK
    string default_currency FK
  }
  entity_markets {
    uuid id PK
    uuid legal_entity_id FK
    uuid market_id FK
  }
  app_users {
    uuid id PK
    string auth_subject UK
    string display_name
  }
  entity_memberships {
    uuid id PK
    uuid user_id FK
    uuid legal_entity_id FK
    string role
  }
  currencies ||--o{ legal_entities : "functional currency"
  currencies ||--o{ markets : "default currency"
  legal_entities ||--o{ entity_markets : "operates in"
  markets ||--o{ entity_markets : "available to"
  legal_entities ||--o{ entity_memberships : "grants access"
  app_users ||--o{ entity_memberships : "has access"
%% portable-canonical-v2:eyJhY2Nlc3NpYmlsaXR5IjpudWxsLCJkYXRhIjp7ImVudGl0aWVzIjpbeyJhdHRyaWJ1dGVzIjpbInN0cmluZyBjb2RlIFBLIiwiaW50IG1pbm9yX3VuaXRfc2NhbGUiXSwiaWQiOiJjdXJyZW5jaWVzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJzdHJpbmcgbGVnYWxfbmFtZSIsInN0cmluZyBmdW5jdGlvbmFsX2N1cnJlbmN5IEZLIl0sImlkIjoibGVnYWxfZW50aXRpZXMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsInN0cmluZyBjb2RlIFVLIiwic3RyaW5nIGRlZmF1bHRfY3VycmVuY3kgRksiXSwiaWQiOiJtYXJrZXRzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIGxlZ2FsX2VudGl0eV9pZCBGSyIsInV1aWQgbWFya2V0X2lkIEZLIl0sImlkIjoiZW50aXR5X21hcmtldHMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsInN0cmluZyBhdXRoX3N1YmplY3QgVUsiLCJzdHJpbmcgZGlzcGxheV9uYW1lIl0sImlkIjoiYXBwX3VzZXJzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIHVzZXJfaWQgRksiLCJ1dWlkIGxlZ2FsX2VudGl0eV9pZCBGSyIsInN0cmluZyByb2xlIl0sImlkIjoiZW50aXR5X21lbWJlcnNoaXBzIn1dLCJyZWxhdGlvbnNoaXBzIjpbeyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJjdXJyZW5jaWVzIiwibGFiZWwiOiJmdW5jdGlvbmFsIGN1cnJlbmN5IiwidG8iOiJsZWdhbF9lbnRpdGllcyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoiY3VycmVuY2llcyIsImxhYmVsIjoiZGVmYXVsdCBjdXJyZW5jeSIsInRvIjoibWFya2V0cyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoibGVnYWxfZW50aXRpZXMiLCJsYWJlbCI6Im9wZXJhdGVzIGluIiwidG8iOiJlbnRpdHlfbWFya2V0cyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoibWFya2V0cyIsImxhYmVsIjoiYXZhaWxhYmxlIHRvIiwidG8iOiJlbnRpdHlfbWFya2V0cyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoibGVnYWxfZW50aXRpZXMiLCJsYWJlbCI6ImdyYW50cyBhY2Nlc3MiLCJ0byI6ImVudGl0eV9tZW1iZXJzaGlwcyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoiYXBwX3VzZXJzIiwibGFiZWwiOiJoYXMgYWNjZXNzIiwidG8iOiJlbnRpdHlfbWVtYmVyc2hpcHMifV19LCJkZXNjcmlwdGlvbiI6IkEgbGVnYWwgZW50aXR5IG93bnMgYSBzZXQgb2YgYm9va3MuIFVTLCBVSyBhbmQgQVUgYXJlIG1hcmtldHMsIG5vdCBhdXRvbWF0aWNhbGx5IHNlcGFyYXRlIGNvbXBhbmllcy4gR2xvYmFsIGlzIGEgZmlsdGVyLiBNZW1iZXJzaGlwcyBjb25uZWN0IGxvZ2lucyB0byB0aGUgYnVzaW5lc3MgYm9va3MgdGhleSBtYXkgYWNjZXNzLiBDdXJyZW5jeSBpcyBhIGxvb2t1cCB1c2VkIHRocm91Z2hvdXQgdGhlIG1vZGVsLiIsImlkIjoic2NoZW1hXzAxX2FjY2Vzc19tYXJrZXRzIiwia2luZCI6ImVyIiwic291cmNlU2hhMjU2IjoiNTg5ZTkwM2YzOGIzZWRhMTAzMTdkNzY1ZDIzYWY2NGY0YzllNmU1ZDQ0Y2E4MWUxMWFkNjE5ZmFmZTNkZTg3YiIsInN0eWxlcyI6W10sInRpdGxlIjoiMS4gQnVzaW5lc3MsIG1hcmtldHMgYW5kIGFjY2VzcyIsInZlcnNpb24iOjF9
```

### 2. Bookkeeping configuration

These are background configuration tables. Document sequences number invoices, jobs and receipts. Periods control closing dates; policy versions preserve the rules used by historical postings. GL accounts classify assets, liabilities, equity, income and expenses.

```mermaid
erDiagram
  legal_entities {
    uuid id PK
  }
  document_sequences {
    uuid id PK
    string document_type
    int fiscal_year
    bigint next_value
  }
  accounting_periods {
    uuid id PK
    date start_date
    date end_date
    string status
  }
  accounting_policies {
    uuid id PK
    int version
    string invoice_posting_mode
    string revenue_basis
  }
  gl_accounts {
    uuid id PK
    uuid parent_id FK
    string code
    string account_type
  }
  tax_codes {
    uuid id PK
    uuid tax_gl_account_id FK
    decimal rate_percent
  }
  legal_entities ||--o{ document_sequences : "numbers documents"
  legal_entities ||--o{ accounting_periods : "closes periods"
  legal_entities ||--o{ accounting_policies : "versions rules"
  legal_entities ||--o{ gl_accounts : "classifies money"
  gl_accounts ||--o{ tax_codes : "posts tax"
%% portable-canonical-v2:eyJhY2Nlc3NpYmlsaXR5IjpudWxsLCJkYXRhIjp7ImVudGl0aWVzIjpbeyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiXSwiaWQiOiJsZWdhbF9lbnRpdGllcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwic3RyaW5nIGRvY3VtZW50X3R5cGUiLCJpbnQgZmlzY2FsX3llYXIiLCJiaWdpbnQgbmV4dF92YWx1ZSJdLCJpZCI6ImRvY3VtZW50X3NlcXVlbmNlcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwiZGF0ZSBzdGFydF9kYXRlIiwiZGF0ZSBlbmRfZGF0ZSIsInN0cmluZyBzdGF0dXMiXSwiaWQiOiJhY2NvdW50aW5nX3BlcmlvZHMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsImludCB2ZXJzaW9uIiwic3RyaW5nIGludm9pY2VfcG9zdGluZ19tb2RlIiwic3RyaW5nIHJldmVudWVfYmFzaXMiXSwiaWQiOiJhY2NvdW50aW5nX3BvbGljaWVzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIHBhcmVudF9pZCBGSyIsInN0cmluZyBjb2RlIiwic3RyaW5nIGFjY291bnRfdHlwZSJdLCJpZCI6ImdsX2FjY291bnRzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIHRheF9nbF9hY2NvdW50X2lkIEZLIiwiZGVjaW1hbCByYXRlX3BlcmNlbnQiXSwiaWQiOiJ0YXhfY29kZXMifV0sInJlbGF0aW9uc2hpcHMiOlt7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6ImxlZ2FsX2VudGl0aWVzIiwibGFiZWwiOiJudW1iZXJzIGRvY3VtZW50cyIsInRvIjoiZG9jdW1lbnRfc2VxdWVuY2VzIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJsZWdhbF9lbnRpdGllcyIsImxhYmVsIjoiY2xvc2VzIHBlcmlvZHMiLCJ0byI6ImFjY291bnRpbmdfcGVyaW9kcyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoibGVnYWxfZW50aXRpZXMiLCJsYWJlbCI6InZlcnNpb25zIHJ1bGVzIiwidG8iOiJhY2NvdW50aW5nX3BvbGljaWVzIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJsZWdhbF9lbnRpdGllcyIsImxhYmVsIjoiY2xhc3NpZmllcyBtb25leSIsInRvIjoiZ2xfYWNjb3VudHMifSx7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6ImdsX2FjY291bnRzIiwibGFiZWwiOiJwb3N0cyB0YXgiLCJ0byI6InRheF9jb2RlcyJ9XX0sImRlc2NyaXB0aW9uIjoiVGhlc2UgYXJlIGJhY2tncm91bmQgY29uZmlndXJhdGlvbiB0YWJsZXMuIERvY3VtZW50IHNlcXVlbmNlcyBudW1iZXIgaW52b2ljZXMsIGpvYnMgYW5kIHJlY2VpcHRzLiBQZXJpb2RzIGNvbnRyb2wgY2xvc2luZyBkYXRlczsgcG9saWN5IHZlcnNpb25zIHByZXNlcnZlIHRoZSBydWxlcyB1c2VkIGJ5IGhpc3RvcmljYWwgcG9zdGluZ3MuIEdMIGFjY291bnRzIGNsYXNzaWZ5IGFzc2V0cywgbGlhYmlsaXRpZXMsIGVxdWl0eSwgaW5jb21lIGFuZCBleHBlbnNlcy4iLCJpZCI6InNjaGVtYV8wMl9ib29ra2VlcGluZ19zZXR1cCIsImtpbmQiOiJlciIsInNvdXJjZVNoYTI1NiI6IjcxOGI4ZDFlMzA3YmI0MDFiYTZjM2M4YjQ2MjU1YzliZGYyNDdlMDEyNjQ3YjEyNjVmMjEzYWJmZDUyYTM4OGMiLCJzdHlsZXMiOltdLCJ0aXRsZSI6IjIuIEJvb2trZWVwaW5nIGNvbmZpZ3VyYXRpb24iLCJ2ZXJzaW9uIjoxfQ
```

### 3. Customers, invoices and corrections

An invoice contains line items and can have replacement payment links. A credit note corrects an issued charge; its lines point back to the original invoice lines. A payment link does not prove that payment was received. Issued documents require at least one valid line.

```mermaid
erDiagram
  customers {
    uuid id PK
    string display_name
  }
  invoices {
    uuid id PK
    uuid customer_id FK
    uuid market_id FK
    bigint total_minor
  }
  invoice_lines {
    uuid id PK
    uuid invoice_id FK
    decimal quantity
    bigint gross_minor
  }
  payment_links {
    uuid id PK
    uuid invoice_id FK
    string url
    string status
  }
  customer_credit_notes {
    uuid id PK
    uuid invoice_id FK
    bigint total_minor
  }
  customer_credit_lines {
    uuid id PK
    uuid credit_note_id FK
    uuid invoice_line_id FK
  }
  customers ||--o{ invoices : "receives"
  invoices ||--o{ invoice_lines : "contains"
  invoices ||--o{ payment_links : "requests payment"
  invoices ||--o{ customer_credit_notes : "corrected by"
  customer_credit_notes ||--o{ customer_credit_lines : "contains"
  invoice_lines ||--o{ customer_credit_lines : "adjusted by"
%% portable-canonical-v2:eyJhY2Nlc3NpYmlsaXR5IjpudWxsLCJkYXRhIjp7ImVudGl0aWVzIjpbeyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJzdHJpbmcgZGlzcGxheV9uYW1lIl0sImlkIjoiY3VzdG9tZXJzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIGN1c3RvbWVyX2lkIEZLIiwidXVpZCBtYXJrZXRfaWQgRksiLCJiaWdpbnQgdG90YWxfbWlub3IiXSwiaWQiOiJpbnZvaWNlcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCBpbnZvaWNlX2lkIEZLIiwiZGVjaW1hbCBxdWFudGl0eSIsImJpZ2ludCBncm9zc19taW5vciJdLCJpZCI6Imludm9pY2VfbGluZXMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsInV1aWQgaW52b2ljZV9pZCBGSyIsInN0cmluZyB1cmwiLCJzdHJpbmcgc3RhdHVzIl0sImlkIjoicGF5bWVudF9saW5rcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCBpbnZvaWNlX2lkIEZLIiwiYmlnaW50IHRvdGFsX21pbm9yIl0sImlkIjoiY3VzdG9tZXJfY3JlZGl0X25vdGVzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIGNyZWRpdF9ub3RlX2lkIEZLIiwidXVpZCBpbnZvaWNlX2xpbmVfaWQgRksiXSwiaWQiOiJjdXN0b21lcl9jcmVkaXRfbGluZXMifV0sInJlbGF0aW9uc2hpcHMiOlt7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6ImN1c3RvbWVycyIsImxhYmVsIjoicmVjZWl2ZXMiLCJ0byI6Imludm9pY2VzIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJpbnZvaWNlcyIsImxhYmVsIjoiY29udGFpbnMiLCJ0byI6Imludm9pY2VfbGluZXMifSx7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6Imludm9pY2VzIiwibGFiZWwiOiJyZXF1ZXN0cyBwYXltZW50IiwidG8iOiJwYXltZW50X2xpbmtzIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJpbnZvaWNlcyIsImxhYmVsIjoiY29ycmVjdGVkIGJ5IiwidG8iOiJjdXN0b21lcl9jcmVkaXRfbm90ZXMifSx7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6ImN1c3RvbWVyX2NyZWRpdF9ub3RlcyIsImxhYmVsIjoiY29udGFpbnMiLCJ0byI6ImN1c3RvbWVyX2NyZWRpdF9saW5lcyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoiaW52b2ljZV9saW5lcyIsImxhYmVsIjoiYWRqdXN0ZWQgYnkiLCJ0byI6ImN1c3RvbWVyX2NyZWRpdF9saW5lcyJ9XX0sImRlc2NyaXB0aW9uIjoiQW4gaW52b2ljZSBjb250YWlucyBsaW5lIGl0ZW1zIGFuZCBjYW4gaGF2ZSByZXBsYWNlbWVudCBwYXltZW50IGxpbmtzLiBBIGNyZWRpdCBub3RlIGNvcnJlY3RzIGFuIGlzc3VlZCBjaGFyZ2U7IGl0cyBsaW5lcyBwb2ludCBiYWNrIHRvIHRoZSBvcmlnaW5hbCBpbnZvaWNlIGxpbmVzLiBBIHBheW1lbnQgbGluayBkb2VzIG5vdCBwcm92ZSB0aGF0IHBheW1lbnQgd2FzIHJlY2VpdmVkLiBJc3N1ZWQgZG9jdW1lbnRzIHJlcXVpcmUgYXQgbGVhc3Qgb25lIHZhbGlkIGxpbmUuIiwiaWQiOiJzY2hlbWFfMDNfaW52b2ljZXMiLCJraW5kIjoiZXIiLCJzb3VyY2VTaGEyNTYiOiI0OGM2ZTYzMDFlOTJjMzg1YTM3NmE5ZDYxODRmYzZmMmI4MzlmOTE3MzEzNzM3MGJkZThmNzY1OTZlZTMwYTRlIiwic3R5bGVzIjpbXSwidGl0bGUiOiIzLiBDdXN0b21lcnMsIGludm9pY2VzIGFuZCBjb3JyZWN0aW9ucyIsInZlcnNpb24iOjF9
```

### 4. One payment system linking both sides

The same payments table records incoming and outgoing money. An allocation says how much of a payment settles one invoice or bill. This supports partial payments, several receipts against one invoice and one payment covering several bills. A positively funded, eligible invoice creates at most one job; an unpaid invoice creates none. jobs.origin_invoice_id is unique.

```mermaid
erDiagram
  money_accounts {
    uuid id PK
    string name
    string currency FK
  }
  payments {
    uuid id PK
    uuid money_account_id FK
    string direction
    bigint amount_minor
  }
  customer_payment_allocations {
    uuid id PK
    uuid payment_id FK
    uuid invoice_id FK
    bigint document_amount_minor
  }
  invoices {
    uuid id PK
    bigint total_minor
  }
  jobs {
    uuid id PK
    uuid origin_invoice_id FK
  }
  vendor_payment_allocations {
    uuid id PK
    uuid payment_id FK
    uuid vendor_bill_id FK
    bigint document_amount_minor
  }
  vendor_bills {
    uuid id PK
    bigint total_minor
  }
  money_accounts ||--o{ payments : "receives or sends"
  payments ||--o{ customer_payment_allocations : "applied to customer"
  invoices ||--o{ customer_payment_allocations : "settled by"
  invoices ||--o| jobs : "funded origin"
  payments ||--o{ vendor_payment_allocations : "applied to vendor"
  vendor_bills ||--o{ vendor_payment_allocations : "settled by"
%% portable-canonical-v2:eyJhY2Nlc3NpYmlsaXR5IjpudWxsLCJkYXRhIjp7ImVudGl0aWVzIjpbeyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJzdHJpbmcgbmFtZSIsInN0cmluZyBjdXJyZW5jeSBGSyJdLCJpZCI6Im1vbmV5X2FjY291bnRzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIG1vbmV5X2FjY291bnRfaWQgRksiLCJzdHJpbmcgZGlyZWN0aW9uIiwiYmlnaW50IGFtb3VudF9taW5vciJdLCJpZCI6InBheW1lbnRzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIHBheW1lbnRfaWQgRksiLCJ1dWlkIGludm9pY2VfaWQgRksiLCJiaWdpbnQgZG9jdW1lbnRfYW1vdW50X21pbm9yIl0sImlkIjoiY3VzdG9tZXJfcGF5bWVudF9hbGxvY2F0aW9ucyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwiYmlnaW50IHRvdGFsX21pbm9yIl0sImlkIjoiaW52b2ljZXMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsInV1aWQgb3JpZ2luX2ludm9pY2VfaWQgRksiXSwiaWQiOiJqb2JzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIHBheW1lbnRfaWQgRksiLCJ1dWlkIHZlbmRvcl9iaWxsX2lkIEZLIiwiYmlnaW50IGRvY3VtZW50X2Ftb3VudF9taW5vciJdLCJpZCI6InZlbmRvcl9wYXltZW50X2FsbG9jYXRpb25zIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJiaWdpbnQgdG90YWxfbWlub3IiXSwiaWQiOiJ2ZW5kb3JfYmlsbHMifV0sInJlbGF0aW9uc2hpcHMiOlt7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6Im1vbmV5X2FjY291bnRzIiwibGFiZWwiOiJyZWNlaXZlcyBvciBzZW5kcyIsInRvIjoicGF5bWVudHMifSx7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6InBheW1lbnRzIiwibGFiZWwiOiJhcHBsaWVkIHRvIGN1c3RvbWVyIiwidG8iOiJjdXN0b21lcl9wYXltZW50X2FsbG9jYXRpb25zIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJpbnZvaWNlcyIsImxhYmVsIjoic2V0dGxlZCBieSIsInRvIjoiY3VzdG9tZXJfcGF5bWVudF9hbGxvY2F0aW9ucyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3wiLCJmcm9tIjoiaW52b2ljZXMiLCJsYWJlbCI6ImZ1bmRlZCBvcmlnaW4iLCJ0byI6ImpvYnMifSx7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6InBheW1lbnRzIiwibGFiZWwiOiJhcHBsaWVkIHRvIHZlbmRvciIsInRvIjoidmVuZG9yX3BheW1lbnRfYWxsb2NhdGlvbnMifSx7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6InZlbmRvcl9iaWxscyIsImxhYmVsIjoic2V0dGxlZCBieSIsInRvIjoidmVuZG9yX3BheW1lbnRfYWxsb2NhdGlvbnMifV19LCJkZXNjcmlwdGlvbiI6IlRoZSBzYW1lIHBheW1lbnRzIHRhYmxlIHJlY29yZHMgaW5jb21pbmcgYW5kIG91dGdvaW5nIG1vbmV5LiBBbiBhbGxvY2F0aW9uIHNheXMgaG93IG11Y2ggb2YgYSBwYXltZW50IHNldHRsZXMgb25lIGludm9pY2Ugb3IgYmlsbC4gVGhpcyBzdXBwb3J0cyBwYXJ0aWFsIHBheW1lbnRzLCBzZXZlcmFsIHJlY2VpcHRzIGFnYWluc3Qgb25lIGludm9pY2UgYW5kIG9uZSBwYXltZW50IGNvdmVyaW5nIHNldmVyYWwgYmlsbHMuIEEgcG9zaXRpdmVseSBmdW5kZWQsIGVsaWdpYmxlIGludm9pY2UgY3JlYXRlcyBhdCBtb3N0IG9uZSBqb2I7IGFuIHVucGFpZCBpbnZvaWNlIGNyZWF0ZXMgbm9uZS4gam9icy5vcmlnaW5faW52b2ljZV9pZCBpcyB1bmlxdWUuIiwiaWQiOiJzY2hlbWFfMDRfcGF5bWVudF9jb25uZWN0aW9ucyIsImtpbmQiOiJlciIsInNvdXJjZVNoYTI1NiI6IjQwMzNjOWNlNDk5ZWU3NjJkOWNkOGQxNjRkYWM1NDMyZWEwNWNkZTY0ZjhlZTczYzIxYTA1NWU0YzAwYTYzYmUiLCJzdHlsZXMiOltdLCJ0aXRsZSI6IjQuIE9uZSBwYXltZW50IHN5c3RlbSBsaW5raW5nIGJvdGggc2lkZXMiLCJ2ZXJzaW9uIjoxfQ
```

### 5. Jobs and production

Jobs is the single order table. Production lives inside the job screen but has its own records. Each run belongs to one job and one vendor. Run items connect the run to particular ordered items and their quantities. Job items also retain their originating invoice-line IDs.

```mermaid
erDiagram
  jobs {
    uuid id PK
    uuid origin_invoice_id FK
    string lifecycle
  }
  job_items {
    uuid id PK
    uuid job_id FK
    uuid origin_invoice_line_id FK
    int ordered_quantity
  }
  production_runs {
    uuid id PK
    uuid job_id FK
    uuid vendor_id FK
    string status
  }
  production_run_items {
    uuid id PK
    uuid production_run_id FK
    uuid job_item_id FK
    int completed_quantity
  }
  vendors {
    uuid id PK
    string name
  }
  jobs ||--o{ job_items : "contains"
  jobs ||--o{ production_runs : "manufactured in"
  vendors ||--o{ production_runs : "manufactures"
  production_runs ||--o{ production_run_items : "contains"
  job_items ||--o{ production_run_items : "produced as"
%% portable-canonical-v2:eyJhY2Nlc3NpYmlsaXR5IjpudWxsLCJkYXRhIjp7ImVudGl0aWVzIjpbeyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIG9yaWdpbl9pbnZvaWNlX2lkIEZLIiwic3RyaW5nIGxpZmVjeWNsZSJdLCJpZCI6ImpvYnMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsInV1aWQgam9iX2lkIEZLIiwidXVpZCBvcmlnaW5faW52b2ljZV9saW5lX2lkIEZLIiwiaW50IG9yZGVyZWRfcXVhbnRpdHkiXSwiaWQiOiJqb2JfaXRlbXMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsInV1aWQgam9iX2lkIEZLIiwidXVpZCB2ZW5kb3JfaWQgRksiLCJzdHJpbmcgc3RhdHVzIl0sImlkIjoicHJvZHVjdGlvbl9ydW5zIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIHByb2R1Y3Rpb25fcnVuX2lkIEZLIiwidXVpZCBqb2JfaXRlbV9pZCBGSyIsImludCBjb21wbGV0ZWRfcXVhbnRpdHkiXSwiaWQiOiJwcm9kdWN0aW9uX3J1bl9pdGVtcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwic3RyaW5nIG5hbWUiXSwiaWQiOiJ2ZW5kb3JzIn1dLCJyZWxhdGlvbnNoaXBzIjpbeyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJqb2JzIiwibGFiZWwiOiJjb250YWlucyIsInRvIjoiam9iX2l0ZW1zIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJqb2JzIiwibGFiZWwiOiJtYW51ZmFjdHVyZWQgaW4iLCJ0byI6InByb2R1Y3Rpb25fcnVucyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoidmVuZG9ycyIsImxhYmVsIjoibWFudWZhY3R1cmVzIiwidG8iOiJwcm9kdWN0aW9uX3J1bnMifSx7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6InByb2R1Y3Rpb25fcnVucyIsImxhYmVsIjoiY29udGFpbnMiLCJ0byI6InByb2R1Y3Rpb25fcnVuX2l0ZW1zIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJqb2JfaXRlbXMiLCJsYWJlbCI6InByb2R1Y2VkIGFzIiwidG8iOiJwcm9kdWN0aW9uX3J1bl9pdGVtcyJ9XX0sImRlc2NyaXB0aW9uIjoiSm9icyBpcyB0aGUgc2luZ2xlIG9yZGVyIHRhYmxlLiBQcm9kdWN0aW9uIGxpdmVzIGluc2lkZSB0aGUgam9iIHNjcmVlbiBidXQgaGFzIGl0cyBvd24gcmVjb3Jkcy4gRWFjaCBydW4gYmVsb25ncyB0byBvbmUgam9iIGFuZCBvbmUgdmVuZG9yLiBSdW4gaXRlbXMgY29ubmVjdCB0aGUgcnVuIHRvIHBhcnRpY3VsYXIgb3JkZXJlZCBpdGVtcyBhbmQgdGhlaXIgcXVhbnRpdGllcy4gSm9iIGl0ZW1zIGFsc28gcmV0YWluIHRoZWlyIG9yaWdpbmF0aW5nIGludm9pY2UtbGluZSBJRHMuIiwiaWQiOiJzY2hlbWFfMDVfcHJvZHVjdGlvbiIsImtpbmQiOiJlciIsInNvdXJjZVNoYTI1NiI6IjJmYzA1NmRmMWY1OWIxNDQwMDdmMGEwYzg5Mzc3ZGIxMGNmYzhiNjBlMGU3M2E3ZDExN2RjNDY2NDY3YzQ2ZGYiLCJzdHlsZXMiOltdLCJ0aXRsZSI6IjUuIEpvYnMgYW5kIHByb2R1Y3Rpb24iLCJ2ZXJzaW9uIjoxfQ
```

### 6. Shipping and vendor roles

One job may ship in several consignments. Shipment items record which ordered items and quantities were shipped and delivered. A vendor can have production, shipping and other roles, so separate vendor tables are unnecessary.

```mermaid
erDiagram
  jobs {
    uuid id PK
  }
  job_items {
    uuid id PK
    uuid job_id FK
  }
  shipments {
    uuid id PK
    uuid job_id FK
    uuid vendor_id FK
    string tracking_reference
  }
  shipment_items {
    uuid id PK
    uuid shipment_id FK
    uuid job_item_id FK
    int delivered_quantity
  }
  vendors {
    uuid id PK
  }
  vendor_roles {
    uuid id PK
    uuid vendor_id FK
    string role
  }
  jobs ||--o{ job_items : "contains"
  jobs ||--o{ shipments : "ships in"
  shipments ||--o{ shipment_items : "contains"
  job_items ||--o{ shipment_items : "delivered as"
  vendors o|--o{ shipments : "ships"
  vendors ||--o{ vendor_roles : "serves as"
%% portable-canonical-v2:eyJhY2Nlc3NpYmlsaXR5IjpudWxsLCJkYXRhIjp7ImVudGl0aWVzIjpbeyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiXSwiaWQiOiJqb2JzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIGpvYl9pZCBGSyJdLCJpZCI6ImpvYl9pdGVtcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCBqb2JfaWQgRksiLCJ1dWlkIHZlbmRvcl9pZCBGSyIsInN0cmluZyB0cmFja2luZ19yZWZlcmVuY2UiXSwiaWQiOiJzaGlwbWVudHMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsInV1aWQgc2hpcG1lbnRfaWQgRksiLCJ1dWlkIGpvYl9pdGVtX2lkIEZLIiwiaW50IGRlbGl2ZXJlZF9xdWFudGl0eSJdLCJpZCI6InNoaXBtZW50X2l0ZW1zIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiXSwiaWQiOiJ2ZW5kb3JzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIHZlbmRvcl9pZCBGSyIsInN0cmluZyByb2xlIl0sImlkIjoidmVuZG9yX3JvbGVzIn1dLCJyZWxhdGlvbnNoaXBzIjpbeyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJqb2JzIiwibGFiZWwiOiJjb250YWlucyIsInRvIjoiam9iX2l0ZW1zIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJqb2JzIiwibGFiZWwiOiJzaGlwcyBpbiIsInRvIjoic2hpcG1lbnRzIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJzaGlwbWVudHMiLCJsYWJlbCI6ImNvbnRhaW5zIiwidG8iOiJzaGlwbWVudF9pdGVtcyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoiam9iX2l0ZW1zIiwibGFiZWwiOiJkZWxpdmVyZWQgYXMiLCJ0byI6InNoaXBtZW50X2l0ZW1zIn0seyJjYXJkaW5hbGl0eSI6Im98LS1veyIsImZyb20iOiJ2ZW5kb3JzIiwibGFiZWwiOiJzaGlwcyIsInRvIjoic2hpcG1lbnRzIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJ2ZW5kb3JzIiwibGFiZWwiOiJzZXJ2ZXMgYXMiLCJ0byI6InZlbmRvcl9yb2xlcyJ9XX0sImRlc2NyaXB0aW9uIjoiT25lIGpvYiBtYXkgc2hpcCBpbiBzZXZlcmFsIGNvbnNpZ25tZW50cy4gU2hpcG1lbnQgaXRlbXMgcmVjb3JkIHdoaWNoIG9yZGVyZWQgaXRlbXMgYW5kIHF1YW50aXRpZXMgd2VyZSBzaGlwcGVkIGFuZCBkZWxpdmVyZWQuIEEgdmVuZG9yIGNhbiBoYXZlIHByb2R1Y3Rpb24sIHNoaXBwaW5nIGFuZCBvdGhlciByb2xlcywgc28gc2VwYXJhdGUgdmVuZG9yIHRhYmxlcyBhcmUgdW5uZWNlc3NhcnkuIiwiaWQiOiJzY2hlbWFfMDZfc2hpcHBpbmciLCJraW5kIjoiZXIiLCJzb3VyY2VTaGEyNTYiOiIwYjdmOGM1Yzc4MDFkY2IxMDM1ZTE1ZmU3ODBkOGM3YzZlNTRkZmIzYWVlNTgxYzEyNmQ5MjYzNjM0NTkwNGEyIiwic3R5bGVzIjpbXSwidGl0bGUiOiI2LiBTaGlwcGluZyBhbmQgdmVuZG9yIHJvbGVzIiwidmVyc2lvbiI6MX0
```

### 7. Expected costs, vendor bills and expenses

Job cost estimates are forecasts. Vendor bills record actual supplier charges. Bill lines optionally point to a job, production run, shipment and estimate. An overhead line has no job and uses its expense GL account. One bill can cover several jobs through separate lines. Paying it does not add the cost again.

```mermaid
erDiagram
  jobs {
    uuid id PK
  }
  job_cost_estimates {
    uuid id PK
    uuid job_id FK
    string category
    bigint estimated_minor
  }
  vendors {
    uuid id PK
  }
  vendor_bills {
    uuid id PK
    uuid vendor_id FK
    string purpose
    bigint total_minor
  }
  vendor_bill_lines {
    uuid id PK
    uuid vendor_bill_id FK
    uuid job_id FK
    uuid estimate_id FK
  }
  production_runs {
    uuid id PK
    uuid job_id FK
  }
  shipments {
    uuid id PK
    uuid job_id FK
  }
  jobs ||--o{ job_cost_estimates : "forecasts"
  jobs ||--o{ production_runs : "produces"
  jobs ||--o{ shipments : "dispatches"
  vendors ||--o{ vendor_bills : "bills"
  vendor_bills ||--o{ vendor_bill_lines : "contains"
  jobs o|--o{ vendor_bill_lines : "costed by"
  job_cost_estimates o|--o{ vendor_bill_lines : "compared with"
  production_runs o|--o{ vendor_bill_lines : "charged on"
  shipments o|--o{ vendor_bill_lines : "charged on"
%% portable-canonical-v2:eyJhY2Nlc3NpYmlsaXR5IjpudWxsLCJkYXRhIjp7ImVudGl0aWVzIjpbeyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiXSwiaWQiOiJqb2JzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIGpvYl9pZCBGSyIsInN0cmluZyBjYXRlZ29yeSIsImJpZ2ludCBlc3RpbWF0ZWRfbWlub3IiXSwiaWQiOiJqb2JfY29zdF9lc3RpbWF0ZXMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyJdLCJpZCI6InZlbmRvcnMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsInV1aWQgdmVuZG9yX2lkIEZLIiwic3RyaW5nIHB1cnBvc2UiLCJiaWdpbnQgdG90YWxfbWlub3IiXSwiaWQiOiJ2ZW5kb3JfYmlsbHMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsInV1aWQgdmVuZG9yX2JpbGxfaWQgRksiLCJ1dWlkIGpvYl9pZCBGSyIsInV1aWQgZXN0aW1hdGVfaWQgRksiXSwiaWQiOiJ2ZW5kb3JfYmlsbF9saW5lcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCBqb2JfaWQgRksiXSwiaWQiOiJwcm9kdWN0aW9uX3J1bnMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsInV1aWQgam9iX2lkIEZLIl0sImlkIjoic2hpcG1lbnRzIn1dLCJyZWxhdGlvbnNoaXBzIjpbeyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJqb2JzIiwibGFiZWwiOiJmb3JlY2FzdHMiLCJ0byI6ImpvYl9jb3N0X2VzdGltYXRlcyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoiam9icyIsImxhYmVsIjoicHJvZHVjZXMiLCJ0byI6InByb2R1Y3Rpb25fcnVucyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoiam9icyIsImxhYmVsIjoiZGlzcGF0Y2hlcyIsInRvIjoic2hpcG1lbnRzIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJ2ZW5kb3JzIiwibGFiZWwiOiJiaWxscyIsInRvIjoidmVuZG9yX2JpbGxzIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJ2ZW5kb3JfYmlsbHMiLCJsYWJlbCI6ImNvbnRhaW5zIiwidG8iOiJ2ZW5kb3JfYmlsbF9saW5lcyJ9LHsiY2FyZGluYWxpdHkiOiJvfC0tb3siLCJmcm9tIjoiam9icyIsImxhYmVsIjoiY29zdGVkIGJ5IiwidG8iOiJ2ZW5kb3JfYmlsbF9saW5lcyJ9LHsiY2FyZGluYWxpdHkiOiJvfC0tb3siLCJmcm9tIjoiam9iX2Nvc3RfZXN0aW1hdGVzIiwibGFiZWwiOiJjb21wYXJlZCB3aXRoIiwidG8iOiJ2ZW5kb3JfYmlsbF9saW5lcyJ9LHsiY2FyZGluYWxpdHkiOiJvfC0tb3siLCJmcm9tIjoicHJvZHVjdGlvbl9ydW5zIiwibGFiZWwiOiJjaGFyZ2VkIG9uIiwidG8iOiJ2ZW5kb3JfYmlsbF9saW5lcyJ9LHsiY2FyZGluYWxpdHkiOiJvfC0tb3siLCJmcm9tIjoic2hpcG1lbnRzIiwibGFiZWwiOiJjaGFyZ2VkIG9uIiwidG8iOiJ2ZW5kb3JfYmlsbF9saW5lcyJ9XX0sImRlc2NyaXB0aW9uIjoiSm9iIGNvc3QgZXN0aW1hdGVzIGFyZSBmb3JlY2FzdHMuIFZlbmRvciBiaWxscyByZWNvcmQgYWN0dWFsIHN1cHBsaWVyIGNoYXJnZXMuIEJpbGwgbGluZXMgb3B0aW9uYWxseSBwb2ludCB0byBhIGpvYiwgcHJvZHVjdGlvbiBydW4sIHNoaXBtZW50IGFuZCBlc3RpbWF0ZS4gQW4gb3ZlcmhlYWQgbGluZSBoYXMgbm8gam9iIGFuZCB1c2VzIGl0cyBleHBlbnNlIEdMIGFjY291bnQuIE9uZSBiaWxsIGNhbiBjb3ZlciBzZXZlcmFsIGpvYnMgdGhyb3VnaCBzZXBhcmF0ZSBsaW5lcy4gUGF5aW5nIGl0IGRvZXMgbm90IGFkZCB0aGUgY29zdCBhZ2Fpbi4iLCJpZCI6InNjaGVtYV8wN19jb3N0cyIsImtpbmQiOiJlciIsInNvdXJjZVNoYTI1NiI6IjM0NDViZmE2NmRjZTZjOTEzMjdiZWY3ZDAwNTMyZDY0YWI5YzIzZTVlYjZkOWYyZmU0ODgwOTljYmVhNTM1OTUiLCJzdHlsZXMiOltdLCJ0aXRsZSI6IjcuIEV4cGVjdGVkIGNvc3RzLCB2ZW5kb3IgYmlsbHMgYW5kIGV4cGVuc2VzIiwidmVyc2lvbiI6MX0
```

### 8. Financial records behind balances and reports

Financial actions create balanced journals automatically. A posted payment references its journal and its exact cash line. Money-account balances come from ledger lines. Adjustments record documented opening balances, accruals and corrections. A posted journal needs at least two lines and equal debits and credits. No separate stored balance or profit table is needed.

```mermaid
erDiagram
  gl_accounts {
    uuid id PK
    string account_type
  }
  money_accounts {
    uuid id PK
    uuid gl_account_id FK
    string currency FK
  }
  journal_entries {
    uuid id PK
    uuid policy_id FK
    date accounting_date
    string status
  }
  journal_lines {
    uuid id PK
    uuid journal_entry_id FK
    uuid gl_account_id FK
    bigint debit_base_minor
    bigint credit_base_minor
  }
  payments {
    uuid id PK
    uuid journal_entry_id FK
    uuid cash_journal_line_id FK
  }
  accounting_adjustments {
    uuid id PK
    uuid journal_entry_id FK
    string kind
  }
  accounting_policies {
    uuid id PK
    int version
  }
  gl_accounts ||--o| money_accounts : "backs account"
  gl_accounts ||--o{ journal_lines : "classified in"
  journal_entries ||--o{ journal_lines : "contains"
  accounting_policies o|--o{ journal_entries : "governs"
  journal_entries ||--o{ accounting_adjustments : "posts adjustment"
  journal_entries o|--o{ payments : "posts payment"
  journal_lines o|--o| payments : "cash leg"
%% portable-canonical-v2:eyJhY2Nlc3NpYmlsaXR5IjpudWxsLCJkYXRhIjp7ImVudGl0aWVzIjpbeyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJzdHJpbmcgYWNjb3VudF90eXBlIl0sImlkIjoiZ2xfYWNjb3VudHMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsInV1aWQgZ2xfYWNjb3VudF9pZCBGSyIsInN0cmluZyBjdXJyZW5jeSBGSyJdLCJpZCI6Im1vbmV5X2FjY291bnRzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIHBvbGljeV9pZCBGSyIsImRhdGUgYWNjb3VudGluZ19kYXRlIiwic3RyaW5nIHN0YXR1cyJdLCJpZCI6ImpvdXJuYWxfZW50cmllcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCBqb3VybmFsX2VudHJ5X2lkIEZLIiwidXVpZCBnbF9hY2NvdW50X2lkIEZLIiwiYmlnaW50IGRlYml0X2Jhc2VfbWlub3IiLCJiaWdpbnQgY3JlZGl0X2Jhc2VfbWlub3IiXSwiaWQiOiJqb3VybmFsX2xpbmVzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIGpvdXJuYWxfZW50cnlfaWQgRksiLCJ1dWlkIGNhc2hfam91cm5hbF9saW5lX2lkIEZLIl0sImlkIjoicGF5bWVudHMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsInV1aWQgam91cm5hbF9lbnRyeV9pZCBGSyIsInN0cmluZyBraW5kIl0sImlkIjoiYWNjb3VudGluZ19hZGp1c3RtZW50cyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwiaW50IHZlcnNpb24iXSwiaWQiOiJhY2NvdW50aW5nX3BvbGljaWVzIn1dLCJyZWxhdGlvbnNoaXBzIjpbeyJjYXJkaW5hbGl0eSI6Inx8LS1vfCIsImZyb20iOiJnbF9hY2NvdW50cyIsImxhYmVsIjoiYmFja3MgYWNjb3VudCIsInRvIjoibW9uZXlfYWNjb3VudHMifSx7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6ImdsX2FjY291bnRzIiwibGFiZWwiOiJjbGFzc2lmaWVkIGluIiwidG8iOiJqb3VybmFsX2xpbmVzIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJqb3VybmFsX2VudHJpZXMiLCJsYWJlbCI6ImNvbnRhaW5zIiwidG8iOiJqb3VybmFsX2xpbmVzIn0seyJjYXJkaW5hbGl0eSI6Im98LS1veyIsImZyb20iOiJhY2NvdW50aW5nX3BvbGljaWVzIiwibGFiZWwiOiJnb3Zlcm5zIiwidG8iOiJqb3VybmFsX2VudHJpZXMifSx7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6ImpvdXJuYWxfZW50cmllcyIsImxhYmVsIjoicG9zdHMgYWRqdXN0bWVudCIsInRvIjoiYWNjb3VudGluZ19hZGp1c3RtZW50cyJ9LHsiY2FyZGluYWxpdHkiOiJvfC0tb3siLCJmcm9tIjoiam91cm5hbF9lbnRyaWVzIiwibGFiZWwiOiJwb3N0cyBwYXltZW50IiwidG8iOiJwYXltZW50cyJ9LHsiY2FyZGluYWxpdHkiOiJvfC0tb3wiLCJmcm9tIjoiam91cm5hbF9saW5lcyIsImxhYmVsIjoiY2FzaCBsZWciLCJ0byI6InBheW1lbnRzIn1dfSwiZGVzY3JpcHRpb24iOiJGaW5hbmNpYWwgYWN0aW9ucyBjcmVhdGUgYmFsYW5jZWQgam91cm5hbHMgYXV0b21hdGljYWxseS4gQSBwb3N0ZWQgcGF5bWVudCByZWZlcmVuY2VzIGl0cyBqb3VybmFsIGFuZCBpdHMgZXhhY3QgY2FzaCBsaW5lLiBNb25leS1hY2NvdW50IGJhbGFuY2VzIGNvbWUgZnJvbSBsZWRnZXIgbGluZXMuIEFkanVzdG1lbnRzIHJlY29yZCBkb2N1bWVudGVkIG9wZW5pbmcgYmFsYW5jZXMsIGFjY3J1YWxzIGFuZCBjb3JyZWN0aW9ucy4gQSBwb3N0ZWQgam91cm5hbCBuZWVkcyBhdCBsZWFzdCB0d28gbGluZXMgYW5kIGVxdWFsIGRlYml0cyBhbmQgY3JlZGl0cy4gTm8gc2VwYXJhdGUgc3RvcmVkIGJhbGFuY2Ugb3IgcHJvZml0IHRhYmxlIGlzIG5lZWRlZC4iLCJpZCI6InNjaGVtYV8wOF9sZWRnZXIiLCJraW5kIjoiZXIiLCJzb3VyY2VTaGEyNTYiOiI5YTY2YmJhZWNmN2Y3MTI1NjQyYjA3MGYyNzdlMzFmNmQ0NGExZDBmNTU1YTY1MjcwY2U1M2M0MjFhM2FhMDI0Iiwic3R5bGVzIjpbXSwidGl0bGUiOiI4LiBGaW5hbmNpYWwgcmVjb3JkcyBiZWhpbmQgYmFsYW5jZXMgYW5kIHJlcG9ydHMiLCJ2ZXJzaW9uIjoxfQ
```

### 9. Transfers, payment disputes and account checks

A transfer has a source and destination account and two payment legs. Transfer principal is neither sales nor expense. A dispute is a case record; an actual debit or recovery is a separate linked payment. Reconciliation matches statement evidence to existing cash journal lines without creating duplicate payments.

```mermaid
erDiagram
  money_accounts {
    uuid id PK
    string currency FK
  }
  account_transfers {
    uuid id PK
    uuid from_account_id FK
    uuid to_account_id FK
  }
  payments {
    uuid id PK
    uuid transfer_id FK
  }
  payment_disputes {
    uuid id PK
    uuid payment_id FK
    string status
  }
  reconciliation_sessions {
    uuid id PK
    uuid money_account_id FK
    string status
  }
  reconciliation_items {
    uuid id PK
    uuid session_id FK
    uuid cash_journal_line_id FK
  }
  journal_lines {
    uuid id PK
  }
  money_accounts ||--o{ account_transfers : "source"
  money_accounts ||--o{ account_transfers : "destination"
  account_transfers o|--o{ payments : "has two legs"
  payments ||--o{ payment_disputes : "disputed in"
  money_accounts ||--o{ reconciliation_sessions : "checked against statement"
  reconciliation_sessions ||--o{ reconciliation_items : "contains matches"
  journal_lines ||--o| reconciliation_items : "matched cash line"
%% portable-canonical-v2:eyJhY2Nlc3NpYmlsaXR5IjpudWxsLCJkYXRhIjp7ImVudGl0aWVzIjpbeyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJzdHJpbmcgY3VycmVuY3kgRksiXSwiaWQiOiJtb25leV9hY2NvdW50cyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCBmcm9tX2FjY291bnRfaWQgRksiLCJ1dWlkIHRvX2FjY291bnRfaWQgRksiXSwiaWQiOiJhY2NvdW50X3RyYW5zZmVycyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCB0cmFuc2Zlcl9pZCBGSyJdLCJpZCI6InBheW1lbnRzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIHBheW1lbnRfaWQgRksiLCJzdHJpbmcgc3RhdHVzIl0sImlkIjoicGF5bWVudF9kaXNwdXRlcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCBtb25leV9hY2NvdW50X2lkIEZLIiwic3RyaW5nIHN0YXR1cyJdLCJpZCI6InJlY29uY2lsaWF0aW9uX3Nlc3Npb25zIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIHNlc3Npb25faWQgRksiLCJ1dWlkIGNhc2hfam91cm5hbF9saW5lX2lkIEZLIl0sImlkIjoicmVjb25jaWxpYXRpb25faXRlbXMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyJdLCJpZCI6ImpvdXJuYWxfbGluZXMifV0sInJlbGF0aW9uc2hpcHMiOlt7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6Im1vbmV5X2FjY291bnRzIiwibGFiZWwiOiJzb3VyY2UiLCJ0byI6ImFjY291bnRfdHJhbnNmZXJzIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJtb25leV9hY2NvdW50cyIsImxhYmVsIjoiZGVzdGluYXRpb24iLCJ0byI6ImFjY291bnRfdHJhbnNmZXJzIn0seyJjYXJkaW5hbGl0eSI6Im98LS1veyIsImZyb20iOiJhY2NvdW50X3RyYW5zZmVycyIsImxhYmVsIjoiaGFzIHR3byBsZWdzIiwidG8iOiJwYXltZW50cyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoicGF5bWVudHMiLCJsYWJlbCI6ImRpc3B1dGVkIGluIiwidG8iOiJwYXltZW50X2Rpc3B1dGVzIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJtb25leV9hY2NvdW50cyIsImxhYmVsIjoiY2hlY2tlZCBhZ2FpbnN0IHN0YXRlbWVudCIsInRvIjoicmVjb25jaWxpYXRpb25fc2Vzc2lvbnMifSx7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6InJlY29uY2lsaWF0aW9uX3Nlc3Npb25zIiwibGFiZWwiOiJjb250YWlucyBtYXRjaGVzIiwidG8iOiJyZWNvbmNpbGlhdGlvbl9pdGVtcyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3wiLCJmcm9tIjoiam91cm5hbF9saW5lcyIsImxhYmVsIjoibWF0Y2hlZCBjYXNoIGxpbmUiLCJ0byI6InJlY29uY2lsaWF0aW9uX2l0ZW1zIn1dfSwiZGVzY3JpcHRpb24iOiJBIHRyYW5zZmVyIGhhcyBhIHNvdXJjZSBhbmQgZGVzdGluYXRpb24gYWNjb3VudCBhbmQgdHdvIHBheW1lbnQgbGVncy4gVHJhbnNmZXIgcHJpbmNpcGFsIGlzIG5laXRoZXIgc2FsZXMgbm9yIGV4cGVuc2UuIEEgZGlzcHV0ZSBpcyBhIGNhc2UgcmVjb3JkOyBhbiBhY3R1YWwgZGViaXQgb3IgcmVjb3ZlcnkgaXMgYSBzZXBhcmF0ZSBsaW5rZWQgcGF5bWVudC4gUmVjb25jaWxpYXRpb24gbWF0Y2hlcyBzdGF0ZW1lbnQgZXZpZGVuY2UgdG8gZXhpc3RpbmcgY2FzaCBqb3VybmFsIGxpbmVzIHdpdGhvdXQgY3JlYXRpbmcgZHVwbGljYXRlIHBheW1lbnRzLiIsImlkIjoic2NoZW1hXzA5X3RyYW5zZmVyc19yZWNvbmNpbGlhdGlvbiIsImtpbmQiOiJlciIsInNvdXJjZVNoYTI1NiI6IjUwMTNkZmQ3Y2U3NjA3ZmUxMDliZTJlNTcwNDFjYmRlNTM0ZDQ2Zjc1ZmZkMDA5ZjQ5ZTNiMTkzNjM3OTI1OWEiLCJzdHlsZXMiOltdLCJ0aXRsZSI6IjkuIFRyYW5zZmVycywgcGF5bWVudCBkaXNwdXRlcyBhbmQgYWNjb3VudCBjaGVja3MiLCJ2ZXJzaW9uIjoxfQ
```

### 10. Job progress and recognised profit

Recognition records connect earned revenue and released costs to the relevant job items and financial journal under the chosen policy. A customer receipt alone does not decide when revenue is earned. Forecast margin, recognised profit and cash flow are different views.

```mermaid
erDiagram
  jobs {
    uuid id PK
  }
  job_items {
    uuid id PK
    uuid job_id FK
  }
  recognition_events {
    uuid id PK
    uuid job_id FK
    uuid policy_id FK
    uuid journal_entry_id FK
  }
  recognition_lines {
    uuid id PK
    uuid recognition_event_id FK
    uuid job_item_id FK
    decimal recognised_quantity
  }
  journal_entries {
    uuid id PK
  }
  accounting_policies {
    uuid id PK
  }
  jobs ||--o{ job_items : "contains"
  jobs ||--o{ recognition_events : "recognises value"
  recognition_events ||--o{ recognition_lines : "contains"
  job_items ||--o{ recognition_lines : "recognised as"
  journal_entries ||--o{ recognition_events : "posts"
  accounting_policies ||--o{ recognition_events : "governs"
%% portable-canonical-v2:eyJhY2Nlc3NpYmlsaXR5IjpudWxsLCJkYXRhIjp7ImVudGl0aWVzIjpbeyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiXSwiaWQiOiJqb2JzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIGpvYl9pZCBGSyJdLCJpZCI6ImpvYl9pdGVtcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCBqb2JfaWQgRksiLCJ1dWlkIHBvbGljeV9pZCBGSyIsInV1aWQgam91cm5hbF9lbnRyeV9pZCBGSyJdLCJpZCI6InJlY29nbml0aW9uX2V2ZW50cyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCByZWNvZ25pdGlvbl9ldmVudF9pZCBGSyIsInV1aWQgam9iX2l0ZW1faWQgRksiLCJkZWNpbWFsIHJlY29nbmlzZWRfcXVhbnRpdHkiXSwiaWQiOiJyZWNvZ25pdGlvbl9saW5lcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIl0sImlkIjoiam91cm5hbF9lbnRyaWVzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiXSwiaWQiOiJhY2NvdW50aW5nX3BvbGljaWVzIn1dLCJyZWxhdGlvbnNoaXBzIjpbeyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJqb2JzIiwibGFiZWwiOiJjb250YWlucyIsInRvIjoiam9iX2l0ZW1zIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJqb2JzIiwibGFiZWwiOiJyZWNvZ25pc2VzIHZhbHVlIiwidG8iOiJyZWNvZ25pdGlvbl9ldmVudHMifSx7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6InJlY29nbml0aW9uX2V2ZW50cyIsImxhYmVsIjoiY29udGFpbnMiLCJ0byI6InJlY29nbml0aW9uX2xpbmVzIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJqb2JfaXRlbXMiLCJsYWJlbCI6InJlY29nbmlzZWQgYXMiLCJ0byI6InJlY29nbml0aW9uX2xpbmVzIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJqb3VybmFsX2VudHJpZXMiLCJsYWJlbCI6InBvc3RzIiwidG8iOiJyZWNvZ25pdGlvbl9ldmVudHMifSx7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6ImFjY291bnRpbmdfcG9saWNpZXMiLCJsYWJlbCI6ImdvdmVybnMiLCJ0byI6InJlY29nbml0aW9uX2V2ZW50cyJ9XX0sImRlc2NyaXB0aW9uIjoiUmVjb2duaXRpb24gcmVjb3JkcyBjb25uZWN0IGVhcm5lZCByZXZlbnVlIGFuZCByZWxlYXNlZCBjb3N0cyB0byB0aGUgcmVsZXZhbnQgam9iIGl0ZW1zIGFuZCBmaW5hbmNpYWwgam91cm5hbCB1bmRlciB0aGUgY2hvc2VuIHBvbGljeS4gQSBjdXN0b21lciByZWNlaXB0IGFsb25lIGRvZXMgbm90IGRlY2lkZSB3aGVuIHJldmVudWUgaXMgZWFybmVkLiBGb3JlY2FzdCBtYXJnaW4sIHJlY29nbmlzZWQgcHJvZml0IGFuZCBjYXNoIGZsb3cgYXJlIGRpZmZlcmVudCB2aWV3cy4iLCJpZCI6InNjaGVtYV8xMF9yZWNvZ25pdGlvbiIsImtpbmQiOiJlciIsInNvdXJjZVNoYTI1NiI6ImQzOTBmYjE0Mzk5MmQzZDQ5OTAzZjc2OTZkMDkwOThlYTNjM2JlZTM5ODZiZjFmNzBjZDM5N2JlOWU1N2VjNGEiLCJzdHlsZXMiOltdLCJ0aXRsZSI6IjEwLiBKb2IgcHJvZ3Jlc3MgYW5kIHJlY29nbmlzZWQgcHJvZml0IiwidmVyc2lvbiI6MX0
```

### 11. Partner allocations and cash movements

Partner accounts separate capital, loans and distributable balances. A profit distribution allocates an approved amount from a closed period; its lines give each partner their amount. This creates no payment. A contribution or withdrawal uses the shared payments table and a partner event. No 50/50 ownership assumption is built in.

```mermaid
erDiagram
  partners {
    uuid id PK
    string display_name
  }
  partner_accounts {
    uuid id PK
    uuid partner_id FK
    string bucket
    uuid gl_account_id FK
  }
  partner_events {
    uuid id PK
    uuid partner_account_id FK
    uuid payment_id FK
    uuid distribution_line_id FK
  }
  profit_distributions {
    uuid id PK
    uuid period_id FK
    bigint distributed_minor
  }
  profit_distribution_lines {
    uuid id PK
    uuid distribution_id FK
    uuid partner_account_id FK
    bigint amount_minor
  }
  accounting_periods {
    uuid id PK
    string status
  }
  payments {
    uuid id PK
    uuid partner_account_id FK
  }
  partners ||--o{ partner_accounts : "holds"
  partner_accounts ||--o{ partner_events : "records movements"
  accounting_periods ||--o{ profit_distributions : "earnings allocated"
  profit_distributions ||--o{ profit_distribution_lines : "splits amount"
  partner_accounts ||--o{ profit_distribution_lines : "receives allocation"
  profit_distribution_lines o|--o{ partner_events : "records entitlement"
  payments o|--o{ partner_events : "records cash movement"
  partner_accounts o|--o{ payments : "contributes or withdraws"
%% portable-canonical-v2:eyJhY2Nlc3NpYmlsaXR5IjpudWxsLCJkYXRhIjp7ImVudGl0aWVzIjpbeyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJzdHJpbmcgZGlzcGxheV9uYW1lIl0sImlkIjoicGFydG5lcnMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsInV1aWQgcGFydG5lcl9pZCBGSyIsInN0cmluZyBidWNrZXQiLCJ1dWlkIGdsX2FjY291bnRfaWQgRksiXSwiaWQiOiJwYXJ0bmVyX2FjY291bnRzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIHBhcnRuZXJfYWNjb3VudF9pZCBGSyIsInV1aWQgcGF5bWVudF9pZCBGSyIsInV1aWQgZGlzdHJpYnV0aW9uX2xpbmVfaWQgRksiXSwiaWQiOiJwYXJ0bmVyX2V2ZW50cyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCBwZXJpb2RfaWQgRksiLCJiaWdpbnQgZGlzdHJpYnV0ZWRfbWlub3IiXSwiaWQiOiJwcm9maXRfZGlzdHJpYnV0aW9ucyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCBkaXN0cmlidXRpb25faWQgRksiLCJ1dWlkIHBhcnRuZXJfYWNjb3VudF9pZCBGSyIsImJpZ2ludCBhbW91bnRfbWlub3IiXSwiaWQiOiJwcm9maXRfZGlzdHJpYnV0aW9uX2xpbmVzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJzdHJpbmcgc3RhdHVzIl0sImlkIjoiYWNjb3VudGluZ19wZXJpb2RzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIHBhcnRuZXJfYWNjb3VudF9pZCBGSyJdLCJpZCI6InBheW1lbnRzIn1dLCJyZWxhdGlvbnNoaXBzIjpbeyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJwYXJ0bmVycyIsImxhYmVsIjoiaG9sZHMiLCJ0byI6InBhcnRuZXJfYWNjb3VudHMifSx7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6InBhcnRuZXJfYWNjb3VudHMiLCJsYWJlbCI6InJlY29yZHMgbW92ZW1lbnRzIiwidG8iOiJwYXJ0bmVyX2V2ZW50cyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoiYWNjb3VudGluZ19wZXJpb2RzIiwibGFiZWwiOiJlYXJuaW5ncyBhbGxvY2F0ZWQiLCJ0byI6InByb2ZpdF9kaXN0cmlidXRpb25zIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJwcm9maXRfZGlzdHJpYnV0aW9ucyIsImxhYmVsIjoic3BsaXRzIGFtb3VudCIsInRvIjoicHJvZml0X2Rpc3RyaWJ1dGlvbl9saW5lcyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoicGFydG5lcl9hY2NvdW50cyIsImxhYmVsIjoicmVjZWl2ZXMgYWxsb2NhdGlvbiIsInRvIjoicHJvZml0X2Rpc3RyaWJ1dGlvbl9saW5lcyJ9LHsiY2FyZGluYWxpdHkiOiJvfC0tb3siLCJmcm9tIjoicHJvZml0X2Rpc3RyaWJ1dGlvbl9saW5lcyIsImxhYmVsIjoicmVjb3JkcyBlbnRpdGxlbWVudCIsInRvIjoicGFydG5lcl9ldmVudHMifSx7ImNhcmRpbmFsaXR5Ijoib3wtLW97IiwiZnJvbSI6InBheW1lbnRzIiwibGFiZWwiOiJyZWNvcmRzIGNhc2ggbW92ZW1lbnQiLCJ0byI6InBhcnRuZXJfZXZlbnRzIn0seyJjYXJkaW5hbGl0eSI6Im98LS1veyIsImZyb20iOiJwYXJ0bmVyX2FjY291bnRzIiwibGFiZWwiOiJjb250cmlidXRlcyBvciB3aXRoZHJhd3MiLCJ0byI6InBheW1lbnRzIn1dfSwiZGVzY3JpcHRpb24iOiJQYXJ0bmVyIGFjY291bnRzIHNlcGFyYXRlIGNhcGl0YWwsIGxvYW5zIGFuZCBkaXN0cmlidXRhYmxlIGJhbGFuY2VzLiBBIHByb2ZpdCBkaXN0cmlidXRpb24gYWxsb2NhdGVzIGFuIGFwcHJvdmVkIGFtb3VudCBmcm9tIGEgY2xvc2VkIHBlcmlvZDsgaXRzIGxpbmVzIGdpdmUgZWFjaCBwYXJ0bmVyIHRoZWlyIGFtb3VudC4gVGhpcyBjcmVhdGVzIG5vIHBheW1lbnQuIEEgY29udHJpYnV0aW9uIG9yIHdpdGhkcmF3YWwgdXNlcyB0aGUgc2hhcmVkIHBheW1lbnRzIHRhYmxlIGFuZCBhIHBhcnRuZXIgZXZlbnQuIE5vIDUwLzUwIG93bmVyc2hpcCBhc3N1bXB0aW9uIGlzIGJ1aWx0IGluLiIsImlkIjoic2NoZW1hXzExX3BhcnRuZXJzIiwia2luZCI6ImVyIiwic291cmNlU2hhMjU2IjoiZjE0MWI2ZDA0YjU3YTZmNDAyNzM1NjA3MzIwMzAzYmRhNWQzMjZmODBlMTc5ZWU4MTJiOTlkZGYyYTBhYmNlNCIsInN0eWxlcyI6W10sInRpdGxlIjoiMTEuIFBhcnRuZXIgYWxsb2NhdGlvbnMgYW5kIGNhc2ggbW92ZW1lbnRzIiwidmVyc2lvbiI6MX0
```

### 12. PDFs, receipts, artwork and delivery evidence

A generated document has exactly one subject: invoice, credit note or payment receipt. Its file can be absent while generation is pending. Sending attempts are separate from document generation. Job attachments reuse the files table. Actual file bytes live in private object storage; these tables store metadata and links.

```mermaid
erDiagram
  invoices {
    uuid id PK
  }
  payments {
    uuid id PK
  }
  customer_credit_notes {
    uuid id PK
  }
  generated_documents {
    uuid id PK
    string kind
    uuid file_id FK
    string document_number
  }
  document_deliveries {
    uuid id PK
    uuid document_id FK
    string channel
    string status
  }
  files {
    uuid id PK
    string object_key
    string original_name
  }
  jobs {
    uuid id PK
  }
  job_file_links {
    uuid id PK
    uuid job_id FK
    uuid file_id FK
    string purpose
  }
  invoices o|--o{ generated_documents : "invoice PDF"
  payments o|--o{ generated_documents : "receipt PDF"
  customer_credit_notes o|--o{ generated_documents : "credit PDF"
  files o|--o{ generated_documents : "stores output"
  generated_documents ||--o{ document_deliveries : "sent through"
  jobs ||--o{ job_file_links : "has attachments"
  files ||--o{ job_file_links : "linked to job"
%% portable-canonical-v2:eyJhY2Nlc3NpYmlsaXR5IjpudWxsLCJkYXRhIjp7ImVudGl0aWVzIjpbeyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiXSwiaWQiOiJpbnZvaWNlcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIl0sImlkIjoicGF5bWVudHMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyJdLCJpZCI6ImN1c3RvbWVyX2NyZWRpdF9ub3RlcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwic3RyaW5nIGtpbmQiLCJ1dWlkIGZpbGVfaWQgRksiLCJzdHJpbmcgZG9jdW1lbnRfbnVtYmVyIl0sImlkIjoiZ2VuZXJhdGVkX2RvY3VtZW50cyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCBkb2N1bWVudF9pZCBGSyIsInN0cmluZyBjaGFubmVsIiwic3RyaW5nIHN0YXR1cyJdLCJpZCI6ImRvY3VtZW50X2RlbGl2ZXJpZXMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsInN0cmluZyBvYmplY3Rfa2V5Iiwic3RyaW5nIG9yaWdpbmFsX25hbWUiXSwiaWQiOiJmaWxlcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIl0sImlkIjoiam9icyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCBqb2JfaWQgRksiLCJ1dWlkIGZpbGVfaWQgRksiLCJzdHJpbmcgcHVycG9zZSJdLCJpZCI6ImpvYl9maWxlX2xpbmtzIn1dLCJyZWxhdGlvbnNoaXBzIjpbeyJjYXJkaW5hbGl0eSI6Im98LS1veyIsImZyb20iOiJpbnZvaWNlcyIsImxhYmVsIjoiaW52b2ljZSBQREYiLCJ0byI6ImdlbmVyYXRlZF9kb2N1bWVudHMifSx7ImNhcmRpbmFsaXR5Ijoib3wtLW97IiwiZnJvbSI6InBheW1lbnRzIiwibGFiZWwiOiJyZWNlaXB0IFBERiIsInRvIjoiZ2VuZXJhdGVkX2RvY3VtZW50cyJ9LHsiY2FyZGluYWxpdHkiOiJvfC0tb3siLCJmcm9tIjoiY3VzdG9tZXJfY3JlZGl0X25vdGVzIiwibGFiZWwiOiJjcmVkaXQgUERGIiwidG8iOiJnZW5lcmF0ZWRfZG9jdW1lbnRzIn0seyJjYXJkaW5hbGl0eSI6Im98LS1veyIsImZyb20iOiJmaWxlcyIsImxhYmVsIjoic3RvcmVzIG91dHB1dCIsInRvIjoiZ2VuZXJhdGVkX2RvY3VtZW50cyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoiZ2VuZXJhdGVkX2RvY3VtZW50cyIsImxhYmVsIjoic2VudCB0aHJvdWdoIiwidG8iOiJkb2N1bWVudF9kZWxpdmVyaWVzIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJqb2JzIiwibGFiZWwiOiJoYXMgYXR0YWNobWVudHMiLCJ0byI6ImpvYl9maWxlX2xpbmtzIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJmaWxlcyIsImxhYmVsIjoibGlua2VkIHRvIGpvYiIsInRvIjoiam9iX2ZpbGVfbGlua3MifV19LCJkZXNjcmlwdGlvbiI6IkEgZ2VuZXJhdGVkIGRvY3VtZW50IGhhcyBleGFjdGx5IG9uZSBzdWJqZWN0OiBpbnZvaWNlLCBjcmVkaXQgbm90ZSBvciBwYXltZW50IHJlY2VpcHQuIEl0cyBmaWxlIGNhbiBiZSBhYnNlbnQgd2hpbGUgZ2VuZXJhdGlvbiBpcyBwZW5kaW5nLiBTZW5kaW5nIGF0dGVtcHRzIGFyZSBzZXBhcmF0ZSBmcm9tIGRvY3VtZW50IGdlbmVyYXRpb24uIEpvYiBhdHRhY2htZW50cyByZXVzZSB0aGUgZmlsZXMgdGFibGUuIEFjdHVhbCBmaWxlIGJ5dGVzIGxpdmUgaW4gcHJpdmF0ZSBvYmplY3Qgc3RvcmFnZTsgdGhlc2UgdGFibGVzIHN0b3JlIG1ldGFkYXRhIGFuZCBsaW5rcy4iLCJpZCI6InNjaGVtYV8xMl9kb2N1bWVudHMiLCJraW5kIjoiZXIiLCJzb3VyY2VTaGEyNTYiOiIxOTAyZGI3ZGM0Y2YyMjAwMGUyZWUzZWU2MTIxM2I1ZTY5YzMxYzkyNjBkOTgwMzI2NWMwYzJmZmJkY2RlZDIyIiwic3R5bGVzIjpbXSwidGl0bGUiOiIxMi4gUERGcywgcmVjZWlwdHMsIGFydHdvcmsgYW5kIGRlbGl2ZXJ5IGV2aWRlbmNlIiwidmVyc2lvbiI6MX0
```

### 13. Audit and reliable commands

These tables run in the background. Audit records who changed something; idempotency prevents a repeated click or retry from posting twice; outbox tasks retry PDF generation and other follow-up work after the business transaction commits. Audit subjects and task payload references are application metadata, not invented financial foreign keys.

```mermaid
erDiagram
  legal_entities {
    uuid id PK
  }
  app_users {
    uuid id PK
  }
  audit_events {
    uuid id PK
    uuid actor_id FK
    string subject_type
    uuid subject_id
    string action
  }
  idempotency_requests {
    uuid id PK
    string command
    string key
    string request_hash
  }
  outbox_tasks {
    uuid id PK
    string task_type
    string dedupe_key
    string status
  }
  legal_entities ||--o{ audit_events : "audits"
  app_users o|--o{ audit_events : "performed action"
  legal_entities ||--o{ idempotency_requests : "deduplicates commands"
  legal_entities ||--o{ outbox_tasks : "queues follow-up work"
%% portable-canonical-v2:eyJhY2Nlc3NpYmlsaXR5IjpudWxsLCJkYXRhIjp7ImVudGl0aWVzIjpbeyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiXSwiaWQiOiJsZWdhbF9lbnRpdGllcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIl0sImlkIjoiYXBwX3VzZXJzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIGFjdG9yX2lkIEZLIiwic3RyaW5nIHN1YmplY3RfdHlwZSIsInV1aWQgc3ViamVjdF9pZCIsInN0cmluZyBhY3Rpb24iXSwiaWQiOiJhdWRpdF9ldmVudHMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsInN0cmluZyBjb21tYW5kIiwic3RyaW5nIGtleSIsInN0cmluZyByZXF1ZXN0X2hhc2giXSwiaWQiOiJpZGVtcG90ZW5jeV9yZXF1ZXN0cyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwic3RyaW5nIHRhc2tfdHlwZSIsInN0cmluZyBkZWR1cGVfa2V5Iiwic3RyaW5nIHN0YXR1cyJdLCJpZCI6Im91dGJveF90YXNrcyJ9XSwicmVsYXRpb25zaGlwcyI6W3siY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoibGVnYWxfZW50aXRpZXMiLCJsYWJlbCI6ImF1ZGl0cyIsInRvIjoiYXVkaXRfZXZlbnRzIn0seyJjYXJkaW5hbGl0eSI6Im98LS1veyIsImZyb20iOiJhcHBfdXNlcnMiLCJsYWJlbCI6InBlcmZvcm1lZCBhY3Rpb24iLCJ0byI6ImF1ZGl0X2V2ZW50cyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoibGVnYWxfZW50aXRpZXMiLCJsYWJlbCI6ImRlZHVwbGljYXRlcyBjb21tYW5kcyIsInRvIjoiaWRlbXBvdGVuY3lfcmVxdWVzdHMifSx7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6ImxlZ2FsX2VudGl0aWVzIiwibGFiZWwiOiJxdWV1ZXMgZm9sbG93LXVwIHdvcmsiLCJ0byI6Im91dGJveF90YXNrcyJ9XX0sImRlc2NyaXB0aW9uIjoiVGhlc2UgdGFibGVzIHJ1biBpbiB0aGUgYmFja2dyb3VuZC4gQXVkaXQgcmVjb3JkcyB3aG8gY2hhbmdlZCBzb21ldGhpbmc7IGlkZW1wb3RlbmN5IHByZXZlbnRzIGEgcmVwZWF0ZWQgY2xpY2sgb3IgcmV0cnkgZnJvbSBwb3N0aW5nIHR3aWNlOyBvdXRib3ggdGFza3MgcmV0cnkgUERGIGdlbmVyYXRpb24gYW5kIG90aGVyIGZvbGxvdy11cCB3b3JrIGFmdGVyIHRoZSBidXNpbmVzcyB0cmFuc2FjdGlvbiBjb21taXRzLiBBdWRpdCBzdWJqZWN0cyBhbmQgdGFzayBwYXlsb2FkIHJlZmVyZW5jZXMgYXJlIGFwcGxpY2F0aW9uIG1ldGFkYXRhLCBub3QgaW52ZW50ZWQgZmluYW5jaWFsIGZvcmVpZ24ga2V5cy4iLCJpZCI6InNjaGVtYV8xM19yZWxpYWJpbGl0eSIsImtpbmQiOiJlciIsInNvdXJjZVNoYTI1NiI6IjdkZDRhYjlhNzNmYWM2N2E0ZDlkYzQyMzA2NDEwMzhiMjg1OTUyNTRlODE4ZDUxNGY2Mzg1MjE1YjMzODkxMWMiLCJzdHlsZXMiOltdLCJ0aXRsZSI6IjEzLiBBdWRpdCBhbmQgcmVsaWFibGUgY29tbWFuZHMiLCJ2ZXJzaW9uIjoxfQ
```

### 14. Four later additions

These four new tables are outside the 53-table complete basic release: recurring_expense_templates, profit_policies, profit_policy_shares and webhook_events. Templates create draft bills, not payments. Policy shares automate future allocation rules. A webhook event is an incoming provider message; it must be verified and processed through the same existing payment command.

```mermaid
erDiagram
  vendors {
    uuid id PK
  }
  recurring_expense_templates {
    uuid id PK
    uuid vendor_id FK
    date next_due_date
  }
  vendor_bills {
    uuid id PK
    uuid recurring_template_id FK
    date recurrence_period
  }
  legal_entities {
    uuid id PK
  }
  profit_policies {
    uuid id PK
    uuid legal_entity_id FK
    int version
  }
  profit_policy_shares {
    uuid id PK
    uuid policy_id FK
    uuid partner_id FK
    decimal share_percent
  }
  partners {
    uuid id PK
  }
  webhook_events {
    uuid id PK
    uuid legal_entity_id FK
    string external_event_id
    string processing_status
  }
  vendors ||--o{ recurring_expense_templates : "recurs for"
  recurring_expense_templates o|--o{ vendor_bills : "creates draft bills"
  legal_entities ||--o{ profit_policies : "defines future rules"
  profit_policies ||--o{ profit_policy_shares : "splits pool"
  partners ||--o{ profit_policy_shares : "assigned share"
  legal_entities ||--o{ webhook_events : "receives provider event"
%% portable-canonical-v2:eyJhY2Nlc3NpYmlsaXR5IjpudWxsLCJkYXRhIjp7ImVudGl0aWVzIjpbeyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiXSwiaWQiOiJ2ZW5kb3JzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiLCJ1dWlkIHZlbmRvcl9pZCBGSyIsImRhdGUgbmV4dF9kdWVfZGF0ZSJdLCJpZCI6InJlY3VycmluZ19leHBlbnNlX3RlbXBsYXRlcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCByZWN1cnJpbmdfdGVtcGxhdGVfaWQgRksiLCJkYXRlIHJlY3VycmVuY2VfcGVyaW9kIl0sImlkIjoidmVuZG9yX2JpbGxzIn0seyJhdHRyaWJ1dGVzIjpbInV1aWQgaWQgUEsiXSwiaWQiOiJsZWdhbF9lbnRpdGllcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCBsZWdhbF9lbnRpdHlfaWQgRksiLCJpbnQgdmVyc2lvbiJdLCJpZCI6InByb2ZpdF9wb2xpY2llcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIiwidXVpZCBwb2xpY3lfaWQgRksiLCJ1dWlkIHBhcnRuZXJfaWQgRksiLCJkZWNpbWFsIHNoYXJlX3BlcmNlbnQiXSwiaWQiOiJwcm9maXRfcG9saWN5X3NoYXJlcyJ9LHsiYXR0cmlidXRlcyI6WyJ1dWlkIGlkIFBLIl0sImlkIjoicGFydG5lcnMifSx7ImF0dHJpYnV0ZXMiOlsidXVpZCBpZCBQSyIsInV1aWQgbGVnYWxfZW50aXR5X2lkIEZLIiwic3RyaW5nIGV4dGVybmFsX2V2ZW50X2lkIiwic3RyaW5nIHByb2Nlc3Npbmdfc3RhdHVzIl0sImlkIjoid2ViaG9va19ldmVudHMifV0sInJlbGF0aW9uc2hpcHMiOlt7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6InZlbmRvcnMiLCJsYWJlbCI6InJlY3VycyBmb3IiLCJ0byI6InJlY3VycmluZ19leHBlbnNlX3RlbXBsYXRlcyJ9LHsiY2FyZGluYWxpdHkiOiJvfC0tb3siLCJmcm9tIjoicmVjdXJyaW5nX2V4cGVuc2VfdGVtcGxhdGVzIiwibGFiZWwiOiJjcmVhdGVzIGRyYWZ0IGJpbGxzIiwidG8iOiJ2ZW5kb3JfYmlsbHMifSx7ImNhcmRpbmFsaXR5IjoifHwtLW97IiwiZnJvbSI6ImxlZ2FsX2VudGl0aWVzIiwibGFiZWwiOiJkZWZpbmVzIGZ1dHVyZSBydWxlcyIsInRvIjoicHJvZml0X3BvbGljaWVzIn0seyJjYXJkaW5hbGl0eSI6Inx8LS1veyIsImZyb20iOiJwcm9maXRfcG9saWNpZXMiLCJsYWJlbCI6InNwbGl0cyBwb29sIiwidG8iOiJwcm9maXRfcG9saWN5X3NoYXJlcyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoicGFydG5lcnMiLCJsYWJlbCI6ImFzc2lnbmVkIHNoYXJlIiwidG8iOiJwcm9maXRfcG9saWN5X3NoYXJlcyJ9LHsiY2FyZGluYWxpdHkiOiJ8fC0tb3siLCJmcm9tIjoibGVnYWxfZW50aXRpZXMiLCJsYWJlbCI6InJlY2VpdmVzIHByb3ZpZGVyIGV2ZW50IiwidG8iOiJ3ZWJob29rX2V2ZW50cyJ9XX0sImRlc2NyaXB0aW9uIjoiVGhlc2UgZm91ciBuZXcgdGFibGVzIGFyZSBvdXRzaWRlIHRoZSA1My10YWJsZSBjb21wbGV0ZSBiYXNpYyByZWxlYXNlOiByZWN1cnJpbmdfZXhwZW5zZV90ZW1wbGF0ZXMsIHByb2ZpdF9wb2xpY2llcywgcHJvZml0X3BvbGljeV9zaGFyZXMgYW5kIHdlYmhvb2tfZXZlbnRzLiBUZW1wbGF0ZXMgY3JlYXRlIGRyYWZ0IGJpbGxzLCBub3QgcGF5bWVudHMuIFBvbGljeSBzaGFyZXMgYXV0b21hdGUgZnV0dXJlIGFsbG9jYXRpb24gcnVsZXMuIEEgd2ViaG9vayBldmVudCBpcyBhbiBpbmNvbWluZyBwcm92aWRlciBtZXNzYWdlOyBpdCBtdXN0IGJlIHZlcmlmaWVkIGFuZCBwcm9jZXNzZWQgdGhyb3VnaCB0aGUgc2FtZSBleGlzdGluZyBwYXltZW50IGNvbW1hbmQuIiwiaWQiOiJzY2hlbWFfMTRfbGF0ZXJfYWRkaXRpb25zIiwia2luZCI6ImVyIiwic291cmNlU2hhMjU2IjoiOTQxYWVjOTU4OTA3ZGY0MjM5ZDVlMWE4OGQ1N2M0ZjE5NDFmYWVjYWNlOWYwYzI0NjJiYzNlODYyZGI0NmI1YiIsInN0eWxlcyI6W10sInRpdGxlIjoiMTQuIEZvdXIgbGF0ZXIgYWRkaXRpb25zIiwidmVyc2lvbiI6MX0
```

## What stays separate and what shares a screen

| Screen or action | Separate underlying records |
| --- | --- |
| Invoice → Record payment | Payment, invoice allocation, journal, generated receipt and eligible job |
| Job → Production | Production runs, run items and cost estimates |
| Job → Shipping | Shipments, shipment items and shipping cost records |
| Costs → Paid now | Vendor bill, bill lines, payment and vendor allocation |
| Partners → Withdraw | Payment and partner event sharing one journal |
| Partners → Allocate profit | Distribution, distribution lines and partner events; no cash payment |
| Reports and balances | Queries over existing invoices, bills, jobs, allocations and journal lines |

Use one `jobs` table for orders. Production and shipping vendors share `vendors` plus `vendor_roles`. Expenses are overhead vendor-bill lines; the recurring template is optional later. Receipts are generated documents linked to payments. Dashboards, global/market views, P&L, account balances and vendor ledgers are queries/views, not additional independent records of money.

## Rules that the diagram alone cannot enforce

1. A verified payment, allocations, financial posting and any newly eligible job commit together. Retries must return the existing result. `jobs.origin_invoice_id` is unique.
2. No verified funds means no automatic job. Partial receipts remain recorded without triggering production. Credit notes or write-offs do not count as real funding. Refunds and reversals retain job history and recalculate funding.
3. A payment records money moving. An allocation records how that money settles a charge. A bill records an amount owed. Their values cannot be counted twice.
4. Allocations cannot exceed the remaining usable payment or eligible invoice/bill balance. Currency conversions preserve both amounts and the exchange-rate snapshot.
5. Every linked financial record belongs to the permitted legal entity. Enforce composite entity foreign keys where needed. A market filter is not an access-control boundary.
6. Every posted journal balances. Posted financial history is corrected through linked reversals or adjustment records, not deletion or overwriting.
7. Invoice parties, amounts, taxes and specifications are frozen on issue. Job items retain their originating invoice lines. Production and shipment items must belong to the same job as their parent run/shipment.
8. Profit allocation and cash withdrawal are separate actions. Both can affect a partner account, but only the withdrawal moves business cash.
9. Invoice and bill payment status is derived from posted allocations and corrections. Account balances and reported profit come from financial postings.
10. Store money as integer minor units and use exact decimal calculations for rates and intermediate prices. Do not add different currencies without an explicit reporting conversion.

## Full logical field register

This is a planning contract, not executable SQL. Ordinary tables have `id uuid PK`, `created_at timestamptz` and an appropriate actor reference such as `created_by → app_users`. The currency lookup uses `code` as its primary key. Mutable draft/configuration records also have `updated_at` and an optimistic-lock `version` where applicable. These shared columns are omitted below. `ID` means UUID foreign key, `?` means nullable, `ccy` references `currencies.code`, `money` means bigint minor units, `rate` means numeric(24,12), `qty` means numeric(18,3), `json` means jsonb, and `time` means timestamptz. Physical quantities are whole units initially.

Common ownership columns required for composite entity scoping must be added to the relevant child tables during implementation. A diagram line does not replace the aggregate checks, constraints and transactional locks described in the source model.

### Business, markets and access

| Table | Stage | Important columns | Rules |
| --- | --- | --- | --- |
| `currencies` | F | `code char(3) PK`, `minor_unit_scale int`, `name text` | ISO codes; scale validated; no surrogate UUID required |
| `legal_entities` | F | `legal_name text`, `registered_address json`, `registration_ref text?`, `tax_registration_refs json`, `functional_currency ccy`, `timezone text`, `fiscal_year_start_month int`, `active bool` | One set of books per entity; functional-currency change is a migration/policy event, not an ordinary settings edit |
| `markets` | F | `code text UQ`, `name text`, `default_currency ccy` | Seed US, UK, AU; no Global row |
| `entity_markets` | F | `legal_entity_id ID → legal_entities`, `market_id ID → markets`, `active bool` | UQ entity + market; invoice combination must exist |
| `app_users` | F | `auth_subject text UQ`, `display_name text`, `email text`, `active bool` | Provider-managed identities map here; no app-owned password table |
| `entity_memberships` | F | `user_id ID → app_users`, `legal_entity_id ID → legal_entities`, `role text`, `active bool` | UQ user + entity; start with explicit access for Momin and Zohaib |

### Bookkeeping configuration

| Table | Stage | Important columns | Rules |
| --- | --- | --- | --- |
| `document_sequences` | F | `legal_entity_id ID`, `document_type text`, `fiscal_year int`, `prefix text`, `next_value bigint` | UQ entity + type + fiscal year; allocate under lock, never `MAX(number)+1` |
| `accounting_periods` | F | `legal_entity_id ID`, `start_date date`, `end_date date`, `status text`, `closed_at time?`, `closed_by ID?` | Date intervals do not overlap per entity; reject postings to a closed period |
| `accounting_policies` | F | `legal_entity_id ID`, `version int`, `effective_from date`, `effective_to date?`, `invoice_posting_mode text`, `revenue_basis text`, `tax_timing_rules json`, `cost_rules json`, `approved_by ID?`, `approved_at time?` | Policies are versioned and referenced by posted documents; no silent retrospective change |
| `gl_accounts` | F | `legal_entity_id ID`, `code text`, `name text`, `account_type text`, `subtype text`, `parent_id ID → gl_accounts?`, `is_control bool`, `active bool` | UQ entity + code; types asset, liability, equity, income, expense; control accounts restrict manual postings |
| `tax_codes` | F | `legal_entity_id ID`, `code text`, `name text`, `jurisdiction text?`, `rate_percent numeric(9,6)`, `direction text`, `tax_gl_account_id ID`, `recoverable_percent numeric(9,6)`, `effective_from date`, `effective_to date?` | Effective-dated; no hardcoded country tax percentage; distinguish zero-rated, exempt and out-of-scope where applicable |

### Customers and invoicing

| Table | Stage | Important columns | Rules |
| --- | --- | --- | --- |
| `customers` | I | `legal_entity_id ID`, `display_name text`, `company_name text?`, `email text?`, `phone text?`, `billing_address json`, `default_delivery_address json?`, `tax_ref text?`, `default_currency ccy?`, `active bool`, `notes text?` | Scope to entity in v1; customer identity can later be shared through a separate group record |
| `invoices` | I | `legal_entity_id ID`, `market_id ID`, `customer_id ID`, `number text?`, `document_kind text`, `status text`, `issue_date date?`, `due_date date?`, `currency ccy`, `seller_snapshot json`, `customer_snapshot json`, `billing_snapshot json`, `delivery_snapshot json`, `terms_snapshot json`, `subtotal_minor money`, `discount_minor money`, `tax_minor money`, `total_minor money`, `policy_id ID`, `issue_journal_entry_id ID?`, `void_journal_entry_id ID?`, `issued_at time?`, `last_sent_at time?` | Kind invoice/proforma; status draft/issued/voided; UQ entity + final number; frozen financial content after issue |
| `invoice_lines` | I | `invoice_id ID`, `position int`, `kind text`, `description text`, `quantity qty`, `unit_price_major numeric(24,8)`, `discount_minor money`, `net_minor money`, `tax_code_id ID?`, `tax_snapshot json`, `tax_minor money`, `gross_minor money`, `fulfilment_required bool`, `specification_snapshot json` | UQ invoice + position; positive physical quantities; service/charge lines do not automatically create manufactured job items |
| `payment_links` | I | `invoice_id ID`, `provider text`, `external_reference text?`, `url text`, `currency ccy`, `amount_minor money`, `status text`, `created_for_version int`, `expires_at time?` | Several links can exist over time; HTTPS; audit replacements; status does not establish invoice funding |
| `customer_credit_notes` | I | `legal_entity_id ID`, `invoice_id ID`, `number text?`, `status text`, `credit_date date`, `currency ccy`, `reason text`, `net_minor money`, `tax_minor money`, `total_minor money`, `journal_entry_id ID?`, `approved_by ID?` | Positive credit amounts; same customer/entity/currency as invoice; freeze after issue; aggregate credit cannot exceed eligible original charges |
| `customer_credit_lines` | I | `credit_note_id ID`, `invoice_line_id ID`, `quantity qty?`, `net_minor money`, `tax_minor money`, `gross_minor money`, `tax_snapshot json` | Line-level references keep tax and quantity corrections attributable |

### Payments and cash controls

| Table | Stage | Important columns | Rules |
| --- | --- | --- | --- |
| `money_accounts` | F | `legal_entity_id ID`, `name text`, `kind text`, `currency ccy`, `gl_account_id ID → gl_accounts`, `provider text?`, `masked_reference text?`, `active bool` | UQ entity + GL account; kind bank, processor or cash; each account has one currency and one owner |
| `payments` | I | `legal_entity_id ID`, `money_account_id ID`, `direction text`, `kind text`, `status text`, `amount_minor money`, `currency ccy`, `received_or_paid_at time`, `accounting_date date`, `method text`, `external_provider text?`, `external_id text?`, `reference text?`, `customer_id ID?`, `vendor_id ID?`, `partner_account_id ID?`, `transfer_id ID?`, `original_payment_id ID?`, `reversal_of_id ID?`, `journal_entry_id ID?`, `cash_journal_line_id ID?`, `evidence_file_id ID?`, `verified_by ID?` | Amount > 0; currency equals money account currency; posted record requires journal/cash line; UQ money account + provider + external_id when present; UQ reversal_of_id |
| `customer_payment_allocations` | I | `legal_entity_id ID`, `payment_id ID`, `invoice_id ID`, `payment_amount_minor money`, `document_amount_minor money`, `effect int`, `purpose text`, `original_allocation_id ID?`, `journal_entry_id ID`, `allocated_at time` | Positive amount columns; effect +1 settlement / -1 reduction; explicit FX when payment and invoice currencies differ; append-only |
| `vendor_payment_allocations` | O | `legal_entity_id ID`, `payment_id ID`, `vendor_bill_id ID`, `payment_amount_minor money`, `document_amount_minor money`, `effect int`, `purpose text`, `original_allocation_id ID?`, `journal_entry_id ID`, `allocated_at time` | Same allocation/FX rules as customer side; no cross-vendor settlement; cap against both available funds and payable/credit |
| `account_transfers` | B | `legal_entity_id ID`, `from_account_id ID`, `to_account_id ID`, `from_amount_minor money`, `to_amount_minor money`, `effective_at time`, `fx_rate rate?`, `journal_entry_id ID`, `reference text?` | Different accounts; same entity; two linked payment legs; preserve both currencies and amounts; fees are separate |
| `payment_disputes` | B | `payment_id ID`, `provider_case_id text?`, `status text`, `opened_at time`, `resolved_at time?`, `amount_minor money`, `currency ccy`, `outcome text?`, `settlement_payment_id ID?` | A dispute flag alone is not a cash movement; actual debits/recoveries have payment/journal records |
| `reconciliation_sessions` | B | `money_account_id ID`, `from_date date`, `to_date date`, `statement_opening_minor money`, `statement_closing_minor money`, `status text`, `statement_file_id ID?`, `closed_by ID?`, `closed_at time?` | Currency from account; explain differences and reconcile opening before close |
| `reconciliation_items` | B | `session_id ID`, `cash_journal_line_id ID`, `matched_amount_minor money`, `note text?` | Same account; each cash line cleared once in basic manual reconciliation; partial/grouped statement matching is later work |

### Jobs, production and shipping

| Table | Stage | Important columns | Rules |
| --- | --- | --- | --- |
| `jobs` | I | `legal_entity_id ID`, `market_id ID`, `customer_id ID`, `origin_invoice_id ID`, `number text`, `lifecycle text`, `opened_at time`, `requested_delivery_date date?`, `delivery_snapshot json`, `artwork_status text`, `approved_artwork_file_id ID?`, `artwork_approved_by ID?`, `artwork_approved_at time?`, `hold_reason text?`, `financial_close_status text` | UQ origin_invoice_id; UQ entity + number; originating invoice matches customer/entity/market; financial close separate from fulfilment |
| `job_items` | I | `job_id ID`, `origin_invoice_line_id ID`, `description text`, `ordered_quantity int`, `unit text`, `specification_snapshot json`, `artwork_file_id ID?` | Quantity > 0; immutable original agreement, audited amendments; UQ job + origin line in v1 |
| `production_runs` | O | `job_id ID`, `vendor_id ID`, `status text`, `planned_start date?`, `expected_finish date?`, `started_at time?`, `completed_at time?`, `released_specification json`, `released_artwork_file_id ID?`, `notes text?` | Many runs per job; require funding/artwork checks before release; only approved vendor roles |
| `production_run_items` | O | `production_run_id ID`, `job_item_id ID`, `planned_quantity int`, `completed_quantity int`, `rejected_quantity int` | UQ run + job item; all items belong to run's job; aggregate overproduction/rework requires documented treatment |
| `shipments` | O | `job_id ID`, `vendor_id ID?`, `carrier_name text?`, `status text`, `tracking_reference text?`, `tracking_url text?`, `destination_snapshot json`, `planned_dispatch date?`, `dispatched_at time?`, `delivered_at time?`, `exception_reason text?`, `replacement_for_shipment_id ID?` | Many shipments per job; delivered timestamp follows dispatch unless an explained import/correction |
| `shipment_items` | O | `shipment_id ID`, `job_item_id ID`, `shipped_quantity int`, `delivered_quantity int`, `returned_quantity int` | Same job; quantities non-negative; no duplicate delivery counting; replacements do not count as additional ordered quantity |

### Vendors, bills and expected costs

| Table | Stage | Important columns | Rules |
| --- | --- | --- | --- |
| `vendors` | O | `legal_entity_id ID`, `name text`, `contact_name text?`, `email text?`, `phone text?`, `address json?`, `default_currency ccy?`, `payment_terms_days int?`, `tax_ref text?`, `active bool` | One vendor may serve several markets; current banking instructions must be protected and changes audited |
| `vendor_roles` | O | `vendor_id ID → vendors`, `role text` | Role is production, shipping or other; UQ vendor + role |
| `vendor_bills` | O | `legal_entity_id ID`, `vendor_id ID`, `document_kind text`, `original_bill_id ID?`, `supplier_reference text`, `purpose text`, `status text`, `bill_date date`, `due_date date`, `currency ccy`, `vendor_snapshot json`, `net_minor money`, `tax_minor money`, `total_minor money`, `policy_id ID`, `journal_entry_id ID?`, `evidence_file_id ID?`, `recurring_template_id ID?`, `recurrence_period date?` | Kind bill/credit_note; amounts positive, kind supplies financial sign; same-entity/vendor/currency original for initial credits; UQ vendor + kind + normalised supplier reference |
| `vendor_bill_lines` | O | `vendor_bill_id ID`, `position int`, `description text`, `quantity qty`, `unit_price_major numeric(24,8)`, `net_minor money`, `tax_minor money`, `gross_minor money`, `tax_snapshot json`, `recognition_treatment text`, `debit_gl_account_id ID`, `market_id ID?`, `job_id ID?`, `production_run_id ID?`, `shipment_id ID?`, `estimate_id ID?`, `clears_adjustment_id ID?` | Explicit job/expense allocation; referenced run/shipment/estimate must belong to job; account and tax treatment validated by policy |
| `job_cost_estimates` | O | `job_id ID`, `series_key text`, `category text`, `vendor_id ID?`, `production_run_id ID?`, `shipment_id ID?`, `currency ccy`, `estimated_minor money`, `fx_rate rate`, `estimate_version int`, `is_current bool`, `basis text`, `valid_at time` | Production, shipping or other direct cost; UQ job + series + version and one current version per job/series; no journal posting |

### Ledger and recognition

| Table | Stage | Important columns | Rules |
| --- | --- | --- | --- |
| `journal_entries` | F | `legal_entity_id ID`, `accounting_date date`, `status text`, `posting_key text`, `description text`, `policy_id ID → accounting_policies?`, `reversal_of_id ID → journal_entries?`, `posted_at time?`, `posted_by ID?` | UQ entity + posting key; UQ reversal_of_id for whole-entry corrections; same-entity reversal |
| `journal_lines` | F | `legal_entity_id ID`, `journal_entry_id ID`, `gl_account_id ID`, `debit_base_minor money`, `credit_base_minor money`, `transaction_currency ccy`, `transaction_amount_minor money`, `fx_rate rate`, `fx_rate_date date`, `fx_source text`, `market_id ID?`, `customer_id ID?`, `vendor_id ID?`, `job_id ID?`, `partner_account_id ID?`, `memo text?` | Base amounts use entity functional currency; positive native amount means debit, negative means credit; dimensions must belong to entity |
| `accounting_adjustments` | B | `legal_entity_id ID`, `kind text`, `effective_date date`, `reason text`, `evidence_file_id ID?`, `journal_entry_id ID`, `reversal_due_date date?`, `reversal_journal_entry_id ID?`, `approved_by ID`, `related_bill_id ID?` | Supports documented opening balances, accruals, FX revaluation and close adjustments; no undocumented balance overrides |
| `recognition_events` | B | `job_id ID`, `event_type text`, `effective_date date`, `policy_id ID`, `reason text`, `journal_entry_id ID`, `approved_by ID` | Revenue recognition, cost release or subsequent adjustment; journal is authoritative for reported value |
| `recognition_lines` | B | `recognition_event_id ID`, `job_item_id ID`, `recognised_quantity qty`, `revenue_document_minor money`, `document_currency ccy`, `revenue_base_minor money`, `cost_base_minor money` | Limits against remaining unrecognised quantities/value; permit later cost true-up with zero new revenue quantity |

### Partner accounts

| Table | Stage | Important columns | Rules |
| --- | --- | --- | --- |
| `partners` | B | `display_name text`, `contact_email text?`, `app_user_id ID?`, `active bool` | Partner identity independent of login and legal-entity membership |
| `partner_accounts` | B | `legal_entity_id ID`, `partner_id ID`, `bucket text`, `gl_account_id ID`, `currency ccy` | Capital, loan or current/distribution; use functional currency initially; UQ entity + partner + bucket + currency |
| `partner_events` | B | `partner_account_id ID`, `kind text`, `effective_date date`, `amount_minor money`, `payment_id ID?`, `vendor_bill_id ID?`, `distribution_line_id ID?`, `journal_entry_id ID`, `reason text`, `approved_by ID` | Cash events share the payment's journal; non-cash events have their own balanced journal; never post both twice |
| `profit_distributions` | B | `legal_entity_id ID`, `period_id ID`, `status text`, `earnings_basis_minor money`, `retained_minor money`, `distributed_minor money`, `basis_snapshot json`, `journal_entry_id ID?`, `approved_by ID?`, `approved_at time?`, `policy_version text?` | Closed-period remaining basis after prior allocations and applicable losses/reserves; retained + distributed matches that approved basis; no automatic cash transfer |
| `profit_distribution_lines` | B | `distribution_id ID`, `partner_account_id ID`, `amount_minor money`, `share_snapshot numeric(9,6)?` | UQ distribution + partner account; line sum equals distribution total |

### Documents and reliability

| Table | Stage | Important columns | Rules |
| --- | --- | --- | --- |
| `files` | F | `legal_entity_id ID`, `storage_provider text`, `bucket text`, `object_key text`, `original_name text`, `mime_type text`, `byte_size bigint`, `sha256 text`, `status text`, `uploaded_by ID`, `uploaded_at time` | UQ provider + bucket + key; metadata only; private storage; allowed type/size limits |
| `generated_documents` | I | `legal_entity_id ID`, `kind text`, `invoice_id ID?`, `credit_note_id ID?`, `payment_id ID?`, `document_number text`, `version int`, `snapshot json`, `template_version text`, `file_id ID?`, `status text`, `generated_at time?` | Exactly one subject FK; invoice/credit/receipt kinds; UQ subject + document kind + version; receipt number uniquely scoped to entity |
| `document_deliveries` | I | `document_id ID`, `channel text`, `recipient_snapshot text`, `status text`, `provider_message_id text?`, `sent_at time?`, `error_summary text?`, `recorded_by ID?` | Manual sending is explicitly marked manual; no assumption that a provider has delivered it |
| `job_file_links` | O | `job_id ID`, `file_id ID`, `purpose text`, `version_label text?` | UQ job + file + purpose; drawing/artwork/proof/POD roles; issued approvals point to exact file |
| `audit_events` | F | `legal_entity_id ID`, `actor_id ID?`, `action text`, `subject_type text`, `subject_id uuid`, `before json?`, `after json?`, `reason text?`, `occurred_at time`, `request_id text` | Append-only; typed subjects resolved by app for display; never use audit metadata as a financial foreign-key relationship |
| `idempotency_requests` | F | `legal_entity_id ID`, `command text`, `key text`, `request_hash text`, `status text`, `result_references json`, `created_at time`, `completed_at time?` | UQ entity + command + key; claim and result commit with business write; mismatch hash rejects |
| `outbox_tasks` | F | `legal_entity_id ID`, `task_type text`, `dedupe_key text`, `payload json`, `status text`, `attempts int`, `available_at time`, `lease_until time?`, `last_error text?`, `completed_at time?` | UQ task_type + dedupe_key; task payload contains IDs, not unnecessary customer data |

### Later automation

| Table | Stage | Important columns | Rules |
| --- | --- | --- | --- |
| `recurring_expense_templates` | L | `legal_entity_id ID`, `vendor_id ID`, `name text`, `currency ccy`, `schedule_rule json`, `next_due_date date`, `end_date date?`, `line_defaults json`, `active bool` | Produces draft bills only; UQ template + recurrence_period on generated bills |
| `profit_policies` | L | `legal_entity_id ID`, `effective_from date`, `effective_to date?`, `version int`, `basis text` | Future automated allocation only; effective periods must not overlap |
| `profit_policy_shares` | L | `policy_id ID`, `partner_id ID`, `share_percent numeric(9,6)` | Shares total 100% of the declared distributable pool; retention is a separate decision |
| `webhook_events` | L | `provider text`, `provider_account_ref text`, `external_event_id text`, `legal_entity_id ID`, `payload_hash text`, `received_at time`, `processing_status text`, `processed_at time?`, `error text?` | UQ provider + provider_account + event_id; verify authenticity before applying business effects |

## Explicit foreign-key register

This register covers every explicitly declared ID/ccy relationship in the field register, including optional links and self-references omitted from the diagrams. All references point to `id` unless `code` is shown. “Nullable” describes the logical column; posting rules can require a value later. The standard actor fields and extra composite entity-scoping columns follow the shared rules above. The audit `subject_id`, auth subject, external provider references, task payloads and other JSON metadata are not financial foreign keys.

| Referencing column | References | Nullable |
| --- | --- | --- |
| `account_transfers.from_account_id` | `money_accounts.id` | No |
| `account_transfers.journal_entry_id` | `journal_entries.id` | No |
| `account_transfers.legal_entity_id` | `legal_entities.id` | No |
| `account_transfers.to_account_id` | `money_accounts.id` | No |
| `accounting_adjustments.approved_by` | `app_users.id` | No |
| `accounting_adjustments.evidence_file_id` | `files.id` | Yes |
| `accounting_adjustments.journal_entry_id` | `journal_entries.id` | No |
| `accounting_adjustments.legal_entity_id` | `legal_entities.id` | No |
| `accounting_adjustments.related_bill_id` | `vendor_bills.id` | Yes |
| `accounting_adjustments.reversal_journal_entry_id` | `journal_entries.id` | Yes |
| `accounting_periods.closed_by` | `app_users.id` | Yes |
| `accounting_periods.legal_entity_id` | `legal_entities.id` | No |
| `accounting_policies.approved_by` | `app_users.id` | Yes |
| `accounting_policies.legal_entity_id` | `legal_entities.id` | No |
| `audit_events.actor_id` | `app_users.id` | Yes |
| `audit_events.legal_entity_id` | `legal_entities.id` | No |
| `customer_credit_lines.credit_note_id` | `customer_credit_notes.id` | No |
| `customer_credit_lines.invoice_line_id` | `invoice_lines.id` | No |
| `customer_credit_notes.approved_by` | `app_users.id` | Yes |
| `customer_credit_notes.currency` | `currencies.code` | No |
| `customer_credit_notes.invoice_id` | `invoices.id` | No |
| `customer_credit_notes.journal_entry_id` | `journal_entries.id` | Yes |
| `customer_credit_notes.legal_entity_id` | `legal_entities.id` | No |
| `customer_payment_allocations.invoice_id` | `invoices.id` | No |
| `customer_payment_allocations.journal_entry_id` | `journal_entries.id` | No |
| `customer_payment_allocations.legal_entity_id` | `legal_entities.id` | No |
| `customer_payment_allocations.original_allocation_id` | `customer_payment_allocations.id` | Yes |
| `customer_payment_allocations.payment_id` | `payments.id` | No |
| `customers.default_currency` | `currencies.code` | Yes |
| `customers.legal_entity_id` | `legal_entities.id` | No |
| `document_deliveries.document_id` | `generated_documents.id` | No |
| `document_deliveries.recorded_by` | `app_users.id` | Yes |
| `document_sequences.legal_entity_id` | `legal_entities.id` | No |
| `entity_markets.legal_entity_id` | `legal_entities.id` | No |
| `entity_markets.market_id` | `markets.id` | No |
| `entity_memberships.legal_entity_id` | `legal_entities.id` | No |
| `entity_memberships.user_id` | `app_users.id` | No |
| `files.legal_entity_id` | `legal_entities.id` | No |
| `files.uploaded_by` | `app_users.id` | No |
| `generated_documents.credit_note_id` | `customer_credit_notes.id` | Yes |
| `generated_documents.file_id` | `files.id` | Yes |
| `generated_documents.invoice_id` | `invoices.id` | Yes |
| `generated_documents.legal_entity_id` | `legal_entities.id` | No |
| `generated_documents.payment_id` | `payments.id` | Yes |
| `gl_accounts.legal_entity_id` | `legal_entities.id` | No |
| `gl_accounts.parent_id` | `gl_accounts.id` | Yes |
| `idempotency_requests.legal_entity_id` | `legal_entities.id` | No |
| `invoice_lines.invoice_id` | `invoices.id` | No |
| `invoice_lines.tax_code_id` | `tax_codes.id` | Yes |
| `invoices.currency` | `currencies.code` | No |
| `invoices.customer_id` | `customers.id` | No |
| `invoices.issue_journal_entry_id` | `journal_entries.id` | Yes |
| `invoices.legal_entity_id` | `legal_entities.id` | No |
| `invoices.market_id` | `markets.id` | No |
| `invoices.policy_id` | `accounting_policies.id` | No |
| `invoices.void_journal_entry_id` | `journal_entries.id` | Yes |
| `job_cost_estimates.currency` | `currencies.code` | No |
| `job_cost_estimates.job_id` | `jobs.id` | No |
| `job_cost_estimates.production_run_id` | `production_runs.id` | Yes |
| `job_cost_estimates.shipment_id` | `shipments.id` | Yes |
| `job_cost_estimates.vendor_id` | `vendors.id` | Yes |
| `job_file_links.file_id` | `files.id` | No |
| `job_file_links.job_id` | `jobs.id` | No |
| `job_items.artwork_file_id` | `files.id` | Yes |
| `job_items.job_id` | `jobs.id` | No |
| `job_items.origin_invoice_line_id` | `invoice_lines.id` | No |
| `jobs.approved_artwork_file_id` | `files.id` | Yes |
| `jobs.artwork_approved_by` | `app_users.id` | Yes |
| `jobs.customer_id` | `customers.id` | No |
| `jobs.legal_entity_id` | `legal_entities.id` | No |
| `jobs.market_id` | `markets.id` | No |
| `jobs.origin_invoice_id` | `invoices.id` | No |
| `journal_entries.legal_entity_id` | `legal_entities.id` | No |
| `journal_entries.policy_id` | `accounting_policies.id` | Yes |
| `journal_entries.posted_by` | `app_users.id` | Yes |
| `journal_entries.reversal_of_id` | `journal_entries.id` | Yes |
| `journal_lines.customer_id` | `customers.id` | Yes |
| `journal_lines.gl_account_id` | `gl_accounts.id` | No |
| `journal_lines.job_id` | `jobs.id` | Yes |
| `journal_lines.journal_entry_id` | `journal_entries.id` | No |
| `journal_lines.legal_entity_id` | `legal_entities.id` | No |
| `journal_lines.market_id` | `markets.id` | Yes |
| `journal_lines.partner_account_id` | `partner_accounts.id` | Yes |
| `journal_lines.transaction_currency` | `currencies.code` | No |
| `journal_lines.vendor_id` | `vendors.id` | Yes |
| `legal_entities.functional_currency` | `currencies.code` | No |
| `markets.default_currency` | `currencies.code` | No |
| `money_accounts.currency` | `currencies.code` | No |
| `money_accounts.gl_account_id` | `gl_accounts.id` | No |
| `money_accounts.legal_entity_id` | `legal_entities.id` | No |
| `outbox_tasks.legal_entity_id` | `legal_entities.id` | No |
| `partner_accounts.currency` | `currencies.code` | No |
| `partner_accounts.gl_account_id` | `gl_accounts.id` | No |
| `partner_accounts.legal_entity_id` | `legal_entities.id` | No |
| `partner_accounts.partner_id` | `partners.id` | No |
| `partner_events.approved_by` | `app_users.id` | No |
| `partner_events.distribution_line_id` | `profit_distribution_lines.id` | Yes |
| `partner_events.journal_entry_id` | `journal_entries.id` | No |
| `partner_events.partner_account_id` | `partner_accounts.id` | No |
| `partner_events.payment_id` | `payments.id` | Yes |
| `partner_events.vendor_bill_id` | `vendor_bills.id` | Yes |
| `partners.app_user_id` | `app_users.id` | Yes |
| `payment_disputes.currency` | `currencies.code` | No |
| `payment_disputes.payment_id` | `payments.id` | No |
| `payment_disputes.settlement_payment_id` | `payments.id` | Yes |
| `payment_links.currency` | `currencies.code` | No |
| `payment_links.invoice_id` | `invoices.id` | No |
| `payments.cash_journal_line_id` | `journal_lines.id` | Yes |
| `payments.currency` | `currencies.code` | No |
| `payments.customer_id` | `customers.id` | Yes |
| `payments.evidence_file_id` | `files.id` | Yes |
| `payments.journal_entry_id` | `journal_entries.id` | Yes |
| `payments.legal_entity_id` | `legal_entities.id` | No |
| `payments.money_account_id` | `money_accounts.id` | No |
| `payments.original_payment_id` | `payments.id` | Yes |
| `payments.partner_account_id` | `partner_accounts.id` | Yes |
| `payments.reversal_of_id` | `payments.id` | Yes |
| `payments.transfer_id` | `account_transfers.id` | Yes |
| `payments.vendor_id` | `vendors.id` | Yes |
| `payments.verified_by` | `app_users.id` | Yes |
| `production_run_items.job_item_id` | `job_items.id` | No |
| `production_run_items.production_run_id` | `production_runs.id` | No |
| `production_runs.job_id` | `jobs.id` | No |
| `production_runs.released_artwork_file_id` | `files.id` | Yes |
| `production_runs.vendor_id` | `vendors.id` | No |
| `profit_distribution_lines.distribution_id` | `profit_distributions.id` | No |
| `profit_distribution_lines.partner_account_id` | `partner_accounts.id` | No |
| `profit_distributions.approved_by` | `app_users.id` | Yes |
| `profit_distributions.journal_entry_id` | `journal_entries.id` | Yes |
| `profit_distributions.legal_entity_id` | `legal_entities.id` | No |
| `profit_distributions.period_id` | `accounting_periods.id` | No |
| `profit_policies.legal_entity_id` | `legal_entities.id` | No |
| `profit_policy_shares.partner_id` | `partners.id` | No |
| `profit_policy_shares.policy_id` | `profit_policies.id` | No |
| `recognition_events.approved_by` | `app_users.id` | No |
| `recognition_events.job_id` | `jobs.id` | No |
| `recognition_events.journal_entry_id` | `journal_entries.id` | No |
| `recognition_events.policy_id` | `accounting_policies.id` | No |
| `recognition_lines.document_currency` | `currencies.code` | No |
| `recognition_lines.job_item_id` | `job_items.id` | No |
| `recognition_lines.recognition_event_id` | `recognition_events.id` | No |
| `reconciliation_items.cash_journal_line_id` | `journal_lines.id` | No |
| `reconciliation_items.session_id` | `reconciliation_sessions.id` | No |
| `reconciliation_sessions.closed_by` | `app_users.id` | Yes |
| `reconciliation_sessions.money_account_id` | `money_accounts.id` | No |
| `reconciliation_sessions.statement_file_id` | `files.id` | Yes |
| `recurring_expense_templates.currency` | `currencies.code` | No |
| `recurring_expense_templates.legal_entity_id` | `legal_entities.id` | No |
| `recurring_expense_templates.vendor_id` | `vendors.id` | No |
| `shipment_items.job_item_id` | `job_items.id` | No |
| `shipment_items.shipment_id` | `shipments.id` | No |
| `shipments.job_id` | `jobs.id` | No |
| `shipments.replacement_for_shipment_id` | `shipments.id` | Yes |
| `shipments.vendor_id` | `vendors.id` | Yes |
| `tax_codes.legal_entity_id` | `legal_entities.id` | No |
| `tax_codes.tax_gl_account_id` | `gl_accounts.id` | No |
| `vendor_bill_lines.clears_adjustment_id` | `accounting_adjustments.id` | Yes |
| `vendor_bill_lines.debit_gl_account_id` | `gl_accounts.id` | No |
| `vendor_bill_lines.estimate_id` | `job_cost_estimates.id` | Yes |
| `vendor_bill_lines.job_id` | `jobs.id` | Yes |
| `vendor_bill_lines.market_id` | `markets.id` | Yes |
| `vendor_bill_lines.production_run_id` | `production_runs.id` | Yes |
| `vendor_bill_lines.shipment_id` | `shipments.id` | Yes |
| `vendor_bill_lines.vendor_bill_id` | `vendor_bills.id` | No |
| `vendor_bills.currency` | `currencies.code` | No |
| `vendor_bills.evidence_file_id` | `files.id` | Yes |
| `vendor_bills.journal_entry_id` | `journal_entries.id` | Yes |
| `vendor_bills.legal_entity_id` | `legal_entities.id` | No |
| `vendor_bills.original_bill_id` | `vendor_bills.id` | Yes |
| `vendor_bills.policy_id` | `accounting_policies.id` | No |
| `vendor_bills.recurring_template_id` | `recurring_expense_templates.id` | Yes |
| `vendor_bills.vendor_id` | `vendors.id` | No |
| `vendor_payment_allocations.journal_entry_id` | `journal_entries.id` | No |
| `vendor_payment_allocations.legal_entity_id` | `legal_entities.id` | No |
| `vendor_payment_allocations.original_allocation_id` | `vendor_payment_allocations.id` | Yes |
| `vendor_payment_allocations.payment_id` | `payments.id` | No |
| `vendor_payment_allocations.vendor_bill_id` | `vendor_bills.id` | No |
| `vendor_roles.vendor_id` | `vendors.id` | No |
| `vendors.default_currency` | `currencies.code` | Yes |
| `vendors.legal_entity_id` | `legal_entities.id` | No |
| `webhook_events.legal_entity_id` | `legal_entities.id` | No |

## Validation and scope

All 57 tables in the source model appear in the diagrams. Every drawn relationship is checked against an actual declared foreign key. The canonical structured JSON passes the Mermaid plugin schema and static lint. This is static source validation, not a live database test or a claim that every host renders identically. No local Mermaid rendering engine was used. Additional ideas mentioned in the roadmap, such as quotes or inventory, have no fully specified schema in this release and are outside the 57-table count.
