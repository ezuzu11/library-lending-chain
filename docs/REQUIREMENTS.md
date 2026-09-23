# Requirements Checklist — Phase 1

Source of truth: `individual assignment.pdf` (graded spec + rubric). `prompt.pdf` (engineering workflow) cross-checked against it — where they conflict, the assignment PDF wins, per the project plan's Standing Rule 1.

Naming note: the assignment's block table uses `Member_name` and calls this an "attendance chain" — both are leftover template wording. This checklist and the implementation use `member_name` and "lending chain" throughout. See the project plan for the full note.

## R1 — Registries

- [ ] R1.1 Load `books.txt` into an array of `Book` structs at startup (`book_id[20], title[80], author[50]`)
- [ ] R1.2 Load `members.txt` into an array of `Member` structs at startup (`member_id[20], full_name[50], course_code[10]`)
- [ ] R1.3 Reject a borrow/return referencing an unknown `book_id`
- [ ] R1.4 Reject a borrow/return referencing an unknown `member_id`
- [ ] R1.5 Print a clear error if `books.txt` is missing
- [ ] R1.6 Print a clear error if `members.txt` is missing
- [ ] R1.7 Print a clear error if either file is present but empty
- [ ] R1.8 Malformed lines (wrong field count, oversized fields) are handled without crashing (prompt.pdf 2A — not explicit in assignment PDF but consistent with its "error handling and data validation" objective; adopted)

## R2 — Block structure

- [ ] R2.1 Block contains: `index (int)`, `timestamp (time_t)`, `book_id[20]`, `book_title[80]`, `member_id[20]`, `member_name[50]`, `action[10]`, `previous_hash[65]`, `signature (unsigned char[72])`, `hash[65]`
- [ ] R2.2 `action` is one of `BORROWED`, `RETURNED`, `OVERDUE`
- [ ] R2.3 `previous_hash` holds the SHA-256 hash of the previous block
- [ ] R2.4 `hash` is the SHA-256 hash of the rest of the block's fields combined
- [ ] R2.5 `signature` is an ECDSA signature of the block data
- [ ] R2.6 Genesis block: `index = 0`, `previous_hash` = 64 ASCII zeros

## R3 — Borrow

- [ ] R3.1 Validate `book_id` against the book registry
- [ ] R3.2 Validate `member_id` against the member registry
- [ ] R3.3 On either invalid: print `ERROR: Book or Member not found`, abort — no block created
- [ ] R3.4 Reject if the book is already on loan (no matching RETURNED after the last BORROWED)
- [ ] R3.5 Copy canonical `book_title` / `member_name` from the registries into the block (not user-supplied)
- [ ] R3.6 Set timestamp, `previous_hash`, compute `hash`, compute `signature`
- [ ] R3.7 Append to chain, persist to disk, report success clearly

## R4 — Return

- [ ] R4.1 Validate `book_id`
- [ ] R4.2 Find the book's most recent `BORROWED` entry on the chain
- [ ] R4.3 If none exists (never borrowed, or already returned), print an error — no block created
- [ ] R4.4 Create a `RETURNED` block, preserving book/member info
- [ ] R4.5 Set `previous_hash`, compute `hash`, compute `signature`
- [ ] R4.6 Append, persist, report success

## R5 — Validate chain

- [ ] R5.1 Verify genesis block correctness (`index = 0`, `previous_hash` = 64 zeros)
- [ ] R5.2 Recompute and compare every block's `hash`
- [ ] R5.3 Verify every block's `previous_hash` matches the prior block's `hash`
- [ ] R5.4 Verify signature validity per block
- [ ] R5.5 Verify index/order consistency
- [ ] R5.6 Report clearly whether the chain is VALID or COMPROMISED

## R6 — View records

- [ ] R6.1 Print, per record: index, book title, book ID, member name, member ID, action, timestamp, hash, previous hash, signature validity
- [ ] R6.2 Output is readable in a terminal

