# Corpus duplicates — data-cleanup backlog

**Status:** open · **Owner:** CPMT (source-data decision, not a code fix)
**Found:** RAG quality eval (16 queries) after the full 123-doc CPMT ingest.

## The issue

The source folder `~/Downloads/CPMT` contains **versioned and near-identical PDFs**. They
were all ingested as `ACTIVE`, so when a query matches a document that has several versions,
the **sources panel shows 2–3 copies of the same document** (e.g. `Comprehensive Deposit
Policy`, `… - V2.2`, and `… - V3.1` all surface together).

Answers are still **correct** — the model synthesizes across the duplicates — but the
sources list looks noisy and a staff member can't tell which version is authoritative.

This is **source-data hygiene**, not a retrieval/pipeline bug. The fix is to decide which
version is current and de-duplicate the corpus.

## Scope

`84 docs · 15 duplicate clusters covering 31 files` (as of the post-ingest eval).

### Type A — true version pairs (keep latest ACTIVE, mark older OBSOLETE)

These are real revisions. Recommended: keep the highest version `ACTIVE`; retag the older
one(s) `OBSOLETE` from the CPMT Hub so the ⚠️ obsolete caveat flow handles them correctly
(rather than deleting — keeps history and tests the Active→Obsolete→Flag routing).

| Cluster (dept / category) | Versions present | Keep ACTIVE |
|---|---|---|
| Comprehensive Deposit Policy (BB / Policy) | *(unversioned)*, V2.2, V3.1 | V3.1 |
| Resident & Non-Resident Cross Border Remittance SOP (BB / Process) | V3.0, V3.1 | V3.1 |
| Product Program – Privilege Savings Account (BB / Product) | V2.0, V3.0 | V3.0 |
| Variable Pay Calculation SOP (BB / Process) | V1.0, V2.0 | V2.0 |
| Product Program – Platina Deposit (BB / Product) | V1.0, V2.0 | V2.0 |
| Product Program – Garima Savings Account (BB / Product) | V1.0, V2.0 | V2.0 |

> Note: there are additional Platina files in the corpus (`…Platina Deposits_V4`) and the
> scanned `…- V3.0` is on the OCR-deferred list — reconcile the true latest version when
> cleaning this cluster.

### Type B — exact-name duplicates (de-dupe; keep one)

Same title ingested twice (duplicate files across source subfolders, or a `- Duplicate`
suffix). Recommended: keep one `ACTIVE`, delete the other from the Hub.

- Dealer Onboarding & Payout Process Note (VF / Process)
- Staff Vehicle Loan Process Note (VF / Process)
- Product Program – Used Car Loan (VF / Product)
- Product Program – Small Commercial Vehicle Loan (VF / Product)
- Product Program – Staff Vehicle Loan (VF / Product)
- Process Flow for Availing Staff Vehicle Loan – Balance Transfer from Other Financier Manual (VF / Process)
- Product Program Non Residents (BB / Product)
- Product Program – Collection Account – Branch Banking (BB / Product)
- Product Program – Safe Deposit Lockers – V1.0 (BB / Product)

> `Product_Program_Classic_SA_v1 - Duplicate` is another explicit duplicate seen in the
> sources during the eval — confirm and remove its redundant copy too.

### Found by the task #13 overlap detector (6 Jul 2026)

Surfaced while verifying the new upload-time overlap detection — both are Type B
(same content; keep one, delete the other from the Hub):

- **Product Program – Platina Deposit – V3.0** (BB / Product) — ingested **twice**
  (identical 225-chunk copies; docIds `6ae86801…` and `c61a98bb…`). The source folder has
  the same file in both `Product/` and `Product/Product Extra Docs/`.
- **NB36_Comprehensive_Deposit_Policy – V3.1** (BB / Policy, in `Policy Extra Docs/`) —
  same content as **Comprehensive Deposit Policy – V3.1** under a different filename
  (~82–86% chunk coverage both directions). Adds a fourth member to the Deposit Policy
  cluster in Type A above.

## How to clean it up

Two paths, no code change required:

1. **Via the CPMT Hub (recommended for a few docs):**
   - *Type A* → open the older version → **retag to Obsolete**.
   - *Type B* → **delete** the redundant copy (removes it from Qdrant + Firestore + Storage).

2. **Re-ingest a cleaned folder (recommended if many):**
   - Remove duplicate/superseded PDFs from `~/Downloads/CPMT` (or mark intended-obsolete ones).
   - Re-run a fresh ingest:
     `cd backend && uv run python -m scripts.ingest_folder "/Users/huleshjangde/Downloads/CPMT" --recreate`
     (⚠️ `--recreate` wipes the current 84 good docs — only do this against a corrected folder.)

## Why we're not auto-deduping in code

De-duplication needs a human "which version is authoritative" call — title/version parsing
is unreliable (mixed naming: `V2.2`, `_V4`, `- Duplicate`, unversioned). Picking the wrong
"latest" would silently drop a current policy, so this stays a CPMT data decision.

**Update (6 Jul 2026, task #13):** *new* uploads are now checked automatically — the CPMT
Hub upload compares content against the ACTIVE corpus and pauses on strong overlap so CPMT
can review the matches (with page-level preview) and retire superseded versions in the same
step. The human stays in the loop; nothing is auto-deleted or auto-retagged. The backlog
above (duplicates already in the corpus) is unaffected and still needs the manual cleanup
described below.

## Related

- `backend/.ingest_deferred.json` — 39 scanned/image-only PDFs awaiting an OCR decision
  (separate backlog item; some overlap with the Platina cluster above).
