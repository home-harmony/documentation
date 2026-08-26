# 0007. Loan Kind as Extensible Dictionary (System + Family-Defined)

* **Status**: Accepted
* **Date**: 2026-08-26
* **Deciders**: Dmitri Astafiev

## Context and Problem Statement

The original `debt_loans.kind` column was constrained by a `VARCHAR CHECK(kind IN ('mortgage','car_loan','personal_loan','credit_card','other'))` clause. This hard-coded set omits many common real-world loan types: `student_loan`, `medical_debt`, `payday_loan`, `heloc`, `business_loan`, and `family_loan`, among others. Each of these has distinct repayment dynamics important to the debt planner (e.g., payday loans carry extreme APRs that the avalanche strategy must prioritise; HELOCs have a revolving draw period). Additionally, different families may use different terminology — a family in Romania might want "Credit Prima Casa" as a loan kind, while a US household might want "PSLF-eligible student loan". A `CHECK` constraint cannot accommodate per-family vocabulary, and extending the enum requires a schema migration for every addition.

## Decision Drivers

* **Extensibility** — new system-wide loan kinds must be addable without a schema migration touching `debt_loans`.
* **Localisation and family autonomy** — families must be able to define their own loan kind labels in their preferred language or terminology.
* **Debt planner accuracy** — the planner's avalanche/snowball logic must be able to identify high-interest categories (e.g., payday loans) by code, not free-text label.
* **Ubiquitous language** — the domain should not force users to map their real-world instruments to an ill-fitting fixed vocabulary.

## Considered Options

1. **Hard-coded `CHECK` constraint enum** — extend the existing `CHECK` list with additional values.
2. **Single global dictionary table** — `debt_loan_kinds` with system-wide entries only (`family_id` always NULL); families cannot add custom kinds.
3. **Dictionary table with system and family rows** — `debt_loan_kinds` with `family_id NULL` for system-wide entries and a specific `family_id` for per-family custom entries.

## Decision Outcome

Chosen option: **Option 3 — Dictionary table with system and family rows**, because it simultaneously satisfies the system taxonomy requirement (debt planner can reference stable `code` values) and the family autonomy requirement (custom rows scoped by `family_id`).

Option 1 is rejected because every new loan type requires a schema migration and a code deployment; it cannot support per-family terminology.

Option 2 is rejected because it does not allow families to create custom labels or localise kind names.

### Schema Change

Replace `debt_loans.kind VARCHAR(30) CHECK(...)` with `debt_loans.loan_kind_id UUID NOT NULL` (FK → `debt_loan_kinds.id`).

Add the dictionary table:

```sql
CREATE TABLE debt_loan_kinds (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  family_id   UUID,              -- NULL = system-wide standard kind
  code        VARCHAR(50)  NOT NULL,
  name        VARCHAR(200) NOT NULL,
  is_system   BOOLEAN      NOT NULL DEFAULT false,
  sort_order  SMALLINT     NOT NULL DEFAULT 0,
  deleted_at  TIMESTAMPTZ
);
```

**System seed rows** (`family_id = NULL`, `is_system = TRUE`) inserted in the initial migration:

| code | name |
|:---|:---|
| `mortgage` | Mortgage |
| `car_loan` | Car Loan |
| `personal_loan` | Personal Loan |
| `credit_card` | Credit Card |
| `student_loan` | Student Loan |
| `medical_debt` | Medical Debt |
| `payday_loan` | Payday Loan |
| `heloc` | Home Equity Line of Credit (HELOC) |
| `business_loan` | Business Loan |
| `family_loan` | Family Loan |
| `other` | Other |

Family-defined kinds have `family_id` set to the creating family's UUID, `is_system = FALSE`, and a `name` provided by the family administrator. They are soft-deleted via `deleted_at`.

### Positive Consequences

* Families can name loan types in their own language or terminology without modifying any system tables.
* System taxonomy is richer and covers the most common loan types the debt planner needs to identify by stable `code` values.
* New system kinds can be added by inserting seed rows in a data migration rather than altering the `CHECK` constraint.
* `deleted_at` supports soft-deleting custom kinds without breaking historical loans that referenced them.

### Negative Consequences / Trade-offs

* A `JOIN` to `debt_loan_kinds` is now required to display the loan kind name in the API response.
* The seed migration must be maintained; if the seed is absent, loan registration fails at the FK constraint.
* Family-kind `code` values are not globally unique, so the debt planner must use system kinds (identified by `is_system = TRUE`) for business logic, not arbitrary family codes.

## Compliance & Invariants

* `debt_loans` **must** reference `loan_kind_id UUID NOT NULL` (FK → `debt_loan_kinds.id`); the old `kind VARCHAR CHECK(...)` column must be dropped in the same migration that creates `debt_loan_kinds`.
* All system seed rows **must** be inserted in the schema migration and must not be hard-coded in application code.
* Family kinds are exposed and managed via the API:
  * `GET /loan-kinds` — returns system kinds merged with the requesting family's custom kinds.
  * `POST /loan-kinds` — creates a custom kind scoped to the requesting family; requires `Admin` or `Owner` role.
* The debt planner **must** identify high-priority categories (e.g., payday loans) by the stable system `code` field, never by the mutable `name` field.
* System kinds (`is_system = TRUE`) **must not** be modifiable or deletable via the API.