## R7 — Tamper detection

- [ ] R7.1 Chain persists to disk in a format that can be modified externally (single byte)
- [ ] R7.2 Reloading a tampered file and running Validate Chain reports failure, not silent acceptance
- [ ] R7.3 Demo must show a working chain, then a deliberately broken one, side by side

## R8 — CLI

- [ ] R8.1 `borrow`, `return`, `view records`, `validate chain` minimum commands
- [ ] R8.2 Load/init on startup
- [ ] R8.3 Exit command
- [ ] R8.4 Invalid commands produce a helpful message, never silently pass
- [ ] R8.5 (optional, adopted) `help`, `list books`, `list members`, `chain status`

## R9 — Persistence

- [ ] R9.1 Chain survives program termination (saved to disk)
- [ ] R9.2 On startup: load registries → load chain if present → else create genesis → validate loaded chain before trusting it
- [ ] R9.3 Corrupted/tampered persisted data is flagged, not silently trusted
- [ ] R9.4 Serialization format choice documented (struct padding / endianness / fixed-width types addressed if raw structs are written)

## R10 — Cryptography

- [ ] R10.1 SHA-256 used correctly (via OpenSSL EVP API, not deprecated low-level calls)
- [ ] R10.2 ECDSA used correctly for signing and verification
- [ ] R10.3 Private key never stored in source code
- [ ] R10.4 Private key never printed unnecessarily
- [ ] R10.5 What is hashed vs. what is signed is documented and distinct

## R11 — Memory safety / C quality

- [ ] R11.1 No buffer overflows; all fixed buffers bounds-checked
- [ ] R11.2 No unsafe functions (`strcpy`, `strcat`, `gets`, unchecked `scanf`)
- [ ] R11.3 All allocations checked for failure; all allocations freed exactly once
- [ ] R11.4 Clean under AddressSanitizer / valgrind (no leaks, no use-after-free, no double-free)

## Deliverables (Compliance criterion, 4 pts)

- [ ] D1 Complete C source code, pushed to a GitHub repo, link included
- [ ] D2 Demo video, 3–5 min: startup, registry load, valid + invalid IDs, borrow, return, view records, validate chain, tamper detection
- [ ] D3 Technical report (Intro, blockchain implementation, security mechanisms, persistence approach, error handling strategy, screenshots, challenges/solutions, system design diagram)
- [ ] D4 README (compilation instructions, build/run instructions, required libraries/dependencies)

## Ambiguities / risks identified

1. **"Attendance chain" / `Member_name`** — resolved: library lending chain, `member_name` lowercase. (see plan)
2. **`action[10]` (char array, not `char[10]`)** — spec typo; treated as `char action[10]`, matching every other field's declared style.
3. **OVERDUE is listed as a valid action but no functional requirement defines when/how it's triggered** — no automatic overdue logic is specified anywhere (no due-date field exists on the block or registries). Resolution: implement `action` as an enum-safe string field that accepts OVERDUE for future extensibility, but do not build automatic overdue detection — it's not a stated requirement and inventing one would violate Standing Rule 3. Note this explicitly in the report as an intentionally unimplemented, spec-absent feature.
4. **Signature buffer `unsigned char[72]`** — ECDSA (P-256) DER signatures are variable-length, up to ~72 bytes; store the DER signature left-aligned in the fixed buffer with an explicit length tracked separately (needed since the struct doesn't have a signature-length field — flagged for report as a struct-fidelity vs. correctness tradeoff, resolved by keeping the buffer as specified and tracking length in memory only, not scoring against the spec's fixed struct).
5. **"Digital signatures to authenticate lending actions" vs. rubric "proper use of cryptographic keys"** — implies persistent ECDSA keypair(s), stored outside source (in `keys/`), not regenerated per run. Adopted.

## Status

All items above are unchecked — this file is the tracking checklist. Check items off in the same session as their corresponding phase's implementation, and never check one off before its test in Phase 8 actually passes.
