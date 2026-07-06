# Upload-time document overlap detection (Task #13)

**Date:** 2026-07-06 · **Status:** approved
**Task:** #13 — Add document overlap / consolidation detection. "On upload, compare
to existing docs; highlight overlaps and consolidation candidates."

## Scope

Upload-time check only (CPMT Hub `Upload SoP` flow). No corpus-wide scan; the
existing backlog in `docs/CORPUS-DUPLICATES.md` stays a manual cleanup. The check
is **blocking**: when a strong overlap is found the upload pauses and CPMT decides —
upload anyway, upload and mark older doc(s) Obsolete, or cancel. Detection only;
never auto-delete/auto-retag without an explicit human choice (per the
CORPUS-DUPLICATES reasoning that "which version is authoritative" is a human call).

`/ingest-text` (CPMT clarifications) bypasses the check — those are *supposed* to
embed close to existing content.

## Flow

```
Upload SoP (dialog) ──POST /ingest──▶ parse → chunk → embed
                                          │
                                 overlap check (Qdrant batch query)
                                          │
                          ┌───────────────┴───────────────┐
                    no strong overlap               candidates found
                          │                               │
                    commit ingest                 stash pending (TTL 15 min)
                    (storage+Qdrant+FS)           return 409 + overlap report
                          │                               │
                       200 OK                    dialog shows candidates:
                                                 coverage %, matched sections
                                                 [Cancel] [Upload anyway]
                                                 [☑ mark old Obsolete + Upload]
                                                          │
                                          POST /ingest/confirm/{pendingId}
                                                          │
                                            commit ingest (+ optional retags)
```

## Detection algorithm (chunk-neighbor voting)

Runs after `embed_documents()` and before any commit. Zero extra embedding cost —
the chunk vectors are already in hand.

1. **Batch NN query** — one `query_batch_points` call: for each new chunk's dense
   vector, dense-only, `limit=3`, payload `doc_id` + `section` only. Filter
   `status=ACTIVE` — overlapping an already-OBSOLETE doc is expected (new version
   replacing old) and must not block.
2. **Per-candidate coverage** — a new chunk "matches" candidate doc D if any of its
   top-3 neighbors from D scores cosine ≥ `OVERLAP_SIM` (0.80). Then
   `coverage = matched_new_chunks / total_new_chunks` per candidate `doc_id`.
3. **Candidate gate** — report candidates with coverage ≥ `OVERLAP_COVERAGE`
   (0.50), sorted by coverage desc, max 5. Below that, ingest proceeds silently.
4. **Matched sections** — per candidate, the distinct `section` names of the new
   doc's matching chunks (capped at 8) → the "highlight overlaps" part.

Thresholds are module constants in `backend/app/overlap.py`; per-candidate scores
are logged for tuning against the known duplicate clusters.

## Backend

- **New `app/overlap.py`** — `find_overlaps(chunks, embeddings)` +
  in-memory pending store `{pending_id: (prepared_args, expires_at)}`, TTL 15 min,
  swept lazily. Assumes a single backend instance (current deployment); a shared
  store would be needed if it scales out.
- **Refactor `ingest.py`** — split `ingest_document()` into `prepare_ingest()`
  (parse/chunk/embed — the expensive part) and `commit_ingest()` (Storage upload,
  Qdrant upsert, Firestore mirror). `ingest_document()` = prepare + commit, so
  `/ingest-text` and `scripts/ingest_folder.py` are untouched.
- **`POST /ingest`** — after prepare, run `find_overlaps`. If candidates → stash
  pending, return **409**:

  ```json
  {
    "pendingId": "…",
    "candidates": [{
      "docId": "…", "title": "Comprehensive Deposit Policy - V2.2",
      "department": "BRANCH_BANKING", "uploadedAt": "…",
      "coverage": 0.92,
      "matchedSections": ["Eligibility", "Interest Rates", "Page 4"]
    }]
  }
  ```

- **`POST /ingest/confirm/{pending_id}`** — body `{"obsoleteDocIds": ["…"]}`
  (empty = plain "upload anyway"). Commits the cached ingest, then retags each
  listed doc OBSOLETE (Qdrant payload + Firestore, same as PATCH /documents).
  Returns the same record shape as `/ingest`. Expired/unknown id → **404**
  (UI asks the user to re-upload).
- **Fail-open** — if the overlap query itself errors, log a warning and proceed
  with normal ingest. The new feature must never block uploads.

## Frontend (admin_dashboard only)

- `lib/documents.ts` — `uploadDocument()` recognizes the 409 payload and throws a
  typed `OverlapDetectedError { pendingId, candidates }`; new
  `confirmIngest(pendingId, obsoleteDocIds)`.
- `UploadDialog` (hub/documents/page.tsx) — on `OverlapDetectedError` the dialog
  body swaps to an overlap review step: amber warning header, per-candidate row
  (title, department, `92% overlap`, matched-section chips, unchecked
  "Mark obsolete" checkbox), footer **Cancel** · **Upload anyway** /
  **Upload & mark N obsolete** (label reflects checked count).
- No staff-frontend changes.

## Testing

- **Unit** — coverage math in `overlap.py` with synthetic vectors (full dupe ≈ 1.0,
  disjoint ≈ 0, partial ≈ mid).
- **Integration (manual, dev backend)** — re-upload an already-ingested corpus PDF
  → 409 with coverage near 1.0 + sensible sections; confirm with retag → new doc
  ACTIVE, old OBSOLETE everywhere; unrelated PDF → clean 200.
- **Tuning check** — one known Type-A pair (Deposit Policy V2.2 vs V3.1) to
  sanity-check 0.80/0.50 before calling it done.
