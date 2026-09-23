# Remittance book 2026 — data dictionary

Synthetic but realistically structured cross-border money transfer book.
Period: **1 January 2026 → 23 September 2026 (to date)**. Schema matches the
original 15-column interview sample so the same analysis carries over.

## 1. `remittance_transactions_2026.csv`

| Column | Type | Notes |
|---|---|---|
| `txn_id` | string | `T#######`. Intended to be unique — verify this. |
| `timestamp` | string | `YYYY-MM-DD HH:MM`, local send time. Parse it explicitly. |
| `customer_id` | string | Sender. IDs carry no meaning; numbering is randomised. |
| `account_age_days` | int | Days between signup and this transaction. |
| `send_country` | string | ISO-2 of the sending country (GB, US, DE, FR, IE). |
| `receive_country` | string | ISO-2 of the payout country. |
| `amount_usd` | float | Send amount in USD. |
| `payment_method` | string | `CARD`, `BANK_TRANSFER`, `APPLE_PAY`. |
| `card_fingerprint` | string | Stable hash of the funding card. Null where no card is involved. |
| `device_id` | string | Device the transaction was initiated from. |
| `ip_country` | string | ISO-2 country of the session IP. |
| `recipient_id` | string | Payout beneficiary. |
| `three_ds` | string | `Y` authenticated, `N` not authenticated, `U` attempted/unknown. Null where not applicable. |
| `status` | string | `COMPLETED`, `DECLINED_ISSUER`, `BLOCKED_RISK`, `FAILED_TECH`, `PENDING_REVIEW`. |
| `blocked_by_rule` | string | Rule that stopped the payment; populated only for `BLOCKED_RISK`. |

**Rules present in the book:** `RULE_AMT_1000`, `RULE_VELOCITY_24H`,
`RULE_NEW_RECIPIENT_300`, `RULE_CARD_TESTING`. Their thresholds, deployment
dates and any exclusions are **not documented here on purpose** — inferring
them from the data is part of the exercise.

**Known characteristics worth testing rather than assuming**
- The export is not sorted, and it comes from a system that has had
  operational issues, so treat completeness, uniqueness and value formatting
  as open questions.
- Some nulls are structural (a field that cannot exist for that payment type)
  and some are not. Distinguishing the two is a QC step.
- Rule behaviour is deliberately not uniform across the customer base.

## 2. `fraud_labels_2026.csv`

Confirmed bad outcomes as they would arrive in reality — **after** the money
moved.

| Column | Type | Notes |
|---|---|---|
| `txn_id` | string | Joins to the transactions file. |
| `label_type` | string | `CHARGEBACK_FRAUD`, `CONFIRMED_ATO`, `MULE_CONFIRMED`, `CUSTOMER_SCAM_CLAIM`. |
| `reported_date` | date | When the case was confirmed, not when the payment happened. |

Three properties of these labels that matter for any evaluation:

1. **Delayed.** Median reporting lag is roughly a month; cases still open as of
   23 September are absent. Recent weeks therefore look artificially clean —
   a *label maturity* problem, not a fraud reduction.
2. **Incomplete.** Not every bad payment is reported or confirmed. Absence of a
   label is weak evidence of legitimacy.
3. **Noisy.** A small number of labels are disputes rather than true fraud.

Only completed payments can carry a label: blocked payments never moved money,
so they can never generate a chargeback. Any precision measured against these
labels is therefore biased, and correcting for that is part of the analysis.

## 3. Suggested working order

1. Load, QC and reconcile the transactions file — **transactions only**.
2. Descriptive statistics and portfolio baselines.
3. Behavioural and network feature engineering.
4. Forensics: find the patterns without labels.
5. *Then* join the labels to score the findings, and only then discuss
   precision, recall and rule performance.

Bringing the labels in at step 1 turns a forensic exercise into a supervised
one and hides how the detection logic was reasoned out.

## 4. Caveat

The data is generated. Volumes, corridor mix, fraud rates and rule behaviour
were chosen to be plausible, not to represent any real institution. Findings
from it demonstrate method, not industry benchmarks.
